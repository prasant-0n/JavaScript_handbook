# Chapter 126 — Module Linking, Resolution & Async Module Evaluation

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Understand ECMAScript modules as a dependency graph and execution system: Module Records, module requests, host loading, linking, environment initialization, live bindings, cyclic dependencies, evaluation order, top-level `await`, asynchronous module dependencies, and failure propagation.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Module Specialist · Runtime Engineer · Bundler/Tooling Engineer · Platform Architect
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **An `import` statement is not simply “load a file.” ECMAScript modules form a graph whose dependencies are loaded, linked, instantiated, and evaluated under defined semantic rules.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain Source Text Module Records
[ ] explain Module Records and module requests
[ ] distinguish module loading from linking and evaluation
[ ] explain the host's role in locating/loading modules
[ ] explain requested modules
[ ] explain imported/exported names
[ ] explain module environments
[ ] explain live bindings
[ ] explain ResolveExport
[ ] distinguish direct and indirect exports
[ ] explain namespace exports
[ ] explain star exports
[ ] explain re-export chains
[ ] explain cyclic module graphs
[ ] explain strongly connected components conceptually
[ ] explain depth-first linking/evaluation
[ ] understand why cycles do not automatically imply failure
[ ] understand temporal dead zones across modules
[ ] explain module evaluation errors
[ ] explain top-level await
[ ] explain asynchronous module dependencies
[ ] explain pending async dependencies
[ ] explain async parent module relationships
[ ] explain evaluation Promises for asynchronous module graphs
[ ] distinguish module syntax from host module resolution
[ ] distinguish ECMAScript semantics from Node/browser/bundler resolution
[ ] explain import maps conceptually
[ ] explain package/bare-specifier resolution as host/toolchain behavior
[ ] debug module-loading failures systematically
[ ] implement a toy module linker
[ ] reason about production ESM architecture
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 01 — JavaScript / ECMAScript / Runtime Landscape
Chapter 10 — Scope / Lexical Environments
Chapter 11 — Hoisting / TDZ
Chapter 12 — Execution Contexts
Chapter 13 — Closures
Chapter 31 — Async Fundamentals
Chapter 32 — ECMAScript Jobs / Promise Reactions
Chapter 36 — Async / Await
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 44 — Realms / Agents / Execution Isolation
Chapter 64 — ES Modules
Chapter 66 — package.json / Resolution
Chapter 68 — Transpilation / Compilation
Chapter 69 — Bundlers / Build Systems
Chapter 91 — TC39 Proposal Tracking
Chapter 123 — Grammar / Parsing / Early Errors
Chapter 124 — Execution Records / Completion Records / References
Chapter 125 — Promise Internals / Capability Machinery
```

---

# 3. Why Module Linking Matters

Developers often model:

```js
import { x } from "./x.js";
```

as:

```text
read file
→ execute file
→ get x
```

That model is incomplete.

The module system needs to establish:

```text
module identity
dependency graph
export/import relationships
bindings
cycles
evaluation order
```

before normal execution can proceed.

With top-level `await`, the evaluation model adds:

```text
async dependency tracking
completion Promises
parent/child relationships
```

---

# 4. Mental Model

Use:

```text
source
  ↓
parse as Module
  ↓
Module Record
  ↓
load dependency graph
  ↓
link graph
  ↓
initialize module environments
  ↓
resolve imports/exports
  ↓
evaluate graph
  ↓
synchronous completion
      OR
asynchronous module evaluation
```

The three words to keep distinct are:

```text
Load
Link
Evaluate
```

---

# 5. Load

Loading answers:

```text
Where are the requested modules?
```

The host participates in module loading.

A module request such as:

```js
import x from "./dep.js";
```

contains a module specifier.

The ECMAScript specification does not universally define:

```text
filesystem lookup
HTTP fetching
package.json lookup
node_modules lookup
browser URL resolution
```

Those are host/toolchain concerns.

---

# 6. Link

Linking answers:

```text
Which imported names correspond to which exported bindings?

Can the complete module graph establish its environments and dependencies?
```

The specification's `InnerModuleLinking` recursively traverses dependencies, initializes environments, and handles cyclic graphs. It uses DFS bookkeeping to identify strongly connected components. citeturn149700search1

---

# 7. Evaluate

Evaluation answers:

```text
What happens when the module bodies execute?
```

For a synchronous graph:

```text
evaluate dependencies
→ evaluate module
```

For graphs involving top-level `await`:

```text
evaluate dependencies
→ wait for asynchronous dependencies
→ continue evaluation
```

The result of evaluating a cyclic module graph can involve a Promise representing completion of asynchronous module evaluation. citeturn149700search0turn149700search1

---

# 8. Module Record

A Module Record is a specification abstraction representing a module.

A Source Text Module Record carries information derived from module source, including concepts such as:

```text
requested modules
import entries
local export entries
indirect exports
star exports
environment
status
```

Do not confuse:

```text
Module Record
```

with:

```text
JavaScript module object
```

It is specification-level state.

---

# 9. Module Requests

A module can request dependencies:

```js
import { a } from "./a.js";
import { b } from "./b.js";
```

The specification tracks these module requests as structured records.

Conceptually:

```text
A
├── ./a.js
└── ./b.js
```

The source string alone is not the complete dependency graph.

The host must resolve each request to an actual module.

---

# 10. Host Loading Boundary

The host provides mechanisms around module loading.

Conceptually:

```text
Module Request
      ↓
Host module loading
      ↓
