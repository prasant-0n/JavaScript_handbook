# Chapter 90 — Modern ECMAScript Features

> **Part XVII — Modern JavaScript / Evolution**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Mastering modern standardized ECMAScript features, understanding what changed and why, distinguishing language evolution from host APIs and proposals, and adopting new capabilities without creating compatibility or maintenance debt.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what “modern ECMAScript” means.
2. Distinguish ECMAScript editions from JavaScript runtime support.
3. Distinguish standardized features from Stage 4 proposals and earlier proposals.
4. Explain why TC39 evolves the language incrementally.
5. Understand the relationship between TC39 proposals and annual ECMAScript editions.
6. Identify the current ECMAScript 2026 additions.
7. Understand modern iterator capabilities.
8. Use `Iterator` and iterator helper methods appropriately.
9. Use `Iterator.concat()` appropriately.
10. Use `Array.fromAsync()` correctly.
11. Understand `Promise.withResolvers()`.
12. Understand `Promise.try()`.
13. Use modern `Set` composition methods.
14. Use `RegExp.escape()` safely.
15. Understand RegExp modifier syntax.
16. Understand JSON module imports and import attributes.
17. Use `Float16Array`, `Math.f16round()`, and related APIs.
18. Understand `Math.sumPrecise()`.
19. Understand `Error.isError()`.
20. Understand `Map.prototype.getOrInsert()`-style modern APIs where supported by the relevant ECMAScript edition/runtime.
21. Understand new `Uint8Array` Base64/hex conversion methods.
22. Understand `JSON.rawJSON()` and `JSON.parse()` reviver source-context capabilities.
23. Connect new features to older language concepts.
24. Understand when a modern feature improves correctness or expressiveness.
25. Understand when a modern feature adds unnecessary complexity.
26. Evaluate runtime compatibility before adoption.
27. Choose native support versus transpilation/polyfilling deliberately.
28. Understand semantic differences between syntax transforms and runtime polyfills.
29. Analyze performance implications of modern language features.
30. Analyze memory implications.
31. Analyze security implications.
32. Write migration strategies for older codebases.
33. Review modern JavaScript changes for API and compatibility impact.
34. Build practical examples using modern ECMAScript features.
35. Track feature support without confusing browser/runtime support with language standardization.
36. Use modern features without treating all new syntax as automatically superior.
37. Explain feature provenance during technical interviews.
38. Defend modernization decisions at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript values and types.
- Functions and closures.
- Objects and prototypes.
- Iterables and iterators.
- Generators.
- Promises.
- Async/await.
- Regular expressions.
- JSON.
- Typed arrays and binary data.
- Modules.
- Error handling.
- Resource management.
- Node.js and browser runtime distinctions.
- Transpilation and compilation.
- Bundlers and build systems.
- Compatibility engineering.

Recommended prior chapters:

- **22–28** — Data Structures / Iterators / Generators / Typed Arrays / JSON
- **29–30** — Errors / Resource Management
- **31–40** — Async / Promises / Cancellation / Streaming
- **41–44** — ECMAScript Specification Architecture
- **64–70** — Modules / Packages / Transpilation / Bundlers / Source Maps
- **90** onward — Modern ECMAScript evolution
- **94** — Compatibility Engineering

---

# 3. What Is It?

Modern ECMAScript is the evolving standardized language commonly implemented by JavaScript engines.

The key distinction is:

```text
JavaScript
≈ common name for the language ecosystem

ECMAScript
= standardized language specification

Node.js / browsers / Deno / other runtimes
= hosts that implement ECMAScript plus host APIs
```

The most recent finalized annual ECMAScript specification is the **ECMAScript 2026 Language Specification**, the 17th edition. The TC39 specification site describes the current specification as the most accurate/up-to-date snapshot and explains that finished Stage 4 proposals become part of the yearly specification. citeturn749000search8turn749000search7

The ECMAScript 2026 additions include:

```text
Math.sumPrecise()
Iterator.concat()
Array.fromAsync()
Error.isError()
Map/WeakMap get-or-insert methods
Uint8Array Base64/hex conversions
JSON.parse reviver source context
JSON.rawJSON()
```

along with other standardized changes included in the edition. citeturn749000search7

---

# 4. Why Does It Exist?

Language evolution solves recurring developer problems.

Examples:

```text
repeated boilerplate
unsafe patterns
missing primitives
awkward asynchronous composition
insufficient data-processing tools
missing low-level capabilities
```

A standardized feature can replace:

```text
custom helper
polyfill
utility package
home-grown abstraction
```

But a new language feature also has costs:

```text
learning
compatibility
documentation
tooling
migration
code review complexity
```

Modern does not mean:

```text
always better
```

The engineering question is:

> Does the standardized feature improve this codebase's correctness, expressiveness, compatibility, or operational properties enough to justify adoption?

---

# 5. Mental Model

Think of a modern ECMAScript feature's lifecycle:

```text
idea
  ↓
proposal
  ↓
Stage 1
  ↓
Stage 2
  ↓
Stage 3
  ↓
Stage 4
  ↓
annual ECMAScript edition
  ↓
engine/runtime implementation
  ↓
application adoption
```

Important:

```text
standardized
≠
supported everywhere
```

and:

```text
implemented somewhere
≠
standardized
```

---

# 6. Core Rules

## Rule 1 — Learn the feature's semantics, not just syntax

For:

```js
Array.fromAsync(source)
```

learn:

```text
what counts as source
awaiting behavior
ordering
error propagation
memory implications
```

---

## Rule 2 — Verify actual support

Before using a new feature:

```text
target browsers
Node versions
other runtimes
bundlers
test environments
TypeScript/lib definitions
```

---

## Rule 3 — Distinguish language from host

For example:

```text
Array.fromAsync()
```

is language-level ECMAScript.

But:

```text
fetch()
```

is a host/runtime web API rather than an ECMAScript language primitive.

---

## Rule 4 — Prefer native semantics when available

Avoid reproducing standardized functionality with fragile custom utilities once your support baseline can safely use the standard API.

---

## Rule 5 — Do not polyfill blindly

A polyfill may emulate:

```text
surface syntax
```

but can have limitations around:

```text
performance
intrinsics
engine optimizations
exact semantics
native object branding
```

---

## Rule 6 — New syntax is not automatically clearer

A feature should reduce conceptual complexity, not merely reduce characters.

---

## Rule 7 — Preserve compatibility deliberately

A modernization may require:

```text
runtime upgrade
bundle target change
polyfill
feature detection
dual implementation
migration
```

---

## Rule 8 — Use feature provenance

For every modern feature know:

```text
proposal / edition
specification status
runtime support
host dependencies
```

---

## Rule 9 — Read normative semantics when behavior is subtle

Especially for:

```text
iterators
Promises
RegExp
JSON
typed arrays
numeric precision
error identity
```

---

## Rule 10 — Modernization should reduce debt, not move it

A rewrite using every new feature can create a different kind of complexity.

---

# 7. Syntax

Modern ECMAScript includes syntax and built-ins.

Examples:

```js
const doubled = values.map(x => x * 2);
```

Modern standard APIs:

```js
const set = new Set([1, 2, 3]);

const overlap = set.intersection(
  new Set([2, 3, 4]),
);
```

Async conversion:

```js
const values = await Array.fromAsync(source);
```

Promise construction:

```js
const {
  promise,
  resolve,
  reject,
} = Promise.withResolvers();
```

Regex escaping:

```js
const pattern = new RegExp(
  RegExp.escape(userText),
);
```

---

# 8. Basic Examples

## Example 1 — Promise.withResolvers()

Before:

```js
let resolve;
let reject;

const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});
```

Modern:

```js
const {
  promise,
  resolve,
  reject,
} = Promise.withResolvers();
```

MDN documents `Promise.withResolvers()` as a standardized method available broadly since 2024. citeturn749000search1

---

## Example 2 — Set intersection

```js
const userPermissions = new Set([
  "read",
  "write",
]);

const requiredPermissions = new Set([
  "read",
  "delete",
]);

const overlap =
  userPermissions.intersection(
    requiredPermissions,
  );

console.log(overlap);
```

ECMAScript 2025 added Set methods for common set operations. citeturn749000search2

---

## Example 3 — RegExp.escape()

```js
const literal = "a+b";

const regex = new RegExp(
  RegExp.escape(literal),
);

console.log(regex.test("a+b"));
```

`RegExp.escape()` is standardized in ECMAScript 2025 and available broadly in modern environments. citeturn749000search0turn749000search2

---

# 9. Execution Walkthrough

Consider:

```js
const values = await Array.fromAsync(
  source,
);
```

Conceptually:

```text
evaluate source
  ↓
obtain async iteration behavior where applicable
  ↓
retrieve values
  ↓
await async values
  ↓
collect values
  ↓
create array
  ↓
resolve Promise
```

The important observation:

```text
Array.fromAsync()
```

is not merely:

```text
Array.from()
+
await
```

It has defined behavior around async sources and awaiting elements.

---

# 10. Internal Mechanics

## 10.1 Iterator helpers

Modern iterator facilities add methods directly to iterator-processing workflows.

Conceptually:

```js
const result =
  Iterator
    .from(values)
    .map(x => x * 2)
    .filter(x => x > 10);
```

Iterator-based processing can remain lazy until consumption.

This differs from:

```js
values
  .map(...)
  .filter(...);
```

because arrays eagerly materialize intermediate results.

The ECMAScript 2025 edition introduced the `Iterator` global and associated static/prototype methods. citeturn749000search2

---

## 10.2 `Iterator` versus `Array`

Array:

```text
materialized collection
```

Iterator:

```text
potentially lazy sequence
```

This affects:

```text
memory
timing
termination
side effects
```

---

## 10.3 Set methods

Modern Set composition includes operations conceptually such as:

```text
union
intersection
difference
symmetricDifference
isSubsetOf
isSupersetOf
isDisjointFrom
```

These make set algebra directly expressible using standard primitives.

ECMAScript 2025 standardized these Set methods. citeturn749000search2

---

# 11. ECMAScript / Specification Semantics

This chapter is specifically about language-level evolution.

The TC39 specification site describes the latest specification as including the current yearly snapshot plus finished Stage 4 proposals. citeturn749000search8

Therefore classify:

```text
Stage 4
→ standardized feature

Stage 3
→ advanced proposal, not yet final language standard

Stage 2 and earlier
→ proposal, not a production language guarantee
```

Do not write:

> “JavaScript has feature X”

when X is only a proposal in your target environment.

For modern feature adoption, use:

```text
spec status
+
runtime support
+
compatibility target
```

---

# 12. Advanced Behavior

## 12.1 Promise.withResolvers()

`Promise.withResolvers()` returns:

```js
{
  promise,
  resolve,
  reject,
}
```

MDN notes that it is equivalent in purpose to manually capturing Promise constructor callbacks, while also being generic and supporting Promise subclassing under the relevant constructor requirements. citeturn749000search1

Useful when completion is controlled by events:

```text
stream event
queue event
callback API
manual state transition
```

Example:

```js
function waitForEvent(emitter, name) {
  const {
    promise,
    resolve,
    reject,
  } = Promise.withResolvers();

  const onValue = value => {
    cleanup();
    resolve(value);
  };

  const onError = error => {
    cleanup();
    reject(error);
  };

  const cleanup = () => {
    emitter.off(name, onValue);
    emitter.off("error", onError);
  };

  emitter.on(name, onValue);
  emitter.on("error", onError);

  return promise;
}
```

---

## 12.2 Promise.try()

ECMAScript 2025 added:

```js
Promise.try(fn)
```

It provides a convenient way to turn a callback that may:

```text
return
throw synchronously
return a Promise
```

into a Promise-based result. The feature is part of the ECMAScript 2025 edition. citeturn749000search2turn749000search4

Conceptually:

```js
Promise.try(() => possiblyThrows());
```

This can simplify Promise-oriented APIs around mixed sync/async callbacks.

Do not use it just because the word “Promise” looks modern.

---

## 12.3 Array.fromAsync()

ECMAScript 2024 added `Array.fromAsync()`. citeturn749000search2

Conceptually:

```js
const array =
  await Array.fromAsync(asyncIterable);
```

Useful for:

```text
async generators
streams exposed as async iterables
paginated sources
async data transforms
```

Important:

```text
collection result = fully materialized array
```

So this is not a replacement for streaming when the source is huge.

---

## 12.4 Iterator.concat()

ECMAScript 2026 adds:

```js
Iterator.concat(...)
```

to sequence iterators. citeturn749000search7

Conceptually:

```js
const combined =
  Iterator.concat(
    [1, 2],
    [3, 4],
  );
```

This supports composition without requiring an eagerly merged array.

Use when:

```text
sequence composition
lazy traversal
potentially large sources
```

are important.

---

## 12.5 Math.sumPrecise()

ECMAScript 2026 adds:

```js
Math.sumPrecise(iterable)
```

to sum Numbers while reducing precision loss across values with varying magnitude. citeturn749000search7

This matters when:

```text
small + large magnitudes
```

could accumulate meaningful floating-point error.

It is not:

```text
arbitrary precision arithmetic
```

and does not replace:

```text
Decimal-like exact monetary modeling
BigInt for integer-domain quantities
```

---

## 12.6 Error.isError()

ECMAScript 2026 adds:

```js
Error.isError(value)
```

for identifying error objects according to the standard's semantics. citeturn749000search7

This is useful in boundaries that receive unknown values.

Example:

```js
function classify(value) {
  if (Error.isError(value)) {
    return "error";
  }

  return "other";
}
```

---

## 12.7 Map / WeakMap get-or-insert methods

ECMAScript 2026 adds methods that provide a default value when a key is absent. citeturn749000search7

The important architectural idea is:

```text
lookup-or-initialize
```

becomes more directly expressible than repeated:

```js
if (!map.has(key)) {
  map.set(key, createValue());
}
```

Check the exact method names and availability in the runtime baseline you target before using them in production libraries.

---

## 12.8 Uint8Array binary conversions

ECMAScript 2026 adds `Uint8Array` methods for hexadecimal and Base64 string conversion. citeturn749000search7

Conceptually:

```js
const bytes =
  Uint8Array.fromHex("ff00aa");

const text =
  bytes.toHex();
```

and corresponding Base64 methods.

This can eliminate repeated custom binary encoding utilities.

