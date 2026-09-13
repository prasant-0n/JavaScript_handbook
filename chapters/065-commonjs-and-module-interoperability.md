\
# Chapter 65 — CommonJS and Interoperability

> **Curriculum position:** Part XII — Modules / Tooling  
> **Previous chapter:** Chapter 64 — ES Modules  
> **Next chapter:** Chapter 66 — `package.json` and Module Resolution  
> **Primary environment:** Modern Node.js, with current Node.js 26.x behavior treated as the reference point where version-sensitive details matter.

---

# Chapter Mission

Master **CommonJS (CJS)** as Node.js's original module system and, more importantly, master the boundary between CommonJS and ECMAScript Modules (ESM).

This chapter is not a historical tour of:

```js
const x = require('x');
module.exports = x;
```

It is a runtime and architecture chapter.

You should be able to reason through:

```text
CommonJS source
   ↓
module wrapper
   ↓
require()
   ↓
CommonJS resolution
   ↓
module loading
   ↓
module.exports
   ↓
cache
   ↓
execution
```

and then:

```text
CommonJS
   ↕
interop boundary
   ↕
ESM
```

The central questions are:

- What exactly does Node do with a CommonJS file?
- What are `exports`, `module`, `require`, `__filename`, and `__dirname`?
- Why does `exports = ...` behave differently from `module.exports = ...`?
- How does CommonJS resolution differ from ESM resolution?
- How does the CommonJS cache work?
- How do circular dependencies behave?
- What does `require()` return?
- What does ESM `import` observe when importing a CommonJS module?
- Why are CommonJS named exports from ESM compatibility detection heuristics rather than native CommonJS live bindings?
- When can CommonJS `require()` load an ESM module in modern Node?
- Why does top-level `await` matter to CJS ↔ ESM interoperability?
- What does `module.createRequire()` solve?
- What problems arise when a package publishes both CJS and ESM?
- How should a large production codebase migrate without creating duplicate state, inconsistent APIs, or impossible debugging conditions?

The principal-level goal is:

> **Understand both module systems deeply enough that you can design, migrate, debug, and defend their boundary.**

---

# 1. Learning Objectives

After this chapter, you should be able to:

## CommonJS theory

- Define CommonJS in the context of Node.js.
- Explain the Node CommonJS module wrapper.
- Explain `module.exports`.
- Explain the `exports` shortcut.
- Explain `require()`.
- Explain `require.resolve()`.
- Explain `require.cache`.
- Explain `require.main`.
- Explain `module.children`.
- Explain `module.parent` / parent-module concepts and current limitations.
- Explain `__filename`.
- Explain `__dirname`.
- Explain CommonJS module-local scope.
- Explain synchronous CommonJS loading.
- Explain CommonJS evaluation timing.

## Resolution

- Explain relative, absolute, core, and package specifiers.
- Explain Node's CommonJS file lookup.
- Explain directory lookup.
- Explain `package.json` handling.
- Explain `node_modules` traversal.
- Explain package `"exports"` and `"imports"` at the CommonJS boundary.
- Explain `require.resolve()`.

## Caching and cycles

- Explain CommonJS module caching.
- Explain partially initialized modules.
- Explain circular dependencies.
- Explain why CommonJS cycles can expose incomplete exports.
- Explain why path identity and symlinks can affect caching.

## Interoperability

- Import CommonJS from ESM.
- Explain default import behavior.
- Explain named-export detection for CommonJS.
- Explain limitations of CommonJS named-export interop.
- Use `module.createRequire()` from ESM.
- Use dynamic `import()` to load ESM from CommonJS.
- Understand modern `require(esm)` support and its synchronous/top-level-await constraint.
- Explain the `"module.exports"` interop export feature in modern Node.
- Identify dual-package hazards.

## Production engineering

- Design module boundaries.
- Migrate CJS to ESM incrementally.
- Avoid duplicate singleton state.
- Maintain public API stability.
- Test both module loaders.
- Diagnose resolution failures.
- Diagnose circular dependency failures.
- Diagnose interoperability failures.
- Evaluate package export maps.

## Principal judgment

- Decide whether a package should remain CJS, become ESM-only, or support both.
- Defend a migration strategy.
- Design compatibility layers.
- Identify hidden operational costs of dual publishing.
- Establish rules for module-system boundaries in large repositories.

---

# 2. Prerequisites

Recommended:

- Chapter 10 — Scope and Lexical Environments
- Chapter 12 — Execution Contexts
- Chapter 15 — Objects and Property Semantics
- Chapter 17 — Prototypes
- Chapter 28 — JSON and Serialization
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — Abstract Operations
- Chapter 44 — Realms and Agents
- Chapter 58 — Node.js Architecture
- Chapter 59 — Node Core APIs
- Chapter 62 — Process Lifecycle
- Chapter 64 — ES Modules

---

# 3. What Is CommonJS?

CommonJS is Node.js's original module system.

Example:

```js
// math.js
function add(a, b) {
  return a + b;
}

module.exports = {
  add,
};
```

Consumer:

```js
const { add } = require('./math.js');

console.log(add(2, 3));
```

Conceptually:

```text
math.js
  ↓
CommonJS loader
  ↓
module.exports
  ↓
require()
```

Unlike ESM, CommonJS is not the ECMAScript language's standardized module system.

It is a Node.js module/runtime mechanism.

---

# 4. Why Does CommonJS Exist?

Historically, server-side JavaScript needed a practical module system before standardized ESM existed.

CommonJS provided:

- synchronous loading,
- explicit exports,
- module-local scope,
- dependency reuse,
- package resolution,
- module caching,
- Node-friendly filesystem semantics.

The result became foundational to the Node ecosystem.

Millions of packages were built around:

```js
require()
module.exports
```

Even after ESM became standardized, CommonJS remains an important compatibility system.

---

# 5. Mental Model

Think of CommonJS as:

```text
file
 ↓
Node wraps file in function
 ↓
Node creates module object
 ↓
Node provides require/exports
 ↓
Node executes wrapper
 ↓
module.exports becomes the public value
 ↓
require() returns that value
 ↓
module is cached
```

This is fundamentally different from ESM's:

```text
source text
 ↓
module record
 ↓
dependency graph
 ↓
linking
 ↓
instantiation
 ↓
evaluation
```

---

# 6. The CommonJS Wrapper

Node conceptually wraps CommonJS source with:

```js
(function(exports, require, module, __filename, __dirname) {
  // module source
});
```

Node's current documentation describes this wrapper and explains that it creates module-local scope and supplies convenience variables. citeturn215289search1

This explains why:

```js
console.log(__filename);
console.log(__dirname);
console.log(module);
console.log(require);
```

can work without those identifiers being normal ECMAScript globals.

They are local to the CommonJS module wrapper.

---

# 7. The Five Famous CommonJS Variables

Within a CommonJS module:

```text
exports
require
module
__filename
__dirname
```

are available through the CommonJS wrapper.

### Key warning

They are not all equivalent.

Especially:

```text
exports
```

is a local reference initially pointing at:

```text
module.exports
```

---

# 8. `module.exports`

The authoritative export value is:

```js
module.exports
```

Example:

```js
module.exports = function add(a, b) {
  return a + b;
};
```

Consumer:

```js
const add = require('./add.js');
```

The returned value is the function assigned to `module.exports`.

Node's CommonJS docs explicitly define `module.exports` as the object/value made available through `require()`. citeturn215289search1

---

# 9. The `exports` Shortcut

At the start of a CommonJS module, conceptually:

```js
exports === module.exports
```

Therefore:

```js
exports.add = add;
```

works.

But:

```js
exports = add;
```

does not replace the exported value.

Why?

Because:

```text
exports
   │
   └── initially points to ──→ module.exports
```

Then:

```js
exports = add;
```

only changes the local variable.

The relationship becomes:

```text
exports ──→ add

module.exports ──→ original export object
```

Node's current documentation explicitly warns about this distinction. citeturn215289search1

---

# 10. Code → Prediction → Result → Trace

## Example

```js
exports.a = 1;
exports.b = 2;
```

### Prediction

What does `require()` return?

### Actual Result

Conceptually:

```js
{
  a: 1,
  b: 2,
}
```

### Trace

```text
module.exports = {}
        ↑
        │
     exports
        │
exports.a = 1
exports.b = 2
```

### Rule

Mutating `exports` mutates the object currently referenced by `module.exports`.

---

# 11. Assignment Trap

## Code

```js
exports = {
  a: 1,
};
```

### Prediction

Does the consumer receive `{ a: 1 }`?

### Actual Result

No.

