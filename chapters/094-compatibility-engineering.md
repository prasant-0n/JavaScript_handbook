# Chapter 94 — Compatibility Engineering

> **JavaScript Mastery — Part XVII: Modern ECMAScript & Language Evolution**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-10
>
> **Core rule:** **Compatibility is an engineering property of a target system, not a property you can infer from a feature name or proposal stage alone.**

The official ECMAScript specification is the language authority, while browsers and runtimes determine when that language surface is actually usable in a deployment. MDN explicitly notes that browser documentation can cover proposed ECMAScript features before official publication, often between Stages 3 and 4; therefore proposal maturity and implementation availability must be tracked separately. citeturn959902search4turn959902search12

MDN's current Baseline program summarizes availability across a defined set of major browsers, but it is explicitly a summary rather than a substitute for project-specific accessibility, usability, performance, security, or other testing. citeturn959902search0


# 0. Chapter Mission

Compatibility engineering is the discipline of making JavaScript software behave acceptably across a defined set of:

- runtimes,
- browsers,
- devices,
- language levels,
- host APIs,
- operating systems,
- toolchains,
- dependencies,
- and deployment environments.

The work begins with a contract:

```text
Who must run this software?
What must it support?
What can it refuse to support?
How will unsupported capability be detected?
What fallback is acceptable?
What performance cost is acceptable?
How will compatibility regressions be caught?
```

A principal engineer does not say:

> “This feature is modern, so it is safe.”

The stronger statement is:

> “This feature is supported by the exact runtime/toolchain/browser baseline we operate, has the required semantic behavior, passes our compatibility tests, and has an explicit fallback or exclusion policy.”

Compatibility engineering connects nearly every previous chapter:

```text
ECMAScript specification
        ↓
engine implementation
        ↓
host runtime
        ↓
toolchain
        ↓
application
        ↓
production fleet
```

A defect at any boundary can make a feature unusable.


# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define compatibility precisely.
2. Separate standards compatibility from runtime compatibility.
3. Separate runtime compatibility from tooling compatibility.
4. Explain syntax, API, semantic, and platform compatibility.
5. Build a browser support policy.
6. Build a Node.js/runtime support policy.
7. Use feature detection correctly.
8. Explain why user-agent detection is often inferior to capability detection.
9. Distinguish transpilation from polyfilling.
10. Explain when a syntax transform can work and when it cannot.
11. Explain why some APIs require true runtime support.
12. Use compatibility matrices.
13. Build a capability matrix from runtime versions.
14. Reason about Baseline and browser compatibility data.
15. Understand the difference between “available” and “safe for our users.”
16. Design progressive enhancement.
17. Design graceful degradation.
18. Design compatibility adapters.
19. Define minimum runtime versions.
20. Manage deprecation and support windows.
21. Handle mixed-version fleets.
22. Test compatibility in CI.
23. Debug environment-specific failures.
24. Prevent accidental use of unsupported language features.
25. Evaluate package engine constraints.
26. Understand transitive dependency compatibility.
27. Design dual builds when necessary.
28. Design server/browser compatibility boundaries.
29. Measure compatibility risk.
30. Build a production compatibility governance system.
31. Explain compatibility decisions in principal-level interviews.


# 2. Prerequisites

Recommended prerequisite chapters:

```text
Chapter 01 — ECMAScript and Runtime Landscape
Chapter 02 — Values, Types, Type System
Chapter 03 — Numbers, Floating Point, BigInt
Chapter 04 — Strings, Unicode, Text Semantics
Chapter 05 — Variables, Declarations, Assignment
Chapter 07 — Type Conversion, Coercion, Equality
Chapter 14 — this / Invocation / Binding
Chapter 19 — Proxy / Reflect / Metaprogramming
Chapter 28 — JSON / Serialization / Structured Clone
Chapter 33 — Browser Event Loop
Chapter 41 — Spec Architecture
Chapter 47 — JavaScript Engine Architecture
Chapter 48 — V8 Internals and Optimization
Chapter 55 — Fetch / HTTP Networking
Chapter 64 — ES Modules
Chapter 65 — CommonJS and Interoperability
Chapter 66 — package.json and Module Resolution
Chapter 67 — Dependency Management and Supply Chain
Chapter 68 — Transpilation and Compilation
Chapter 69 — Bundlers and Build Systems
Chapter 70 — Source Maps and Production Debugging
Chapter 90 — Modern ECMAScript Features
Chapter 91 — TC39 Proposal Tracking
Chapter 92 — Temporal
Chapter 93 — Decorators


# 3. What Is Compatibility?

Compatibility answers whether a program, feature, API, or package can be used correctly in a target environment.

A useful model is:

```text
Compatibility
=
Syntax
+
Semantics
+
Runtime APIs
+
Host behavior
+
Toolchain
+
Dependency graph
+
Operational constraints
```

A feature may be compatible at one layer and incompatible at another.

Example:

```text
Browser parses syntax        YES
Browser implements API       NO
Bundler parses syntax        NO
Type checker understands it  YES
Production fleet supports it NO
```

The correct result is:

```text
Not production-compatible.
```

Compatibility therefore has a **vector**, not a single Boolean.


# 4. Why Compatibility Engineering Exists

JavaScript ecosystems are heterogeneous.

A single product can simultaneously contain:

```text
latest desktop Chrome
older iOS Safari
Android WebView
embedded browser
Node.js LTS
serverless runtime
CI runner
developer workstation
build container
third-party SDK
```

Each may expose a different capability set.

Compatibility engineering exists to transform:

```text
heterogeneous environments
```

into:

```text
explicit support guarantees
```

Without that discipline, teams accidentally make support decisions through:

- developer machine versions,
- latest-browser assumptions,
- dependency updates,
- accidental syntax adoption,
- transitive package behavior,
- CI image changes.


# 5. Mental Model — The Compatibility Stack

Use five layers:

```text
Layer 1 — Standard
        Is the behavior defined by ECMAScript / relevant platform standard?

Layer 2 — Implementation
        Does the engine/runtime/browser implement it?

Layer 3 — Host
        Does the host expose the required APIs and behavior?

Layer 4 — Toolchain
        Can parser/compiler/bundler/type-checker/test tools process it?

Layer 5 — Fleet
        Does the actual production population support it?
```

A feature is production-safe only when the required layers line up.

```text
standardized
     ≠
implemented
     ≠
tool-supported
     ≠