Module Record
```

Browser and Node.js systems can resolve the same-looking source specifier differently.

Examples:

```text
./local.js
../shared.js
https://example.com/x.js
package-name
node:fs
```

Not all are handled identically by all hosts.

---

# 11. Resolution vs Linking

These are different.

### Resolution

Maps a module specifier to a module.

### Linking

Connects imports to exports and initializes environments.

For example:

```text
"./x.js"
```

may be resolved to a URL/path/module identity.

Then linking determines:

```text
which export binding in that module corresponds to this import
```

Do not merge both concepts into:

```text
"import resolution."
```

---

# 12. Module Identity

A module graph depends on module identity.

If the host identifies:

```text
A
```

and:

```text
A
```

as the same module, they should share the same Module Record identity in the relevant graph.

Different host resolution can therefore produce:

```text
same source text
```

but:

```text
different module identities
```

This is one reason URL/path/package resolution matters operationally.

---

# 13. Module Environment

When a module is linked, its environment is initialized.

Bindings are created for:

```text
imports
local declarations
exports
```

The module environment provides the lexical binding structure used by module code.

This is why imported names can be used without creating ordinary local copies.

---

# 14. Live Bindings

Consider:

```js
// counter.js
export let count = 0;

export function increment() {
  count++;
}
```

and:

```js
// app.js
import { count, increment } from "./counter.js";

console.log(count);
increment();
console.log(count);
```

The import observes the exporting module's binding.

Conceptually:

```text
imported name
→ linked binding
→ current value
```

It is not equivalent to:

```js
const count = exportedValueAtImportTime;
```

---

# 15. Imported Bindings Are Read-Only From Importing Module

An importing module cannot assign directly to an imported binding:

```js
import { count } from "./counter.js";

count = 10;
```

The imported binding is not a locally mutable binding owned by the importing module.

The source module controls its own binding.

This is part of the live-binding model.

---

# 16. Live Binding vs Object Mutation

Do not confuse:

```text
live binding
```

with:

```text
deep immutable data
```

Example:

```js
export const state = {
  count: 0
};
```

The binding:

```text
state
```

is not reassigned.

But the referenced object may still be mutable:

```js
state.count++;
```

The module system controls bindings.

It does not automatically freeze reachable object graphs.

---

# 17. Export Entries

A module's exports can originate from:

```text
local declarations
direct exports
re-exports
star exports
```

Conceptually:

```text
source binding
→ export name
```

or:

```text
dependency export
→ re-export name
```

This allows module API construction independent of physical file boundaries.

---

# 18. Direct Export

Example:

```js
const x = 1;

export { x };
```

The module directly exports a local binding.

Likewise:

```js
export const x = 1;
```

creates a local binding with export exposure.

---

# 19. Indirect Export

Example:

```js
export { x } from "./other.js";
```

The current module does not own the underlying `x` binding.

It exposes another module's export.

Conceptually:

```text
current module
→ re-export entry
→ dependency's x export
```

---

# 20. Star Export

Example:

```js
export * from "./other.js";
```

The module exposes eligible exports from the dependency.

The language specifies collision/ambiguity rules.

Do not model this simply as:

```text
copy every property into an object
```

because module exports are bindings, not a plain object-copy operation.

---

# 21. Namespace Export

A module can expose a namespace-like view:

```js
import * as ns from "./module.js";
```

The namespace object provides access to the module's exported names.

But:

```text
namespace object
```

is a view over module exports.

It is not the module's internal environment.

---

# 22. `export *` Name Conflicts

Consider:

```text
A exports x
B exports x
C:
  export * from A
  export * from B
```

The module graph must resolve the export name correctly.

Ambiguous star exports can produce a linking failure.

This is another reason:

```text
linking
```

is not equivalent to:

```text
execute everything.
```

Some problems are discovered before evaluation.

---

# 23. `ResolveExport`

`ResolveExport` is a key specification operation.

Conceptually:

```text
module
+
exportName
→
binding location or ambiguous/not-found result
```

It can recursively follow:

```text
local exports
indirect exports
star exports
```

This is how an importer can determine which underlying binding it should connect to.

---

# 24. Export Resolution Graph

Imagine:

```text
A exports x
↑
B re-exports x
↑
C re-exports x
↑
D imports x
```

Resolution can conceptually traverse:

```text
D
→ C
→ B
→ A
→ x
```

The binding remains connected to:

```text
A's x
```

rather than becoming a copied value in each module.

---

# 25. Initialization vs Evaluation

A linked module may have its environment initialized before its body has fully evaluated.

This distinction enables:

```text
cycles
live bindings
hoisting behavior
```

A useful model is:

```text
Link:
create/connect binding structure