The module still exports the original `module.exports` value unless you also replace it.

### Correct

```js
module.exports = {
  a: 1,
};
```

---

# 12. Exporting Functions

CommonJS is flexible:

```js
module.exports = function createClient() {
  // ...
};
```

Consumer:

```js
const createClient = require('./client.js');
```

Or:

```js
module.exports.createClient = createClient;
```

Consumer:

```js
const { createClient } = require('./client.js');
```

Or:

```js
module.exports = {
  createClient,
  ClientError,
};
```

---

# 13. CommonJS Export Shape

CommonJS exports one value:

```text
module.exports → arbitrary JavaScript value
```

That value can be:

- object,
- function,
- class,
- primitive,
- array,
- instance,
- callable object.

This is one conceptual difference from ESM's named-export binding model.

---

# 14. Functions with Properties

CommonJS permits:

```js
function createClient() {}

createClient.VERSION = '1.0.0';

module.exports = createClient;
```

Consumer:

```js
const createClient = require('./client.js');

createClient();
console.log(createClient.VERSION);
```

This style appears in older Node libraries.

It can be concise but may be less explicit than a structured named-export API.

---

# 15. `require()`

Basic form:

```js
const fs = require('node:fs');
```

Relative:

```js
const helper = require('./helper.js');
```

Package:

```js
const express = require('express');
```

Node's CommonJS documentation states that `require()` uses the CommonJS loader. `import()` uses the ECMAScript module loader. citeturn215289search1

---

# 16. Synchronous Loading

CommonJS `require()` is historically synchronous:

```js
const config = require('./config.js');
```

This means dependency loading happens as part of synchronous module execution.

Advantages:

- simple startup sequencing,
- straightforward module initialization,
- easy API design.

Costs:

- startup can block,
- synchronous filesystem resolution/loading,
- difficult integration with inherently asynchronous module graphs.

---

# 17. CommonJS Resolution

A high-level model for:

```js
require(X)
```

is:

```text
1. built-in?
2. relative/absolute?
3. package imports?
4. package self-reference?
5. node_modules traversal?
6. package exports?
7. file lookup?
8. directory lookup?
9. module not found
```

Node publishes the detailed CommonJS resolver algorithm in its current documentation. citeturn215289search1

---

# 18. Core Modules

```js
require('node:fs');
require('node:path');
```

Core/built-in modules have special resolution behavior.

Use `node:` when you want the built-in dependency to be explicit.

---

# 19. Relative Modules

```js
require('./utils');
```

CommonJS resolution has historically been more permissive than Node ESM.

It can perform file and directory lookup.

This is one major reason:

```text
CJS import paths
```

cannot be blindly copied into:

```text
ESM import paths
```

---

# 20. File Lookup

A conceptual CommonJS lookup can attempt:

```text
X
X.js
X.json
X.node
```

The exact behavior is governed by the current Node resolution algorithm.

The fact that CommonJS supports extension searching is an important interoperability distinction.

---

# 21. Directory Lookup

CommonJS can resolve directories according to its package/main/index resolution rules.

Conceptually:

```text
./feature/
    ↓
package.json main
    ↓
index.js
```

Modern package `"exports"` can provide a more controlled package API.

---

# 22. `node_modules` Traversal

For:

```js
require('some-package');
```

Node searches upward through `node_modules` locations.

Conceptually:

```text
/app/src/node_modules
/app/node_modules
/node_modules
```

plus relevant package-resolution rules.

Node documents this search behavior and recommends local dependency installation for reliability. citeturn215289search1

---

# 23. `require.resolve()`

Use:

```js
const filename = require.resolve('./config.js');
```

This resolves the request without loading the module.

It returns the resolved filename or throws if resolution fails.

Node's documentation defines it as a lookup mechanism based on CommonJS resolution. citeturn215289search1

---

# 24. `require.resolve()` vs `import.meta.resolve()`

These are related but not identical.

| API | Loader model |
|---|---|
| `require.resolve()` | CommonJS |
| `import.meta.resolve()` | ESM |

They can produce different results for the same-looking specifier because their resolution algorithms differ.

---

# 25. CommonJS Cache

Node caches modules after their first successful load.

Example:

```js
const a = require('./counter.js');
const b = require('./counter.js');

console.log(a === b);
```

Typically:

```text
true
```

for the same resolved module.

Node documents that `require()` returns the same cached object for the same resolved filename unless cache behavior is changed. citeturn215289search1

---

# 26. Why Caching Matters

Without caching:

```text
require A
  ↓
execute file

require A
  ↓
execute file again

require A
  ↓
execute file again
```

With caching:

```text
require A
  ↓
execute once
  ↓
cache export

require A
  ↓
return cached export
```

This makes CommonJS modules naturally capable of process-wide singleton-like state.

---

# 27. Module Cache Is Not a Universal Singleton Guarantee

The cache key depends on resolved identity.

Node's current documentation notes that different resolved filenames can result in different module cache entries. It also describes case-sensitivity and filesystem-specific caveats. citeturn215289search1

Therefore:

```text
same source file
```

does not always imply:

```text
same module instance
```

---

# 28. `require.cache`

You can inspect:

```js
console.log(require.cache);
```

and delete an entry:

```js
delete require.cache[require.resolve('./module.js')];
```

Then a future `require()` may reload it.

This is often useful in development/test environments.

It is dangerous to treat cache mutation as ordinary production application state.

---

# 29. Cache Invalidation Hazards

Deleting one module does not necessarily reset all transitive dependencies.

Example:

```text
A
↓
B
↓
C
```

If only:

```text
B
```

is removed from cache, `C` may remain cached.

Now you can construct mixed generations of module state.

### Principal lesson

> Manual CommonJS cache invalidation is a graph problem, not a single-object problem.

---

# 30. Circular Dependencies

CommonJS supports cycles.

Example:

```text
a.js → b.js
b.js → a.js
```

Because modules are cached as loading progresses, one side may receive a partially initialized export.

Node's current documentation explicitly notes that partially done objects can be returned in cycles. citeturn215289search1

---

# 31. Circular Example

```js
// a.js
exports.name = 'A';

const b = require('./b.js');

exports.bName = b.name;
```

```js
// b.js
exports.name = 'B';

const a = require('./a.js');

exports.aName = a.name;
```

Depending on evaluation timing, each module can observe an incomplete snapshot of the other's export object.

---

# 32. Why CommonJS Cycles Differ From ESM Cycles

CommonJS:

```text
exports object
+
incremental mutation
+
partial initialization
```

ESM:

```text
module bindings
+
linking
+
evaluation
+
TDZ behavior
```

Therefore the same graph:

```text
A ↔ B
```

can fail or behave differently depending on the module system.

---

# 33. CommonJS Module Evaluation

When:

```js
require('./a.js');
```

causes a new module load, Node:

```text
resolve
 ↓
create module object
 ↓
cache module as loading
 ↓
execute wrapper
 ↓
populate module.exports
 ↓
return exports
```

The timing of cache registration is important for cycle handling.

---

# 34. Partial Initialization

Consider:

```js
module.exports.first = true;

const other = require('./other.js');

module.exports.second = true;
```

If `other.js` requires this module during the middle of initialization, it may observe:

```js
{
  first: true
}
```

without:

```js
second: true
```

This is a central CommonJS cycle behavior.

---

# 35. Design Rule for Cycles

Avoid APIs whose correctness depends on another module's export object being completely initialized during a circular load.

Prefer:

```text
A → shared abstraction ← B
```

over:

```text
A ↔ B
```

when practical.

---

# 36. `__filename`

In CommonJS:

```js
console.log(__filename);
```

gives the current module's absolute filename under Node's CommonJS model.

Node documents that the value represents the current module file path and resolves symlinks in the documented CommonJS behavior. citeturn215289search1

---

# 37. `__dirname`

```js
console.log(__dirname);
```

is the directory containing the current CommonJS module.

It is not:

```js
process.cwd()
```

These can be different.

Example:

```text
module:
  /app/src/main.js

cwd:
  /app
```

Then:

```text
__dirname = /app/src
process.cwd() = /app
```

---

# 38. `process.cwd()` vs `__dirname`

This distinction is important in production.

```js
fs.readFileSync('./config.json');
```

is relative to:

```text
process.cwd()
```

while:

```js
path.join(__dirname, 'config.json')
```

is relative to the module file.

Therefore deployment behavior can differ depending on current working directory.

---

# 39. `module` Object

CommonJS exposes:

```js
module
```

The current module object contains metadata and the export object.

Useful properties include:

```js
module.exports
module.filename
module.id
module.loaded
module.children
module.paths
```