deployed
```


# 6. The Three Truths

Maintain three independent truths.

### Standards truth

What the standards bodies specify.

Sources include:

```text
TC39
ECMAScript specification
Web standards
```

### Implementation truth

What software actually implements.

Sources include:

```text
engine/runtime release notes
browser compatibility data
runtime documentation
```

### Deployment truth

What your own fleet supports.

Sources include:

```text
telemetry
supported-version policy
client distribution
CI matrix
production inventory
```

A principal engineer reconciles all three instead of substituting one for another.


# 7. Compatibility Dimensions

Track at least:

| Dimension | Main question |
|---|---|
| Syntax | Can the source be parsed? |
| Runtime API | Does the required object/function exist? |
| Semantics | Does behavior match the required contract? |
| Host | Does the browser/runtime expose required platform behavior? |
| Toolchain | Can build/test/lint/type tools process it? |
| Dependency | Do packages work with our baseline? |
| Performance | Is implementation speed acceptable? |
| Memory | Is allocation/retention acceptable? |
| Security | Is the compatibility path safe? |
| Operational | Can the organization support the baseline? |
| UX | Can users still complete the task? |
| Accessibility | Does the fallback remain usable? |

A feature is not compatible merely because:

```js
typeof SomeFeature === "function"
```

Existence proves very little about full behavioral compatibility.


# 8. Syntax Compatibility

Syntax compatibility is the first gate:

```text
Can the parser understand the source?
```

Examples of syntax-sensitive features include:

- optional chaining,
- nullish coalescing,
- class fields,
- private fields,
- import attributes,
- newer regex syntax,
- newer expression/statement forms.

If a runtime cannot parse the program, no runtime feature detection inside that program can save it.

This leads to the classic rule:

> **You cannot feature-detect syntax after a parser has already rejected the script.**

Therefore syntax compatibility can require:

```text
transpilation
or
version-bounded source delivery
```


# 9. API Compatibility

API compatibility is different:

```js
if ("someMethod" in target) {
  // capability exists
}
```

A runtime can parse the source but lack a required API.

Example:

```js
if (typeof globalThis.SomeAPI?.doThing === "function") {
  // capability path
} else {
  // fallback
}
```

The check should reflect the exact operation you need.

Prefer:

```text
feature detection
```

over:

```text
browser-name detection
```

when the problem is capability availability.


# 10. Semantic Compatibility

Semantic compatibility asks:

> Does the feature behave the way the application requires?

A method can exist but still be insufficient because of:

- partial implementation,
- browser-specific bugs,
- legacy quirks,
- missing options,
- different error behavior,
- timing differences,
- unsupported edge cases.

Therefore:

```text
API existence
```

is weaker than:

```text
API behavioral conformance
```

For high-risk domains, write behavior tests against your support matrix.


# 11. Host Compatibility

JavaScript the language is not the entire platform.

Browsers add:

```text
DOM
Fetch
Streams
Workers
Storage
Web Crypto
Web Components
```

Node.js adds:

```text
fs
net
HTTP
streams
workers
process APIs
```

A language feature can be supported while a required host API is not.

Always ask:

```text
Is this ECMAScript?
or
is this a host capability?
```


# 12. Toolchain Compatibility

Tooling is part of the executable system.

Check:

```text
parser
transformer
bundler
type checker
linter
formatter
test runner
coverage tool
minifier
source-map processor
deployment platform
```

Example:

```text
Node runtime: supports syntax
ESLint parser: old
CI: fails before tests
```

The application is still incompatible with its engineering system.

MDN's documentation reinforces the need to treat browser compatibility as an evidence problem rather than relying on a single informal status label. citeturn959902search1


# 13. Fleet Compatibility

Your actual support contract may be narrower than global support.

For example:

```text
Supported:
Chrome current - 2
Safari current - 2
Firefox current - 2
Node.js active LTS
```

This becomes the organization's **compatibility baseline**.

The baseline should be:

- explicit,
- versioned,
- reviewed,
- testable,
- communicated.

Never let a developer's local environment silently define production compatibility.


# 14. Browser Baseline

MDN's current Baseline concept groups browser availability into statuses such as:

```text
Widely available
Newly available
Limited availability
Deprecated
```

The current Baseline browser set includes:

```text
Safari macOS
Safari iOS
Chrome desktop
Chrome Android
Edge desktop
Firefox desktop
Firefox Android
```

The current MDN definition describes “widely available” features as having a consistent history of support in the core browsers for at least 2.5 years. citeturn959902search0

Use Baseline as a useful web-compatibility signal.

Do not treat it as a guarantee for:

- your exact older-device population,
- embedded web views,
- accessibility,
- performance,
- application-specific behavior.


# 15. Browser Compatibility Data

MDN publishes structured Browser Compatibility Data (BCD).

The data can represent browser support for:

- APIs,
- properties,
- JavaScript,
- HTML,
- CSS,
- SVG,
- and related technologies.

It is useful for automation because it is structured data rather than only prose. citeturn959902search1

Engineering use:

```text
BCD
 ↓
support matrix
 ↓
build target policy
 ↓
CI matrix
```

But validate important decisions against authoritative runtime/browser evidence.


# 16. Feature Detection

Feature detection asks:

> Can this capability be used here?

Basic form:

```js
if (typeof globalThis.structuredClone === "function") {
  // use capability
}
```

But robust feature detection can require more.

Example:

```js
function hasUsableStructuredClone() {
  if (typeof globalThis.structuredClone !== "function") {
    return false;
  }

  try {
    const result = globalThis.structuredClone({ value: 1 });
    return result?.value === 1;
  } catch {
    return false;
  }
}
```

Do not over-test every capability.

Test the behavior that actually matters.


# 17. Feature Detection vs Version Detection

Version detection answers:

```text
What version claims to be running?
```

Feature detection answers:

```text
Can this capability be used?
```

Capability detection is often more robust because:

- vendors backport features,
- enterprise runtimes can be modified,
- compatibility modes exist,
- browsers can ship features independently,
- user agents can misrepresent.

Version detection is still useful when:

```text
a known implementation bug
or
a required minimum release
```

must be enforced.

Prefer capability checks for capability decisions.
Use version checks for known version-specific constraints.


# 18. User-Agent Detection

User-agent sniffing is fragile.

Bad:

```js
if (navigator.userAgent.includes("Chrome")) {
  // assume feature support
}
```

Problems:

- compatibility tokens,
- browser forks,
- embedded browsers,
- spoofing,
- version parsing,
- unrelated implementation details.

Prefer:

```text
capability detection
```

when the decision is truly:

```text
can the operation execute?
```


# 19. Progressive Enhancement

Progressive enhancement starts with a broadly compatible core and adds capabilities when available.

Model:

```text
Baseline experience
      ↓
detect capability
      ↓
enhanced experience
```

Example:

```js
if (typeof globalThis.someEnhancement === "function") {
  enableEnhancement();
} else {
  enableBaseline();
}
```

The fallback should be a complete user flow where required.

Progressive enhancement is especially valuable on the web because the browser population is heterogeneous.


# 20. Graceful Degradation

Graceful degradation starts from an advanced implementation and deliberately defines what happens when parts fail.

Example:

```text
full feature
   ↓
partial feature
   ↓
basic feature
   ↓
recoverable error
```

Do not confuse:

```text
progressive enhancement
```

with:

```text
silent failure
```

A graceful fallback must preserve correctness and user understanding.


# 21. Capability Adapters

Centralize compatibility logic.

Instead of:

```text
feature detection
feature detection
feature detection
```

throughout the codebase, create:

```text
platform/
  capabilities.js
  storage.js
  crypto.js
  streams.js
```

Then application code depends on stable internal contracts.

Example:

```js
export function createStorage() {
  if (typeof localStorage !== "undefined") {
    return localStorage;
  }

  return createMemoryStorage();
}
```

The adapter becomes the compatibility boundary.


# 22. Polyfills

A polyfill supplies missing runtime behavior using existing primitives.

Conceptually:

```text
missing API
   ↓
compatibility implementation
   ↓
application
```

Polyfills are useful for runtime APIs.

But there are limits.

A polyfill may not be able to reproduce:

- new syntax parsing,
- engine-level optimization,
- hidden internal behavior,
- true host integration,
- performance characteristics,
- security properties.

Therefore:

```text
polyfill support
≠
native support
```


# 23. Transpilation

Transpilation transforms source code.

```text
new syntax
   ↓
transformer
   ↓
older syntax
```

This can make unsupported syntax parseable by older engines.

But a transformer cannot automatically invent every runtime primitive.

Therefore:

```text
syntax compatibility
and
runtime API compatibility
```

must be handled separately.


# 24. What Transpilers Cannot Magically Solve

Suppose code requires:

```js
globalThis.someNewApi()
```

A syntax transformer may leave it intact.

If the runtime lacks the API:

```text
transformed code
still fails
```

You may need:

```text
polyfill
or
feature branch
or
alternative implementation
```

This distinction is one of the most important compatibility concepts.


# 25. Syntax + Runtime Compatibility Pipeline

A mature build can look like:

```text
author modern JS
        ↓
parse
        ↓
transform unsupported syntax
        ↓
bundle
        ↓
inject required runtime helpers/polyfills
        ↓
ship target-specific build
        ↓
run in target
```

Every stage should have explicit assumptions.

Do not call a build “compatible” without knowing the target.


# 26. Build Targets

A build target describes the environment your output is intended to run in.

Examples:

```text
browser-modern
browser-legacy
node-current
node-active-lts
serverless-X
embedded-webview
```

Targets influence:

- syntax transforms,
- polyfills,
- module format,
- bundle structure,
- runtime APIs,
- dependency resolution.

A single universal build can be correct, but it is not always optimal.


# 27. Dual Builds

A dual-build strategy provides different artifacts.

Example:

```text
dist/
  modern/
  legacy/
```

Modern:

```text
less transpilation
smaller output
native features
```

Legacy:

```text
more transforms
more compatibility helpers
broader support
```

Selection can happen through:

- conditional package exports,
- script/module strategies,
- deployment routing,
- runtime selection.

Dual builds increase complexity, so justify the trade-off.


# 28. Differential Serving

Differential serving means users receive artifacts appropriate for their capabilities.

Benefits:

```text
modern users
→ smaller/faster code

older users
→ compatibility build
```

Costs:

```text
build complexity
testing matrix
deployment complexity
cache variation
observability complexity
```

Principal decision:

```text
performance benefit
vs
operational complexity
```


# 29. Module Compatibility

Module compatibility includes:

```text
ESM parsing
CommonJS loading
conditional exports
package exports
module resolution
dynamic import
import attributes
```

A package can support one runtime and fail in another because the module loader differs.

Always test the actual consumption mode.

```text
Node ESM
≠
Browser ESM
≠
bundler ESM
```

They share concepts but have host-specific behavior.


# 30. Package Engine Constraints

Packages can declare runtime requirements.

Example:

```json
{
  "engines": {
    "node": ">=22"
  }
}
```

This is a contract signal.

But do not assume a package manager or runtime will always enforce it exactly the way your organization expects.

CI should test the declared baseline.

Also inspect transitive dependencies.


# 31. Dependency Compatibility

Your application is constrained by the deepest dependency that matters.

```text
Application
 ↓
