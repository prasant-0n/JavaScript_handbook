# Chapter 80 — Library Authoring

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing, implementing, packaging, publishing, testing, evolving, and maintaining JavaScript libraries that other developers can safely depend on.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what makes a JavaScript library different from an application.
2. Design a public API that minimizes accidental coupling.
3. Separate public API from implementation details.
4. Design package entry points and export maps.
5. Understand ESM, CommonJS, and interoperability choices for libraries.
6. Design package boundaries and dependency graphs.
7. Choose dependencies deliberately.
8. Understand semantic versioning as a communication contract.
9. Distinguish source compatibility, runtime compatibility, and behavioral compatibility.
10. Design stable error behavior.
11. Design library configuration APIs.
12. Avoid global mutable state in reusable libraries.
13. Design deterministic APIs.
14. Design synchronous, asynchronous, and streaming interfaces.
15. Design cancellation and resource ownership.
16. Build library abstractions that compose with application code.
17. Design browser, Node.js, and cross-runtime compatibility boundaries.
18. Package types, declarations, source maps, and build artifacts correctly.
19. Test library contracts rather than only implementation details.
20. Design documentation that teaches correct usage and constraints.
21. Design examples that do not accidentally establish unsupported guarantees.
22. Design deprecation and migration strategies.
23. Prevent supply-chain and dependency risks.
24. Measure and improve startup, memory, bundle, and runtime costs.
25. Publish a production-style JavaScript package.
26. Review a library API for maintainability, compatibility, security, and operational risk.
27. Defend library-design decisions at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript modules and exports.
- Objects, functions, closures, classes, and prototypes.
- Promises and async/await.
- Errors and cleanup.
- Node.js runtime architecture.
- ESM and CommonJS.
- package.json and module resolution.
- dependency management and supply-chain risks.
- bundling/transpilation/source maps.
- testing and debugging.
- production architecture.
- API design.

Recommended prior chapters:

- **15–21** — Objects / Prototypes / Metaprogramming
- **29–30** — Errors / Resource Management
- **31–40** — Async / Concurrency / Cancellation / Streaming
- **58–63** — Node.js Runtime
- **64–70** — Modules / Packages / Tooling
- **78** — Production JavaScript Architecture
- **79** — API Design

---

# 3. What Is It?

A library is software designed to be **used by other software**.

An application owns:

```text
the process
the deployment
the configuration
the primary user experience
```

A library usually does not.

A library becomes a dependency of a host application.

Therefore the library author must reason about:

```text
unknown callers
unknown environments
unknown execution timing
unknown application architecture
unknown error policies
unknown dependency versions
unknown performance constraints
```

This makes library authoring fundamentally about **contracts and restraint**.

---

## 3.1 Library versus application

Application:

```text
main()
  ↓
configure
  ↓
start
  ↓
serve
  ↓
shutdown
```

Library:

```text
host application
      ↓
    import
      ↓
   library API
      ↓
 library internals
```

The host owns lifecycle.

A well-designed library should not unexpectedly take ownership of:

```text
ports
process termination
global logging
global environment state
global timers
global signal handlers
```

unless explicitly designed and documented to do so.

---

## 3.2 Public API

A public API includes more than exported names.

Potential public contract:

```text
function names
classes
method names
argument meaning
return values
error types
error codes
timing
side effects
mutation behavior
resource ownership
stream semantics
events
configuration
environment assumptions
package entry points
supported runtimes
```

If consumers can reasonably depend on it, treat it as part of the contract.

---

# 4. Why Does It Exist?

Libraries allow teams to:

```text
reuse
standardize
encapsulate
share infrastructure
reduce duplicate implementation
```

But every public abstraction creates a dependency.

Therefore:

> **Library authoring is the art of making useful guarantees without creating unnecessary permanent obligations.**

A bad library can be harder to change than a bad application because the consumers may be outside your team.

---

# 5. Mental Model

Think of a library as a contract wrapped around an implementation:

```text
         Consumer
             │
             ▼
   ┌───────────────────┐
   │   Public Contract  │
   │ API / behavior     │
   └─────────┬─────────┘
             │
   ┌─────────▼─────────┐
   │    Compatibility  │
   │ runtime / deps    │
   └─────────┬─────────┘
             │
   ┌─────────▼─────────┐
   │   Implementation  │
   └─────────┬─────────┘
             │
   ┌─────────▼─────────┐
   │ External Runtime  │
   │ Node / Browser    │
   └───────────────────┘
```

The most important boundary is:

```text
What consumers may rely on
versus
what the author is still free to change
```

---

# 6. Core Rules

## Rule 1 — Export less