Exact fields should be treated as Node runtime behavior.

---

# 40. `module.children`

A module can inspect modules it required:

```js
console.log(module.children);
```

This forms part of CommonJS's runtime module graph.

It can be useful for diagnostics.

Do not build core architecture around mutable internals of the module graph.

---

# 41. `require.main`

Traditional CommonJS entry-point check:

```js
if (require.main === module) {
  main();
}
```

This distinguishes the directly launched CommonJS main module from a module imported by another CommonJS module.

Node's current docs define `require.main` as the entry module object when the entry point is CommonJS. citeturn215289search1

---

# 42. ESM Equivalent

Modern ESM provides:

```js
if (import.meta.main) {
  main();
}
```

Node explicitly documents this as an ESM replacement concept for `require.main === module`. citeturn215289search0

---

# 43. CommonJS and Global Scope

CommonJS top-level variables are scoped to the module wrapper.

Example:

```js
const secret = 123;
```

does not create:

```js
global.secret
```

This is a major misconception about old Node code.

---

# 44. Strict Mode

Unlike ESM, CommonJS is not automatically strict mode.

This means legacy code can rely on sloppy-mode behavior unless it opts into strict mode:

```js
'use strict';
```

Modern production code should generally prefer strict semantics.

---

# 45. Top-Level `this`

CommonJS wrapper semantics mean top-level `this` can behave differently from ESM.

Do not port code like:

```js
this === undefined
```

assumptions between module systems without testing the intended environment.

---

# 46. CJS → ESM: Default Import

Suppose:

```js
// legacy.cjs
module.exports = function createClient() {};
```

ESM can typically consume it as:

```js
import createClient from './legacy.cjs';
```

Node provides the CommonJS `module.exports` value as the default export when importing CommonJS from ESM. citeturn215289search0

---

# 47. CJS Named-Export Interoperability

Suppose:

```js
// legacy.cjs
exports.add = (a, b) => a + b;
exports.multiply = (a, b) => a * b;
```

ESM may allow:

```js
import { add, multiply } from './legacy.cjs';
```

Node performs static analysis of CommonJS source to attempt to detect likely named exports.

This is an interoperability convenience.

It is not equivalent to native ESM live bindings.

Node's current documentation explicitly describes named-export detection for CommonJS and warns that it is based on static analysis. citeturn215289search0

---

# 48. The Important Limitation

Consider:

```js
// legacy.cjs
module.exports = {};

setTimeout(() => {
  module.exports.dynamic = 123;
}, 100);
```

A statically detected named export may not behave like a true live ESM export binding.

For reliable interop, treat:

```text
default export
```

as the primary bridge for arbitrary CommonJS `module.exports` values.

---

# 49. ESM Named Imports from CJS: Mental Model

Do not think:

```text
CJS object properties
=
ESM live bindings
```

Think:

```text
CJS module.exports
       ↓
Node compatibility analysis
       ↓
possible named import surface
```

This distinction is critical when migrating libraries.

---

# 50. Importing CJS as a Namespace

ESM can also do:

```js
import * as legacy from './legacy.cjs';
```

The resulting namespace contains the CommonJS-exported value and compatibility properties according to Node's interop rules.

Do not assume this namespace behaves identically to a native ESM module namespace.

---

# 51. `module.createRequire()`

ESM does not natively provide:

```js
require
module.exports
__dirname
```

Node provides:

```js
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);
```

Now:

```js
const legacyConfig = require('./legacy.cjs');
```

Node's ESM documentation identifies `module.createRequire()` as the mechanism for constructing a `require` function inside an ES module when needed. citeturn215289search0

---

# 52. Why `createRequire()` Exists

Useful when ESM needs:

- legacy CJS packages,
- native addons supported through CJS loading paths,
- JSON or legacy loader behavior,
- APIs that genuinely require CommonJS.

Architectural rule:

> Use it at clear compatibility boundaries rather than spreading it through the entire ESM codebase.

---

# 53. CJS → ESM with Dynamic `import()`

CommonJS can use:

```js
async function loadFeature() {
  const module = await import('./feature.mjs');
  return module;
}
```

Node's current docs state that dynamic `import()` is supported in CommonJS. citeturn215289search0turn215289search1

This is often the cleanest way for legacy CJS to consume an ESM-only dependency.

---

# 54. Why `require()` Historically Could Not Load ESM

Classic CommonJS expects synchronous module loading.

ESM can have:

```js
await something;
```

at top level.

A synchronous CommonJS caller cannot pause for an arbitrarily asynchronous module graph.

Historically this led to:

```text
CJS → ESM
    ↓
dynamic import()
```

as the compatibility bridge.

---

# 55. Modern `require(esm)`

Modern Node can synchronously load certain ESM graphs using `require()`.

Node's current CommonJS documentation states that this support is available for fully synchronous ES modules, meaning the graph must not contain top-level await. The feature has become stable in recent Node versions. citeturn215289search1

Conceptually:

```text
require()
   ↓
ESM graph
   ↓
must be synchronously evaluable
```

This does not eliminate all differences between CJS and ESM.

---

# 56. `require(esm)` Constraint

If the ESM graph contains:

```js
export const config = await loadConfig();
```

then synchronous `require()` cannot satisfy the module's asynchronous evaluation contract.

Use:

```js
const module = await import('./config.mjs');
```

from CJS instead.

---

# 57. `module.exports` Interop Export

Modern Node also supports a special ESM export name:

```js
export { value as 'module.exports' };
```

This can customize the value exposed directly by `require(esm)`. Node documents this as a compatibility mechanism. citeturn215289search1

Example:

```js
const client = {
  connect() {},
};

export { client as 'module.exports' };
```

Then a compatible `require()` consumer can receive the selected value directly rather than the ordinary namespace object.

Use this cautiously because it changes the interoperability contract.

---

# 58. The `__esModule` Marker

Modern Node may include:

```js
__esModule: true
```

on the namespace returned by `require(esm)` when there is a default export, to support transpiler conventions.

Node's documentation explicitly describes this as compatibility behavior for tools that translate ESM to CommonJS. Code authored directly in CommonJS should not depend on it. citeturn215289search1

---

# 59. Do Not Depend on `__esModule` Manually

Avoid:

```js
if (module.__esModule) {
  // assume...
}
```

as a universal module-system detector.

It is an ecosystem convention and compatibility marker, not a clean architectural source of truth.

---

# 60. Interoperability Matrix

| Direction | Main mechanism |
|---|---|
| ESM → CJS | `import` |
| ESM → CJS with explicit require semantics | `createRequire()` |
| CJS → ESM | dynamic `import()` |
| CJS → synchronous ESM | modern `require(esm)` when graph is eligible |
| ESM → native addon/CJS-oriented loader | `createRequire()` / appropriate Node API |
| Browser ESM → CJS | not natively equivalent |

---

# 61. ESM Importing CJS: Default vs Named

Prefer:

```js
import legacy from './legacy.cjs';
```

when the CommonJS package has an arbitrary `module.exports` value.

Named imports:

```js
import { helper } from './legacy.cjs';
```

can be convenient but depend on Node's static detection behavior.

For package API stability, be explicit about what you guarantee.

---

# 62. CommonJS Requiring ESM: Dynamic Import

Example:

```js
async function main() {
  const { createServer } = await import('./server.js');
  const server = createServer();
  await server.start();
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

This is a practical migration pattern.

---

# 63. Migration Architecture

A large CJS system can migrate incrementally:

```text
legacy CJS
     │
     ├── stable package boundaries
     │
     ▼
new ESM modules
     │
     ├── dynamic import for one direction
     ├── createRequire at narrow edges
     └── explicit tests
```

Avoid converting every file in one enormous change unless the repository and release strategy genuinely support it.

---

# 64. The Migration Direction

A practical strategy can be:

```text
new modules → ESM
legacy modules → CJS
boundary → explicit compatibility
```

Then gradually shrink the CJS region.

This creates an observable migration surface.

---

# 65. Migration Anti-Pattern

Do not create:

```text
every ESM module
   ↕
every CJS module
```

with compatibility calls scattered everywhere.

Instead:

```text
ESM core
   ↓
compatibility adapter
   ↓