Verify exact method names and available encoding modes against the target runtime/spec version.

---

## 12.9 JSON.rawJSON()

ECMAScript 2026 adds:

```js
JSON.rawJSON(...)
```

for controlling parts of `JSON.stringify()` output at the JSON boundary. citeturn749000search7

This is advanced functionality.

Do not use it merely to “make JSON faster.”

Understand:

```text
why raw representation is required
where validation occurs
how downstream parsers interpret output
```

---

## 12.10 JSON.parse reviver source context

ECMAScript 2026 adds source-context information to JSON parse revivers. citeturn749000search7

This can support workflows where the original JSON representation matters during transformation.

Use cases can include:

```text
numeric preservation decisions
source-aware transformation
loss-sensitive parsing
```

Treat it as a specialized capability, not a default parsing strategy.

---

# 13. Edge Cases

## 13.1 RegExp.escape() is not equivalent to manual backslash insertion

`RegExp.escape()` handles more cases than:

```js
text.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")
```

MDN specifically documents additional handling such as leading alphanumeric escaping and `\x` handling for punctuators that cannot safely be escaped by simply prefixing `\`. citeturn749000search0

Therefore:

> Prefer the standardized API over a homemade approximation when supported.

---

## 13.2 Iterator laziness

This:

```js
const iter =
  Iterator.from(values)
    .map(fn);
```

does not necessarily execute `fn` for every item immediately.

Consumption drives execution.

Compare:

```js
const array =
  values.map(fn);
```

which eagerly processes all elements.

---

## 13.3 Array.fromAsync() materializes

Even though the source may be lazy:

```js
const array =
  await Array.fromAsync(source);
```

eventually creates a full array.

For large streams:

```text
Array.fromAsync()
```

can become a memory problem.

---

## 13.4 Set operations and object identity

Set operations use Set equality semantics.

Objects:

```js
const a = {};
const b = {};

new Set([a]).has(b);
```

is:

```text
false
```

even if they look structurally identical.

---

## 13.5 Promise.withResolvers() lifecycle

Once created:

```js
const deferred =
  Promise.withResolvers();
```

the resolve/reject functions can outlive the local scope.

Store and expose them carefully.

They become a new state-management mechanism.

---

## 13.6 Float16 limits

Half-precision floating point has lower range and precision than Float32/Float64.

Use it where:

```text
memory/bandwidth
interop
ML/GPU-related representation
```

matter.

Do not silently replace financial numeric representations with Float16.

---

# 14. Common Misconceptions

### “ECMAScript 2026 means every browser supports it.”

No. Specification publication and runtime deployment are separate.

### “Stage 3 is standardized.”

No. Stage 3 means the proposal is advanced but not yet finalized.

### “New syntax is always faster.”

No.

### “Array.fromAsync() streams.”

It consumes an async source and materializes an array.

### “Iterator helpers always use less memory.”

They can preserve laziness, but downstream consumers can still materialize everything.

### “Set intersection is just an API convenience.”

It also makes intent and set algebra more explicit.

### “Promise.withResolvers() creates a more powerful Promise.”

It exposes the controls separately; it also makes it easier to retain them too long or create complicated state machines.

### “RegExp.escape() means regex injection is solved.”

It safely treats text as a literal regex pattern. It does not solve every regular-expression denial-of-service problem or unsafe pattern construction issue.

### “Math.sumPrecise() gives exact decimal arithmetic.”

No. Floating-point semantics remain floating-point semantics.

### “JSON.rawJSON() should be used to optimize JSON.”

It is a representation-control feature and should be used only when raw JSON insertion is genuinely required.

---

# 15. Common Mistakes

## Mistake 1 — Adopting by hype

```text
feature is new
→ use everywhere
```

---

## Mistake 2 — Ignoring baseline

```text
local runtime supports it
→ production does too
```

---

## Mistake 3 — Confusing proposal with standard

```text
Stage 3 blog post
→ production guarantee
```

---

## Mistake 4 — Polyfill everywhere

A polyfill can increase:

```text
bundle
startup
maintenance
```

---

## Mistake 5 — Modern syntax with legacy architecture

Changing:

```text
callbacks
→ async/await
```

does not fix:

```text
bad ownership
bad transactions
bad error handling
```

---

## Mistake 6 — Iterator without consumption understanding

A lazy pipeline can hide when work actually occurs.

---

## Mistake 7 — Materializing huge async sources

```js
await Array.fromAsync(hugeSource);
```

---

## Mistake 8 — Feature detection too late

Applications discover unsupported behavior only in production.

---

# 16. Comparison With Related Concepts

| Feature / Approach | Strength | Trade-off |
|---|---|---|
| `Promise.withResolvers()` | Explicit Promise controls | Easier to create retained deferred state |
| `Promise.try()` | Normalizes mixed sync/async callbacks | Can obscure where sync work occurs |
| Iterator helpers | Lazy composition | Requires iterator mental model |
| Array methods | Familiar/eager | Intermediate allocations |
| `Array.fromAsync()` | Simple async materialization | Memory grows with result |
| Set methods | Clear set algebra | Requires Set inputs |
| `RegExp.escape()` | Safer literal embedding | Only escapes literal data |
| Float16Array | Compact half precision | Lower precision/range |
| `Math.sumPrecise()` | Reduced floating-point summation error | Still numeric floating-point semantics |
| `JSON.rawJSON()` | Raw JSON representation control | Advanced correctness/security concerns |
| Custom helper | Full control | Maintenance and correctness burden |
| Polyfill | Compatibility | Added runtime/bundle cost |
| Runtime upgrade | Native semantics | Deployment coordination |

---


---

# 16A. Feature Adoption Matrix

Use the matrix before adopting a language feature:

| Dimension | Question |
|---|---|
| Problem | What real problem does this solve? |
| Semantics | What exactly changes? |
| Standard | Which ECMAScript edition? |
| Stage | Is it Stage 4? |
| Runtime | Which production runtimes support it? |
| Browser | Which browser baseline supports it? |
| Tooling | Can build/test tooling parse and bundle it? |
| Types | Are type definitions aligned? |
| Polyfill | Is a fallback practical? |
| Performance | What is the measured effect? |
| Memory | Does allocation/materialization change? |
| Security | Does the feature affect trust boundaries? |
| Migration | How much code changes? |
| Rollback | Can adoption be reversed safely? |
| Maintenance | What custom code can disappear? |

The matrix prevents:

```text
new feature
→ immediate adoption
```

and encourages:

```text
need
→ evidence
→ compatibility
→ adoption decision
```

---

# 16B. Native Feature Versus Polyfill

A native feature can benefit from:

```text
engine implementation
intrinsics
optimization opportunities
standardized semantics
runtime maintenance
```

A polyfill may provide:

```text
similar API surface
portable fallback
older-runtime support
```

But a polyfill may not reproduce:

```text
engine-level performance
internal slots/branding
native memory behavior
new runtime capabilities
```

Use a polyfill when it provides meaningful compatibility value.

Do not assume it provides identical implementation characteristics.

---

# 16C. Syntax Transformation Versus Runtime Capability

Transpilation can transform syntax:

```text
new syntax
→ older syntax
```

But it cannot magically create every runtime capability.

Example:

```text
new syntax
```

may transpile.

A new built-in:

```js
Promise.withResolvers()
```

may require a runtime implementation or polyfill.

Therefore classify a feature as:

```text
syntax-level
runtime built-in
host API
both syntax and runtime
```

before choosing a compatibility strategy.

---

# 16D. Feature Detection

Prefer capability detection when runtime variation is expected.

Conceptually:

```js
if (typeof Promise.withResolvers === "function") {
  // modern path
} else {
  // fallback
}
```

Feature detection should be paired with tests.

Do not infer support from:

```text
browser name
Node major version string alone
user-agent
```

when direct capability detection is practical.

---

# 16E. Minimum Runtime Baseline

A team can choose:

```text
support every currently deployed runtime
```

or:

```text
raise the supported runtime baseline
```

The latter can reduce:

```text
polyfills
transpilation
conditional code
documentation burden
testing combinations
```

but creates:

```text
deployment coordination
consumer upgrade requirement
legacy environment exclusion
```

Baseline changes are architectural decisions.

---

# 16F. Modernization by Risk

Prioritize modernization where:

```text
legacy code creates real correctness/security/maintenance risk
```

Example:

```text
manual regex escaping
→ RegExp.escape()
```

may reduce correctness risk.

Whereas:

```text
replace all old syntax with newest syntax
```

may produce little measurable value.

---

# 16G. Feature Interaction

Modern features can compose.

Example:

```js
const result =
  Iterator.concat(
    sourceA,
    sourceB,
  )
  .filter(predicate)
  .map(transform);