Every export is a compatibility promise.

Prefer:

```js
export { createParser };
```

over exposing:

```text
internal parser classes
token utilities
debugging helpers
temporary constants
```

unless consumers genuinely need them.

---

## Rule 2 — Stable contracts beat clever APIs

A simple:

```js
parse(input, options)
```

may be easier to evolve than:

```js
new Parser(...)
  .configure(...)
  .withStrategy(...)
  .withResolver(...)
  .register(...)
  .compile(...)
  .execute(...)
```

More API surface means more compatibility obligations.

---

## Rule 3 — Explicit ownership

For every resource ask:

```text
Who created it?
Who closes it?
Who may mutate it?
Who may retain it?
```

---

## Rule 4 — Avoid hidden global state

Bad:

```js
let currentConfig = {};
```

inside a library module if consumers can indirectly affect one another.

Prefer explicit instances:

```js
const clientA = createClient(configA);
const clientB = createClient(configB);
```

---

## Rule 5 — Preserve determinism

A library function should be predictable regarding:

```text
input
output
mutation
time
randomness
I/O
```

Where nondeterminism is required, make it explicit or injectable.

---

## Rule 6 — Do not assume the host application

A library should avoid silently assuming:

```text
specific logger
specific framework
specific environment variables
specific global state
specific process lifecycle
specific database
```

---

## Rule 7 — Make expensive behavior visible

Consumers should be able to tell when an operation:

```text
does network I/O
allocates large buffers
starts timers
creates workers
opens sockets
uses disk
```

---

## Rule 8 — Compatibility is multidimensional

A change can be compatible in one dimension and breaking in another.

Consider:

```text
syntax
runtime
behavior
performance
memory
types
module format
dependency graph
security
```

---

## Rule 9 — Defaults are contracts

This is public behavior:

```js
createClient()
```

if it implicitly means:

```text
timeout = 30 seconds
retry = 3
logging = enabled
```

Changing defaults can break consumers even when function signatures stay identical.

---

## Rule 10 — Documentation is part of the product

A technically correct library with misleading examples is still a bad library.

---

# 7. Syntax

## 7.1 Minimal public entry point

```js
// src/index.js
export { createParser } from "./parser.js";
```

Consumers:

```js
import { createParser } from "my-parser";
```

Internal modules stay private.

---

## 7.2 Factory-based API

```js
export function createClient({
  baseUrl,
  fetchImpl = fetch,
  timeout = 5000,
}) {
  return {
    async get(path) {
      const response = await fetchImpl(
        new URL(path, baseUrl),
      );

      return response.json();
    },
  };
}
```

Factories make instance-specific configuration explicit.

---

## 7.3 Class-based API

```js
export class Cache {
  constructor(options) {
    this.options = options;
  }

  get(key) {
    // ...
  }
}
```

Classes are useful when:

```text
identity
mutable state
lifecycle
inheritance or subclassing
```

are meaningful.

Do not use classes merely because a library feels “more serious” with classes.

---

# 8. Basic Examples

## Example 1 — Small public API

```js
export function parse(input, options = {}) {
  // ...
}
```

Potential advantages:

- low ceremony;
- easy testing;
- easy composition.

---

## Example 2 — Instance API

```js
const client = createClient({
  baseUrl: "https://api.example.com",
});

await client.get("/users");
```

Useful when:

```text
multiple configurations
multiple tenants
test doubles
different credentials
```

must coexist.

---

## Example 3 — Streaming API

```js
for await (const chunk of parseStream(stream)) {
  consume(chunk);
}
```

Streaming APIs communicate an important architectural property:

```text
work happens incrementally
memory need not scale with full input
consumer controls iteration
```

---

# 9. Execution Walkthrough

Consider:

```js
import { createClient } from "my-client";

const client = createClient({
  baseUrl,
});
```

A library call may execute:

```text
module resolution
  ↓
package export resolution
  ↓
module evaluation
  ↓
factory invocation
  ↓
configuration normalization
  ↓
instance creation
  ↓
consumer method call
  ↓
request construction
  ↓
network I/O
  ↓
response parsing
  ↓
error translation
  ↓
consumer
```

Every stage can become part of the user's observed behavior.

---

# 10. Internal Mechanics

## 10.1 Public entry point design

A package may have:

```text
package.json
src/
  index.js
  parser.js
  tokenizer.js
  internal/
    cursor.js
    diagnostics.js
```

Consumers should ideally depend on:

```text
index.js
```

not:

```text
internal/cursor.js
```

---

## 10.2 Export maps