CJS legacy boundary
```

This reduces architectural entropy.

---

# 66. Dual-Package Hazard

Consider:

```text
package/index.js → ESM
package/index.cjs → CJS
```

Both maintain:

```js
const registry = new Map();
```

If a process loads both:

```text
ESM registry
+
CJS registry
```

you now have two logical singleton states.

Node package documentation and ecosystem guidance treat this as a major dual-package hazard.

---

# 67. Why Duplicate State Is Dangerous

Suppose:

```js
// registry
export const clients = new Map();
```

and the CJS build has its own copy.

Then:

```text
ESM caller registers client A
CJS caller queries clients
```

The CJS caller may see:

```text
empty
```

even though “the package” registered a client.

This causes subtle cross-boundary bugs.

---

# 68. Avoiding Duplicate State

Possible strategies:

### Strategy A

Make one canonical implementation and adapt the other format to it.

### Strategy B

Move shared state into a separate single runtime service/module.

### Strategy C

Expose stateless APIs.

### Strategy D

Use process-global coordination only when carefully justified.

The best answer depends on the package.

---

# 69. Canonical Implementation Pattern

Example architecture:

```text
src/
  core.js       ← canonical logic
  index.js      ← ESM entry
  index.cjs     ← CJS compatibility entry
```

But be careful that both entries do not independently create state.

A stronger pattern can route both loaders toward one canonical implementation when Node's module-boundary constraints allow it.

---

# 70. `"exports"` for Dual Packages

Example:

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

This can select loader-specific entry points.

However:

```text
two entry files
=
potentially two module graphs
```

Design state ownership accordingly.

---

# 71. Conditional Export Testing

Test:

```text
Node ESM consumer
Node CJS consumer
bundler consumer
browser-oriented consumer
```

where supported.

Check:

```text
same API
same semantics
same error behavior
same state model
same side effects
```

---

# 72. Package `"type"` Boundary

Example:

```json
{
  "type": "module"
}
```

Then:

```text
.js → ESM
.cjs → CJS
```

A nested package can change interpretation through another `package.json`.

This means module classification depends on package scope.

---

# 73. Mixed Source Tree

Example:

```text
package.json
{
  "type": "module"
}

src/
  index.js
  legacy.cjs
```

This is valid.

Use:

```js
import legacy from './legacy.cjs';
```

or:

```js
const legacy = require('./legacy.cjs');
```

from a CJS context.

---

# 74. CJS Inside ESM Package

If a package is:

```json
{
  "type": "module"
}
```

then:

```text
.cjs
```

remains CommonJS.

This makes `.cjs` valuable as an explicit compatibility boundary.

---

# 75. ESM Inside CommonJS Package

If:

```json
{
  "type": "commonjs"
}
```

then:

```text
.mjs
```

remains ESM.

This gives gradual migration room.

---

# 76. JSON Interoperability

CommonJS historically supports:

```js
const config = require('./config.json');
```

ESM requires the explicit JSON import type syntax:

```js
import config from './config.json' with { type: 'json' };
```

A migration therefore cannot blindly transform every `require()` into `import`.

---

# 77. Native Addons

Some native addons are designed around CommonJS loading.

Node's current ESM documentation states that addons are not currently supported directly through ESM `import` and can instead be loaded using `module.createRequire()` or `process.dlopen`. citeturn215289search0

This is a practical reason a modern ESM application may still need a narrow CJS bridge.

---

# 78. CommonJS Built-ins and ESM Named Exports

Node built-in modules have CommonJS-oriented implementations with ESM named-export compatibility.

Node provides:

```js
import { readFile } from 'node:fs';
```

and also:

```js
import fs from 'node:fs';
```

The built-in module behavior is special and should not be generalized to arbitrary CommonJS packages.

---

# 79. `module.syncBuiltinESMExports()`

Node exposes a mechanism to synchronize named ESM exports of built-in modules after CommonJS-side mutations.

This is an advanced compatibility concern.

Do not use mutable built-in-module APIs as an ordinary application design strategy.

---

# 80. CommonJS and Tooling

CommonJS remains important to:

- older test runners,
- legacy CLIs,
- Node libraries,
- transpiled output,
- older bundler configurations.

Modern tooling often supports both but may have different assumptions.

Always test the exact build/test runtime rather than trusting file extensions alone.

---

# 81. Babel / TypeScript Interop Warning

Transpilers may generate patterns such as:

```js
Object.defineProperty(exports, "__esModule", {
  value: true,
});
```

or:

```js
exports.default = ...
```

This can make generated CommonJS look unlike hand-written CommonJS.

Do not infer source module semantics solely from emitted helper patterns.

---

# 82. Default Export Transpilation Confusion

A transpiler may turn:

```js
export default function foo() {}
```

into something resembling:

```js
exports.default = foo;
```

Then a raw CommonJS consumer may write:

```js
const foo = require('./module');
```

and get:

```js
{ default: foo, ... }
```

instead of the function itself.

This is one reason package interop contracts must be deliberate.

---

# 83. Native ESM vs Transpiled CJS

These are not interchangeable:

```text
source ESM
      ↓
transpiler
      ↓
CJS output
```

versus:

```text
native ESM
```

The runtime module system is determined by the loaded artifact and Node configuration, not merely by what syntax the source author intended.

---

# 84. Debugging Module-System Mismatch

Symptoms:

```text
ReferenceError: require is not defined
```

Likely:

```text
ESM context
```

Symptoms:

```text
Cannot use import statement outside a module
```

Likely:

```text
CJS classification
```

Symptoms:

```text
require is undefined
```

Check:

```text
package "type"
.mjs / .cjs
entry point
loader configuration
```

---

# 85. Debugging “module.exports Is Empty”

Possible reasons:

```text
1. assigned exports instead of module.exports
2. export assignment happens asynchronously
3. circular dependency exposes partial initialization
4. another loader transformed the module
5. package condition selected a different entry
```

Inspect the actual loaded artifact.

---

# 86. Debugging `exports =`

Bad:

```js
exports = {
  run,
};
```

Correct:

```js
module.exports = {
  run,
};
```

Or:

```js
exports.run = run;
```

---

# 87. Debugging Circular CJS

When seeing:

```text
undefined
```

during module initialization, draw:

```text
module graph
+
evaluation order
+
export mutation order
```

Then identify the exact point where a partially initialized object is read.

---

# 88. Debugging `ERR_REQUIRE_ESM`-Style Failures

Legacy CJS code may fail when loading a package that is ESM-only.

Modern choices include:

```js
const module = await import('package');
```

or migrating the consuming module boundary.

Do not solve the problem by randomly changing extensions.

Identify the module-system boundary first.

---

# 89. Debugging `ERR_REQUIRE_ASYNC_MODULE`

Modern Node can synchronously require some ESM but not a graph containing top-level await.

When synchronous `require()` encounters an asynchronous ESM graph, use:

```js
await import(...)
```

or redesign the boundary.

The exact current error and diagnostic behavior should be verified against the Node version used in production.

---

# 90. `createRequire()` Debugging

From ESM:

```js
const require = createRequire(import.meta.url);

const legacy = require('./legacy.cjs');
```

The starting location matters.

Using the wrong filename can alter package resolution semantics.

---

# 91. Security Considerations

## 91.1 Dynamic `require`

Avoid:

```js
require(userInput);
```

without a strict allowlist.

---

## 91.2 Dynamic `import`

Likewise:

```js
await import(userInput);
```

must not become an untrusted module-loader interface.

---

## 91.3 Package Evaluation

Loading a package can execute arbitrary module initialization.

```text
require(package)
   ↓
resolve
   ↓
load
   ↓
execute package code
```

Treat dependencies as executable supply-chain inputs.

---

# 92. Security and Package Exports

A package with:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

limits supported public paths.

Do not bypass the contract using filesystem tricks or undocumented deep imports.

---

# 93. Memory Considerations

CommonJS caching can intentionally retain:

```text
module state
+
closures
+
exports
+
dependency graph references
```

for the life of the process.

Be cautious about module-level caches:

```js
const giantCache = new Map();
```

They can become effectively process-lifetime storage.

---

# 94. Performance Considerations

CommonJS `require()` can synchronously perform:

```text
filesystem lookup
file read
parse
execute
```

on first load.

Therefore startup performance can be affected by:

- dependency count,
- module graph depth,
- filesystem behavior,
- package resolution,
- expensive module initialization.

---

# 95. Performance: Dynamic Interop

Dynamic:

```js
await import(...)
```

adds an asynchronous boundary.

Do not use it merely to avoid learning the module system.

Use it where lazy loading or a genuine compatibility boundary provides value.

---

# 96. Performance: `require.resolve()`

Repeated:

```js
require.resolve('some-package');
```

can perform resolution work.

Cache stable results when appropriate.

---

# 97. Production Architecture

A large application might deliberately separate:

```text
ESM application core
       │
       ├── ESM domain
       ├── ESM services
       ├── ESM transport
       │
       ▼