```

The value comes from the composition:

```text
sequence
→ lazy filtering
→ lazy mapping
```

But complexity can also compose.

A principal engineer asks:

```text
Can the next engineer understand the pipeline?
Can it be debugged?
Can it be profiled?
Can it be terminated?
```

---

# 16H. Lazy Versus Eager Reasoning

When modernizing collection code, write down:

```text
When is work performed?
How much memory is retained?
When are errors raised?
Can processing stop early?
How many values are created?
```

Compare:

```js
const output =
  values
    .map(transform)
    .filter(predicate);
```

with:

```js
const output =
  Iterator
    .from(values)
    .map(transform)
    .filter(predicate);
```

The second can defer processing until consumed.

That changes:

```text
timing
side effects
memory
error location
```

So it is not merely a syntactic rewrite.

---

# 16I. Error Semantics During Modernization

When replacing helpers, compare:

```text
invalid input
throw behavior
Promise rejection
error identity
error cause
message
timing
```

Example:

```js
JSON.parse(input);
```

versus:

```js
someCustomParser(input);
```

A standard replacement is only safe when its observable failure semantics meet the application's needs.

---

# 16J. Numeric Modernization

For numeric features, define:

```text
domain
precision requirement
range
rounding
serialization
storage
interop
```

Example:

```text
money
```

usually needs an explicit monetary representation strategy.

Do not infer:

```text
Math.sumPrecise()
→ money is now exact
```

It is a floating-point summation facility, not a decimal money type.

---

# 16K. Unicode and Modern Strings

Modern ECMAScript should be evaluated with Unicode semantics in mind.

Useful existing capabilities include:

```text
String.prototype.isWellFormed()
String.prototype.toWellFormed()
```

introduced in ECMAScript 2024. citeturn749000search2

When processing external text, consider:

```text
well-formed UTF-16
Unicode normalization
code points
grapheme clusters
serialization
security-sensitive identifiers
```

Standardization does not make all Unicode problems disappear.

---

# 16L. Modern Regular Expressions

Modern RegExp capabilities include:

```text
/v flag
modifier groups
RegExp.escape()
```

The ECMAScript 2024 and 2025 specifications include these evolving regular-expression capabilities. citeturn749000search2

Treat advanced regex as a language with a cost model:

```text
clarity
compile cost
match cost
backtracking
input size
security
```

A safer literal-escaping API does not make arbitrary regex patterns automatically safe from pathological runtime behavior.

---

# 16M. Modern JSON and Modules

ECMAScript 2025 standardized JSON module imports and syntax for import attributes. citeturn749000search2

Before adoption, verify:

```text
module system
Node version
browser bundler
test runner
package exports
deployment
```

A source file may be valid ECMAScript while the host/build environment still rejects its module-loading behavior.

---

# 16N. Float16 Engineering

`Float16Array` can be useful when memory/bandwidth matters.

Example:

```js
const values =
  new Float16Array(1024);
```

But lower precision means:

```text
more rounding
smaller dynamic range
different numerical error
```

Evaluate:

```text
precision requirement
interoperability
conversion cost
storage savings
```

before adoption.

---

# 16O. Iterator Resource Management

Iterators can represent resources indirectly:

```text
file
stream
database cursor
network source
```

If iteration stops early:

```js
for (const item of iterable) {
  if (done(item)) {
    break;
  }
}
```

the iterator's cleanup semantics matter.

Modern iterator usage must therefore be understood alongside:

```text
IteratorClose
return()
resource management
streams
```

from earlier chapters.

---

# 16P. Modern Feature and Garbage Collection

A language feature can change object allocation patterns.

Example:

```text
array pipeline
→ intermediate arrays
```

versus:

```text
iterator pipeline
→ fewer intermediate materializations
```

But retaining a lazy pipeline too long can also retain:

```text
source
closures
captured state
```

Therefore:

```text
less allocation
≠
always less memory retention
```

Measure both allocation and retained memory.

---

# 16Q. Debugging Modern Features

When a modern feature behaves differently across environments, collect:

```text
runtime version
engine version
feature detection result
build target
transformed artifact
polyfill presence
test environment
```

Then classify:

```text
language mismatch
runtime mismatch
build mismatch
polyfill mismatch
application bug
```

---

# 16R. Production Rollout Strategy

For a large organization:

```text
experimental service
→ internal package
→ small production cohort
→ wider services
→ default standard
```

Track:

```text
compatibility failures
developer feedback
performance
bundle/runtime behavior
incidents
```

Do not roll a language baseline to hundreds of services with no compatibility inventory.

---

# 16S. Codebase Modernization Scorecard

| Area | Legacy State | Modern Target | Evidence |
|---|---|---|---|
| Iteration | custom helpers | standard iterators | ____ |
| Promise construction | manual deferred | `withResolvers` | ____ |
| Set algebra | array utilities | Set methods | ____ |
| Regex escaping | custom helper | `RegExp.escape` | ____ |
| Async collection | loops | `Array.fromAsync` where suitable | ____ |
| Binary encoding | custom codec | typed-array APIs | ____ |
| Numeric summation | naive reduction | evaluated summation strategy | ____ |
| Errors | ad-hoc checks | standardized/error API strategy | ____ |

---

# 16T. Modernization Exit Criteria

A modernization is complete when:

```text
feature used intentionally
compatibility verified
tests updated
documentation updated
old helper removed
fallback strategy documented
performance measured where relevant
security reviewed
rollback understood
```

Do not leave:

```text
new API
+
old helper
+
half-used fallback
```

forever.

---

# 16U. Modern ECMAScript Interview Reasoning

A strong answer to:

> “Should we adopt feature X?”

should include:

```text
problem
semantics
standard status
runtime support
compatibility
migration
performance
memory
security
maintenance
rollback
```

This is better than:

> “It is the latest feature.”

---

# 16V. Modern Feature Anti-Patterns

Avoid:

```text
feature-driven development
syntax worship
proposal-driven architecture
automatic codemods without tests
polyfill-everything strategy
runtime-version guessing
ignoring build tools
ignoring package consumers
removing compatibility too early
modernizing without measurable benefit
```

---

# 16W. Principal Modernization Review

Before merge, ask:

```text
Why are we changing this?
What old problem disappears?
What new behavior appears?
What environments change?
What compatibility contract changes?
What test proves correctness?
What performance evidence exists?
What memory evidence exists?
What security implications exist?
What fallback exists?
What is the removal plan for compatibility code?
```

Modernization should create a simpler long-term system.


# 17. Performance Considerations

## 17.1 Modern feature performance is workload-dependent

Do not assume:

```text
new API
→ faster
```

Measure:

```text
CPU
allocations
GC
latency
throughput
memory
```

---

## 17.2 Iterator laziness

Lazy pipelines can reduce intermediate arrays:

```js
Iterator.from(values)
  .map(...)
  .filter(...);