A
 ↓
B
 ↓
C
```

If C requires a newer runtime than your fleet, your application may become incompatible even if your own source is conservative.

Dependency review should track:

```text
engine requirements
syntax requirements
host API requirements
native modules
optional dependencies
package exports
```


# 32. Supply-Chain Compatibility

Compatibility and supply-chain risk overlap.

A dependency update can:

```text
raise minimum Node version
introduce new syntax
drop a browser
require a new Web API
switch module format
change native binaries
```

Therefore dependency updates should include compatibility review.

Treat:

```text
dependency upgrade
```

as a potential platform change, not only a feature update.


# 33. Semantic Versioning Is Not a Runtime Contract

Semantic versioning tells you something about package API intent.

It does not guarantee:

```text
your runtime can parse the package
```

or:

```text
your browser supports its APIs
```

A minor dependency update can introduce a newly allowed platform baseline depending on project policy.

Read:

```text
release notes
engines
exports
build targets
compatibility documentation
```


# 34. Browsers and JavaScript Language Evolution

Modern ECMAScript features often move from proposal to implementation before official yearly specification publication.

MDN explicitly states that some proposed ECMAScript features may already be implemented and documented before official publication, most often around Stages 3 and 4. citeturn959902search4

Therefore:

```text
“MDN has a page”
```

does not mean:

```text
“the feature is already in my support baseline.”
```

Check compatibility data and the target browser versions.


# 35. ECMAScript Standard vs Browser Platform

ECMAScript defines the language.

Browsers additionally implement:

```text
DOM
Fetch
Web Crypto
Web Streams
Workers
Storage
```

A feature's presence in ECMAScript does not imply equivalent browser-host behavior.

Conversely, a browser API may be standardized outside ECMA-262.

Always identify the correct specification owner.


# 36. Runtime Compatibility — Node.js

Node.js compatibility depends on:

```text
Node version
V8 version
Node-specific APIs
module loader
OS support
build configuration
```

Do not say:

> “V8 supports it, therefore Node supports it.”

Check the exact Node release behavior.

For a production service, define:

```text
minimum Node
recommended Node
maximum tested Node
deprecation date
upgrade owner
```


# 37. Runtime Compatibility — Embedded Environments

Embedded JavaScript environments can diverge sharply from mainstream browsers/Node.

Examples:

```text
smart-TV browser
in-app WebView
game runtime
device firmware
automation host
database JavaScript runtime
```

Never extrapolate support from desktop browser data.

Maintain an explicit environment inventory.


# 38. Compatibility Matrix

A useful matrix:

| Capability | Chrome | Firefox | Safari | Node | Tooling |
|---|---|---|---|---|---|
| Feature A | ✓ | ✓ | ✓ | ✓ | ✓ |
| Feature B | ✓ | ✓ | ~ | ✓ | ✓ |
| Feature C | ~ | ✗ | ✗ | ✗ | ✓ |

Legend:

```text
✓ supported
~ partial / constrained
✗ unsupported
? unverified
```

Always record:

```text
version
observation date
source
```

A matrix without version/date information becomes misleading.


# 39. Version Matrix

Use:

```text
Browser / Runtime
Minimum
Preferred
Latest tested
```

Example:

```text
Chrome:
minimum 120
preferred latest supported
latest tested 142

Node:
minimum 22
preferred active LTS
latest tested 26
```

These values are examples of structure, not universal recommendations.

The exact baseline must come from your product.


# 40. Support Policy

A support policy should answer:

```text
Which environments are supported?
For how long?
What happens when a user is unsupported?
Who approves baseline changes?
How are exceptions handled?
How are compatibility regressions detected?
```

Good policies are:

```text
explicit
versioned
owned
tested
communicated
```


# 41. Support Window

Define:

```text
N = supported version window
```

For example:

```text
latest two major browser versions
```

or:

```text
active Node LTS releases
```

The policy should account for:

- customer constraints,
- security support,
- framework support,
- dependency support,
- cost to test,
- market distribution.


# 42. Minimum Version Selection

Do not pick a minimum version because it sounds modern.

Evaluate:

```text
user distribution
business requirements
security requirements
support cost
dependency constraints
framework baseline
feature needs
```

Then choose:

```text
minimum supported version
```

with a documented rationale.


# 43. Compatibility Budget

Every additional target creates costs:

```text
testing
build complexity
fallback code
bug surface
support burden
observability
documentation
```

Think of compatibility as a budget.

Example decision:

```text
Support +1 legacy browser
→
+X CI minutes
+Y bundle size
+Z engineering complexity
```

The question is not:

> “Can we support it?”

The question is:

> “Is supporting it worth its product value?”


# 44. Feature Cost Model

For every compatibility feature, consider:

```text
Source complexity
Build complexity
Runtime complexity
Fallback complexity
Bundle cost
Performance cost
Test cost
Operational cost
Future removal cost
```

This is the bridge to principal engineering.

Compatibility is a system trade-off.


# 45. Feature Policy Categories

Classify capabilities:

```text
A — universally supported in baseline
B — supported but requires guarded use
C — transform/polyfill required
D — unsupported; alternative required
E — unsupported and product flow must change
```

Use the policy consistently.


# 46. Feature Gates

Feature gates can control behavior:

```js
if (capabilities.newFeature) {
  useNewPath();
} else {
  useStablePath();
}
```

For syntax, the gate may need to exist at build/deployment level because an old parser cannot execute code containing unsupported syntax.

Feature gates should have:

- owner,
- default,
- rollback,
- removal date.


# 47. Progressive Enhancement Architecture

A robust architecture:

```text
Request
 ↓
capability profile
 ↓
baseline path
 ├── supported → enhancement
 └── unsupported → fallback
```

The application should not repeatedly rediscover environment state.

Create a stable capability profile once.


# 48. Behavioral Detection

Sometimes simple existence checks are inadequate.

Example shape:

```js
function supportsRequiredBehavior() {
  if (!("someApi" in globalThis)) {
    return false;
  }

  try {
    const value = globalThis.someApi.create(...);
    return value?.requiredProperty === expected;
  } catch {
    return false;
  }
}
```

Do this only when there is a known compatibility distinction.

Avoid expensive behavioral probes on hot paths.


# 49. Compatibility Shims

A shim can normalize differences:

```js
export function getNormalizedValue(input) {
  if (nativePathAvailable()) {
    return nativePath(input);
  }

  return fallbackPath(input);
}
```

A good shim:

```text
small
isolated
tested
observable
removable
```

A bad shim becomes permanent infrastructure nobody understands.


# 50. The Compatibility Debt Problem

Compatibility code accumulates.

Example:

```text
legacy Safari workaround
old Node workaround
old parser workaround
old polyfill
obsolete dependency branch
```

Each adds code paths.

Therefore every compatibility workaround needs:

```text
why
target
introduced
owner
removal condition
```

Track expiration.


# 51. Compatibility Debt Register

Create:

```text
compatibility/
  README.md
  known-workarounds.md
  baseline.md
  removal-plan.md
```

For each workaround:

```text
Issue:
Affected versions:
Detection:
Fallback:
Test:
Owner:
Review date:
Removal trigger:
```

This turns compatibility debt into managed engineering work.


# 52. Deprecating Old Runtimes

Removing old support can simplify:

```text
source
toolchain
polyfills
tests
CI
documentation
```

But it can affect customers.

A deprecation process should include:

```text
usage measurement
notice period
support-policy update
migration guidance
release planning
observability
```

Do not remove compatibility support merely because engineers are tired of it.


# 53. Compatibility Analytics

Measure:

```text
active client versions
unsupported client attempts
fallback frequency
feature-detection path frequency
error rates by environment
performance by environment
```

This tells you whether compatibility code is still valuable.


# 54. Real User Distribution

Browser support decisions should use actual user distribution where possible.

A feature may be:

```text
widely supported globally
```

but still poorly supported by:

```text
your customer base
```

This is why Baseline is useful context rather than a substitute for product telemetry. citeturn959902search0


# 55. Compatibility Testing Strategy

Use multiple levels:

```text
unit tests
feature tests
browser matrix
runtime matrix
integration tests
end-to-end tests
production telemetry
```

Each catches different classes of failure.

Do not try to test every version combination equally.
Prioritize by risk and user distribution.


# 56. CI Matrix Design

A CI matrix might cover:

```text
minimum supported
middle supported
latest supported
future/latest preview
```

For browsers:

```text
core supported browser set
one legacy edge
one mobile representative
```

For Node:

```text
minimum
active LTS
latest
```

The exact matrix must reflect your contract.


# 57. Contract Tests

Write tests that express your support policy.

Example:

```js
test("required crypto capability exists", () => {
  expect(typeof globalThis.crypto?.subtle).toBe("object");
});
```

But compatibility tests should verify behavior, not only object presence.

Use contract tests for the critical platform APIs.


# 58. Canary Testing

Canary environments can detect compatibility regressions before full rollout.

```text
new runtime
 ↓