Evaluate:
execute module code
```

---

# 26. Cyclic Module Graphs

Consider:

```text
A → B
↑   ↓
└── C
```

Cycles are allowed in the module graph.

They do not automatically mean:

```text
infinite execution
```

The specification has explicit machinery for cyclic Module Records.

---

# 27. Why Cycles Are Hard

Suppose:

```js
// a.js
import { b } from "./b.js";
export const a = b + 1;
```

and:

```js
// b.js
import { a } from "./a.js";
export const b = a + 1;
```

The modules depend on bindings whose initialization/evaluation may not yet have completed.

The result can involve:

```text
TDZ / uninitialized binding access
```

rather than an abstract “cycle error.”

---

# 28. Cycles and Strongly Connected Components

The current specification's linking algorithm uses depth-first traversal with fields such as:

```text
[[DFSIndex]]
[[DFSAncestorIndex]]
```

to identify strongly connected components (SCCs). citeturn149700search1

Conceptually:

```text
cycle
→ SCC
→ linked/evaluated as a coordinated graph unit
```

This is an algorithmic technique, not a JavaScript API.

---

# 29. Linking Status

Cyclic module records have statuses including:

```text
new
unlinked
linking
linked
evaluating
evaluating-async
evaluated
```

The specification uses these states to coordinate module graph processing. citeturn149700search1

Do not map these directly to:

```text
file loaded / file executed
```

They represent richer module lifecycle state.

---

# 30. `InnerModuleLinking`

Conceptually:

```text
module
→ mark linking
→ DFS dependencies
→ initialize environment
→ identify SCC boundary
→ transition SCC to linked
```

If a link operation fails:

```text
currently linking modules in the active stack
```

can be reset to the appropriate earlier state. The specification describes this rollback behavior for linking failures. citeturn149700search1

---

# 31. Linking Failure

Examples include:

```text
missing exported name
ambiguous export
invalid import/export relationship
```

A linking failure occurs before ordinary evaluation of the successfully linked graph can proceed.

This distinction is important:

```text
syntax/early error
→ link error
→ evaluation error
```

are different failure layers.

---

# 32. Evaluation Order

For an acyclic synchronous graph:

```text
A imports B
A imports C
```

the dependency graph establishes evaluation ordering constraints.

Conceptually:

```text
B
C
A
```

rather than:

```text
A
B
C
```

The exact order among independent branches follows the module evaluation algorithm and dependency traversal rather than arbitrary textual intuition.

---

# 33. Module Evaluation Is Not File Concatenation

Do not model modules as:

```text
concatenate A.js + B.js + C.js
```

because that loses:

```text
lexical scope
binding identity
imports/exports
module environments
cycles
module metadata
```

The module system provides semantic boundaries.

---

# 34. Module Scope

Top-level declarations in modules are scoped to the module.

Example:

```js
const secret = 42;
```

does not automatically become a global variable merely because it is top-level module code.

This is a fundamental distinction from legacy script behavior.

---

# 35. Realm Relationship

Modules execute in a Realm.

The module's:

```text
Realm
```

provides its intrinsic/environmental context.

Cross-realm module loading can therefore affect:

```text
constructor identity
intrinsics
global object
built-in behavior
```

This connects module semantics to:

```text
Chapter 44 — Realms / Agents / Execution Isolation
```

---

# 36. Top-Level Await

A module can contain:

```js
const response = await fetch(url);
```

at top level when the module context permits it.

This changes the evaluation graph.

A module with top-level `await` can become:

```text
evaluating-async
```

instead of finishing evaluation synchronously.

---

# 37. Why Top-Level Await Is Different

Without top-level await:

```text
module evaluation
→ completes synchronously
```

With top-level await:

```text
module evaluation
→ suspends at async boundary
→ waits
→ resumes
→ completes
```

Dependent modules may need to wait.

This creates dependency-level asynchronous behavior.

---

# 38. Async Module Dependencies

Suppose:

```text
A imports B
B has top-level await
```

Then:

```text
A
↓
B
```

creates an asynchronous evaluation dependency.

A cannot necessarily complete its own evaluation until the relevant asynchronous dependency has completed successfully.

The specification tracks such dependencies explicitly.

---

# 39. `[[HasTLA]]`

Cyclic Module Records track whether a module contains top-level await through:

```text
[[HasTLA]]
```

This contributes to determining how evaluation should proceed for the module graph.

Do not interpret this as:

```text
"module is asynchronous because it uses any Promise."
```

Top-level await is specifically relevant to module evaluation semantics.

---

# 40. `[[PendingAsyncDependencies]]`

For modules with asynchronous dependencies, the specification tracks:

```text
[[PendingAsyncDependencies]]
```

This represents how many async dependencies remain before the module can proceed.

Conceptually:

```text
A depends on B and C
B async
C async

pending = 2

B finishes
pending = 1

C finishes
pending = 0

A can continue
```

The current specification documents this field and its role in async module execution. citeturn149700search1

---

# 41. `[[AsyncParentModules]]`

Asynchronous module graphs also track parent relationships.

Conceptually:

```text
child async module
       ↑
parent importer
       ↑
parent importer
```

These relationships allow async completion or failure to propagate through the graph.

This becomes important for:

```text
top-level await
```

and failure propagation.

---

# 42. `[[TopLevelCapability]]`

When a strongly connected asynchronous module component is evaluated, a Promise can represent completion of that evaluation.

The current specification stores this through the cycle root's:

```text
[[TopLevelCapability]]
```

The first relevant `Evaluate()` call creates the Promise and later evaluations return the same Promise for that component. citeturn149700search0

---

# 43. Evaluate for Cyclic Modules

For cyclic Module Records:

```text
Evaluate()
```

transitions the module graph through evaluation states.

For asynchronous evaluation, the resulting Promise resolves/rejects according to the eventual module completion.

This gives top-level module loading a Promise-shaped completion mechanism.

---

# 44. Evaluation Error

If module source throws during evaluation:

```js
throw new Error("startup failure");
```

the module's evaluation fails.

For synchronous module evaluation:

```text
throw
→ evaluation failure
```

For asynchronous graphs:

```text
failure
→ record evaluation error
→ propagate through async parent relationships
→ reject module evaluation Promise
```

The specification explicitly tracks `[[EvaluationError]]` on Cyclic Module Records. citeturn149700search1

---

# 45. Linking Error vs Evaluation Error

### Linking error

```text
module graph cannot establish valid bindings
```

Example:

```js
import { missing } from "./module.js";
```

where the export cannot be resolved.

### Evaluation error

```text
bindings are valid
but executing module code throws/rejects
```

Example:

```js
throw new Error("startup failed");
```

These happen at different stages.

---

# 46. Top-Level Await Failure

Consider:

```js
// config.js
export const config = await loadConfig();
```

If:

```text
loadConfig()
```

rejects, dependent module evaluation can fail.

The failure is not merely:

```text
"the imported value is undefined."
```

The module graph has an asynchronous evaluation failure.

---

# 47. Async Parent Propagation

Conceptually:

```text
leaf module rejects
        ↓