```

But repeated iterator layers can also increase call overhead in some workloads.

Measure before claiming a performance gain.

---

## 17.3 Array.fromAsync()

This is convenient:

```js
const values =
  await Array.fromAsync(source);
```

but full materialization costs:

```text
memory
allocation
GC
```

Use streaming when:

```text
source is large
consumer can process incrementally
```

---

## 17.4 Promise.withResolvers()

The performance difference from manually capturing callbacks is not usually the primary reason to use it.

Use it for:

```text
clarity
event integration
lifecycle architecture
```

rather than speculative micro-optimization.

---

## 17.5 Set operations

Set-based algorithms can be dramatically clearer and often have better expected behavior than repeated array scanning.

Example:

```js
const allowed = new Set(
  permissions,
);

return required.every(
  permission => allowed.has(permission),
);
```

This is still workload-dependent.

---

## 17.6 Float16

Float16 can reduce storage and bandwidth compared with wider representations.

But conversion can add CPU cost and precision loss.

Use it when the numeric error budget and interoperability requirements allow it.

---

# 18. Memory Considerations

Modern features can change allocation patterns.

## Array pipeline

```js
values
  .map(...)
  .filter(...)
  .map(...);
```

can create several arrays.

Iterator pipeline can reduce intermediate materialization:

```js
Iterator.from(values)
  .map(...)
  .filter(...);
```

But eventual collection:

```js
Array.from(iter)
```

restores memory cost.

---

## Async materialization

```js
const all =
  await Array.fromAsync(source);
```

may retain:

```text
every result
```

until completion.

---

## Deferred Promise retention

```js
const { promise, resolve } =
  Promise.withResolvers();
```

If retained in a queue forever, the Promise and associated closures can retain resources.

---

# 19. Security Considerations

## 19.1 RegExp.escape()

Useful when user text becomes a literal regex component.

```js
new RegExp(
  `^${RegExp.escape(input)}$`,
);
```

MDN documents this as a primary use case for safely embedding user input as literal pattern text. citeturn749000search0

Still consider:

```text
regex complexity
input length
resource limits
```

---

## 19.2 JSON.rawJSON()

Raw JSON insertion creates a powerful representation boundary.

Potential risks:

```text
invalid JSON
unexpected semantic representation
downstream parser differences
security-sensitive payload manipulation
```

Use only with trusted/validated raw content.

---

## 19.3 Base64/hex APIs

Binary encodings are not encryption.

Do not treat:

```text
toBase64()
```

as:

```text
toSecret()
```

---

## 19.4 Modern feature availability

Feature-detection logic can itself become a security boundary.

Do not assume:

```js
if ("newFeature" in object) {
  // safe
}
```

means the environment is trustworthy.

Validate inputs independently.

---

# 20. Production Usage

## 20.1 Modernization policy

Create a baseline:

```text
minimum Node
minimum browser
build target
test target
```

Then define:

```text
features allowed natively
features requiring transpilation
features requiring polyfills
features prohibited
```

---

## 20.2 Feature adoption record

For each feature:

```text
Feature:
ECMAScript Edition:
Spec Status:
Runtime Support:
Browser Support:
Node Support:
Polyfill:
Bundle Cost:
Performance:
Memory:
Security:
Migration:
Decision:
```

---

## 20.3 Modernization stages

```text
Stage 1
identify legacy pattern

Stage 2
verify runtime support

Stage 3
characterize behavior

Stage 4
migrate small module

Stage 5
run compatibility tests

Stage 6
measure

Stage 7
roll out

Stage 8
remove obsolete compatibility code
```

---

## 20.4 Set modernization

Legacy:

```js
function intersection(a, b) {
  return [...a].filter(
    value => b.has(value),
  );
}
```

Modern:

```js
const result = a.intersection(b);
```

This can improve:

```text
intent clarity
standardization
maintenance
```

But check runtime baseline first.

---

## 20.5 Promise modernization

Legacy:

```js
let resolve;

const promise = new Promise(
  r => {
    resolve = r;
  },
);
```

Modern:

```js
const {
  promise,
  resolve,
} = Promise.withResolvers();
```

Use the latter when separate Promise completion control is genuinely useful.

---

## 20.6 Async source modernization

Legacy:

```js
const results = [];

for await (const item of source) {
  results.push(item);
}
```

Modern:

```js
const results =
  await Array.fromAsync(source);
```

The modern version is concise.

The legacy version may offer better places for:

```text
filtering
early termination
incremental processing
side effects
```

So do not modernize mechanically.

---

# 21. Implementation From Scratch

Build a **Modern ECMAScript Feature Laboratory**.

## Stage 1 — Guided

Implement:

```text
Set algebra helper
Promise deferred helper
async iterable collection
literal regex matching
```

Then replace each with the standardized API where the target baseline supports it.

---

## Stage 2 — Iterator pipeline

Create:

```js
const result =
  Iterator.from(values)
    .map(value => value * 2)
    .filter(value => value > 10);
```

Then design:

```text
early termination
consumption
memory comparison
```

against an array pipeline.

---

## Stage 3 — Async collection

Create:

```js
async function* source() {
  yield 1;
  yield 2;
  yield 3;
}
```

Use:

```js
const values =
  await Array.fromAsync(source());