small fleet
 ↓
observe
 ↓
expand
```

Track:

```text
error rates
latency
memory
CPU
startup
feature-specific failures
```

This is especially important during runtime upgrades.


# 59. Runtime Upgrade Engineering

A runtime upgrade changes more than a version number.

Potential effects:

```text
engine optimization
GC behavior
Web/Node APIs
module loader
security defaults
TLS
OpenSSL
native dependencies
error messages
timers
diagnostics
```

Run a regression suite.

A runtime upgrade is a platform change.


# 60. Browser Upgrade Engineering

Browsers update independently.

A product should generally expect:

```text
continuous evolution
```

rather than treating browser upgrades as rare releases.

The compatibility contract should therefore be:

```text
range-oriented
capability-oriented
test-backed
```

not based on frozen assumptions about one browser build forever.


# 61. Feature Rollout Strategy

For a new ECMAScript/platform capability:

```text
1. Verify standard status.
2. Verify target support.
3. Prototype.
4. Add compatibility detection.
5. Add tests.
6. Pilot.
7. Measure.
8. Expand.
9. Remove fallback later if justified.
```

This is the safe adoption pipeline.


# 62. Security Considerations

Compatibility layers can introduce security differences.

A fallback may:

- use weaker cryptography,
- parse input differently,
- expose a broader attack surface,
- lack hardened native implementation behavior,
- change origin/security assumptions.

Never silently fall back from a security-critical API to an insecure approximation.

For critical security features:

```text
unsupported
→ fail safely
```

may be preferable.


# 63. Performance Considerations

Compatibility mechanisms can cost:

```text
parse time
bundle size
startup time
runtime CPU
memory
network transfer
cache efficiency
```

For example, a broad polyfill bundle can increase payload size even when most users do not need it.

Prefer targeted, data-driven compatibility.


# 64. Memory Considerations

Compatibility helpers may retain:

- closures,
- caches,
- alternate implementations,
- transformed module graphs.

Measure:

```text
heap
retained size
startup allocation
long-lived caches
```

Do not assume a compatibility layer is negligible because the code is small.


# 65. Observability

Log compatibility-path selection when useful:

```json
{
  "feature": "new-api",
  "path": "fallback",
  "runtime": "browser",
  "reason": "capability-missing"
}
```

Do not collect sensitive browser/device data unnecessarily.

The objective is to answer:

```text
How often is fallback used?
Where?
Why?
```


# 66. Debugging Methodology

When a bug occurs only in one environment:

```text
1. Freeze the exact version.
2. Reproduce.
3. Identify language vs host API.
4. Check parser support.
5. Check API support.
6. Check semantic behavior.
7. Check toolchain output.
8. Inspect polyfills/shims.
9. Compare with baseline environment.
10. Add a regression test.
```

Never start with:

> “Safari is broken.”

First isolate the capability and the layer.


# 67. Debugging — Syntax Failure

Symptom:

```text
Unexpected token '?'
```

Possible causes:

```text
old parser
old runtime
wrong build target
untransformed dependency
embedded WebView
```

The first question:

> Which parser produced the error?

That tells you whether the failure occurred in:

```text
build
test
runtime
```


# 68. Debugging — Missing API

Symptom:

```text
TypeError: x.someMethod is not a function
```

Investigate:

```js
typeof x.someMethod
```

Then determine:

```text
which runtime
which version
which object implementation
which polyfills
which bundle
```

Do not patch blindly.


# 69. Debugging — Semantic Drift

Symptom:

```text
same code
different result
different browser
```

Check:

```text
spec behavior
known implementation bugs
input normalization
time zones/locales
host APIs
polyfills
```

A feature can exist everywhere but behave differently due to implementation defects or environment-dependent inputs.


# 70. Debugging — Dependency

Symptom:

```text
app works locally
deployment fails
```

Investigate:

```text
lockfile
resolved dependency versions
package exports
engine constraints
build image
Node version
bundler
optional dependencies
```

The deployed dependency graph is the artifact that matters.


# 71. Code Review Exercise

Review:

```js
if (navigator.userAgent.includes("Safari")) {
  useLegacy();
} else {
  useModern();
}
```

Find the problems:

1. Browser identity is being used as a proxy for capability.
2. Safari versions differ.
3. WebViews may differ.
4. Other browsers may share the same capability gap.
5. Known bugs may not align with browser family.
6. The fallback condition is not directly tied to required behavior.
7. The code is difficult to remove when assumptions change.

Better question:

```text
What capability actually determines which path is safe?
```


# 72. Code Review Exercise — Stage Confusion

Review:

```js
if (proposal.stage >= 3) {
  enableFeature();
}
```

Why is this incorrect?

Because:

```text
proposal stage
```

is standards maturity, not a production capability guarantee.

The correct policy combines:

```text
standard status
+
runtime support
+
toolchain support
+
fleet baseline
```

This revisits Chapter 91.


# 73. Code Review Exercise — Polyfill Assumption

Review:

```js
import "future-feature-polyfill";

useFutureSyntax();
```

Potential issue:

```text
polyfill may implement runtime APIs,
but it cannot make an old JavaScript parser understand arbitrary new syntax.
```

The source may need:

```text
transpilation
```

in addition to:

```text
runtime compatibility support
```


# 74. Implementation — Guided Compatibility Detector

Build:

```js
export function detectCapabilities() {
  return {
    structuredClone:
      typeof globalThis.structuredClone === "function",

    abortController:
      typeof globalThis.AbortController === "function",

    webCrypto:
      typeof globalThis.crypto?.subtle === "object"
  };
}
```

Then extend it with behavioral checks only where necessary.


# 75. Implementation — Capability Registry

Build:

```js
const capabilities = new Map([
  ["structuredClone", () =>
    typeof globalThis.structuredClone === "function"
  ]
]);

export function supports(name) {
  const detector = capabilities.get(name);
  return detector ? detector() : false;
}
```

Requirements:

```text
explicit names
cached results where appropriate
testable detectors
no user-agent parsing
```


# 76. Implementation — Compatibility Adapter

Create:

```js
export function createAbortController() {
  if (typeof globalThis.AbortController === "function") {
    return new AbortController();
  }

  return createFallbackAbortController();
}
```

Then write tests for:

```text
native path
fallback path
unsupported environment
```

Do not expose environment branching throughout the application.


# 77. Implementation — Browser Matrix Generator

Create a JSON input:

```json
{
  "feature": "example",
  "browsers": {
    "chrome": 120,
    "firefox": 120,
    "safari": 17,
    "edge": 120
  }
}
```

Generate:

```text
supported
unsupported
unknown
```

Then add:

```text
observedAt
source
notes
```


# 78. Implementation — Production Compatibility Report

Generate a report:

```text
Feature
Standard status
Minimum runtime
Browser support
Node support
Toolchain support
Fallback
User coverage
Performance cost
Security assessment
Recommendation
```

Recommendation values:

```text
ADOPT
ADOPT WITH GUARD
TRANSFORM/POLYFILL
FALLBACK
REJECT
```


# 79. Implementation — Edge-Case Hardened Tracker

Your tracker must handle:

```text
unknown browser
missing version
partial support
future version
embedded browser
unverified runtime
conflicting sources
stale data
deprecated feature
runtime bug
```

Never convert “unknown” into “supported.”


# 80. Implementation — Production Grade

Architecture:

```text
sources/
  mdn-bcd.js
  runtime-release-notes.js
  internal-fleet.js

normalization/
  versions.js
  capabilities.js

policy/
  support-baseline.js
  risk.js

reports/
  compatibility-matrix.js
  exceptions.js

ci/
  matrix.js
  contract-tests.js