async parent cannot continue
        ↓
parent evaluation rejects
        ↓
parent's async parents are notified
```

This is why one failed top-level-await dependency can prevent a larger module graph from completing evaluation.

---

# 48. Cycles + Top-Level Await

Cyclic module graphs become significantly more complex when top-level await is present.

Consider:

```text
A → B
↑   ↓
└── C
```

with:

```text
B has top-level await
```

The graph may require coordinated asynchronous evaluation.

The specification's SCC and async-parent machinery exists specifically to avoid treating each module as an isolated file.

---

# 49. Deadlocks and Top-Level Await

Poorly designed asynchronous module cycles can create problematic initialization behavior.

Example conceptually:

```text
A awaits B
B awaits A
```

The exact outcome depends on the actual module/evaluation graph and operations.

The architectural lesson is:

```text
avoid cyclic startup dependencies that require each side
to finish only after the other side finishes.
```

Do not confuse:

```text
module graph cycle
```

with:

```text
necessarily deadlock
```

They are not equivalent.

---

# 50. Static Imports vs Dynamic Imports

Static:

```js
import { x } from "./x.js";
```

participates in the module dependency graph and linking/evaluation semantics.

Dynamic:

```js
import("./x.js")
```

returns a Promise and initiates loading/evaluation dynamically.

These are related but not interchangeable mechanisms.

---

# 51. Dynamic Import and Module Records

Conceptually:

```text
import(specifier)
→ host/module loading
→ module graph processing
→ evaluate module
→ Promise settlement
```

The exact host loading/resolution path depends on environment.

The returned Promise provides application-visible asynchronous completion.

---

# 52. Dynamic Import Failure Layers

Dynamic import can fail because of:

```text
specifier/resolution failure
module loading failure
linking failure
evaluation failure
```

The application often sees:

```text
Promise rejection
```

but debugging should identify which underlying stage failed.

---

# 53. Module Namespace Object

A module namespace object is a special object exposing module exports.

Example:

```js
import * as namespace from "./module.js";
```

Access:

```js
namespace.value
```

reflects the module export.

Its property behavior differs from an ordinary mutable object used as an ad hoc API registry.

---

# 54. Namespace Snapshot vs Live Binding

The namespace view reflects module exports.

Consider:

```js
export let count = 0;

setInterval(() => {
  count++;
}, 1000);
```

An importer observing:

```js
import * as ns from "./counter.js";
```

can see the changing export value through the live binding.

This is another reason:

```text
module namespace
```

is not simply:

```text
object copied at import time
```

---

# 55. Default Export

Default exports are a module syntax concept:

```js
export default function () {}
```

Internally, the exported binding is associated with:

```text
"default"
```

The syntactic form can hide this binding-oriented model from application developers.

---

# 56. Re-Exporting Default

Patterns such as:

```js
export { default as Button } from "./Button.js";
```

build module APIs without moving the underlying implementation.

This is useful for:

```text
public entrypoints
barrel modules
package APIs
```

but can create complex resolution graphs when heavily chained.

---

# 57. Barrel Modules

Example:

```js
// index.js
export { A } from "./a.js";
export { B } from "./b.js";
export { C } from "./c.js";
```

Benefits:

```text
stable public API
centralized exports
consumer convenience
```

Costs can include:

```text
dependency graph complexity
accidental broad imports
tooling analysis complexity
cyclic graph risk
```

Use deliberately.

---

# 58. Module Evaluation Side Effects

A module can perform work at top level:

```js
connect();
registerPlugin();
initialize();
```

Importing it can trigger those effects as part of module evaluation.

This creates architectural coupling:

```text
import dependency
→ startup side effect
```

Prefer explicit initialization when startup order/ownership needs to be controlled.

---

# 59. Side-Effect Ordering

If:

```text
A depends on B
```

B's evaluation can occur before A's own evaluation completes.

Therefore module imports can implicitly establish:

```text
startup order
```

Do not hide critical startup dependencies inside large module graphs without documenting them.

---

# 60. Tree Shaking vs Module Semantics

ESM enables static analysis of:

```text
imports
exports
```

because static module dependencies are part of the syntax/graph model.

Bundlers can use this to remove unreachable code.

However:

```text
tree shaking
```

is a build optimization.

It must respect:

```text
module side effects
```

and language semantics.

---

# 61. Cycles and Bundlers

A bundler may transform:

```text
module graph
```

into:

```text
one or more runtime chunks
```

while attempting to preserve observable module semantics.

Therefore:

```text
bundled execution structure
```

may look very different from:

```text
source module graph
```

but correct output must preserve required behavior.

---

# 62. Code Splitting

Dynamic import often supports code splitting.

Conceptually:

```text
initial graph
    ↓
dynamic boundary
    ↓