A package can explicitly define exports:

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./parse": "./dist/parse.js"
  }
}
```

This can prevent consumers from importing arbitrary internal paths.

Package `"exports"` also gives authors control over public subpaths and can support different conditions for environments. Node's package documentation should be treated as the authority for the exact resolution behavior of the Node version you support.

---

## 10.3 Dependency graph

Imagine:

```text
library
 ├─ parser
 ├─ logger
 └─ helper
```

If `logger` depends on:

```text
100 KB helper
→ 40 transitive packages
```

you are not merely adding a logger.

You are adding:

```text
security surface
install cost
update burden
compatibility surface
```

---

## 10.4 Dependency injection

Instead of importing a hard-coded runtime dependency:

```js
import fetch from "some-fetch-package";
```

you may design:

```js
createClient({
  fetchImpl: fetch,
});
```

This improves:

```text
testing
portability
host control
```

But do not inject every tiny function.

Abstraction itself has cost.

---

# 11. ECMAScript / Specification Semantics

Library authors must distinguish language guarantees from package/runtime behavior.

## ECMAScript guarantees

Examples:

```text
module syntax
Promise behavior
object semantics
function calls
iteration
language-level errors
```

## Runtime guarantees

Examples:

```text
Node fs
Node HTTP
worker_threads
process lifecycle
```

## Package-system behavior

Examples:

```text
exports
imports
conditional exports
package resolution
peer dependencies
bundler behavior
```

## Library policy

Examples:

```text
default timeout
retry count
cache duration
error codes
supported Node versions
```

Do not describe an application/library policy as an ECMAScript requirement.

---

# 12. Advanced Behavior

## 12.1 Semantic Versioning

Semantic Versioning communicates intended compatibility expectations through:

```text
MAJOR
MINOR
PATCH
```

Common model:

```text
MAJOR → breaking API changes
MINOR → backward-compatible functionality
PATCH → backward-compatible fixes
```

The important question is not:

> Did the source file change?

It is:

> Did a supported consumer contract change incompatibly?

---

## 12.2 What can be breaking?

Potential breaking changes include:

```text
remove export
rename function
change argument meaning
change default behavior
change error shape
change sync API to async
change promise resolution meaning
change mutation semantics
remove runtime support
require a newer module environment
change package entry point incompatibly
```

A subtle behavioral break:

```js
// Before
parse("x")
```

returns:

```js
"value"
```

After:

```js
Promise.resolve("value")
```

Even though `"value"` is still eventually produced, timing and type changed:

```text
string
→ Promise
```

That is a contract change.

---

## 12.3 Error compatibility

Consumers may rely on:

```js
error instanceof ValidationError
```

or:

```js
error.code === "INVALID_INPUT"
```

Therefore error design is part of library API design.

Prefer stable machine-readable codes:

```js
const error = new Error("Invalid input");
error.code = "INVALID_INPUT";
```

For richer libraries, custom error classes can carry structured metadata.

---

## 12.4 Sync versus async API evolution

Changing:

```js
const value = parse(input);
```

to:

```js
const value = await parse(input);
```

is usually a major API redesign.

It affects:

```text
call sites
error propagation
timing
stack behavior
resource lifetime
composition
```

Design async behavior early when asynchronous I/O is intrinsic.

---

# 13. Edge Cases

## 13.1 Empty versus omitted options

These can differ:

```js
createClient()
```

and:

```js
createClient({})
```

Define whether they mean the same thing.

---

## 13.2 `null` versus `undefined`

Potentially:

```text
undefined → use default
null → disable feature
```

or they may mean the same thing.

Choose intentionally.

---

## 13.3 Mutation

Bad surprise:

```js
const options = { headers: {} };

createClient(options);