CJS compatibility adapters
       │
       ├── legacy libraries
       └── native integrations
```

This is preferable to random bidirectional crossings.

---

# 98. Adapter Pattern

Example:

```js
// legacy-adapter.js
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);

const legacyClient = require('legacy-client');

export function createClient(options) {
  return legacyClient.createClient(options);
}
```

Now the rest of the ESM codebase does not need to know:

```text
legacyClient is CommonJS
```

---

# 99. Why Adapters Matter

An adapter creates:

```text
compatibility boundary
```

which provides:

- isolation,
- easier migration,
- clearer ownership,
- centralized testing,
- simpler future removal.

---

# 100. CommonJS Compatibility Layer

Design:

```text
legacy system
      │
      ▼
CJS adapter
      │
      ▼
stable internal API
      │
      ▼
ESM application
```

Do not allow every business module to call `createRequire()` independently.

---

# 101. Testing Interoperability

A package supporting both systems should test:

```text
CJS consumer → CJS entry
CJS consumer → ESM entry where supported
ESM consumer → CJS entry
ESM consumer → ESM entry
```

Then verify:

```text
exports
errors
types
state
side effects
initialization
caching
```

---

# 102. State Consistency Test

For dual packages, explicitly test:

```text
import package
require package
```

in the same process.

Then verify whether:

```js
esmInstance === cjsInstance
```

or, more importantly:

```text
shared registry behavior
```

matches the contract.

---

# 103. API Shape Test

Test:

```js
const cjs = require('pkg');
```

and:

```js
import esm from 'pkg';
```

and:

```js
import { named } from 'pkg';
```

where supported.

Confirm exactly what each returns.

Do not assume default/named interop based on a transpiler you are not actually using.

---

# 104. Migration Plan — Phase 1

Inventory:

```text
files
packages
entry points
exports
require calls
dynamic imports
native addons
tests
build scripts
CLI entry points
```

Generate a graph.

---

# 105. Migration Plan — Phase 2

Classify nodes:

```text
CJS-only
ESM-only
dual-compatible
generated
external
unknown
```

Then identify strongly connected components/cycles.

---

# 106. Migration Plan — Phase 3

Define policy:

```text
new application code → ESM
legacy code → CJS
boundary → adapter
package API → exports map
```

Document exceptions.

---

# 107. Migration Plan — Phase 4

Convert leaf modules first.

Why?

```text
leaf
 ↓
few dependents
 ↓
lower blast radius
```

Then move upward toward application roots.

---

# 108. Migration Plan — Phase 5

Convert public package boundaries last.

Public APIs have the widest blast radius.

Maintain compatibility tests throughout.

---

# 109. Migration Anti-Pattern: Big-Bang Rewrite

Potential failure:

```text
10,000 files
      ↓
automated conversion
      ↓
module classification errors
      ↓
dependency cycles
      ↓
test failures
      ↓
runtime-only failures
```

The more module-system assumptions exist, the more valuable incremental migration becomes.

---

# 110. Migration Anti-Pattern: Mechanical `require` → `import`

This transformation is not always valid:

```js
const value = require('./value');
```

to:

```js
import value from './value.js';
```

because:

- resolution differs,
- export shape differs,
- timing differs,
- cycles differ,
- JSON loading differs,
- dynamic paths differ,
- side-effect timing can change.

---

# 111. Production Failure Modes

## Failure Mode 1 — Duplicate singleton

Cause:

```text
CJS + ESM separate package instances
```

## Failure Mode 2 — Missing extension

Cause:

```text
CJS path copied into ESM
```

## Failure Mode 3 — `exports` bug

Cause:

```text
exports = value
```

instead of:

```text
module.exports = value
```

## Failure Mode 4 — Async ESM through sync CJS

Cause:

```text
top-level await
```

## Failure Mode 5 — Circular partial state

Cause:

```text
A ↔ B
```

with initialization-dependent exports.

## Failure Mode 6 — Public API accidental deep import

Cause:

```text
no exports map / undocumented path
```

---

# 112. Common Misconceptions

## Misconception 1

> CommonJS is the JavaScript module system.

No. It is a Node.js module system.

## Misconception 2

> `exports` and `module.exports` are always the same.

Only initially/reference-wise. Reassigning `exports` breaks the local alias.

## Misconception 3

> CommonJS imports are live bindings like ESM.

No.

## Misconception 4

> `require()` and `import()` resolve identically.

No.

## Misconception 5

> ESM named imports from CJS are real ESM named exports.

Node can synthesize compatibility named exports through static analysis.

## Misconception 6

> Dual publishing automatically means one module instance.

No.

## Misconception 7

> `createRequire()` converts CJS into ESM.

No.

It provides a CJS loading mechanism inside an ESM module.

## Misconception 8

> `require(esm)` removes the need to understand module systems.

No.

It has constraints and distinct semantics.

---

# 113. Common Mistakes

```text
[ ] Reassigning exports instead of module.exports
[ ] Copying CJS specifiers into ESM
[ ] Assuming named CJS imports are live
[ ] Using createRequire everywhere
[ ] Ignoring package exports
[ ] Ignoring duplicate singleton state
[ ] Treating module cache mutation as normal logic
[ ] Assuming __dirname === process.cwd()
[ ] Using dynamic paths without validation
[ ] Ignoring native addon boundaries
[ ] Treating transpiled CJS as identical to native CJS
```

---

# 114. Comparison: CJS vs ESM

| Dimension | CommonJS | ESM |
|---|---|---|
| Origin | Node ecosystem | ECMAScript standard |
| Loading | synchronous `require()` | static imports + async dynamic import |
| Export model | `module.exports` value | named/default bindings |
| Live bindings | no native ESM-style live binding model | yes |
| Module wrapper | Node wrapper | module semantics |
| `__dirname` | ✅ | ❌ native CJS variable |
| `import.meta` | ❌ | ✅ |
| Top-level await | ❌ syntax/model | ✅ |
| Relative resolution | CJS algorithm | ESM URL-oriented resolver |
| Extension searching | ✅ historically | ❌ default ESM relative lookup |
| Cache | `require.cache` | loader/module identity model |
| Circular behavior | partially initialized exports | linked bindings + evaluation semantics |
| Browser-native | ❌ | ✅ |
| `require` | ✅ | ❌ native |
| `createRequire` | unnecessary | bridge into CJS |

---

# 115. Comparison: `module.exports` vs ESM Named Export

```text
CommonJS

module.exports
      ↓
one exported value
      ↓
consumer receives value
```

versus:

```text
ESM

export const a
export const b
      ↓
export bindings
      ↓
consumer imports bindings
```

This distinction explains many interop surprises.

---

# 116. Comparison: CJS Cache vs ESM Identity

CJS:

```text
resolved filename
      ↓
require.cache
```

ESM:

```text
resolved module URL / loader context
      ↓
module identity
```

The two systems should not be assumed to share one universal cache model.

---

# 117. Specification Boundary

CommonJS is Node host behavior.

ECMAScript specifies:

```text
ESM
```

but not:

```text
require()
module.exports
__dirname
node_modules resolution
```

Therefore:

```text
ECMAScript specification
      ≠
Node CommonJS specification
```

For CommonJS behavior, use Node's documentation and implementation references.

---

# 118. Node Resolution Algorithm — High-Level

For:

```js
require(X)
```

think:

```text
builtin
 ↓
absolute/relative path
 ↓
package imports
 ↓
package self
 ↓
package exports
 ↓
node_modules traversal
 ↓
file/directory resolution
 ↓
throw
```

The current Node documentation provides the full algorithm and should be consulted for version-sensitive details. citeturn215289search1

---

# 119. Package `"exports"` and CJS

The `"exports"` field is not ESM-only.

It can affect:

```js
require('package/subpath');
```

as well as:

```js
import 'package/subpath';
```

Condition selection matters.

For CJS, conditions can include:

```text
node
require
```

and, in current Node's synchronous ESM-loading path, additional conditions can participate depending on loader mode. citeturn215289search1

---

# 120. Conditional Exports Pitfall

Example:

```json
{
  "exports": {
    ".": {
      "import": "./esm.js",
      "require": "./cjs.cjs"
    }
  }
}
```

This is useful.

But now the package has two entry artifacts.

You must test:

```text
API equivalence
state equivalence
error equivalence
```

---

# 121. `exports` Contract and Deep Imports

Without an export map:

```js
require('pkg/lib/internal');
```

might work accidentally.

Once:

```json
"exports": {
  ".": "./index.js"
}
```

exists, that internal path can become unsupported.

This is intentional encapsulation.

---

# 122. CJS `main`

Older packages commonly use:

```json
{
  "main": "./index.js"
}
```

This remains relevant for CommonJS compatibility when `"exports"` is absent or when tools use legacy package metadata.

Modern package architecture should generally define an explicit `"exports"` contract when appropriate.

---

# 123. `main` vs `exports`

Think:

```text
main
  = historical primary entry hint