lazy chunk
```

The browser/runtime loads and evaluates the dynamic module when requested.

This creates performance trade-offs:

```text
smaller initial payload
vs
additional network/request/evaluation latency
```

---

# 63. Module Caching

Hosts typically avoid repeatedly evaluating the same resolved module identity as if it were a fresh independent module for every import.

The exact caching/storage model is host-dependent.

The application-level implication is:

```text
module import is not equivalent to "execute this file again from scratch every time."
```

---

# 64. Module Singleton Assumption

It is common for application developers to rely on:

```text
one module instance per resolved module identity
```

for stateful modules.

This should be discussed with awareness of:

```text
different URLs/specifiers
different realms
different workers
different processes
different package resolution paths
```

A module singleton is scoped by the relevant runtime/module graph, not globally across a distributed system.

---

# 65. Browser Module Resolution

Browsers commonly resolve relative/absolute module specifiers using URL semantics.

Example:

```js
import "./utils.js";
```

is resolved relative to the importing module's URL in the browser environment.

Bare package-like specifiers require host-supported mechanisms such as import maps or environment-specific tooling.

This is host behavior, not universal ECMAScript syntax semantics.

---

# 66. Node.js Module Resolution

Node.js adds its own resolution ecosystem for:

```text
package.json
exports
imports
node_modules
file extensions
node:
URL schemes
```

These are not all defined by ECMAScript itself.

This is why:

```text
ESM semantics
```

and:

```text
Node resolution
```

must be documented separately.

---

# 67. Import Maps

Browsers can use import maps conceptually to map:

```text
specifier
→ URL
```

Example:

```json
{
  "imports": {
    "utils": "/assets/utils.js"
  }
}
```

Then:

```js
import "utils";
```

can resolve through the host's import-map mechanism.

This changes resolution, not ECMAScript module-linking semantics.

---

# 68. Package Resolution Boundary

When writing:

```js
import express from "express";
```

the spelling:

```text
"express"
```

does not by itself specify:

```text
where the package lives
```

That is a host/toolchain resolution concern.

The module linker then works with the resolved module identity.

---

# 69. Resolution Debugging

When an import fails:

```text
1. What is the exact specifier?
2. What is the importing module identity?
3. Which environment is running?
4. Which resolver is used?
5. What module identity was produced?
6. Was loading successful?
7. Was linking successful?
8. Did evaluation fail?
```

Do not jump directly to:

```text
"ESM is broken."
```

---

# 70. Module Graph Debugging

Visualize:

```text
entry
├── A
│   ├── C
│   └── D
└── B
    └── C
```

Then identify:

```text
shared dependency
cycle
async dependency
side effect
failure
```

Graph reasoning is more reliable than reasoning from filenames alone.

---

# 71. Static Dependency Analysis

ESM allows tooling to statically discover:

```text
import declarations
export declarations
```

Tooling can build:

```text
module graph
```

before execution.

This supports:

```text
bundling
tree shaking
dependency visualization
cycle detection
preloading
code splitting
```

Dynamic import introduces runtime graph edges that may not be known statically in the same way.

---

# 72. Common Misconceptions

### Misconception 1

> “Import means execute the imported file immediately.”

Correction:

```text
Load, link, and evaluate are distinct stages.
```

### Misconception 2

> “ESM defines how Node finds packages.”

Correction:

```text
module semantics and host resolution are separate.
```

### Misconception 3

> “Imports copy values.”

Correction:

```text
module imports are live bindings.
```

### Misconception 4

> “Cycles are always errors.”

Correction:

```text
cycles are supported; invalid evaluation can still fail.
```

### Misconception 5

> “Top-level await only affects one module.”

Correction:

```text
it can affect dependent module evaluation.
```

### Misconception 6

> “Dynamic import is just a delayed static import.”

Correction:

```text
it participates in an asynchronous dynamic loading/evaluation path.
```

### Misconception 7

> “Bundled modules are semantically unrelated to ESM anymore.”

Correction:

```text
correct bundlers must preserve relevant module semantics.
```

---

# 73. Common Mistakes

```text
[ ] mixing resolution with linking
[ ] mixing linking with evaluation
[ ] confusing module records with objects
[ ] treating imports as copied values
[ ] ignoring live bindings
[ ] assuming cycles always fail
[ ] ignoring TDZ in cyclic initialization
[ ] assuming top-level await affects only local code
[ ] forgetting dynamic import returns a Promise
[ ] treating package resolution as ECMAScript semantics
[ ] assuming module caching is global across processes
[ ] hiding important startup side effects behind imports
[ ] ignoring bundler transformations
[ ] ignoring realm/worker/process boundaries
```

---

# 74. Comparison With Related Concepts

| Concept | Main Question |
|---|---|
| Resolution | Which module does this specifier identify? |
| Loading | How is that module obtained? |
| Linking | Which imports connect to which exports? |
| Initialization | Which module bindings are created? |
| Evaluation | When and how does module code execute? |
| Static import | What is declared in the module graph? |
| Dynamic import | How is module loading/evaluation initiated dynamically? |
| Module namespace | How are exports exposed as a namespace view? |
| Bundling | How is the graph transformed for distribution? |
| Code splitting | Which parts of the graph are loaded separately? |

---

# 75. Performance Considerations

Module performance can involve:

```text
module discovery
resolution
loading
parsing
linking
initialization
evaluation
network transfer
code splitting
cache behavior
```

Top-level await can add:

```text
startup latency
dependency waiting
waterfall effects
```

Dynamic imports can improve:

```text
initial payload
```

while adding:

```text
lazy-load latency
```

Measure the actual application.

---

# 76. Memory Considerations

Long-lived module graphs can retain:

```text
module state
closures
registries
singletons
event listeners
caches
```

A module-level global is effectively long-lived for the lifetime of its relevant module environment.

Do not treat:

```js
const cache = new Map();
```

inside a module as automatically short-lived.

---

# 77. Security Considerations

Module boundaries can affect:

```text
code loading
dependency trust
specifier control
dynamic imports
package resolution
supply chain
startup execution
```

Never construct dynamic import targets from untrusted user input without a deliberate security model.

Example risk:

```js
await import(userControlledSpecifier);
```

Potential consequences depend on host resolution and what modules are reachable.

---

# 78. Production Usage

Module architecture is central to:

```text
application startup
package APIs
bundling
code splitting
plugin systems
dependency isolation
testing
deployment
serverless cold starts
browser performance
```

Principal engineers should explicitly decide:

```text
module ownership
public exports
initialization side effects
dependency direction
cycle policy
dynamic-loading boundaries
```

---

# 79. Implementation From Scratch — Mini Module Loader

Build a toy module system supporting:

```text
module IDs
source
static imports
static exports
linking
evaluation
```

Represent:

```js
{
  id,
  requestedModules,
  imports,
  exports,
  environment,
  status
}
```

Do not claim this object model is the exact ECMAScript Module Record representation.

It is an educational implementation.

---

# 80. Mini Module Loader — Resolution

Implement:

```text
specifier + importer
→ module ID
```

Start with:

```text
./relative.js
```

only.

Then extend to:

```text
absolute IDs
aliases
```

Document which resolution rules are toy-runtime rules.

---

# 81. Mini Module Loader — Linking

Implement:

```text
load graph
→ discover dependencies
→ resolve imports
→ create environments
→ connect export bindings
→ detect missing exports
```

Then add cycle detection.

---

# 82. Mini Module Loader — Evaluation

Implement:

```text
evaluate dependencies
→ evaluate module
```

For a simple synchronous graph.

Track:

```text
unlinked
linking
linked
evaluating
evaluated
```

---

# 83. Mini Module Loader — Live Bindings

Implement bindings so that:

```text
module A changes export
→ module B observes new value
```

Do not implement exports as:

```text
copy object properties once.
```

This exercise should make live bindings concrete.

---

# 84. Mini Module Loader — Cycles

Create:

```text
A → B
B → A
```

and make your loader avoid infinite recursion during linking.

Then create a cycle where evaluation reads an uninitialized binding.

Observe and document the failure.

---

# 85. Mini Module Loader — Async Evaluation

Extend the toy loader with:

```text
top-level async initialization
```

Track:

```text
pending async dependencies
async parents
evaluation Promise
```

You are modeling the concepts, not reproducing the complete specification.

---

# 86. Implementation Progression

### Guided

Implement:

```text
module record
resolution
basic linking
```

### Partially Guided

Implement:

```text
live bindings
evaluation ordering
```

### No Reference

Implement:

```text
cycle-aware linker
```

### Edge-Case Hardened

Handle:

```text
missing export
ambiguous re-export
cycle
evaluation throw
dynamic loading failure
```

### Production-Grade Learning Version

Add:

```text
graph visualization
diagnostic codes
timing metrics
cache model
async evaluation tracing
```

---

# 87. Execution Walkthrough — Acyclic Graph

```text
A imports B
B imports C
```

Graph:

```text
A
↓
B
↓
C
```

Conceptual sequence:

```text
load A
→ load B
→ load C