// library modifies caller object
options.headers.Authorization = "...";
```

A library should document and preferably avoid unexpected caller-object mutation.

---

## 13.4 Callback timing

A callback can be:

```text
synchronous
microtask
macrotask
worker task
```

Changing callback timing can break consumers even if the callback still fires.

---

## 13.5 Promise rejection timing

These are observably different:

```js
throw new Error("x");
```

versus:

```js
return Promise.reject(new Error("x"));
```

when called from non-async code.

Design and document sync/async behavior.

---

# 14. Common Misconceptions

### “Semantic versioning guarantees compatibility.”

No. It communicates intended compatibility according to the project’s chosen interpretation. Authors can misclassify changes.

### “Adding a field cannot break consumers.”

It can break consumers with:

```text
strict schemas
exact object comparisons
destructuring assumptions
serialization rules
```

### “More exports make a library flexible.”

They also increase compatibility obligations.

### “Tree-shaking makes dependencies free.”

Tree-shaking can reduce bundled code in suitable environments, but dependency installation, evaluation, side effects, and runtime costs still matter.

### “Peer dependencies are always better than direct dependencies.”

No. The correct dependency type depends on whether the library needs to control the implementation/version or integrate with the host's instance of a package.

### “Types are documentation only.”

Type declarations can become part of the effective developer-facing contract. Breaking type changes can break consumers even when runtime behavior appears unchanged.

---

# 15. Common Mistakes

## Mistake 1 — Deep imports become unofficial API

Consumer:

```js
import Parser from "library/dist/parser.js";
```

Now internal paths are effectively public.

Use export maps and documentation to define supported entry points.

---

## Mistake 2 — Side effects at import time

Bad:

```js
startBackgroundWorker();
```

during module evaluation.

Importing should not unexpectedly:

```text
open sockets
start timers
spawn processes
modify globals
```

unless that behavior is intentional and documented.

---

## Mistake 3 — Process ownership

A library should almost never do:

```js
process.exit(1);
```

because the host application owns the process.

Prefer throwing or rejecting.

---

## Mistake 4 — Global monkey patching

Bad:

```js
Array.prototype.someMethod = ...
```

This creates application-wide coupling.

---

## Mistake 5 — Hidden environment variables

Bad:

```js
const token = process.env.MY_LIBRARY_TOKEN;
```

deep inside random library code.

Prefer explicit configuration where practical.

---

## Mistake 6 — Unbounded caches

A reusable library can accidentally become a memory leak if it stores every key forever.

Provide:

```text
limits
TTL
eviction
clear()
```

when appropriate.

---

# 16. Comparison With Related Concepts

| Design | Strength | Risk |
|---|---|---|
| Function API | Simple, composable | Less explicit lifecycle |
| Class API | State/lifecycle visible | Can encourage inheritance |
| Factory API | Explicit instances | Slightly more ceremony |
| Singleton export | Easy access | Hidden global state |
| Plugin architecture | Extensible | Complex compatibility surface |
| Monolithic package | Easy install | Larger dependency surface |
| Small packages | Focused reuse | Dependency fragmentation |
| Direct dependency | Predictable implementation | Duplicate versions |
| Peer dependency | Host controls shared dependency | Compatibility complexity |
| Bundled dependency | Self-contained runtime | Larger artifact / duplication |
| Deep imports | Fine-grained access | Break internal refactors |
| Explicit exports | Strong boundary | Requires intentional API design |

---

# 17. Performance Considerations

Library performance should be measured from the consumer's perspective.

Relevant dimensions:

```text
import/evaluation cost
startup cost
steady-state throughput
latency
allocations
GC pressure
bundle size
dependency size
I/O
serialization
```

---

## 17.1 Import-time cost

A library that imports:

```text
20 modules
large lookup tables
multiple dependencies
```

may increase application startup cost even if the consumer calls one function.

Prefer lazy initialization when justified:

```js
let decoder;

function decode(input) {
  decoder ??= createDecoder();
  return decoder.decode(input);
}
```

But lazy initialization can add first-call latency and complexity.

---

## 17.2 Avoid needless allocations

Bad:

```js
items
  .map(transform)
  .filter(predicate)
  .map(otherTransform);
```

This can create intermediate arrays.

For performance-sensitive hot paths, a single loop may reduce allocations:

```js
const result = [];

for (const item of items) {
  const transformed = transform(item);

  if (!predicate(transformed)) {
    continue;
  }

  result.push(otherTransform(transformed));
}
```

Do not optimize without measurement.

---

## 17.3 API cost transparency

A method named:

```js
client.resolve()
```

should not unexpectedly perform thousands of network requests.

Consumers need predictable cost models.

---

# 18. Memory Considerations

Library authors must reason about:

```text
retained closures
caches
event listeners
timers
buffers
queues
maps
subscriptions
streams
```

Every retained reference can extend object lifetime.

---

## 18.1 Listener lifecycle

If a library exposes:

```js
const unsubscribe = store.subscribe(listener);
```

the contract should make cleanup obvious:

```js
unsubscribe();
```

or provide:

```js
subscription.close();
```

---

## 18.2 Cache ownership

A cache should specify:

```text
who owns it
maximum size
eviction
TTL
whether values are cloned
whether keys are retained
```

Without this, “convenient caching” can become unbounded memory retention.

---

# 19. Security Considerations

A library can introduce security risk into every application that imports it.

## 19.1 Dependency risk

Minimize:

```text
number of dependencies
dependency privilege
unused dependency surface
unmaintained dependency exposure
```

A package should make its dependency tree understandable.

---

## 19.2 Unsafe defaults

Examples:

```text
TLS verification disabled
insecure random IDs
overly broad permissions
verbose secrets in errors
unbounded input
unsafe deserialization
```

Secure defaults are part of API design.

---

## 19.3 Prototype pollution

Libraries that merge arbitrary objects should be careful with:

```text
__proto__
constructor
prototype
```

Do not assume input object keys are harmless.

---

## 19.4 Supply-chain discipline

Pinning strategy, lockfiles, trusted registries, provenance, dependency review, and automated vulnerability assessment should be treated as part of package maintenance.

See **Chapter 67 — Dependency Management and Supply Chain**.

---

# 20. Production Usage

## 20.1 Package layout

A production package may use:

```text
my-library/
  src/
    index.js
    client.js
    errors.js
    internal/
      normalize.js

  test/
    unit/
    integration/
    contract/

  docs/
    getting-started.md
    api.md
    migration.md

  dist/
    index.js
    index.d.ts

  package.json
  README.md
  LICENSE
  CHANGELOG.md