```

Features:

```text
versioned data
source attribution
change history
freshness checks
policy validation
CI integration
alerts
```


# 81. Testing Exercise — Minimum Baseline

Create a test suite that fails when:

```text
source syntax exceeds target baseline
required API missing
fallback path missing
minimum runtime unsupported
dependency raises runtime requirement
```

This converts the support policy into executable governance.


# 82. Testing Exercise — Browser Contract

Define:

```text
required feature
required behavior
supported browsers
unsupported browsers
fallback behavior
```

Then write end-to-end tests.

Do not test every possible browser version.
Test the contract boundaries and representative versions.


# 83. Testing Exercise — Runtime Upgrade

Before upgrading Node.js:

```text
1. Run unit tests.
2. Run integration tests.
3. Run compatibility tests.
4. Run performance benchmark.
5. Compare memory profile.
6. Run canary.
7. Inspect production metrics.
```

Then document:

```text
new baseline
new risks
rollback plan
```


# 84. Compatibility Decision Matrix

Use:

| Question | Yes | No |
|---|---|---|
| Standardized? | continue | experimental policy |
| Runtime support? | continue | fallback |
| Tooling support? | continue | toolchain work |
| Fleet support? | continue | gate rollout |
| Security approved? | continue | reject/mitigate |
| Performance measured? | continue | benchmark |
| Tests present? | adopt candidate | add tests |

This is an internal decision aid, not a standards rule.


# 85. Compatibility Risk Score

Score 0–5:

```text
standards maturity
runtime coverage
browser coverage
tooling coverage
semantic confidence
test confidence
security confidence
performance confidence
fallback quality
observability
```

Then document:

```text
score
evidence
unknowns
decision
```

Do not hide uncertainty behind one numeric score.


# 86. Principal Trade-Off Framework

Evaluate:

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

Ask:

```text
What does compatibility buy?
What does compatibility cost?
Who benefits?
Who pays?
What is the exit strategy?
```


# 87. Architecture — Modern Web Application

A strong architecture can be:

```text
source
 ↓
modern syntax
 ↓
target-aware build
 ↓
modern artifact
 ↓
compatibility artifact where needed
 ↓
browser capability selection
 ↓
shared application domain logic
```

Keep compatibility code near platform boundaries.

Do not contaminate core domain logic with browser-version checks.


# 88. Architecture — Node.js Service

Define:

```text
Node minimum
Node preferred
dependencies' engine requirements
module format
native dependencies
CI versions
deployment images
serverless restrictions
```

Use runtime upgrade rehearsals.

A service should be reproducible from its lockfile and build image.


# 89. Architecture — Shared Library

A published library has a stronger compatibility responsibility.

Document:

```text
supported Node versions
supported browsers
ESM/CJS strategy
package exports
syntax level
polyfill expectations
peer dependencies
browser globals
```

Do not silently ship syntax newer than the declared consumer baseline.


# 90. Library Authoring Compatibility

A library author should optimize for:

```text
consumer compatibility
tree-shaking
module interoperability
side-effect correctness
runtime assumptions
```

Avoid assuming:

```text
consumer uses your same Node version
consumer has your polyfills
consumer uses your bundler
```

Libraries often need to be more conservative than applications.


# 91. Compatibility and Tree Shaking

Conditional compatibility code can affect bundling.

Bad structure:

```js
if (runtimeSupportsFeature()) {
  importEverything();
} else {
  importFallbackEverything();
}
```

The bundler may retain both paths depending on static analyzability.

Measure resulting:

```text
bundle size
dead code
startup
```

Use build-time target knowledge where practical.


# 92. Compatibility and Code Splitting

Compatibility paths can be loaded lazily:

```js
if (needsFallback()) {
  await import("./legacy-path.js");
}
```

Benefits:

```text
modern users avoid fallback payload
```

Costs:

```text
conditional network request
latency
cache complexity
debugging complexity
```

Choose based on actual user distribution.


# 93. Compatibility and Caching

Multiple artifacts can multiply cache keys.

For example:

```text
app-modern.js
app-legacy.js
```

The routing decision must be deterministic.

CDN caches need correct:

```text
Vary
cache key
content negotiation
feature routing
```

A compatibility strategy that breaks caching can erase its performance benefits.


# 94. Compatibility and Security Headers

Browser security features are not identical to ECMAScript features.

If a compatibility fallback changes:

```text
script source
resource loading
worker strategy
storage
crypto
```

review the resulting security posture.

A fallback should not weaken a security boundary merely to preserve a feature.


# 95. Compatibility and Accessibility

Fallbacks must remain accessible.

Test:

```text
keyboard navigation
screen readers
focus order
input behavior
reduced-motion expectations
error messaging
```

A “working” fallback that blocks assistive technology is not a successful compatibility design.

MDN also cautions that Baseline is not a substitute for accessibility or usability testing. citeturn959902search0


# 96. Compatibility and Internationalization

Environment differences can expose:

```text
Intl
locale data
time zones
number formatting
date formatting
text segmentation
```

Never infer locale or time zone semantics from browser version alone.

Test the actual internationalization behavior you need.


# 97. Compatibility and Temporal

Temporal is a useful case study.

A team should separately verify:

```text
ECMAScript standard status
native Temporal support
polyfill policy
TypeScript/tooling support
browser baseline
Node baseline
time-zone data behavior
```

Do not collapse these into:

```text
“Temporal supported”
```

This directly applies Chapter 92 and Chapter 91.


# 98. Compatibility and Decorators

Decorators are another case study.

Because proposal stage and tooling support can differ, inspect:

```text
proposal status
language parser
transpiler mode
TypeScript semantics
native runtime support
library authoring policy
```

A transpiled decorator feature is not identical to native standardized language behavior.

Use Chapter 93 for semantics; use this chapter for deployment judgment.


# 99. Compatibility and New ECMAScript Features

For any new feature:

```text
1. Identify the exact edition/proposal.
2. Identify required runtime support.
3. Check target browser data.
4. Check Node/runtime support.
5. Check tooling parser support.
6. Check dependency interactions.
7. Decide native vs transform/polyfill.
8. Add CI coverage.
```

This is the operational application of Chapter 90.


# 100. Browser Support Sources

Useful source classes:

```text
ECMAScript specification
TC39 proposal repository
MDN reference + compatibility data
browser vendor release documentation
Web feature compatibility datasets
actual browser CI
```

MDN states that its JavaScript documentation covers ECMAScript as well as browser APIs and that compatibility tables are supported by structured Browser Compatibility Data. citeturn959902search1turn959902search9

For high-risk decisions, triangulate primary sources and empirical tests.


# 101. Node Support Sources

Use:

```text
Node.js release documentation
Node.js API documentation
Node/V8 release notes
CI execution on actual Node versions
container/deployment image definitions
```

Do not infer support from a generic “JavaScript compatibility” chart.


# 102. When Not to Polyfill

Avoid polyfills when:

```text
security properties cannot be reproduced
performance would be unacceptable
semantic fidelity is insufficient
bundle cost is excessive
native support is a hard requirement
```

Failing fast can be safer than silently approximating.


# 103. When Not to Transpile

Avoid aggressive transpilation when:

```text
target fleet already supports the feature
transform greatly increases bundle size
transform harms performance
debugging becomes harder
semantic equivalence is uncertain
```

Modern code is not automatically better if the compatibility transform creates a bigger operational cost.


# 104. When to Raise the Minimum Baseline

Raise the baseline when:

```text
old-user share is low
security support is weak
compatibility code is expensive
modern runtime gains are significant
dependencies have moved
```

Quantify the trade-off.

Do not use developer convenience as the only argument.


# 105. Compatibility Exceptions

Sometimes one customer needs an older environment.

Handle this through an explicit exception:

```text
customer
environment
constraint
business reason
additional engineering cost
expiration
owner
```

Do not silently contaminate the entire platform architecture for one undocumented exception.


# 106. Compatibility Governance

A mature organization has:

```text
platform baseline owner
browser policy
Node policy
library policy
dependency policy
runtime upgrade policy
exception process
deprecation process
compatibility CI
telemetry
```

This transforms compatibility from tribal knowledge into platform governance.


# 107. Compatibility RFC Template

```md
# Compatibility RFC

## Change
What capability is changing?

## Target
Which environments are supported?

## Current Baseline
-

## Proposed Baseline
-

## Evidence
-

## Product Impact
-

## Technical Impact
-

## Performance
-

## Security
-

## Migration
-

## Fallback
-

## CI Changes
-

## Observability
-

## Rollback
-

## Removal Plan
-