```

Then compare with:

```js
for await (...) {
  ...
}
```

for memory and streaming behavior.

---

## Stage 4 — Promise.withResolvers()

Build an event-to-Promise adapter:

```js
function waitForSignal(emitter) {
  const {
    promise,
    resolve,
    reject,
  } = Promise.withResolvers();

  // ...
  return promise;
}
```

Add:

```text
success
error
cleanup
timeout
abort
duplicate event
```

---

## Stage 5 — Set operations

Build permission checks:

```text
required
granted
```

Use:

```text
intersection
difference
subset
disjointness
```

Document the domain meaning.

---

## Stage 6 — RegExp.escape()

Build a search utility:

```js
function containsLiteral(text, query) {
  const pattern =
    new RegExp(
      RegExp.escape(query),
      "i",
    );

  return pattern.test(text);
}
```

Test:

```text
regex metacharacters
Unicode
newlines
leading digits
punctuation
lone surrogates
```

MDN documents the broader escaping behavior and edge cases of `RegExp.escape()`. citeturn749000search0

---

## Stage 7 — Numeric precision

Compare:

```js
values.reduce(
  (sum, value) => sum + value,
  0,
);
```

with:

```js
Math.sumPrecise(values);
```

for datasets with varying magnitudes.

Then explain:

```text
accuracy
performance
domain appropriateness
```

---

## Stage 8 — Binary conversion

Use standardized `Uint8Array` hex/Base64 APIs where your runtime supports them.

Test:

```text
empty
zero bytes
ASCII
binary bytes
invalid text
large arrays
round trips
```

---

## Stage 9 — Production-grade modernization

Take a legacy module and produce:

```text
before
behavior tests
modern API replacement
compatibility test
performance comparison
bundle/runtime comparison
migration notes
```

---

# 22. Debugging Exercises

## Exercise 1 — Runtime mismatch

Local:

```text
feature works
```

Production:

```text
TypeError: method is not a function
```

Find:

```text
runtime version
transpilation
polyfill
bundle target
```

---

## Exercise 2 — Iterator not executing

```js
const pipeline =
  Iterator.from(values)
    .map(expensive);