exports
  = explicit public package boundary
```

`exports` is more expressive because it can define:

- subpaths,
- conditions,
- environment-specific entry points.

---

# 124. Package `imports`

Internal `#` specifiers can be used by CommonJS as well as ESM under Node's package-resolution model.

Example:

```json
{
  "imports": {
    "#db": "./src/db.js"
  }
}
```

Then:

```js
const db = require('#db');
```

where supported by the package/loader configuration.

This enables internal aliases without exposing them as public package paths.

---

# 125. `node_modules` and Monorepos

In monorepos, CJS resolution can be affected by:

- symlinks,
- package-manager layout,
- workspace links,
- package boundaries,
- self references,
- exports maps.

Do not infer resolution from the physical directory tree alone.

---

# 126. Symlinks and Identity

Because module caching and resolution are identity-sensitive, symlinks can produce surprising duplicate instances depending on runtime flags and resolution behavior.

This matters for packages containing:

```text
singleton state
class identity
instance checks
registries
```

---

# 127. `instanceof` Across Duplicate Package Instances

Suppose two copies of a package load:

```js
class Client {}
```

Then:

```text
Client from copy A
```

and:

```text
Client from copy B
```

are distinct constructor identities.

Therefore:

```js
instanceA instanceof ClientFromB
```

can be false even when the source code appears identical.

Module duplication is not merely a memory issue.

---

# 128. Native Addon Interop

Legacy CJS integration may be necessary for:

```text
.node native addons
```

where the current Node ESM loader does not directly support the same import path.

Use explicit compatibility boundaries.

---

# 129. Diagnostics

Useful CommonJS diagnostics:

```js
console.log({
  id: module.id,
  filename: module.filename,
  loaded: module.loaded,
  children: module.children.length,
  paths: module.paths,
});
```

And:

```js
console.log(require.resolve('some-package'));
```

Use these to understand what Node actually loaded.

---

# 130. Production Module Diagnostics

Record when diagnosing startup:

```text
entry point
module format
package type
resolved dependency
loader path
Node version
```

Do not dump an entire environment or secret-bearing configuration merely for module debugging.

---

# 131. Production Performance Model

For CommonJS first load:

```text
resolution
  +
filesystem
  +
parse
  +
execute
  +
cache
```

For cached load:

```text
resolution
  +
cache lookup
```

The exact implementation is Node-version dependent, but the broad distinction is useful for reasoning.

---

# 132. Production Startup Design

Avoid expensive module-level work:

```js
// bad for startup predictability
const data = fs.readFileSync('massive.json');
initializeExpensiveSubsystem(data);
```

Prefer explicit application startup where appropriate:

```js
export async function createApp() {
  const data = await loadData();
  return initializeApp(data);
}
```

This makes lifecycle and module evaluation easier to reason about.

---

# 133. Side Effects

CommonJS modules often execute side effects at `require()` time:

```js
registerMetrics();
startTimer();
patchLibrary();
```

This makes import order operationally significant.

Keep module evaluation side effects small and intentional.

---

# 134. Side-Effect Ordering

This:

```js
require('./polyfill');
require('./app');
```

can intentionally mean:

```text
polyfill side effect
   ↓
app initialization
```

If migrated to ESM, dependency graph semantics can change the way the relationship is expressed.

Prefer explicit imports that communicate the dependency.

---

# 135. CLI Architecture

Traditional CJS:

```js
if (require.main === module) {
  main().catch(...);
}
```

Modern ESM:

```js
if (import.meta.main) {
  await main();
}
```

The actual CLI entry can be a separate thin module:

```text
cli entry
   ↓
application API
```

This reduces test complexity.

---

# 136. Testing CLI Modules

Separate:

```text
program invocation
```

from:

```text
main business logic
```

Then test:

```js
main()
```

directly and perform only a small number of true process-entry tests.

This works well in both CJS and ESM.

---

# 137. CommonJS and Worker Threads

Worker startup can involve:

- `.js`,
- `.cjs`,
- `.mjs`,
- package `"type"`.

The worker's own module classification matters.

Do not assume:

```text
parent loader mode
=
worker module mode
```

without checking the actual worker entry configuration.

---

# 138. CommonJS and Child Processes

Child processes launch independently.

The child gets its own:

```text
module system
working directory
environment
loader state
module cache
```

Do not confuse:

```text
same operating-system process tree
```

with:

```text
same JavaScript module cache
```

---

# 139. Interoperability and Async Context

From Chapter 63:

```text
CJS ↔ ESM
```

does not imply:

```text
automatic distributed context
```

Async context is runtime execution metadata.

Module format is a loading/execution model.

Do not conflate them.

---

# 140. Interoperability and Process Lifecycle

From Chapter 62:

```text
module loading failure
```

can become:

```text
startup failure
```

For production services:

```text
module graph
   ↓
startup initialization
   ↓
readiness
```

Keep module loading and service readiness as separate lifecycle concepts.

---

# 141. Implementation From Scratch

## Stage A — Guided

Create:

```text
math.cjs
app.cjs
```

Use:

```text
module.exports
exports
require
require.resolve
```

Explain every boundary.

---

## Stage B — Partially Guided

Build a CJS package:

```text
src/
  index.cjs
  client.cjs
  errors.cjs
```

with:

```text
public API
private implementation
```

---

## Stage C — No Reference

Build:

```text
ESM application
   ↓
CJS compatibility adapter
   ↓
legacy package
```

using:

```js
createRequire()
```

---

## Stage D — Edge-Case Hardened

Add:

- circular dependency,
- cache inspection,
- cache invalidation test,
- conditional exports,
- dynamic import,
- dual entry points,
- native-addon boundary simulation.

---

## Stage E — Production Grade

Add:

- public API tests,
- CJS consumer tests,
- ESM consumer tests,
- singleton consistency tests,
- package export contract,
- module graph diagnostics,
- startup timing.

---

# 142. Implementation Challenge — CJS Package

Create:

```text
calculator/
├── package.json
├── src/
│   ├── index.cjs
│   ├── add.cjs
│   ├── multiply.cjs
│   └── internal/
│       └── normalize.cjs
└── test/
```

Public API:

```js
const { add, multiply } = require('calculator');
```

Do not expose `internal/`.

---

# 143. Implementation Challenge — ESM Consuming CJS

Create:

```text
legacy-client.cjs
app.js
```

Then consume:

```js
import legacyClient from './legacy-client.cjs';
```

Test both:

```text
default import
namespace import
named import
```

and determine which are actually reliable.

---

# 144. Implementation Challenge — CJS Consuming ESM

Create:

```text
feature.mjs
legacy.cjs
```

Then:

```js
async function main() {
  const feature = await import('./feature.mjs');
  feature.run();
}

main().catch(...);
```

Then create an ESM module with top-level await and observe why synchronous `require()` is inappropriate for that graph.

---

# 145. Implementation Challenge — `createRequire()`

Create an ESM module that loads:

```text
legacy-config.cjs
```

through:

```js
createRequire(import.meta.url)
```

Then explain why the compatibility boundary belongs in one adapter module.

---

# 146. Implementation Challenge — Dual Package

Build:

```text
index.js
index.cjs
```

with conditional exports.

Then:

```text
load from ESM
load from CJS
```

and test whether shared state is duplicated.

---

# 147. Debugging Exercises

## Exercise 1 — Broken `exports`

```js
exports = function run() {};
```

Consumer:

```js
const run = require('./module.cjs');
```

What does `run` actually contain?

---

## Exercise 2 — Partial Cycle

Create:

```text
A → B
B → A
```

and make each module access an export during initialization.

Predict the incomplete state.

---

## Exercise 3 — Resolution Difference

Compare:

```js
require('./utils');
```

with:

```js
import './utils.js';
```

Explain why the first may resolve while the second fails.

---

## Exercise 4 — ESM-only Dependency

A CJS application runs:

```js
const pkg = require('esm-only-package');
```

It fails.

Design the correct bridge.

---

## Exercise 5 — Named CJS Import

A CommonJS package conditionally assigns:

```js
if (condition) {
  exports.feature = feature;
}
```

Can you safely rely on:

```js
import { feature } from 'pkg';
```

Explain the static-analysis issue.

---

## Exercise 6 — Duplicate Singleton

A library has:

```text
dist/index.js
dist/index.cjs
```

Both create a module-level registry.

An ESM consumer and CJS consumer interact with the library in one process.

Find the state consistency bug.

---

## Exercise 7 — Native Addon

An ESM service needs a package that exposes a `.node` native addon through CommonJS.

Design the adapter.

---

## Exercise 8 — Cache Mutation

Write:

```js
delete require.cache[require.resolve('./module.cjs')];
```

Then determine which dependent modules still retain references to old state.

---

# 148. Code Review Exercise

Review:

```js
// index.js
exports = {
  createClient,
};

if (process.env.DEBUG) {
  exports.debug = debug;
}

module.exports.config = require('./config');

setTimeout(() => {
  module.exports.extra = extra;
}, 0);
```

Identify at least 12 issues or questions.

Expected areas:

- `exports` reassignment,
- conditional export detection,
- async export mutation,
- module API shape,
- initialization timing,
- hidden side effects,
- testability,
- caching,
- CJS/ESM interop,
- public contract stability.

---

# 149. Interview Questions

## Foundation

1. What is CommonJS?
2. What is the CommonJS module wrapper?
3. What does `module.exports` mean?
4. What is the `exports` shortcut?
5. Why does `exports = value` not export `value`?
6. What does `require()` return?
7. What is `require.resolve()`?
8. What is `require.cache`?
9. What is `__filename`?
10. What is `__dirname`?

## Intermediate

11. Explain CommonJS module caching.
12. Why can circular dependencies expose partial exports?
13. How does CommonJS resolution differ from ESM resolution?
14. What does `require.main` do?
15. How does package `"exports"` affect CommonJS?
16. Why is `process.cwd()` different from `__dirname`?
17. Why can CommonJS modules maintain singleton state?
18. What does `createRequire()` do?
19. How can CommonJS consume ESM?
20. How can ESM consume CommonJS?

## Advanced

21. Why are ESM named imports from CommonJS not native live bindings?
22. What is Node's static CommonJS named-export detection?
23. What are the constraints on `require(esm)`?
24. Why does top-level await matter?
25. What does the `"module.exports"` ESM export do?
26. What is `__esModule`?
27. Why can dual packages produce duplicate singleton state?
28. How do symlinks affect module identity?
29. Why can manual cache invalidation become graph-inconsistent?
30. Why can a mechanical CJS→ESM conversion fail?

## Principal Level

31. Design a CommonJS → ESM migration for a 5,000-module monorepo.
32. How would you prevent CJS and ESM from creating duplicate state?
33. When should a library remain CommonJS?
34. When should a library become ESM-only?
35. When is dual publishing justified?
36. How would you define compatibility guarantees across both module systems?
37. How would you isolate legacy native addons?
38. How would you test package exports under both loaders?
39. How would you diagnose a module-resolution failure across workspaces?
40. How would you govern module-system usage across a 200-person JavaScript organization?

---

# 150. Predict-the-Output Exercises

## Exercise A

```js
// a.cjs
exports.x = 1;

const b = require('./b.cjs');

exports.y = b.x;
```

```js
// b.cjs
exports.x = 2;

const a = require('./a.cjs');

exports.y = a.x;
```

Predict what each module can observe during initialization.

---

## Exercise B

```js
// module.cjs
exports.a = 1;
exports = {
  b: 2,
};
exports.c = 3;
```

What does the consumer receive?

---

## Exercise C

```js
// module.cjs
module.exports = function run() {};
module.exports.version = '1.0.0';
```

What is the type of:

```js
require('./module.cjs')
```

and how can `version` be accessed?

---

## Exercise D

```js
// module.cjs
console.log('evaluated');
module.exports = {};
```

```js
const a = require('./module.cjs');
const b = require('./module.cjs');
```

How many times is `evaluated` normally printed?

---

## Exercise E

```js
console.log(__dirname === process.cwd());
```

Can this be assumed to be `true`?

Explain.

---

## Exercise F

```js
// legacy.cjs
module.exports = {
  value: 42,
};
```

```js
// app.mjs
import legacy from './legacy.cjs';

console.log(legacy.value);
```

What is the expected interoperability model?

---

## Exercise G

```js
// legacy.cjs
exports.x = 1;

setTimeout(() => {
  exports.x = 2;
}, 0);
```

An ESM module imports:

```js
import { x } from './legacy.cjs';
```

Should you think of `x` as a native ESM live binding?

Why or why not?

---

# 151. Mastery Exercises

## Exercise 1 — CJS Runtime Model

Explain from memory:

```text
require
→ resolve
→ create module
→ cache
→ wrapper
→ execute
→ module.exports
→ return
```

---

## Exercise 2 — Interop Matrix

Build examples for:

```text
ESM → CJS
CJS → ESM via import()
ESM → CJS via createRequire()
CJS → synchronous eligible ESM via require()
```

Document constraints.

---

## Exercise 3 — Dual Package

Publish a local test package with:

```json
{
  "exports": {
    ".": {
      "import": "./index.js",
      "require": "./index.cjs"
    }
  }
}
```

Then prove whether the two entry points share state.

---

## Exercise 4 — Migration Adapter

Create a compatibility adapter around a legacy CJS package and migrate the rest of the application to ESM without exposing `createRequire()` outside that adapter.

---

## Exercise 5 — Circular Dependency Refactor

Build a deliberately broken CJS cycle.

Then refactor to:

```text
A → shared contract ← B
```

and compare reasoning complexity.

---

## Exercise 6 — Package Contract

Use `"exports"` to expose:

```text
.
./errors
```

while keeping:

```text
./internal/*
```

private.

Test both `require` and `import`.

---

# 152. Principal Decision Framework

For every CJS/ESM architecture decision, evaluate:

| Dimension | Question |
|---|---|
| Correctness | Are export shapes identical where promised? |
| Performance | Does loading introduce unacceptable startup cost? |
| Memory | Could dual loaders duplicate state? |
| Security | Are dynamic loading paths controlled? |
| Reliability | Can module loading fail predictably? |
| Maintainability | Is the boundary explicit? |
| Scalability | Does the strategy work across many packages? |
| Observability | Can loader/initialization failures be diagnosed? |
| Developer Experience | Is the module rule easy to teach? |
| Operational Complexity | How many compatibility paths exist? |
| Future Change | Can CJS be removed later without breaking consumers? |

---

# 153. Production Checklist

```text
[ ] CommonJS usage is intentional
[ ] ESM usage is intentional
[ ] package "type" is explicit
[ ] .cjs/.mjs boundaries are documented
[ ] public package exports are explicit
[ ] createRequire() usage is centralized
[ ] dynamic import() usage is justified
[ ] dynamic specifiers are validated
[ ] CJS cycles are tested
[ ] ESM cycles are tested
[ ] CJS/ESM state duplication is tested
[ ] native addons have explicit boundaries
[ ] CJS consumer tests exist
[ ] ESM consumer tests exist
[ ] module resolution is tested
[ ] package export contract is tested
[ ] transpiled output is distinguished from native semantics
[ ] startup behavior is observable
```

---

# 154. Production Scenario

You maintain a large Node platform:

```text
600 CJS packages
180 ESM packages
40 dual packages
5 native addons
```

The goal is:

```text
new code → ESM
```

without breaking production.

A principal strategy:

```text
1. inventory package boundaries
2. classify module systems
3. identify cycles
4. identify singleton/stateful packages
5. identify native addons
6. define export contracts
7. create CJS adapters
8. migrate leaf packages
9. test both loaders
10. monitor runtime failures
11. migrate application roots
12. remove compatibility layers gradually
```

The hardest parts are often not syntax changes.

They are:

```text
state identity
package contracts
initialization timing
resolution differences
tooling
operational risk
```

---

# 155. Migration Completion Criteria

A package is not “migrated” merely because:

```text
require()
```

became:

```text
import
```

A production migration should prove:

```text
[ ] module classification is correct
[ ] resolution is correct
[ ] exports are correct
[ ] cycles are understood
[ ] side effects are understood
[ ] state identity is understood
[ ] tests cover both relevant loaders
[ ] package exports are intentional
[ ] build output is correct
[ ] native dependencies still work
[ ] startup behavior is acceptable
```

---

# 156. Chapter Connections

## Depends On

- Chapter 10 — Scope
- Chapter 12 — Execution Model
- Chapter 17 — Prototypes
- Chapter 28 — Serialization
- Chapter 41 — Spec Architecture
- Chapter 42 — Abstract Operations
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context and Diagnostics
- Chapter 64 — ES Modules

## Builds Toward