link C
→ link B
→ link A

evaluate C
→ evaluate B
→ evaluate A
```

This is a conceptual dependency ordering model.

Actual implementation details depend on the specification algorithm and host.

---

# 88. Execution Walkthrough — Shared Dependency

```text
A imports C
B imports C
```

Graph:

```text
    C
   ↗ ↖
  A   B
```

C is one dependency node for the relevant module identity.

Its exports can be observed by both importers.

Do not assume:

```text
C executes independently once per importer.
```

---

# 89. Execution Walkthrough — Cycle

```text
A → B
↑   ↓
└───┘
```

Conceptual lifecycle:

```text
load
→ link A/B as cyclic graph
→ initialize environments
→ evaluate according to graph rules
```

A cycle may succeed when values are accessed only after initialization.

It may fail when a module reads another module's uninitialized binding.

---

# 90. Execution Walkthrough — Top-Level Await

```text
A imports B
B awaits async initialization
```

Conceptual flow:

```text
link B
link A

evaluate B
→ encounters top-level await
→ B becomes async/pending

A cannot finish dependent evaluation yet

B completes
→ async dependency count updates
→ A continues
→ A completes
```

The current specification models asynchronous dependencies and parent relationships explicitly. citeturn149700search1

---

# 91. Execution Walkthrough — Async Failure

```text
A imports B
B top-level await rejects
```

Conceptually:

```text
B evaluation fails
→ B records evaluation error
→ parent async dependency is informed
→ A cannot successfully complete
→ A's evaluation Promise rejects
```

This is graph-level failure propagation.

---

# 92. Debugging Exercises

## Exercise A — Missing Export

```js
import { missing } from "./module.js";
```

Debug:

```text
resolution
→ linking
→ ResolveExport
```

Determine where the failure occurs.

---

## Exercise B — Cyclic TDZ

Create:

```text
A imports x from B
B imports y from A
```

where one module accesses the other module's uninitialized export during evaluation.

Identify:

```text
link succeeds
evaluation begins
binding still uninitialized
access throws
```

---

## Exercise C — Top-Level Await

Create:

```text
config.js
app.js
```

where:

```text
config.js
```

waits for async configuration.

Measure:

```text
startup timing
```

and trace:

```text
dependent module waiting
```

---

# 93. Code Review Exercise

Review:

```js
// config.js
export const config = await loadConfig();

// app.js
import { config } from "./config.js";

startServer(config);
```

Questions:

```text
What happens before app evaluation completes?

What happens if loadConfig rejects?

What happens to consumers importing app.js?

Does every Promise elsewhere in config.js make the module "async"?

What are the startup-latency implications?