```

---

## 20.2 package.json

Example:

```json
{
  "name": "@example/client",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "engines": {
    "node": ">=20"
  }
}
```

The exact supported Node version should be chosen from your actual compatibility policy and tested in CI.

---

## 20.3 Public API surface

Keep a clear list:

```text
Public:
createClient
ClientError
TimeoutError

Internal:
normalizeHeaders
buildUrl
retryDecision
debugState
```

Review this list before every release.

---

## 20.4 Documentation structure

A strong README often includes:

```text
What it does
Installation
Quick start
Core API
Configuration
Errors
Examples
Compatibility
Performance notes
Security notes
Migration
License
```

---

## 20.5 Changelog discipline

A useful changelog distinguishes:

```text
Added
Changed
Fixed
Deprecated
Removed
Security
```

Consumers need to know what they must evaluate during upgrades.

---

# 21. Implementation From Scratch

Build a production-style JavaScript library.

## Project Goal

Create:

```text
@learning/typed-config
```

A configuration parser that:

```text
accepts environment-like input
validates schema
returns normalized values
reports structured errors
has zero global state
supports sync usage
supports extension hooks
```

---

## Stage 1 — Guided

### Public API

```js
export function createConfig(schema, input) {
  // ...
}
```

Example:

```js
const config = createConfig(
  {
    PORT: {
      type: "number",
      required: true,
    },
  },
  {
    PORT: "3000",
  },
);
```

Result:

```js
{
  PORT: 3000
}
```

---

## Stage 2 — Partially Guided

Add:

```text
string
number
boolean
enum
default
nullable
custom validator
```

Errors should be structured:

```js
{
  code: "INVALID_NUMBER",
  path: ["PORT"],
  message: "PORT must be a number"
}
```

---

## Stage 3 — No Reference

Design your own:

```text
public exports
error hierarchy
schema format
validation rules
package.json
tests
README
```

Do not copy an existing library API.

---

## Stage 4 — Edge-case hardened

Test:

```text
missing property
undefined
null
NaN
Infinity
empty string
large numbers
prototype pollution keys
unexpected nested objects
symbols
arrays
```

---

## Stage 5 — Production-grade

Add:

```text
export map
source maps
types/declarations if applicable
ESM compatibility
Node compatibility matrix
contract tests
benchmark suite
fuzz/property-style tests
changelog
migration guide
security policy
automated release checks
```

---

# 22. Debugging Exercises

## Exercise 1 — Import-time side effect

Importing the library starts a timer.

Find the lifecycle problem.

---

## Exercise 2 — Cross-consumer state leak

```js
const a = createClient({ token: "A" });
const b = createClient({ token: "B" });

a.setHeader("x", "1");

console.log(b.getHeader("x"));
```

If the result is `"1"`, find the shared state.

---

## Exercise 3 — Hidden mutation

The library changes the options object passed by the consumer.

Design a test that catches this.

---

## Exercise 4 — Error compatibility

Version 1:

```js
error.code === "INVALID_INPUT"
```

Version 2 removes `code` and changes the message.

Explain why:

```text
message-only compatibility
```

is weaker than stable machine-readable error codes.

---

## Exercise 5 — Timer leak

A library schedules:

```js
setInterval(...)
```

but never exposes cleanup.

What happens in long-running applications and tests?

---

## Exercise 6 — Dependency explosion

A 4 KB utility package pulls in 200 transitive packages.

Evaluate:

```text
install cost
security
startup
maintenance
bundle
compatibility
```

---

# 23. Code Review Exercise

Review:

```js
let token;

export function configure(options) {
  token = options.token;
}