- Chapter 66 — `package.json` and Resolution
- Chapter 67 — Dependency Management and Supply Chain
- Chapter 68 — Transpilation and Compilation
- Chapter 69 — Bundlers and Build Systems
- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 80 — Library Authoring
- Chapter 90 — Modern ECMAScript Features
- Chapter 94 — Compatibility Engineering
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-Scale JavaScript Platform

## Related Concepts

- ESM
- package resolution
- package exports
- dependency graphs
- module caches
- dynamic import
- native addons
- transpilation
- bundlers
- package encapsulation

## Why This Chapter Matters Later

JavaScript architecture often fails at boundaries, not within isolated functions.

The CJS/ESM boundary is one of the most important Node.js boundaries because it combines:

```text
language semantics
+
host resolution
+
package contracts
+
runtime cache
+
initialization order
+
legacy compatibility
```

Understanding this boundary deeply prepares you for package engineering, dependency management, build systems, compatibility engineering, and large-scale platform design.

---

# 157. Spaced Retrieval Plan

## Day 0

Explain:

```text
exports
module.exports
require()
cache
wrapper
resolution
```

without notes.

## Day 2

Implement:

```text
CJS package
ESM consumer
```

with a deliberate interop boundary.

## Day 7

Debug:

```text
CJS cycle
```

without copying a solution.

## Day 14

Design:

```text
CJS → ESM migration
```

for a multi-package repository.

## Day 30

Defend:

> When is dual publishing the wrong answer?

---

# 158. Dependency Graph

```text
Node architecture
      │
      ▼
CommonJS runtime
      │
 ┌────┼─────────┐
 ▼    ▼         ▼
wrapper cache resolution
 │       │        │
 └───────┼────────┘
         ▼
    module graph
         │
         ▼
   CJS / ESM boundary
       ┌─┴─┐
       ▼   ▼
  createRequire  import()
       │   │
       └─┬─┘
         ▼
 package interoperability
         │
         ▼
 library / platform architecture
```

---

# 159. Completion Criteria

```text
[ ] Explain the CommonJS wrapper
[ ] Explain module.exports
[ ] Explain exports
[ ] Explain the exports alias trap
[ ] Explain require()
[ ] Explain synchronous loading
[ ] Explain CommonJS resolution
[ ] Explain require.resolve()
[ ] Explain require.cache
[ ] Explain module caching
[ ] Explain circular dependencies
[ ] Explain partial initialization
[ ] Explain __filename
[ ] Explain __dirname
[ ] Explain require.main
[ ] Explain module.children
[ ] Explain CJS module scope
[ ] Explain CJS vs ESM
[ ] Explain ESM importing CJS
[ ] Explain CJS importing ESM
[ ] Explain createRequire()
[ ] Explain dynamic import()
[ ] Explain modern require(esm)
[ ] Explain top-level-await constraints
[ ] Explain "module.exports" interop export
[ ] Explain __esModule
[ ] Explain conditional exports
[ ] Explain dual-package hazards
[ ] Explain native addon boundaries
[ ] Explain transpiled-vs-native differences
[ ] Debug module resolution
[ ] Debug cycles
[ ] Design a migration
[ ] Pass principal interview questions
```

---

# 160. Mastery Gate

You have mastered this chapter only when you can:

### Understand

Explain CommonJS from runtime wrapper through cache and resolution.

### Explain

Teach the CJS/ESM boundary without reducing either module system to syntax differences.

### Predict

Predict cache behavior, circular initialization, export shapes, and interop outcomes.

### Implement

Build CJS, ESM, and compatibility adapters from scratch.

### Debug

Find whether a failure is caused by classification, resolution, export shape, cycle, cache, or loader interoperability.

### Apply

Design production package boundaries and incremental migrations.

### Compare

Defend CJS, ESM, or dual publishing for a specific package.

### Defend

Explain your decision to a principal architecture review with correctness, compatibility, performance, and operational evidence.

---

# 161. Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current chapter status:

```text
[ ] Not Started
```

Reading alone does not mark mastery.

---

# 162. Chapter 65 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain CJS wrapper | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain exports trap | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Predict CJS cycle | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain CJS/ESM interop | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design migration | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. Why does exports = value fail?
2. Why does require() return module.exports?
3. Why does CommonJS cache modules?
4. Why can cycles expose partial exports?
5. How does CJS resolution differ from ESM?
6. Why are CJS named imports not native live bindings?
7. Why does top-level await matter for require(esm)?
8. What does createRequire() actually do?
9. How can dual packages duplicate state?
10. When should a package remain CJS?
```

---

# 163. Chapter 65 — Canonical References and Source Discipline

## Primary Node.js references

- Node.js CommonJS modules  
  https://nodejs.org/api/modules.html
- Node.js ECMAScript modules  
  https://nodejs.org/api/esm.html
- Node.js `node:module` APIs  
  https://nodejs.org/api/module.html
- Node.js packages  
  https://nodejs.org/api/packages.html

The current Node.js CommonJS documentation defines:

- the CommonJS module wrapper,
- `module.exports`,
- the `exports` shortcut,
- `require()`,
- `require.resolve()`,
- CommonJS resolution,
- CommonJS caching,
- circular dependency behavior,
- `require.main`,
- and modern `require(esm)` support. citeturn215289search1

The current ESM documentation defines the interoperability model including:

- importing CommonJS from ESM,
- `createRequire()`,
- differences between ESM and CommonJS,
- the absence of CommonJS wrapper variables in ESM,
- and current interoperability guidance. citeturn215289search0

## ECMAScript reference

- ECMA-262  
  https://tc39.es/ecma262/

Use ECMA-262 for ESM language semantics.

Use Node.js documentation for CommonJS and Node-specific interoperability.

## Source-discipline rules

1. Never describe CommonJS as an ECMAScript language feature.
2. Separate CommonJS behavior from ESM semantics.
3. Verify current Node version behavior before relying on `require(esm)` details.
4. Treat named CJS imports as compatibility behavior, not native ESM bindings.
5. Treat package exports/conditions as host package-resolution semantics.
6. Distinguish native Node behavior from transpiler-generated CJS.
7. Treat cache and module identity as runtime-sensitive.
8. Test both loaders whenever a package claims CJS/ESM interoperability.

---

# 164. Chapter 65 — Completion Snapshot

## CommonJS Core

```text
[ ] Wrapper
[ ] module.exports
[ ] exports
[ ] require
[ ] require.resolve
[ ] require.cache
[ ] require.main
[ ] __filename
[ ] __dirname
```

## Resolution

```text
[ ] Core modules
[ ] Relative paths
[ ] File lookup
[ ] Directory lookup
[ ] node_modules
[ ] Package exports
[ ] Package imports
```

## Interoperability

```text
[ ] ESM imports CJS
[ ] CJS imports ESM
[ ] createRequire
[ ] dynamic import
[ ] require(esm)
[ ] top-level await constraint
[ ] named-export detection
[ ] module.exports interop export
```

## Production

```text
[ ] Dual-package state tested
[ ] Native addon boundary tested
[ ] Migration strategy documented
[ ] Public API contract tested
[ ] Resolution tests added
[ ] Cycle behavior tested
[ ] Loader diagnostics available
```

## Mastery

```text
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# Final Principal Perspective

CommonJS is easy to use and difficult to reason about deeply.

The syntax:

```js
const thing = require('thing');
```

hides an entire runtime process:

```text
specifier
   ↓
CommonJS resolution
   ↓
module identity
   ↓
cache lookup
   ↓
module creation
   ↓
wrapper execution
   ↓
module.exports
   ↓
consumer
```

At the boundary with ESM, another model appears:

```text
CommonJS
   ↓
compatibility rules
   ↓
ESM
```

The hardest production problems are usually not syntax errors.

They are:

```text
duplicate state
incorrect resolution
partial initialization
unexpected export shapes
loader mismatches
native addon boundaries
top-level-await incompatibility
package-contract violations
```

A principal engineer therefore asks:

```text
What module system is this artifact actually using?
How was it classified?
How was the specifier resolved?
What exact value is exported?
When is it evaluated?
What is cached?
Can another path create a second instance?
Is this boundary synchronous or asynchronous?
What does the consumer actually receive?
Is the compatibility behavior a guarantee or a heuristic?
Can this package be migrated without changing state identity?
```

The deepest lesson is:

> **Module interoperability is not a syntax conversion problem. It is a runtime, identity, resolution, and contract problem.**

Once you understand that, CommonJS stops being “legacy syntax” and becomes what it actually is:

> **a major Node.js runtime model that must be understood precisely whenever modern JavaScript meets real production systems.**