Would explicit initialization be clearer?
```

---

# 94. Interview Questions

### Fundamentals

```text
1. What is a Module Record?
2. What is the difference between loading, linking, and evaluation?
3. What is a live binding?
4. Why can't import bindings be reassigned locally?
5. What does ResolveExport do?
```

### Advanced

```text
6. How are cyclic modules linked?
7. Why are strongly connected components useful?
8. How can a cycle produce a TDZ error?
9. What is top-level await?
10. How does top-level await affect dependents?
11. What are pending async dependencies?
12. What are async parent modules?
13. What is the purpose of a top-level evaluation Promise?
14. How can module evaluation errors propagate?
15. How does dynamic import differ from static import?
```

### Principal

```text
16. When should top-level await be avoided in a production application?
17. How would you design a package entrypoint with explicit startup side effects?
18. How would you debug an ESM failure that might be resolution, linking, or evaluation?
19. How would you design tooling for module graph analysis?
20. How would you detect dangerous cyclic startup dependencies in a large monorepo?
```

---

# 95. Predict-the-Behavior Exercises

Predict the failure stage:

```text
resolution
linking
evaluation
```

### Exercise 1

```js
import { x } from "./missing-file.js";
```

### Exercise 2

```js
import { x } from "./module.js";
```

where the module exists but does not export:

```text
x
```

### Exercise 3

```js
import { x } from "./module.js";
```

where the export exists but evaluation throws.

### Exercise 4

A dependency contains:

```js
export const value = await load();
```

Predict what happens to an importing module.

### Exercise 5

Create a two-module cycle and determine whether the cycle:

```text
links
evaluates successfully
or fails due to uninitialized access
```

---

# 96. Mastery Exercises

### Exercise 1 — Graph Builder

Create a tool that reads a project and produces:

```text
module graph
dependency edges
cycles
entrypoints
dynamic import edges
```

### Exercise 2 — Export Resolver

Implement a simplified:

```text
ResolveExport(module, name)
```

supporting:

```text
local exports
re-exports
star exports
ambiguity
missing exports
```

### Exercise 3 — Cycle Analyzer

Use DFS/SCC analysis to report:

```text
strongly connected components
```

and rank:

```text
high-risk cycles
```

### Exercise 4 — Startup Analyzer

Find modules containing:

```text
top-level side effects
top-level await
```

and produce a startup dependency report.

### Exercise 5 — Async Module Trace

Instrument a toy runtime to report:

```text
module
status
pending async dependencies
evaluation start
evaluation end
evaluation failure
```

---

# 97. Principal Decision Framework

When choosing module architecture, evaluate:

```text
Correctness
Initialization order
Startup latency
Dependency coupling
Cycle risk
Memory
Security
Tree-shaking
Bundling
Testing
Observability
Deployment
Runtime compatibility
Future change
```

Prefer:

```text
explicit initialization
clear public exports
small dependency direction
low-risk graph structure
```

when startup side effects become difficult to reason about.

---

# 98. Production Checklist

For a large ESM codebase inspect:

```text
module graph
public entrypoints
cycles
barrel modules
top-level side effects
top-level await
dynamic imports
resolution rules
package exports
conditional exports
duplicate module identities
bundler behavior
test/runtime differences
```

---

# 99. Module Graph Metrics

Useful metrics include:

```text
module count
dependency edges
fan-in
fan-out
cycle count
SCC count
startup modules
dynamic import count
top-level-await modules
graph depth
```

These can identify architectural hotspots.

---

# 100. Memory / Lifecycle Warning

A module's top-level state can survive for the lifetime of the relevant module environment.

Therefore:

```js
const subscribers = new Set();
```

can become a long-lived registry.

Ask:

```text
Who removes subscribers?

What happens during hot reload?

What happens in tests?

What happens across workers?

What happens across processes?
```

---

# 101. Specification / Runtime Source Discipline

For module questions, prefer:

```text
1. ECMAScript Module Records
2. module linking algorithms
3. module evaluation algorithms
4. host loading/resolution
5. browser/Node-specific resolver behavior
6. bundler transformations
```

The current ECMAScript specification defines cyclic module linking with DFS/SCC machinery and asynchronous module evaluation state such as `[[PendingAsyncDependencies]]`, `[[AsyncParentModules]]`, and `[[TopLevelCapability]]`. citeturn149700search1turn149700search0

Do not use:

```text
"Node resolves it this way"
```

as proof of:

```text
ECMAScript defines it this way.
```

---

# 102. Common Failure Modes

```text
Failure 1:
Treating module resolution and linking as one step.

Failure 2:
Treating imports as copied values.

Failure 3:
Ignoring live bindings.

Failure 4:
Assuming cycles always fail.

Failure 5:
Ignoring TDZ across cyclic module initialization.

Failure 6:
Confusing linking failure with evaluation failure.

Failure 7:
Ignoring top-level await dependency propagation.

Failure 8:
Assuming dynamic import is merely lazy source execution.

Failure 9:
Treating Node/browser resolution as ECMAScript semantics.

Failure 10:
Ignoring duplicate module identities.

Failure 11:
Hiding startup side effects behind imports.

Failure 12:
Assuming bundler output is structurally identical to source modules.
```

---

# 103. Retrieval Record

```md
# Chapter 126 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Load / Resolve
-

## Link
-

## Evaluate
-

## Live Bindings
-

## Cycles / SCC
-

## Top-Level Await
-

## Async Dependencies
-

## Module Errors
-

## Host Resolution
-

## Bundler Interaction
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

# 104. Spaced Retrieval Schedule

### Day 0

Study:

```text
load
link
evaluate
```

and draw one module graph.

### Day 1

Explain:

```text
live bindings
```

without notes.

### Day 3

Trace:

```text
acyclic graph
cycle
```

by hand.

### Day 7

Implement:

```text
toy linker
```

from requirements.

### Day 14

Analyze:

```text
top-level await
```

with an async dependency graph.

### Day 21

Build a cycle/SCC analyzer.

### Day 30

Explain the complete ESM lifecycle in ten minutes.

---

# 105. Dependency Graph

```text
Chapter 10
scope
   ↓
Chapter 11
TDZ
   ↓
Chapter 12
execution contexts
   ↓
Chapter 31–36
async / Promise / await
   ↓
Chapter 41
specification architecture
   ↓
Chapter 42
abstract operations
   ↓
Chapter 44
realms / agents
   ↓
Chapter 64
ES Modules
   ↓
Chapter 66
package.json / resolution
   ↓
Chapter 69
bundlers
   ↓
Chapter 123
grammar / parsing
   ↓
Chapter 124
completion / reference
   ↓
Chapter 125
Promise internals
   ↓
Chapter 126
module linking / evaluation
   ↓
Chapter 127
SharedArrayBuffer / Atomics / memory model
```