export async function request(path) {
  const response = await fetch(
    `https://api.example.com${path}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    },
  );

  return response.json();
}
```

Identify at least 15 problems.

Possible findings:

```text
global mutable state
multiple consumers interfere
implicit configuration order
no validation
no timeout
no cancellation
no response error handling
no status validation
hard-coded endpoint
poor testability
uncontrolled retry behavior
no resource ownership
no API version strategy
token lifetime unclear
potential secret leakage
```

Refactor toward:

```js
const clientA = createClient({
  baseUrl,
  token,
  fetchImpl,
});
```

---

# 24. Interview Questions

## Fundamental

1. What is a JavaScript library?
2. Library versus application?
3. What is a public API?
4. Why should libraries export less?
5. Why avoid global state?
6. Factory versus class?
7. What belongs in documentation?
8. Why are defaults part of the API?
9. What is semantic versioning?
10. What makes a change breaking?

## Intermediate

11. How do export maps protect a package?
12. What is a peer dependency?
13. How do you support both ESM and CommonJS?
14. What makes an error API stable?
15. Why can adding a field be breaking?
16. How do you design cancellation?
17. How do you design resource ownership?
18. How do you minimize package startup cost?
19. What is an import-time side effect?
20. What should a README communicate?

## Advanced

21. How would you design a library for Node and browsers?
22. How do conditional exports work conceptually?
23. How do you evolve a library without breaking unknown consumers?
24. How do you detect deep-import consumers?
25. How do you design a plugin API?
26. When should a dependency be bundled, direct, or peer?
27. How do you handle transitive vulnerability exposure?
28. How do you test package exports?
29. How do types interact with runtime compatibility?
30. How do you deprecate a public API?

## Principal

31. Which behavior should never be left implicit?
32. What guarantees would you refuse to make?
33. How do you decide whether a concept deserves a public export?
34. When is dependency injection unnecessary?
35. How do you design for unknown consumers without creating abstraction paralysis?
36. How do you measure API stability?
37. What performance cost is acceptable for ergonomic API design?
38. How do you handle a security vulnerability in a widely deployed library?
39. How do organizational ownership and release cadence affect package design?
40. When should a library stop accepting new features and prioritize stability?

---

# 25. Predict-the-Output Exercises

## Exercise A — Closure isolates configuration

Predict:

```js
function createClient(token) {
  return {
    getToken() {
      return token;
    },
  };
}

const a = createClient("A");
const b = createClient("B");

console.log(a.getToken());
console.log(b.getToken());
```

### Actual Result

```text
A
B
```

### Rule

Each factory call creates a separate closure over its own `token`.

---

## Exercise B — Shared object state

Predict:

```js
const options = {
  retries: 3,
};

function configure(input) {
  input.retries += 1;
}

configure(options);

console.log(options.retries);
```

### Actual Result

```text
4
```

### Rule

Objects are mutable reference values. Library APIs should specify whether caller-owned objects may be mutated.

---

## Exercise C — Sync versus async

Predict:

```js
function value() {
  return 42;
}

async function asyncValue() {
  return 42;
}

console.log(typeof value());
console.log(typeof asyncValue());
```

### Actual Result

```text
number
object
```

### Rule

An `async` function returns a Promise, even when its body returns a plain value.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Take an imaginary library with 25 exports.

Reduce it to the smallest public API that still supports the intended use cases.

Explain every removed export.

### A2

Classify each behavior as:

```text
language guarantee
runtime guarantee
package-system behavior
library contract
application policy
```

---

## Track B — Implementation

### B1 — Build a package

Create:

```text
@learning/http-client
```

Requirements:

```text
factory API
request timeout
AbortSignal support
structured errors
JSON parsing
streaming response option
test injection
ESM package
explicit exports
```

### B2 — Compatibility

Support two runtime versions.

Create a CI matrix that tests:

```text
oldest supported
current supported
```

Also test package installation from the published artifact.

### B3 — Release

Create:

```text
CHANGELOG
README
migration guide
API docs
release checklist
```

Perform a simulated:

```text
patch release
minor release
breaking release
```

and justify each classification.

---

## Track C — Interview / Reasoning

### C1

A library has:

```text
40 exports
5 years of history
10,000 consumers
```

The team wants to “clean up the API.”

Design an incremental strategy.

### C2

A library's biggest complaint is memory usage.

Determine whether the cause is:

```text
consumer misuse
library cache
retained listeners
large buffers
dependency
object allocation
```

Design the investigation.

### C3

A widely used library has a severe vulnerability.

Design:

```text
triage
patch
release
communication
backport
consumer migration
verification
```

---

# 27. Key Takeaways

1. Libraries are contracts for unknown consumers.
2. Every public export creates compatibility obligations.
3. Export less and document what is supported.
4. Avoid hidden global state.
5. Make ownership explicit.
6. Defaults are behavioral contracts.
7. Errors are part of the public API.
8. Sync versus async behavior is part of compatibility.
9. Package entry points are architectural boundaries.
10. Export maps can protect internal structure.
11. Dependencies impose security and maintenance costs.
12. Semantic versioning communicates intended compatibility.
13. A semver classification can still be wrong.
14. Performance and memory are user-visible library behavior.
15. Import-time side effects can damage host applications.
16. Libraries should not silently own the host process lifecycle.
17. Documentation is part of the product.
18. Tests should validate the public contract.
19. A package should be tested as consumers actually install and import it.
20. Principal library design is mostly about deciding what the library refuses to promise.

---

# 28. Concept Connections

## Depends On

- **Chapter 15–21** — Objects, prototypes, classes, metaprogramming
- **Chapter 29–30** — Errors and cleanup
- **Chapter 31–40** — Async, promises, cancellation, streaming
- **Chapter 58–63** — Node.js runtime
- **Chapter 64–70** — Modules, packages, dependencies, builds, source maps
- **Chapter 74–77** — Programming paradigms and patterns
- **Chapter 78** — Production JavaScript Architecture
- **Chapter 79** — API Design

## Builds Toward

- **Chapter 81** — Database Integration
- **Chapter 82** — API Architecture
- **Chapter 83** — Observability
- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 89** — Code Review / Refactoring
- **Chapter 94** — Compatibility Engineering
- **Chapter 96** — WebAssembly / Native Interoperability
- **Chapter 97** — Edge / Serverless JavaScript
- **Chapter 104** — Production HTTP Client
- **Chapter 110** — Production JavaScript Backend

## Related Concepts

```text
Library Authoring
  ├─ Public API
  ├─ Package boundaries
  ├─ Compatibility
  ├─ Dependency management
  ├─ Error design
  ├─ Resource ownership
  ├─ Runtime portability
  ├─ Documentation
  ├─ Testing
  └─ Release engineering
```

## Concepts Revisited

```text
modules
closures
objects
promises
errors
streams
cancellation
package resolution
dependency injection
observability
security
performance
```

## Why This Chapter Matters Later

Library authoring is where JavaScript knowledge becomes **reusable engineering capability**.

The same principles govern:

```text
internal packages
shared company libraries
SDKs
frameworks
plugins
command-line tools
browser packages
Node packages
```

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
public API
package boundary
semantic versioning
dependencies
basic publishing
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
exported code vs supported API
semver vs guarantee
runtime behavior vs ECMAScript behavior
dependency injection vs mandatory abstraction
documentation vs actual contract
```

Mark:

```text
[+] Completed
```

when you can:

- design a small public API;
- package it correctly;
- define exports;
- validate configuration;
- design structured errors;
- test public behavior;
- publish a package;
- perform a semver release.

Mark:

```text
[*] Mastered
```

only when you can:

- evolve a library with unknown consumers;
- identify hidden compatibility obligations;
- minimize public surface without reducing usefulness;
- diagnose memory/performance regressions;
- assess dependency risk;
- design migration paths;
- handle security releases;
- defend the library contract at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 80 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain library vs application | ____ | `[ ]` |
| ____ | Define public API | ____ | `[ ]` |
| ____ | Design a minimal package surface | ____ | `[ ]` |
| ____ | Explain semantic versioning | ____ | `[ ]` |
| ____ | Analyze breaking changes | ____ | `[ ]` |
| ____ | Design package exports | ____ | `[ ]` |
| ____ | Diagnose hidden global state | ____ | `[ ]` |
| ____ | Design release/migration strategy | ____ | `[ ]` |

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

1. Define a library contract.
2. Why should a library export less?
3. What makes a change breaking?
4. Why are defaults part of API compatibility?
5. Why is import-time side effect dangerous?
6. Explain direct versus peer dependency.
7. How do export maps protect a package?
8. How do library errors become public API?
9. How do you design for unknown consumers?
10. What would make you refuse a requested API feature?

---

# Chapter 80 — Canonical References and Source Discipline

## 1. ECMAScript

Use the ECMAScript specification for:

```text
language semantics
modules
functions
objects
promises
iteration
```

Primary:

- https://tc39.es/ecma262/

---

## 2. Node.js Package Documentation

Use current Node.js package documentation for:

```text
package exports
imports
module resolution
ESM/CommonJS behavior
conditional exports
package entry points
runtime compatibility
```

Primary:

- https://nodejs.org/docs/latest/api/packages.html

The exact package-resolution behavior depends on the Node.js versions your library supports; test those versions rather than assuming behavior from one local runtime.

---

## 3. npm Package Specification

Use npm's current package documentation for:

```text
package.json
dependencies
peerDependencies
files
publishing
package lifecycle
```

Primary:

- https://docs.npmjs.com/

---

## 4. Semantic Versioning

Use the Semantic Versioning specification for the intended:

```text
MAJOR
MINOR
PATCH
```

compatibility vocabulary.

Primary:

- https://semver.org/

Treat semver as a compatibility communication convention, not proof that a release is actually safe.

---

## 5. OpenTelemetry

For libraries that expose instrumentation hooks or integrate with application telemetry, consult the OpenTelemetry JavaScript documentation.

Primary:

- https://opentelemetry.io/docs/languages/js/

The concrete instrumentation and API surface should match the OpenTelemetry and runtime versions actually supported by the package.

---

## 6. OWASP

Use OWASP guidance for:

```text
input handling
dependency security
unsafe deserialization
secret handling
prototype-related attack surfaces
```

Primary:

- https://owasp.org/

---

## Source Discipline

Every library-authoring decision should be classified:

```text
ECMAScript requirement
Node/runtime behavior
Package-manager behavior
Bundler behavior
Library contract
Application convention
```

Do not say:

> “JavaScript requires this”

when the actual rule is:

> “Our package API chooses this.”

Do not say:

> “Node supports this”

without checking the versions in the package compatibility matrix.

---

# Chapter 80 — Completion Snapshot

## Core Theory

- [ ] Library versus application
- [ ] Public API
- [ ] Public versus internal surface
- [ ] API stability
- [ ] Dependency direction
- [ ] Export maps
- [ ] ESM/CommonJS considerations
- [ ] Semantic versioning
- [ ] Behavioral compatibility
- [ ] Error compatibility
- [ ] Default behavior
- [ ] Configuration
- [ ] Resource ownership
- [ ] Cancellation
- [ ] Streaming
- [ ] Runtime portability
- [ ] Performance
- [ ] Memory
- [ ] Security
- [ ] Documentation
- [ ] Deprecation
- [ ] Release strategy

## Implementation

- [ ] Build public entry point
- [ ] Define explicit exports
- [ ] Implement factory API
- [ ] Design structured errors
- [ ] Implement cancellation
- [ ] Add input validation
- [ ] Add deterministic tests
- [ ] Add integration tests
- [ ] Test package installation
- [ ] Test export map
- [ ] Test multiple runtimes
- [ ] Measure startup cost
- [ ] Measure memory behavior
- [ ] Run benchmarks
- [ ] Audit dependencies
- [ ] Write README
- [ ] Write migration guide
- [ ] Produce changelog
- [ ] Simulate releases

## Interview / Reasoning

- [ ] Explain library/application difference
- [ ] Define supported API
- [ ] Identify breaking changes
- [ ] Defend dependency choices
- [ ] Analyze public exports
- [ ] Diagnose hidden global state
- [ ] Design compatibility policy
- [ ] Design security response
- [ ] Defend package architecture
- [ ] Decide what not to expose

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

Given an unfamiliar JavaScript package, can you determine:

```text
What is actually public?
What is merely accidentally reachable?
Which behaviors are compatibility commitments?
Which defaults are contractual?
Which errors are stable?
Which dependencies create risk?
What happens at import time?
Who owns resources?
How does cancellation work?
What runtimes are supported?
What package entry points are supported?
Can consumers deep-import internals?
What is the startup cost?
What is the memory model?
What are the security assumptions?
How are breaking changes communicated?
How are deprecated APIs removed?
How would you evolve the package without forcing a rewrite?
```

A principal library author understands that the hardest part of reusable software is not implementing functionality.

It is **choosing which behavior deserves to become a permanent promise**.

---

## Principal Library Decision Framework

For every proposed public API, evaluate:

```text
Consumer Need
API Surface
Semantic Clarity
Correctness
Performance
Memory
Security
Reliability
Maintainability
Compatibility
Runtime Support
Dependency Cost
Documentation Cost
Testing Cost
Migration Cost
Operational Complexity
Future Change
```

Then record:

```text
Public API:
Use Case:
Alternatives:
Why Public:
Why Not Internal:
Guarantees:
Non-Guarantees:
Failure Modes:
Performance Characteristics:
Security Assumptions:
Compatibility Impact:
Migration Strategy:
Revisit Trigger:
```

The strongest library is rarely the one with the most features.

It is the one whose **public contract is small, intentional, observable, testable, and trustworthy enough to survive years of unknown consumer usage.**