```

Nothing happens.

Explain why.

---

## Exercise 3 — Array.fromAsync memory spike

A service starts loading millions of records.

Heap increases.

Find the materialization boundary.

---

## Exercise 4 — Promise.withResolvers leak

A server creates one unresolved deferred Promise per client.

Clients disconnect.

Promises remain retained.

Find the lifecycle/cleanup problem.

---

## Exercise 5 — Set semantics

A permission check fails:

```js
required.intersection(granted)
```

but objects represent permissions.

Find the identity/equality assumption.

---

## Exercise 6 — RegExp injection

A search endpoint embeds:

```text
user input
```

into a `RegExp`.

Test:

```text
.
*
+
(
[
\
```

and explain why escaping is necessary.

---

## Exercise 7 — Numeric regression

Legacy totals and new totals differ.

Investigate:

```text
ordering
floating-point accumulation
Math.sumPrecise()
```

---

## Exercise 8 — Polyfill mismatch

Application uses:

```js
Array.fromAsync
```

but only a partial compatibility layer exists.

Find where semantics diverge.

---

## Exercise 9 — Proposal confusion

A team adopts a Stage 2 proposal because a blog claims it is “coming to JavaScript.”

Explain the governance mistake.

---

## Exercise 10 — Modern syntax, old behavior

A refactor replaces a loop with an iterator helper pipeline.

Output is identical.

But latency increases.

Investigate:

```text
iterator overhead
allocation
termination
consumption
```

---

# 23. Code Review Exercise

Review:

```js
const escaped =
  input.replace(
    /[.*+?^${}()|[\]\\]/g,
    "\\$&",
  );

const regex =
  new RegExp(`^${escaped}$`);

const result =
  Array.fromAsync(source);

const deferred =
  new Promise((resolve, reject) => {
    controller.resolve = resolve;
    controller.reject = reject;
  });
```

Identify at least 20 findings.

Consider:

```text
RegExp.escape availability
manual escaping edge cases
unhandled Promise
missing await
controller mutation
Promise lifecycle
cancellation
source materialization
runtime compatibility
semantic intent
```

Then rewrite using modern standardized APIs where the target baseline supports them.

---

# 24. Interview Questions

## Fundamental

1. What is ECMAScript?
2. What is modern ECMAScript?
3. What is a yearly ECMAScript edition?
4. What is TC39?
5. What is Stage 4?
6. What is the difference between language and host APIs?
7. Why does runtime support matter?
8. What is a polyfill?
9. What is transpilation?
10. Why does feature detection matter?

## Intermediate

11. What does `Promise.withResolvers()` solve?
12. What does `Promise.try()` solve?
13. What is `Array.fromAsync()`?
14. What are iterator helpers?
15. Why are Set methods useful?
16. Why is `RegExp.escape()` safer than a simplistic escape regex?
17. What is `Math.sumPrecise()`?
18. What does `Error.isError()` solve?
19. What are the new binary conversion APIs?
20. What is `JSON.rawJSON()`?

## Advanced

21. Iterator versus array pipeline?
22. Why can `Array.fromAsync()` cause memory pressure?
23. How do modern features interact with transpilation?
24. Why are built-in APIs different from polyfills?
25. How do you safely modernize a legacy Node service?
26. How do you manage support baselines?
27. How do modern features affect package exports?
28. How do you distinguish Stage 3 from standardized behavior?
29. How do you test a modernization?
30. When should you avoid a new standard feature?

## Principal

31. What should an organization's ECMAScript adoption policy look like?
32. How fast should a production team adopt new ECMAScript editions?
33. How do you balance language modernization with support obligations?
34. When is a runtime upgrade better than a polyfill?
35. What evidence would justify postponing a modern feature?
36. How do modern language capabilities influence package API design?
37. How do you remove legacy compatibility code safely?
38. How do you detect accidental dependence on unsupported features?
39. How do you evaluate modern feature adoption across 100 services?
40. What is the cost of being permanently one language generation behind?

---

# 25. Predict-the-Output Exercises

## Exercise A — Promise.withResolvers()

Predict:

```js
const {
  promise,
  resolve,
} = Promise.withResolvers();

resolve(42);

console.log(await promise);
```

### Actual Result

```text
42
```

### Rule

Calling `resolve()` settles the Promise.

---

## Exercise B — Promise.try()

Predict conceptually:

```js
const promise =
  Promise.try(() => 42);

console.log(
  promise instanceof Promise,
);

console.log(await promise);
```

### Actual Result

```text
true
42
```

`Promise.try()` normalizes the callback result into a Promise. The feature is standardized in ECMAScript 2025. citeturn749000search2turn749000search4

---

## Exercise C — Set intersection

Predict:

```js
const a =
  new Set([1, 2, 3]);

const b =
  new Set([2, 3, 4]);

console.log(
  [...a.intersection(b)],
);
```

### Actual Result

```text
[2, 3]
```

### Rule

The result contains values present in both sets.

---

## Exercise D — Set object identity

Predict:

```js
const a = {};
const b = {};

const set =
  new Set([a]);

console.log(
  set.has(b),
);
```

### Actual Result

```text
false
```

### Rule

Object identity differs.

---

## Exercise E — Array.fromAsync()

Given:

```js
async function* source() {
  yield 1;
  yield 2;
}
```

Predict:

```js
console.log(
  await Array.fromAsync(source()),
);
```

### Actual Result

```text
[1, 2]
```

But the result is a fully materialized array.

---

## Exercise F — RegExp.escape()

Predict conceptually:

```js
const text = "a+b";

const regex =
  new RegExp(
    RegExp.escape(text),
  );

console.log(
  regex.test("a+b"),
);
```

### Actual Result

```text
true
```

The escaped `+` is interpreted as literal text rather than the regex quantifier. MDN documents `RegExp.escape()` as the standard facility for safely treating input as literal regex pattern text. citeturn749000search0

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Create a feature matrix:

| Feature | Edition | Spec Status | Runtime Baseline | Host? | Compatibility Risk |
|---|---|---|---|---|---|
| `Promise.withResolvers` | ____ | ____ | ____ | ____ | ____ |
| Iterator helpers | ____ | ____ | ____ | ____ | ____ |
| `Array.fromAsync` | ____ | ____ | ____ | ____ | ____ |
| Set methods | ____ | ____ | ____ | ____ | ____ |
| `RegExp.escape` | ____ | ____ | ____ | ____ | ____ |
| `Math.sumPrecise` | ____ | ____ | ____ | ____ | ____ |
| binary conversion APIs | ____ | ____ | ____ | ____ | ____ |

---

### A2

Explain each new feature at three levels:

```text
syntax
semantics
engineering consequence
```

Do not stop at syntax.

---

## Track B — Implementation

### B1 — Modern library

Build:

```text
iterator utilities
async collection
set algebra
regex search
binary codec
```

using standardized APIs where your runtime baseline supports them.

### B2 — Compatibility wrapper

Create a module that supports:

```text
modern runtime
legacy runtime
```

with:

```text
feature detection
fallback
tests
deprecation plan
```

### B3 — Migration

Choose a legacy module and replace:

```text
custom deferred helper
manual regex escaping
custom Set operations
manual async iterable collection
```

with standardized capabilities.

Measure:

```text
lines
complexity
performance
memory
bundle
```

### B4 — Numeric precision

Create a benchmark and correctness suite comparing:

```text
reduce sum
Math.sumPrecise
```

on:

```text
small magnitudes
mixed magnitudes
large datasets
adversarial ordering
```

---

## Track C — Interview / Reasoning

### C1

A company supports:

```text
Node 18
Node 20
Node 22
browser baseline B
```

Design an ECMAScript feature adoption policy.

### C2

A feature removes 200 lines of utility code but requires a runtime upgrade across 80 services.

Would you adopt it?

Defend the decision using:

```text
migration cost
support lifecycle
security
performance
developer productivity
```

### C3

A new feature is standardized but your team's browser support does not include it.

Choose:

```text
polyfill
transpile
feature detection
avoid feature
raise browser baseline
```

and explain why.

---

# 27. Key Takeaways

1. Modern ECMAScript is a continuously evolving standardized language.
2. The ECMAScript 2026 edition is the current finalized yearly specification as of the current 2026 specification snapshot. citeturn749000search8turn749000search7
3. Standardization and runtime availability are separate concerns.
4. Stage 4 proposals are finalized into the language standard.
5. Modern language features should be understood semantically, not memorized as syntax.
6. `Promise.withResolvers()` simplifies controlled Promise completion. citeturn749000search1
7. `Promise.try()` normalizes callbacks that may throw or return Promises. citeturn749000search2turn749000search4
8. Iterator helpers support standard lazy sequence composition. citeturn749000search2turn749000search5
9. `Iterator.concat()` is an ECMAScript 2026 sequence-composition capability. citeturn749000search7
10. Set composition methods provide standard set algebra. citeturn749000search2
11. `Array.fromAsync()` simplifies async-source materialization but can create large in-memory arrays. citeturn749000search2
12. `RegExp.escape()` should replace fragile homemade literal escaping when supported. citeturn749000search0
13. `Math.sumPrecise()` addresses precision-loss concerns for numeric summation but does not provide arbitrary-precision decimal arithmetic. citeturn749000search7
14. `Error.isError()` provides standardized error identification semantics. citeturn749000search7
15. Modern binary encoding methods reduce the need for custom byte/string conversion helpers. citeturn749000search7
16. JSON raw/source-aware capabilities are advanced representation tools, not ordinary JSON defaults. citeturn749000search7
17. Native APIs can be more reliable than hand-written approximations.
18. Polyfills and transpilation are compatibility mechanisms, not guarantees of identical implementation cost.
19. Modernization can improve readability while still harming memory, performance, or compatibility.
20. A production baseline should explicitly define supported runtimes and language capabilities.
21. Principal modernization is about choosing the smallest language change that produces a meaningful engineering benefit.

---

# 28. Concept Connections

## Depends On

- **1–8** — Language foundations
- **9–14** — Functions / Scope / Execution / Closures / `this`
- **22–28** — Arrays / Collections / Iteration / Generators / Typed Arrays / JSON
- **29–30** — Errors / Resource Management
- **31–40** — Async / Promises / Streaming / Concurrency
- **41–44** — Specification Architecture / Abstract Operations / Internal Methods / Realms
- **55** — HTTP
- **58–63** — Node.js runtime
- **64–70** — Modules / Packages / Transpilation / Bundlers / Source Maps
- **78–85** — Production Engineering
- **86–89** — Testing / Debugging / Review / Refactoring

## Builds Toward

- **91** — TC39 Proposal Tracking
- **92** — Temporal
- **93** — Decorators
- **94** — Compatibility Engineering
- **95** — Legacy JavaScript
- **96** — WebAssembly / Native Interoperability
- **97** — Edge / Serverless JavaScript
- **98–101** — Engineering Judgment
- **102–111** — Production Projects
- **112–121** — Assessments / Principal Project

## Related Concepts

```text
Modern ECMAScript
  ├─ language evolution
  ├─ iterators
  ├─ promises
  ├─ async collection
  ├─ Set algebra
  ├─ regex safety
  ├─ numeric precision
  ├─ errors
  ├─ binary data
  ├─ JSON
  └─ compatibility
```

## Concepts Revisited

```text
iterables
iterators
generators
Promises
typed arrays
JSON
RegExp
Errors
modules
transpilation
polyfills
runtime support
```

## Why This Chapter Matters Later

The next evolution chapters move from:

```text
what is standardized
```

to:

```text
how the language is designed
how proposals evolve
how to evaluate future features
```

Modern ECMAScript knowledge is therefore the prerequisite for principal-level language-evolution judgment.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
ECMAScript editions
Stage 4
runtime support
modern APIs
basic migration
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
standardized vs supported
Stage 3 vs Stage 4
language vs host
polyfill vs native
syntax vs semantics
lazy iterator vs materialized array
async vs streaming
```

Mark:

```text
[+] Completed
```

when you can:

- explain current modern ECMAScript capabilities;
- use Iterator helpers;
- use Set methods;
- use `Promise.withResolvers()`;
- use `Promise.try()`;
- use `Array.fromAsync()`;
- use `RegExp.escape()`;
- explain numeric precision improvements;
- evaluate binary/JSON additions;
- build compatibility fallbacks.

Mark:

```text
[*] Mastered
```

only when you can:

- evaluate a new ECMAScript feature proposal;
- determine whether your runtime baseline supports it;
- choose native/polyfill/fallback/avoid;
- migrate legacy code safely;
- measure modernization impact;
- defend adoption policy across an organization;
- explain feature semantics from specification principles;
- challenge “modern syntax” hype with evidence.

Reading alone does not qualify as mastery.

---

# Chapter 90 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define modern ECMAScript | ____ | `[ ]` |
| ____ | Explain annual editions | ____ | `[ ]` |
| ____ | Explain proposal stages | ____ | `[ ]` |
| ____ | Explain runtime support | ____ | `[ ]` |
| ____ | Explain Iterator helpers | ____ | `[ ]` |
| ____ | Explain `Promise.withResolvers` | ____ | `[ ]` |
| ____ | Explain `Promise.try` | ____ | `[ ]` |
| ____ | Explain `Array.fromAsync` | ____ | `[ ]` |
| ____ | Explain Set methods | ____ | `[ ]` |
| ____ | Explain `RegExp.escape` | ____ | `[ ]` |
| ____ | Explain `Math.sumPrecise` | ____ | `[ ]` |
| ____ | Explain 2026 JSON/binary/error additions | ____ | `[ ]` |
| ____ | Design runtime compatibility policy | ____ | `[ ]` |
| ____ | Defend modernization decision | ____ | `[ ]` |

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

1. What is ECMAScript?
2. What is Stage 4?
3. Standardized versus supported?
4. What does `Promise.withResolvers()` solve?
5. What does `Promise.try()` solve?
6. What is an iterator helper?
7. Why can iterator pipelines be lazy?
8. What does `Array.fromAsync()` do?
9. Why can it increase memory?
10. What Set operations are standardized?
11. Why use `RegExp.escape()`?
12. What problem does `Math.sumPrecise()` address?
13. What does `Error.isError()` provide?
14. Why do runtime baselines matter?
15. Polyfill versus transpilation?
16. When should you avoid a new standard feature?

---

# Chapter 90 — Canonical References and Source Discipline

## 1. ECMAScript 2026 Specification

The TC39 specification site currently publishes the ECMAScript 2026 Language Specification as the 17th edition and explains that the current specification contains the latest yearly snapshot plus finished Stage 4 proposals. citeturn749000search8

Primary:

- https://tc39.es/ecma262/2026/multipage/

---

## 2. ECMAScript 2025 Specification Summary

The ECMAScript 2025 edition added:

```text
Iterator
Set methods
JSON module imports
import attributes syntax
RegExp.escape
RegExp modifier syntax
Promise.try
Float16Array and related APIs
```

The specification publication describes these additions directly. citeturn749000search2

Primary:

- https://tc39.es/ecma262/2025/multipage/

---

## 3. ECMAScript 2026 Additions

The current specification's draft/release material documents:

```text
Math.sumPrecise
Iterator.concat
Array.fromAsync
Error.isError
Map/WeakMap get-or-insert methods
Uint8Array binary string conversions
JSON.parse reviver source context
JSON.rawJSON
```

as ECMAScript 2026 additions. citeturn749000search7

Treat exact API names and compatibility as runtime/version-sensitive implementation details.

---

## 4. MDN Reference

MDN provides practical reference and compatibility data.

Relevant pages include:

- `Promise.withResolvers()` citeturn749000search1
- `RegExp.escape()` citeturn749000search0
- `Iterator` citeturn749000search5
- JavaScript Reference citeturn749000search3

Primary:

- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference

---

## 5. TC39 Proposal Process

For future features, use:

- https://tc39.es/process-document/

Do not confuse:

```text
proposal repository
Stage 3
implementation
```

with:

```text
finished language feature
```

This distinction becomes the subject of Chapter 91.

---

## Source Discipline

For every feature, maintain:

```text
Feature name
ECMAScript edition
Proposal stage/history
Normative specification section
Runtime support
Browser support
Build-tool support
Polyfill support
Application usage
```

Never state:

> “This is a JavaScript feature”

without identifying whether you mean:

```text
standardized ECMAScript
runtime extension
browser API
Node API
proposal
```

---

# Chapter 90 — Completion Snapshot

## Core Theory

- [ ] ECMAScript definition
- [ ] JavaScript vs ECMAScript
- [ ] Annual editions
- [ ] TC39
- [ ] Proposal stages
- [ ] Stage 4
- [ ] Runtime implementation
- [ ] Compatibility baseline
- [ ] Iterator
- [ ] Iterator helpers
- [ ] Iterator.concat
- [ ] Array.fromAsync
- [ ] Promise.withResolvers
- [ ] Promise.try
- [ ] Set composition
- [ ] RegExp.escape
- [ ] RegExp modifiers
- [ ] Float16Array
- [ ] Math.f16round
- [ ] Math.sumPrecise
- [ ] Error.isError
- [ ] Map/WeakMap get-or-insert
- [ ] Uint8Array binary conversion
- [ ] JSON.rawJSON
- [ ] JSON.parse source context
- [ ] JSON modules/import attributes
- [ ] Polyfills
- [ ] Transpilation
- [ ] Runtime compatibility
- [ ] Modernization strategy

## Implementation

- [ ] Promise.withResolvers workflow
- [ ] Promise.try workflow
- [ ] Iterator pipeline
- [ ] Iterator.concat
- [ ] Async source conversion
- [ ] Array.fromAsync
- [ ] Set algebra
- [ ] RegExp.escape
- [ ] Float16 experimentation
- [ ] Math.sumPrecise benchmark
- [ ] Error.isError boundary
- [ ] Binary encode/decode
- [ ] JSON.rawJSON experiment
- [ ] Compatibility wrapper
- [ ] Feature detection
- [ ] Polyfill strategy
- [ ] Runtime matrix
- [ ] Modernization migration
- [ ] Performance comparison
- [ ] Memory comparison

## Interview / Reasoning

- [ ] Explain ECMAScript editions
- [ ] Explain Stage 4
- [ ] Explain runtime support
- [ ] Explain Iterator
- [ ] Explain Promise.withResolvers
- [ ] Explain Promise.try
- [ ] Explain Array.fromAsync
- [ ] Explain Set composition
- [ ] Explain RegExp.escape
- [ ] Explain Math.sumPrecise
- [ ] Explain new 2026 additions
- [ ] Evaluate polyfill vs runtime upgrade
- [ ] Evaluate modernization risk
- [ ] Design adoption policy
- [ ] Defend a feature decision

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

Given a new ECMAScript feature, can you answer:

```text
What problem does it solve?
What older patterns does it replace?
What are the exact semantics?
Which ECMAScript edition contains it?
What was its proposal path?
Is it Stage 4?
Which Node versions support it?
Which browser versions support it?
Does our build tool understand it?
Does our test environment support it?
Does it require a polyfill?
Does a polyfill reproduce the important semantics?
What is the performance impact?
What is the memory impact?
What is the security impact?
Does it change error behavior?
Does it change timing?
Does it change laziness/materialization?
Does it affect bundle size?
Does it reduce maintenance?
Does it increase conceptual complexity?
What is the migration plan?
What is the rollback?
When should we deliberately avoid it?
```

A principal JavaScript engineer does not ask:

> **“Is this feature new?”**

They ask:

> **“Is this standardized, supported by our target environments, semantically appropriate for our workload, and valuable enough to become part of our engineering vocabulary and long-term compatibility contract?”**

---

## Principal Modernization Decision Framework

For every modern ECMAScript feature, record:

```text
Feature:
Problem:
ECMAScript Edition:
Proposal Stage:
Normative Semantics:
Target Runtimes:
Target Browsers:
Build Tool Support:
Polyfill:
Transpilation:
Compatibility Risk:
Correctness Benefit:
Performance:
Memory:
Security:
Developer Experience:
Maintenance Reduction:
Migration Cost:
Operational Risk:
Documentation Cost:
Testing:
Rollback:
Decision:
Adoption Scope:
Revisit Trigger:
```

Then ask:

> **Does adopting the standardized capability create more long-term engineering value than the compatibility and learning cost it introduces?**

That is the principal-level modern JavaScript question.