## Decision
-
```


# 108. Incident Scenario — Unsupported Syntax

Incident:

```text
10% of clients receive a blank page.
```

Investigation:

```text
new syntax deployed
↓
old browser parser rejects script
↓
no fallback artifact
```

Corrective actions:

```text
target-aware build
browser compatibility CI
canary test
artifact selection
minimum baseline enforcement
```

Lesson:

> A parser failure happens before application-level recovery logic can execute.


# 109. Incident Scenario — Missing API

Incident:

```text
feature works on developer laptops
fails on older mobile devices
```

Cause:

```text
runtime API missing
```

Corrective architecture:

```text
capability detection
+
polyfill/fallback
+
behavioral test
+
real-device CI
```

Lesson:

> Compile-time success is not runtime compatibility.


# 110. Incident Scenario — Semantic Difference

Incident:

```text
same API
different output
```

Potential causes:

```text
locale
timezone
browser bug
polyfill
input normalization
implementation difference
```

Corrective action:

```text
capture exact environment
reproduce
consult specification
compare native/fallback path
create regression test
```


# 111. Incident Scenario — Runtime Upgrade

Incident:

```text
Node upgrade
→ latency regression
```

The language feature may not be the issue.

Investigate:

```text
V8 optimization
GC
HTTP implementation
TLS
dependency versions
startup behavior
module loading
```

Compatibility engineering includes performance compatibility.


# 112. Anti-Pattern Catalog

Avoid:

```text
latest-browser assumptions
user-agent sniffing
global polyfill dumping
unbounded legacy support
feature detection everywhere
hidden compatibility code
version literals with no reason
unsupported syntax in published packages
ignoring dependency engines
testing only latest runtime
treating Stage 4 as universal support
```

Each is a different expression of the same mistake:

```text
implicit compatibility assumptions
```


# 113. Common Misconceptions

### “Transpilation gives full compatibility.”

No. It primarily addresses source transformation.

### “Polyfills make old runtimes behave exactly like new engines.”

Not necessarily.

### “Baseline means our application is safe.”

No. It is a browser-availability summary.

### “Node.js support equals V8 support.”

No. Node adds host/runtime constraints.

### “Feature detection solves syntax.”

No. Unsupported syntax can fail at parse time.

### “Latest runtime is always fastest for our workload.”

Needs measurement.

### “Compatibility is only frontend work.”

No. Node, libraries, CI, serverless and tooling all have baselines.


# 114. Interview Questions — Fundamentals

1. What is compatibility?
2. What is syntax compatibility?
3. What is API compatibility?
4. What is semantic compatibility?
5. Why is feature detection better than browser-name detection?
6. What can a transpiler solve?
7. What can a polyfill solve?
8. Why can’t a polyfill fix arbitrary syntax?
9. What is progressive enhancement?
10. What is graceful degradation?


# 115. Interview Questions — Senior

1. How do you define a browser support baseline?
2. How do you define a Node.js support baseline?
3. How do you build a compatibility matrix?
4. How do you handle a dependency that raises the runtime requirement?
5. How do you debug a browser-specific failure?
6. What belongs in compatibility CI?
7. When would you raise your minimum runtime?
8. When would you avoid a polyfill?
9. How do you handle partial API support?
10. How do you prevent compatibility debt?


# 116. Interview Questions — Principal

1. Design compatibility governance for a global web platform.
2. How would you balance legacy-browser support against engineering velocity?
3. How would you prove that a baseline increase is safe?
4. How would you migrate a monolith from legacy browser support?
5. How would you support modern and legacy clients without doubling operational complexity?
6. How would you detect an unsupported syntax regression before release?
7. How would you manage compatibility across a multi-year Node.js fleet?
8. How would you evaluate a Stage 3/4 ECMAScript feature?
9. How would you build a compatibility intelligence dashboard?
10. How would you explain compatibility ROI to leadership?


# 117. Predict-the-Result Exercises

Predict first.

### Exercise 1

```js
console.log(typeof globalThis.AbortController);
```

The exact result depends on the runtime.

The lesson is:

```text
runtime capability must be observed in the target environment.
```

### Exercise 2

```js
if (typeof globalThis.SomeFutureApi === "function") {
  useIt();
} else {
  useFallback();
}
```

Question:

> Can this recover from unsupported syntax inside `useIt()`?

No, not if the parser cannot parse the script containing that syntax.

### Exercise 3

```js
const supports = typeof globalThis.Promise === "function";
```

This proves only that `Promise` exists.

It does not prove:

```text
all Promise semantics needed by the application
```


# 118. Mastery Exercise — Build a Compatibility Matrix

Create a matrix for:

```text
Chrome
Firefox
Safari
Edge
Node
your CI runtime
```

Columns:

```text
syntax
API
semantic
tooling
fallback
minimum version
source
observed date
```

Then select one feature from Chapter 90 and fill the matrix from primary/structured sources.


# 119. Mastery Exercise — Build a Baseline Policy

Write a formal policy:

```text
Web:
____________________

Node:
____________________

Library:
____________________

CI:
____________________

Serverless:
____________________
```

Then define:

```text
support window
exception path
deprecation process
upgrade cadence
ownership
```


# 120. Mastery Exercise — Compatibility Incident Drill

Scenario:

```text
new release
↓
older browser crash rate increases
```

Produce:

```text
incident hypothesis
environment capture
compatibility source review
feature detection review
build artifact review
rollback
regression test
long-term fix
removal plan
```


# 121. Mastery Exercise — Principal Compatibility Review

Review a proposed feature using:

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

Final recommendation:

```text
ADOPT
ADOPT WITH GUARD
TRANSFORM/POLYFILL
FALLBACK
RAISE BASELINE
REJECT
```

Defend it with evidence.


# 122. Spaced Retrieval Schedule

### Day 0
Explain the compatibility stack.

### Day 1
Explain syntax vs API compatibility.

### Day 3
Explain transpiler vs polyfill.

### Day 7
Build a browser/runtime matrix.

### Day 14
Design a compatibility policy.

### Day 30
Debug a mixed-version incident.

### Day 60
Rebuild a compatibility adapter from memory.

### Day 90
Defend a principal-level baseline decision.


# 123. Retrieval Prompts

Answer without notes:

```text
What does compatibility mean?
What are the five compatibility layers?
Why does syntax require special treatment?
What can feature detection tell you?
Why is user-agent sniffing fragile?
What can a polyfill do?
What can transpilation do?
What can neither do?
What is Baseline?
Why is Baseline not a product guarantee?
How do you define a fleet baseline?
How do you measure compatibility debt?
How do you test a runtime upgrade?
```


# 124. Dependency Graph

```text
Chapter 41 — Spec Architecture
        ↓
Chapter 91 — TC39 Proposal Tracking
        ↓
Chapter 90 — Modern ECMAScript Features
        ↓
Chapter 94 — Compatibility Engineering
        ├── Chapter 68 — Transpilation / Compilation
        ├── Chapter 69 — Bundlers / Build Systems
        ├── Chapter 67 — Dependency / Supply Chain
        ├── Chapter 70 — Source Maps / Production Debugging
        ├── Chapter 86 — Testing
        ├── Chapter 84 — Reliability
        ├── Chapter 85 — Performance
        └── Chapter 93 — Decorators