---

# 106. Concept Connections

## Depends On

```text
grammar
static semantics
scope
TDZ
execution contexts
Promises
await
realms
module syntax
package resolution
bundling
```

## Builds Toward

```text
package distribution
Node/browser module architecture
dynamic loading
platform tooling
SharedArrayBuffer / Atomics
advanced runtime behavior
```

## Related Concepts

```text
dependency graphs
SCC algorithms
DFS
lazy loading
code splitting
tree shaking
startup architecture
plugin systems
```

## Concepts Revisited

```text
TDZ
live bindings
Promises
async/await
realms
closures
scope
```

## Why This Chapter Matters

ESM is a complete dependency-and-execution system.

The important progression is:

```text
specifier
→ resolution
→ module identity
→ graph
→ linking
→ bindings
→ evaluation
→ async propagation
```

Once that becomes clear, seemingly unrelated behaviors become part of one model.

---

# 107. Track A — Core Theory

Master:

```text
Module Records
module requests
loading
resolution
linking
environment initialization
live bindings
ResolveExport
direct/indirect/star exports
cycles
SCCs
evaluation order
top-level await
async dependencies
async parents
evaluation Promise
module errors
dynamic import
namespace objects
```

Deliverable:

```text
explain an ESM dependency graph from source to evaluation.
```

---

# 108. Track B — Implementation

Build:

```text
toy module loader
resolver
linker
live-binding environment
cycle analyzer
async evaluator
module graph visualizer
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```

Deliverable:

```text
demonstrate module semantics by implementation.
```

---

# 109. Track C — Interview / Reasoning

Practice:

```text
"How does import actually work?"

"What is linking?"

"What is a live binding?"

"Why can cycles work?"

"Why can cycles trigger TDZ?"

"What does top-level await do to dependents?"

"Where does Node's resolver stop and ECMAScript semantics begin?"
```

Deliverable:

```text
precise ESM reasoning.
```

---

# 110. Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] load/link/evaluate distinguished
[ ] Module Record understood
[ ] live bindings understood
[ ] export resolution understood
[ ] cycles understood
[ ] SCC concept understood
[ ] linking failure understood
[ ] evaluation failure understood
[ ] top-level await understood
[ ] async dependency propagation understood
[ ] host resolution boundary understood
[ ] toy module loader implemented
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] trace an ESM graph
[ ] distinguish resolution/link/evaluation failures
[ ] explain live bindings
[ ] explain cycles at binding/evaluation level
[ ] reason about top-level await
[ ] reason about async module propagation
[ ] distinguish browser/Node resolution from ECMAScript semantics
[ ] analyze module graphs algorithmically
[ ] design safe production startup/module boundaries
[ ] read module-linking specification algorithms fluently
```

---

# 111. Completion Snapshot

```md
# Chapter 126 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Primary Gaps:
-

Module Records:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Resolution:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Linking:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Live Bindings:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Cycles / SCC:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Top-Level Await:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Host Boundary:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Module Loader:
[ ] not started
[ ] partial
[ ] complete
[ ] hardened
```

---

# 112. Completion Criteria

```text
[ ] Module Records explained
[ ] module requests explained
[ ] loading distinguished from linking
[ ] linking distinguished from evaluation
[ ] host resolution boundary understood
[ ] module environment explained
[ ] live bindings explained
[ ] imported binding immutability explained
[ ] direct/indirect/star exports explained
[ ] ResolveExport explained
[ ] namespace objects explained
[ ] export ambiguity explained
[ ] cyclic modules explained
[ ] SCC/DFS mechanism understood
[ ] TDZ across modules understood
[ ] link failure distinguished from evaluation failure
[ ] top-level await explained
[ ] HasTLA concept understood
[ ] pending async dependencies understood
[ ] async parent modules understood
[ ] top-level evaluation Promise understood
[ ] async error propagation understood
[ ] dynamic import explained
[ ] browser/Node resolution distinguished
[ ] bundler relationship understood
[ ] module graph debugging practiced
[ ] toy module loader implemented
```

---

# 113. Final Principal Mental Model

Use:

```text
source text
    ↓
Module Record
    ↓
module requests
    ↓
host resolution/loading
    ↓
dependency graph
    ↓
linking
    ↓
environment initialization
    ↓
live binding connections
    ↓
evaluation
    ↓
synchronous completion
or
asynchronous module completion
```

When debugging:

```text
Did resolution identify the module?

Did loading obtain it?

Did linking resolve every import/export?

Did environments initialize?

Did evaluation begin?

Did evaluation throw?

Did top-level await suspend evaluation?

Which parent modules depend on completion?

What Promise represents async module completion?
```

That is the correct foundation for advanced ESM reasoning.

---

# 114. Final Principal Principle

> **ECMAScript modules are a graph-based execution system, not a file-import convenience.**

The mature model is:

```text
module identity
+
dependency graph
+
linking
+
live bindings
+
evaluation
+
async propagation
```

Once you understand this system, you can reason about:

```text
ESM cycles
TDZ across modules
re-exports
barrel modules
dynamic imports
top-level await
startup ordering
bundlers
tree shaking
code splitting
Node/browser resolution boundaries
```

and debug failures by asking:

```text
Which phase failed?
```

rather than:

```text
"Why didn't my import work?"
```

That phase-based model is essential for:

```text
large ESM codebases
package authoring
bundler engineering
startup architecture
runtime debugging
module graph analysis
principal-level platform engineering
```