```

Compatibility engineering operationalizes language evolution.


# 125. Concept Connections

## Depends On

- ECMAScript specification.
- JavaScript engine architecture.
- Browser/runtime behavior.
- Modules.
- Dependency management.
- Build systems.
- Testing.
- Proposal tracking.

## Builds Toward

- production architecture,
- API design,
- library authoring,
- reliability,
- performance,
- security,
- large-scale platform engineering.

## Related Concepts

- graceful degradation,
- progressive enhancement,
- feature detection,
- semantic versioning,
- support policy,
- deprecation,
- canary releases.

## Concepts Revisited

- standard vs implementation,
- language vs host,
- proposal stage vs shipping support,
- tooling vs runtime,
- source vs executable artifact.

## Why This Chapter Matters Later

Every production JavaScript system exists inside a compatibility envelope.

The principal engineer defines that envelope deliberately.


# 126. Principal Decision Framework

For every platform capability, answer:

```text
1. What problem does it solve?
2. What exact semantics are needed?
3. What environments must support it?
4. Is syntax involved?
5. Is a runtime API involved?
6. Is a host API involved?
7. What toolchain support exists?
8. What evidence is available?
9. What fallback exists?
10. What is the performance cost?
11. What is the security cost?
12. What is the maintenance cost?
13. What is the removal plan?
```

Then decide.

This is the compatibility counterpart of the broader principal decision model.


# 127. Production Checklist

Before shipping a new language/platform feature:

```text
[ ] Standards status verified
[ ] Proposal/spec source recorded
[ ] Target runtime support verified
[ ] Browser support verified
[ ] Toolchain support verified
[ ] Dependency graph checked
[ ] Syntax compatibility checked
[ ] API behavior tested
[ ] Fallback defined
[ ] Security reviewed
[ ] Performance measured
[ ] Memory measured where relevant
[ ] CI matrix updated
[ ] Canary plan defined
[ ] Observability available
[ ] Rollback defined
[ ] Removal trigger documented
```


# 128. Canonical Current References

Primary / authoritative sources used in this chapter:

1. **ECMAScript Specification**
   https://tc39.es/ecma262/
   The current specification is the primary source for standardized ECMAScript semantics. citeturn959902search12

2. **TC39 Process**
   https://tc39.es/process-document/

3. **TC39 Proposals**
   https://github.com/tc39/proposals

4. **MDN JavaScript**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript
   MDN distinguishes ECMAScript from browser-specific Web APIs and notes that proposed ECMAScript features can be implemented before official publication. citeturn959902search4turn959902search9

5. **MDN Baseline Compatibility**
   https://developer.mozilla.org/en-US/docs/Glossary/Baseline/Compatibility
   Current definition and browser set verified on 2026-09-10. citeturn959902search0

6. **MDN Compatibility Data Guidance**
   https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Page_structures/Compatibility_tables
   Compatibility data is available programmatically through MDN Browser Compatibility Data. citeturn959902search1

7. **MDN JavaScript Technologies Overview**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/JavaScript_technologies_overview
   Useful for separating ECMAScript from browser APIs and implementations. citeturn959902search9

8. **Node.js Documentation**
   https://nodejs.org/docs/

9. **Can I Use**
   https://caniuse.com/
   Useful as a secondary compatibility discovery source; verify critical claims against primary/structured sources.

10. **Web Features / Baseline ecosystem**
    https://web-platform-dx.github.io/web-features/

Source discipline:

```text
standards claim
→ standards source

browser support claim
→ compatibility data + browser evidence

Node claim
→ Node/runtime evidence

production readiness
→ internal fleet evidence
```


# 129. Source Verification Notes — 2026-09-10

Current-source verification performed for this chapter:

- MDN's current Baseline compatibility glossary was updated August 27, 2026 and defines Baseline as a summary of availability across a core browser set; it explicitly says Baseline does not substitute for other forms of testing. citeturn959902search0
- MDN's compatibility-table guidance confirms structured Browser Compatibility Data as a reusable source for compatibility information. citeturn959902search1
- MDN's JavaScript reference explains that some ECMAScript proposals may already be implemented and documented before official specification publication, especially around Stages 3–4. citeturn959902search4
- The current ECMAScript specification remains the primary standardized language source and contains the most recent yearly snapshot plus finished proposals since that snapshot. citeturn959902search12

These claims describe current source practices. Runtime and browser support are date-sensitive and must be rechecked for production decisions.


# 130. Chapter 94 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I define compatibility as a multi-dimensional property? [ ]
- Could I separate standard status from shipping status? [ ]
- Could I explain syntax vs API compatibility? [ ]
- Could I explain feature detection? [ ]
- Could I explain why transpilation is not a polyfill? [ ]
- Could I build a compatibility matrix? [ ]
- Could I define a fleet baseline? [ ]
- Could I design compatibility CI? [ ]
- Could I explain compatibility debt? [ ]
- Could I defend a baseline change? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```


# 131. Chapter 94 — Completion Snapshot

```text
Track A — Core Theory
[ ] Compatibility stack
[ ] Standards vs implementation vs deployment truth
[ ] Syntax compatibility
[ ] API compatibility
[ ] Semantic compatibility
[ ] Host compatibility
[ ] Toolchain compatibility
[ ] Browser Baseline
[ ] Feature detection
[ ] Polyfills
[ ] Transpilation
[ ] Support policies
[ ] Compatibility debt

Track B — Implementation
[ ] Capability detector
[ ] Capability registry
[ ] Compatibility adapter
[ ] Browser matrix
[ ] Runtime matrix
[ ] Production report
[ ] Compatibility CI
[ ] Behavioral tests
[ ] Stale-data handling
[ ] Exception workflow

Track C — Interview / Reasoning
[ ] Explain support baselines
[ ] Defend feature detection
[ ] Compare polyfill vs transpilation
[ ] Design multi-browser architecture
[ ] Design Node runtime policy
[ ] Evaluate dependency compatibility
[ ] Design runtime upgrade process
[ ] Defend baseline changes
[ ] Perform principal compatibility review

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


# 132. Completion Criteria

Do not mark this chapter mastered because you memorized compatibility vocabulary.

You are ready to move forward when you can independently:

1. Define your product's support baseline.
2. Explain why the baseline exists.
3. Distinguish language, runtime, host, tooling and fleet compatibility.
4. Determine whether a new syntax feature can execute in a target environment.
5. Determine whether a required API can execute.
6. Build a capability detector.
7. Explain when a feature test is insufficient.
8. Explain the limits of a polyfill.
9. Explain the limits of transpilation.
10. Build a browser/runtime compatibility matrix.
11. Design a compatibility CI matrix.
12. Debug a browser-specific failure.
13. Debug a runtime-specific failure.
14. Review a dependency upgrade for compatibility risk.
15. Identify compatibility debt.
16. Define a runtime upgrade strategy.
17. Design a progressive-enhancement architecture.
18. Defend an increase in the minimum supported runtime.
19. Reject a feature despite its standards maturity when deployment evidence is inadequate.
20. Produce a principal-level compatibility decision memo.


# 133. Principal Challenge

Design compatibility governance for a global JavaScript platform serving:

```text
web browsers
mobile browsers
embedded WebViews
Node.js services
serverless functions
internal developer tooling
published npm libraries
```

Your design must contain:

```text
1. Browser baseline
2. Node baseline
3. Library baseline
4. Toolchain baseline
5. Proposal adoption policy
6. Feature-detection standard
7. Polyfill policy
8. Transpilation policy
9. Dependency policy
10. CI matrix
11. Canary strategy
12. Runtime upgrade strategy
13. Compatibility telemetry
14. Exception process
15. Deprecation process
16. Compatibility debt register
17. Security policy
18. Performance policy
19. Rollback strategy
20. Ownership model
```

Then defend the design against this pressure:

> “Supporting one extra old environment will help a few customers. Why not just keep supporting it forever?”

Your answer should quantify:

```text
customer value
engineering cost
security cost
testing cost
performance cost
operational cost
removal cost
```

The principal answer is not automatically “support it” or “drop it.”

The principal answer is:

> **Define the support contract from evidence, make the trade-off explicit, and manage the lifecycle.**


# 134. Final Mental Model

```text
TC39 / Standards
       ↓
Specification semantics
       ↓
Engine implementation
       ↓
Host APIs
       ↓
Toolchain transformation
       ↓
Artifact
       ↓
Target environment
       ↓
Production fleet
```

At every arrow ask:

```text
Is the contract preserved?
```

The most important compatibility distinctions are:

```text
standardized
≠
implemented

implemented
≠
tool-supported

tool-supported
≠
deployed

deployed
≠
universally compatible

available
≠
correct

correct
≠
secure

secure
≠
performant

performant
≠
operationally worthwhile
```

That is compatibility engineering.


# 135. Final Principal Rule

> **Never ask only “Does JavaScript support this?” Ask “Which specification, which implementation, which host, which toolchain, which versions, which users, and which fallback policy are we committing to?”**

That question separates feature adoption from engineering judgment.


# Chapter 94 — Canonical References and Source Discipline

This chapter is intended to be periodically refreshed.

For future reviews, verify:

```text
ECMAScript specification
TC39 proposal state
browser compatibility data
MDN Baseline definition
Node.js support
toolchain support
internal fleet data
```

Record the observation date.

Do not treat this chapter's examples or current-source snapshot as timeless runtime guarantees.


# Chapter 94 — Completion Snapshot

```text
Chapter: 94
Title: Compatibility Engineering
Part: XVII — Modern ECMAScript & Language Evolution
Format: Standalone Markdown chapter
Verification date: 2026-09-10
Status: [ ] Not Started

Core mastery target:
Build and defend an explicit compatibility contract across standards,
engines, hosts, toolchains, dependencies, and production fleets.
```

---

# 136. Deep Drill — Classify the Failure

For each incident, identify the layer:

### A
Old browser rejects a new token.

Answer:

```text
Syntax compatibility
```

### B
Browser parses code but `SomeAPI` is undefined.

Answer:

```text
Runtime API compatibility
```

### C
API exists but an option is ignored.

Answer:

```text
Semantic/implementation compatibility
```

### D
Node runs code but production package loader rejects its exports.

Answer:

```text
Module/runtime/tooling compatibility
```

### E
Everything works but the fallback doubles bundle size.

Answer:

```text
Performance compatibility
```

---

# 137. Deep Drill — Build vs Runtime

Given:

```js
const fn = obj?.method?.();
```

Ask:

```text
Can an old runtime parse it?
Can Babel transform it?
Does the transformed output require helpers?
Does the package publish transformed output?
```

The correct answer depends on the build artifact, not merely the source repository.

---

# 138. Deep Drill — API Detection

Given:

```js
if ("foo" in obj) {
  obj.foo();
}
```

Question:

> Does this guarantee `foo()` can be called successfully?

No.

It may be:

```text
non-function
getter with side effects
partial implementation
buggy implementation
```

Capability detection must match the operation.

---

# 139. Deep Drill — Version Detection

Given:

```js
if (browserVersion >= 120) {
  useFeature();
}
```

Ask:

```text
Where does browserVersion come from?
Can it be spoofed?
Are all variants equivalent?
Is the feature independently backported?
Is there a known browser-specific bug?
```

Then decide whether capability detection or version policy is better.

---

# 140. Deep Drill — Fallback Correctness

Suppose:

```text
native crypto
```

is unavailable.

A fallback uses:

```text
Math.random()
```

Is this compatible?

No.

It may preserve:

```text
“something random-like”
```

but not the required security semantics.

Compatibility means preserving the required contract, not merely making a call succeed.

---

# 141. Deep Drill — Time to Remove a Polyfill

Ask:

```text
What percentage of supported users still need it?
What is the bundle cost?
What maintenance issues exist?
What browsers remain?
Can the support baseline be raised?
```

Do not remove a compatibility layer just because the feature is now standard.

---

# 142. Deep Drill — Dependency Upgrade

A package changes:

```json
"engines": {
  "node": ">=24"
}
```

Your production baseline is:

```text
Node 22
```

Correct response:

```text
Do not merge automatically.
```

Investigate:

```text
Can we stay on prior version?
Can we raise baseline?
Can we isolate the dependency?
Can we replace it?
```

Then make an explicit decision.

---

# 143. Deep Drill — Browser Baseline

A feature becomes Baseline newly available.

Question:

> Can we immediately remove all fallback code?

Not necessarily.

Your product may still support:

```text
older corporate devices
embedded webviews
long-lived OS versions
```

Baseline is a useful web-wide signal, not your application's support contract. citeturn959902search0

---

# 144. Deep Drill — Production Fleet

Suppose:

```text
97% supported
3% unsupported
```

What should you do?

No universal answer.

Evaluate:

```text
3% of what?
revenue?
critical customers?
high-risk devices?
internal users?
```

Compatibility decisions are product decisions with engineering cost.

---

# 145. Deep Drill — Source of Truth

If sources disagree:

```text
Blog says supported.
MDN says limited.
Browser release notes say experimental.
Your CI passes in one version.
```

Do not select the most convenient answer.

Triangulate:

```text
exact version
exact feature
exact behavior
exact source date
```

Then run an empirical test.

# 146. Advanced Compatibility Drill 1

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 147. Advanced Compatibility Drill 2

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 148. Advanced Compatibility Drill 3

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 149. Advanced Compatibility Drill 4

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 150. Advanced Compatibility Drill 5

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 151. Advanced Compatibility Drill 6

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 152. Advanced Compatibility Drill 7

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 153. Advanced Compatibility Drill 8

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 154. Advanced Compatibility Drill 9

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 155. Advanced Compatibility Drill 10

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 156. Advanced Compatibility Drill 11

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 157. Advanced Compatibility Drill 12

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 158. Advanced Compatibility Drill 13

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 159. Advanced Compatibility Drill 14

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 160. Advanced Compatibility Drill 15

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 161. Advanced Compatibility Drill 16

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 162. Advanced Compatibility Drill 17

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 163. Advanced Compatibility Drill 18

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 164. Advanced Compatibility Drill 19

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 165. Advanced Compatibility Drill 20

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 166. Advanced Compatibility Drill 21

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 167. Advanced Compatibility Drill 22

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 168. Advanced Compatibility Drill 23

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 169. Advanced Compatibility Drill 24

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 170. Advanced Compatibility Drill 25

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 171. Advanced Compatibility Drill 26

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 172. Advanced Compatibility Drill 27

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 173. Advanced Compatibility Drill 28

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 174. Advanced Compatibility Drill 29

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 175. Advanced Compatibility Drill 30

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 176. Advanced Compatibility Drill 31

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 177. Advanced Compatibility Drill 32

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 178. Advanced Compatibility Drill 33

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 179. Advanced Compatibility Drill 34

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 180. Advanced Compatibility Drill 35

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 181. Advanced Compatibility Drill 36

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 182. Advanced Compatibility Drill 37

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 183. Advanced Compatibility Drill 38

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 184. Advanced Compatibility Drill 39

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 185. Advanced Compatibility Drill 40

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 186. Advanced Compatibility Drill 41

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 187. Advanced Compatibility Drill 42

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 188. Advanced Compatibility Drill 43

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 189. Advanced Compatibility Drill 44

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

# 190. Advanced Compatibility Drill 45

This drill is designed to reinforce principal-level reasoning.

### Scenario

A team wants to introduce a new JavaScript or Web Platform capability into a heterogeneous fleet.

Evaluate:

```text
1. Standards status
2. Proposal/documentation source
3. Syntax requirements
4. Runtime API requirements
5. Host API requirements
6. Engine implementation
7. Browser/runtime version range
8. Toolchain parser support
9. Build transformation needs
10. Polyfill requirements
11. Dependency constraints
12. Security implications
13. Performance implications
14. Memory implications
15. Observability
16. Test coverage
17. Rollback
18. Support baseline
19. Migration
20. Removal strategy
```

Then write:

```text
Decision:
Evidence:
Unknowns:
Fallback:
Owner:
Review date:
```

#### Principal question

What evidence would cause you to reverse your decision?

Your answer must include at least:

```text
one standards signal
one implementation signal
one fleet signal
one operational signal
```

Do not treat the stage label or browser availability badge as sufficient evidence.

---

# 191. Final Reference Card

```text
Compatibility
=
standards
+
implementation
+
host
+
toolchain
+
fleet

Syntax
→ parser problem

API
→ capability problem

Semantics
→ behavior problem

Polyfill
→ runtime behavior emulation

Transpilation
→ source transformation

Feature detection
→ capability decision

Version detection
→ policy / known-bug decision

Baseline
→ useful web-wide availability signal

Production baseline
→ your explicit support contract

Compatibility debt
→ fallback/workaround cost that needs lifecycle management
```

### Core rule

```text
Do not ask:
“Is this feature supported?”

Ask:
“Supported where, by what version, with what semantics,
through which artifact, for which users, under what policy?”
```

---

# Chapter 94 — Revision / Retrieval Record

```md
### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- I can explain the five compatibility layers. [ ]
- I can distinguish syntax from API compatibility. [ ]
- I can explain feature detection. [ ]
- I can explain polyfills vs transpilation. [ ]
- I can build a browser/runtime matrix. [ ]
- I can define a fleet baseline. [ ]
- I can debug compatibility incidents. [ ]
- I can evaluate dependency compatibility. [ ]
- I can defend a baseline change. [ ]
- I can design compatibility governance. [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 94 — Canonical References and Source Discipline

### Primary

```text
https://tc39.es/ecma262/
https://tc39.es/process-document/
https://github.com/tc39/proposals
https://developer.mozilla.org/en-US/docs/Web/JavaScript
https://developer.mozilla.org/en-US/docs/Glossary/Baseline/Compatibility
https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Page_structures/Compatibility_tables
https://nodejs.org/docs/
```

### Secondary

```text
https://caniuse.com/
https://web-platform-dx.github.io/web-features/
```

### Verification discipline

```text
[ ] Source date recorded
[ ] Runtime version recorded
[ ] Browser version recorded
[ ] Implementation behavior verified
[ ] Toolchain behavior verified
[ ] Internal fleet data verified
[ ] Production decision documented
```

---

# Chapter 94 — Completion Snapshot

```text
Chapter 94 — Compatibility Engineering
[ ] Not Started

Readiness:
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend

Principal outcome:
An explicit, versioned, tested, observable compatibility contract.
```

> **Mastery reminder:** Reading this chapter does not mark it completed. Mastery requires retrieval, implementation, debugging, application, comparison, and defense.