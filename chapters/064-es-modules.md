\
# Chapter 64 — ES Modules

> **Curriculum position:** Part XII — Modules / Tooling  
> **Previous chapter:** Chapter 63 — Async Context and Diagnostics  
> **Next chapter:** Chapter 65 — CommonJS and Interoperability  
> **Primary environment:** Modern ECMAScript + Node.js 26.x documentation baseline.

---

# Chapter Mission

Master **ECMAScript Modules (ESM)** as a language-level module system and understand how Node.js implements, resolves, loads, links, evaluates, caches, and interoperates with it.

This chapter is not simply:

```js
import x from './x.js';
export default x;
```

The goal is to understand the complete model:

```text
source code
   ↓
module classification
   ↓
specifier
   ↓
resolution
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
   ↓
live bindings
   ↓
module namespace
   ↓
runtime execution
```

You should leave this chapter able to explain:

- why ESM exists,
- how static imports differ from dynamic imports,
- why imports are live bindings,
- why import order is dependency-graph driven,
- why circular dependencies can work but still fail,
- why relative ESM imports commonly require file extensions in Node.js,
- what `package.json` `"type"` changes,
- what `exports` and `imports` contribute to package design,
- what `import.meta` represents,
- what top-level `await` changes,
- how module URLs affect identity,
- how ESM differs fundamentally from CommonJS,
- how ESM interoperates with CommonJS,
- how to migrate a production package safely,
- and how a principal engineer should reason about module boundaries.

---

# 1. Learning Objectives

By the end of this chapter you should be able to:

## Theory

- Define an ECMAScript module.
- Explain static `import` and `export`.
- Explain module records and module graphs.
- Explain resolution versus loading versus linking versus evaluation.
- Explain live bindings.
- Explain module namespace objects.
- Explain default exports versus named exports.
- Explain re-exports.
- Explain side-effect-only imports.
- Explain static import restrictions.
- Explain dynamic `import()`.
- Explain top-level `await`.
- Explain circular dependency behavior.
- Explain `import.meta`.
- Explain ESM module identity and URL-based resolution.

## Node.js

- Explain `.mjs`, `.cjs`, and `package.json` `"type"`.
- Explain Node's ESM specifier rules.
- Explain `node:` built-in specifiers.
- Explain package bare specifiers.
- Explain package `"exports"` and `"imports"`.
- Explain `import.meta.url`.
- Explain `import.meta.dirname`, `import.meta.filename`, and `import.meta.main` as current Node facilities.
- Explain `import.meta.resolve()`.
- Explain JSON import attributes.
- Explain ESM/CJS interoperability.
- Explain why `require()` and `import` do not use identical resolution algorithms.

## Implementation

- Design a multi-module application.
- Implement dependency boundaries with named exports.
- Build a package using conditional exports.
- Create a deliberate circular dependency and diagnose it.
- Convert CommonJS code to ESM.
- Use dynamic import for optional functionality.
- Use top-level await deliberately.
- Write ESM code that works predictably in Node and browser-oriented tooling.

## Principal judgment

- Decide when ESM is beneficial.
- Decide how much package encapsulation to enforce.
- Decide between static and dynamic loading.
- Decide how to structure package exports.
- Detect module-boundary smells.
- Evaluate migration risk from CommonJS to ESM.
- Explain module-system choices during architecture review.

---

# 2. Prerequisites

Strongly recommended:

- Chapter 5 — Variables and Declarations
- Chapter 9 — Functions and First-Class Behavior
- Chapter 10 — Scope and Lexical Environments
- Chapter 12 — Execution Contexts
- Chapter 15 — Objects and Property Semantics
- Chapter 17 — Prototypes
- Chapter 28 — JSON and Serialization
- Chapter 31–36 — Asynchronous JavaScript
- Chapter 41 — Specification Architecture
- Chapter 42 — Abstract Operations
- Chapter 58 — Node.js Architecture
- Chapter 59 — Node Core APIs
- Chapter 63 — Async Context and Diagnostics

---

# 3. What Is It?

An ECMAScript module is a JavaScript source unit with module semantics.

Unlike a legacy script, a module has:

- its own top-level lexical environment,
- explicit import/export declarations,
- a module dependency graph,
- defined module-linking semantics,
- strict mode by default,
- and module-specific host integration.

Example:

```js
// math.js
export function add(a, b) {
  return a + b;
}
```

```js
// app.js
import { add } from './math.js';

console.log(add(2, 3));
```

Conceptually:

```text
app.js
  │
  └── imports → math.js
```

The module system makes that relationship explicit.

---

# 4. Why Does It Exist?

Before standardized modules, JavaScript applications often relied on:

- globals,
- immediately invoked function expressions,
- script ordering,
- CommonJS,
- AMD,
- bundler-specific mechanisms,
- custom loaders.

ESM provides a standardized dependency model.

The core benefits include:

- explicit dependencies,
- explicit public exports,
- static analyzability,
- predictable graph construction,
- live bindings,
- composability,
- browser compatibility,
- interoperability with tooling.

The central architectural shift is:

```text
implicit shared global namespace
          ↓
explicit dependency graph
```

---

# 5. Mental Model

Think of ESM as a **graph**, not a collection of files.

```text
                    ┌───────────┐
                    │  app.js   │
                    └─────┬─────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
           config.js   server.js   logging.js
                          │
                          ▼
                       db.js
```

The system must determine:

1. What does each specifier mean?
2. Which module is being referenced?
3. What is its module record?
4. What dependencies does it have?
5. Can the graph be linked?
6. In what order should evaluation happen?
7. What bindings are exported/imported?
8. Which bindings are live?

---

# 6. Static Imports and Dynamic Imports

## Static

```js
import { add } from './math.js';
```

The syntax is part of the module grammar and participates in static module linking.

## Dynamic

```js
const module = await import('./math.js');
```

Dynamic import is an expression that performs asynchronous module loading.

### Architectural difference

```text
static import
  ↓
known dependency graph edge

dynamic import
  ↓
runtime-controlled loading edge
```

---

# 7. Core Rules

## Rule 1 — ESM is not CommonJS with different punctuation

The execution and loading models differ.

## Rule 2 — Imports are bindings

An imported binding reflects the exporting module's binding.

## Rule 3 — ESM modules are evaluated as a graph

Import order is not simply:

```text
top-to-bottom files
```

## Rule 4 — Relative Node ESM imports are fully specified

Typical Node ESM:

```js
import './config.js';
```

rather than:

```js
import './config';
```

Node's current documentation states that relative and absolute ESM specifiers require file extensions, and directory indexes must be fully specified. citeturn279903search0

## Rule 5 — `import` resolves through the ESM loader

`require()` and `import` have distinct Node resolution paths. citeturn279903search1turn279903search2

## Rule 6 — A module has one logical identity per resolved module URL

Different URLs can represent distinct module identities even when they point toward similar source content.

## Rule 7 — Top-level await can make the module graph asynchronous

A module using top-level `await` affects evaluation of dependent modules.

---

# 8. Syntax

## Named export

```js
export const port = 3000;
```

## Named import

```js
import { port } from './config.js';
```

## Export declaration

```js
const port = 3000;

export { port };
```

## Renaming

```js
import { port as serverPort } from './config.js';
```

## Export renaming

```js
export { port as serverPort };
```

## Default export

```js
export default function createServer() {}
```

## Default import

```js
import createServer from './server.js';
```

## Namespace import

```js
import * as math from './math.js';
```

## Side-effect-only import

```js
import './telemetry.js';
```

## Re-export

```js
export { add } from './math.js';
```

## Re-export namespace

```js
export * as math from './math.js';
```

## Dynamic import

```js
const module = await import('./plugin.js');
```

---

# 9. Named Exports

Prefer named exports when the module exposes multiple concepts:

```js
export function encode(value) {
  // ...
}

export function decode(value) {
  // ...
}
```

Consumers:

```js
import { encode, decode } from './codec.js';
```

Named exports make the contract explicit.

---

# 10. Default Exports

Example:

```js
export default class UserRepository {}
```

Consumer:

```js
import UserRepository from './user-repository.js';
```

The importing name is chosen by the importer.

That is different from named imports:

```js
import { UserRepository } from './user-repository.js';
```

where the exported binding name matters.

### Design consideration

Do not use default exports purely because they are shorter.

Choose based on:

- API clarity,
- naming consistency,
- package conventions,
- tooling,
- refactoring behavior.

---

# 11. Live Bindings

This is one of the most important ESM concepts.

```js
// counter.js
export let count = 0;

export function increment() {
  count++;
}
```

```js
// app.js
import { count, increment } from './counter.js';

console.log(count);
increment();
console.log(count);
```

Conceptually:

```text
imported count
      │
      ▼
binding in counter module
```

It is not simply:

```text
copy number 0 into app.js
```

The import refers to the exporting module's binding.

---

# 12. Read-Only Imported Binding

You cannot assign to an imported binding:

```js
import { count } from './counter.js';

count = 100; // SyntaxError
```

The consumer observes the binding but does not own reassignment of that imported binding.

The exporting module controls its own binding.

---

# 13. Mental Model: Reference to a Binding

Do not think:

```text
export = copy value
import = copy value
```

Prefer:

```text
export
  ↓
exposes binding

import
  ↓
creates local reference to exported binding
```

This is why circular dependencies can work at the graph/linking level even when evaluation order creates temporal hazards.

---

# 14. Module Namespace Objects

Consider:

```js
import * as math from './math.js';
```

`math` is a module namespace object.

It provides access to exports of the target module.

Example:

```js
console.log(Object.keys(math));
```

Do not treat it as a normal mutable object containing copied exports.

It represents the module's exported interface.

---

# 15. Re-Exports

Barrel module:

```js
export { createUser } from './user.js';
export { createOrder } from './order.js';
```

Consumer:

```js
import { createUser, createOrder } from './domain/index.js';
```

Useful for:

- public API organization,
- package entry points,
- controlled exposure.

Potential costs:

- dependency graph complexity,
- accidental broad exposure,
- harder tracing of ownership,
- circular dependencies through barrels.

Use barrels intentionally.

---

# 16. Side-Effect Imports

```js
import './register-metrics.js';
```

This means:

> load and evaluate the module for its side effects.

Typical examples:

- polyfill initialization,
- instrumentation registration,
- custom element registration,
- global configuration.

Side-effect modules should be rare and carefully documented.

---

# 17. Execution Walkthrough

Suppose:

```text
app.js
  imports a.js

a.js
  imports b.js

b.js
```

Conceptual process:

```text
1. resolve app.js
2. create app module record
3. discover a.js
4. resolve a.js
5. discover b.js
6. resolve b.js
7. build dependency graph
8. link bindings
9. instantiate environments
10. evaluate modules in dependency-aware order
11. execute application body
```

This is much closer to the ECMAScript specification model than:

```text
read files one by one
execute immediately
```

---

# 18. Resolution

Resolution answers:

> What module does this specifier identify?

Examples:

```js
'node:fs/promises'
'./config.js'
'../util.js'
'lodash'
'#internal-config'
```

Node's current ESM documentation distinguishes:

- relative specifiers,
- bare specifiers,
- absolute specifiers.

Relative and absolute specifiers use URL-style resolution semantics, while bare specifiers go through package resolution. citeturn279903search0

---

# 19. Relative Specifiers

```js
import './startup.js';
```

The `./` means:

```text
relative to the importing module
```

In Node ESM:

```js
import './startup';
```

is generally incomplete because the extension is required for relative file imports. citeturn279903search0

---

# 20. Bare Specifiers

```js
import express from 'express';
```

`express` is a bare specifier.

Node performs package resolution.

Another example:

```js
import 'some-package/feature';
```

Whether that subpath is allowed can depend on the target package's `"exports"` map.

---

# 21. `node:` Specifiers

Use:

```js
import { readFile } from 'node:fs/promises';
```

This explicitly identifies a Node built-in.

Advantages include:

- clearer intent,
- no package-name ambiguity,
- easy distinction from userland dependencies.

---

# 22. URL-Based Module Identity

Node ESM resolves and caches modules as URLs.

For example:

```js
import './foo.mjs?query=1';
```

and:

```js
import './foo.mjs?query=2';
```

can be loaded as distinct module URLs. Node's documentation explicitly describes differing query strings or fragments as producing multiple loads. citeturn279903search0

### Principal insight

> Module identity is not simply “filesystem path string.”

This matters for:

- caching,
- loaders,
- testing,
- dynamic imports,
- virtual modules.

---

# 23. `import.meta`

Inside an ES module:

```js
console.log(import.meta);
```

Node supplies module metadata.

Common properties include:

```js
import.meta.url
```

and, in current Node versions:

```js
import.meta.dirname
import.meta.filename
import.meta.main
```

Node documents `import.meta.dirname` and `import.meta.filename` as stabilized in modern releases, with `import.meta.main` available for detecting whether the module is the application's entry point. citeturn279903search0

---

# 24. `import.meta.url`

Example:

```js
console.log(import.meta.url);
```

Typically:

```text
file:///app/src/index.js
```

Use it when a module-relative URL is needed.

Example:

```js
const url = new URL('./data/config.json', import.meta.url);
```

Then convert to filesystem paths where necessary.

---

# 25. `import.meta.dirname`

Current Node versions expose:

```js
console.log(import.meta.dirname);
```

It provides the module directory for supported `file:` modules. citeturn279903search0

This reduces the need for a common legacy pattern based on:

```js
__dirname
```

which is not an ESM local binding.

---

# 26. `import.meta.filename`

Current Node supports:

```js
console.log(import.meta.filename);
```

for supported file-backed modules.

It corresponds conceptually to the fully resolved file path for the current module. Node documents caveats around the `file:` scheme. citeturn279903search0

---

# 27. `import.meta.main`

A modern Node entry-point pattern:

```js
if (import.meta.main) {
  main();
}
```

Conceptually this asks:

> Is this module the program's main entry module?

Node documents this as an analogue to the CommonJS entry-point test based on `require.main === module`. citeturn279903search0

---

# 28. `import.meta.resolve()`

Current Node exposes:

```js
const resolved = import.meta.resolve('./config.js');
```

It returns the absolute URL string for the specifier relative to the current module.

Node notes that it can involve synchronous filesystem work and can therefore have performance implications similar to `require.resolve()`. citeturn279903search0

Do not repeatedly resolve paths in hot loops without measuring.

---

# 29. Package `"type"`

A package can declare:

```json
{
  "type": "module"
}
```

Then `.js` files in that package scope are interpreted as ES modules.

Alternatively:

```json
{
  "type": "commonjs"
}
```

makes `.js` files CommonJS by default in that package scope.

Node also supports explicit:

```text
.mjs → ESM
.cjs → CommonJS
```

Node's current package documentation recommends being explicit about the `"type"` field because it helps tools and loaders determine interpretation. citeturn279903search1

---

# 30. `.mjs` and `.cjs`

These extensions are explicit signals.

```text
.mjs → ESM
.cjs → CommonJS
```

This can be useful when a project temporarily contains both module systems.

Example:

```text
package.json
src/
  app.js
  legacy.cjs
  plugin.mjs
```

---

# 31. Ambiguous `.js`

Modern Node can inspect explicit markers and, in some situations, syntax when no explicit marker determines the system.

Do not build production architecture around heuristic detection.

Prefer:

```json
{
  "type": "module"
}
```

or explicit extensions.

Node's documentation explains the classification rules and recommends explicit markers. citeturn279903search0turn279903search1turn279903search2

---

# 32. Importing JSON

Modern ESM JSON imports use an import attribute:

```js
import config from './config.json' with { type: 'json' };
```

Node's current ESM documentation states that JSON modules require the import type attribute. citeturn279903search1

Do not confuse this with older experimental import-assertion syntax.

---

# 33. Dynamic Import

```js
const module = await import('./feature.js');
```

The expression returns a Promise.

This is useful for:

- optional features,
- lazy loading,
- plugin systems,
- environment-dependent modules,
- code that should not be loaded until needed.

It is not equivalent to:

```js
require('./feature.js');
```

even when both appear to “load a module.”

---

# 34. Dynamic Import from CommonJS

Modern Node permits dynamic `import()` in CommonJS for loading ESM.

Conceptually:

```text
CommonJS
   │
   └── import()
           ↓
         ESM
```

The direction is important for migration architecture.

Node's current documentation explicitly states that dynamic `import()` is supported in both CommonJS and ES modules. citeturn279903search0

---

# 35. Top-Level Await

ESM permits:

```js
const config = await loadConfig();
```

at module top level.

Example:

```js
const config = await loadRemoteConfig();

export { config };
```

This can simplify initialization.

But it also changes the module graph's evaluation behavior.

---

# 36. Top-Level Await Trade-Off

Useful:

```text
configuration
credential loading
startup initialization
```

Dangerous when:

```text
module A waits forever
    ↓
dependent modules cannot finish evaluation
```

A module graph can therefore inherit asynchronous startup latency.

Use timeouts and explicit initialization policy for external dependencies.

---

# 37. Circular Dependencies

Consider:

```text
a.js → b.js
b.js → a.js
```

ESM supports cycles at the graph level.

But evaluation can still fail if a binding is accessed before initialization.

Example:

```js
// a.js
import { valueB } from './b.js';

export const valueA = valueB + 1;
```

```js
// b.js
import { valueA } from './a.js';

export const valueB = valueA + 1;
```

This can trigger a temporal dead zone / uninitialized-binding failure during evaluation.

### Lesson

> Circular graph support does not mean circular initialization is safe.

---

# 38. Stronger Circular Example

A safer cycle can sometimes work when access happens only after initialization:

```js
// a.js
import { getB } from './b.js';

export function getA() {
  return 'A' + getB();
}
```

```js
// b.js
import { getA } from './a.js';

export function getB() {
  return 'B';
}
```

The graph is cyclic, but actual evaluation-time use can be deferred.

Still, cycles increase reasoning cost.

---

# 39. Circular Dependency Rule

At architecture level:

```text
cycle in runtime graph
    ≠
automatic bug

but

cycle
    = higher cognitive + initialization risk
```

Prefer a directed acyclic dependency graph unless a cycle is genuinely justified.

---

# 40. Strict Mode

ES modules are always strict mode.

Therefore:

```js
x = 10;
```

without declaration will fail rather than create an accidental global.

This eliminates a broad class of sloppy-mode behavior.

---

# 41. Top-Level `this`

In an ES module:

```js
console.log(this);
```

at top level does not behave like CommonJS's module wrapper `this`.

Do not port CommonJS assumptions blindly.

---

# 42. No CommonJS Wrapper Variables

ESM does not provide the traditional:

```js
require
module
exports
__filename
__dirname
```

as local CommonJS module bindings.

Use ESM-native facilities such as:

```js
import ...
export ...
import.meta.url
import.meta.filename
import.meta.dirname
```

where supported.

---

# 43. Module Scope

Top-level variables are module-scoped:

```js
const secret = 'internal';

export function run() {}
```

Another module cannot access:

```js
secret
```

unless it is exported.

This creates a strong encapsulation boundary.

---

# 44. Module Environment vs Global Scope

```js
// module-a.js
const value = 10;
```

does not imply:

```js
globalThis.value === 10
```

Modules reduce accidental global coupling.

This is one reason ESM is much safer for large applications than informal script concatenation.

---

# 45. Tree Shaking

ESM's statically analyzable structure is highly useful to bundlers.

For:

```js
// math.js
export function add() {}
export function multiply() {}
```

and:

```js
import { add } from './math.js';
```

a bundler may determine that `multiply` is unused.

This can enable dead-code elimination.

Important:

> Tree shaking is a bundler optimization, not a magical runtime behavior of Node's ESM loader.

---

# 46. Side Effects and Tree Shaking

This can inhibit elimination:

```js
export function add() {
  console.log('side effect');
}
```

More importantly:

```js
// module.js
registerGlobalPlugin();
```

has module evaluation side effects.

Bundlers may require package metadata such as:

```json
{
  "sideEffects": false
}
```

but this is a bundler contract, not an ECMAScript language feature.

Never declare side-effect freedom unless it is actually true.

---

# 47. Static Analysis

Static ESM syntax enables tooling to answer questions such as:

```text
What does this module import?
What does it export?
Which dependency graph edges exist?
Which exports are unused?
Where are cycles?
```

This powers:

- IDEs,
- linters,
- bundlers,
- dependency analyzers,
- code search,
- refactoring tools.

---

# 48. Export Surface Design

Bad package API:

```text
export everything
```

Better:

```text
public API
   ↓
small stable surface
   ↓
private internals
```

Use package `"exports"` to enforce boundaries.

---

# 49. Package `"exports"`

Example:

```json
{
  "name": "my-package",
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js"
  }
}
```

Consumers can import:

```js
import pkg from 'my-package';
```

and:

```js
import { SomeError } from 'my-package/errors';
```

but not arbitrary internal files outside the allowed export map.

Node's packages documentation describes `"exports"` as defining package entry points and subpaths. citeturn279903search1

---

# 50. Conditional Exports

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

This can support both module systems.

Node documents built-in conditions including `"import"` and `"require"`; these conditions are mutually exclusive based on how the package is loaded. citeturn279903search1

---

# 51. `"imports"`

A package can define internal aliases:

```json
{
  "imports": {
    "#config": "./src/config.js",
    "#db": "./src/db/index.js"
  }
}
```

Then:

```js
import config from '#config';
```

The `#` prefix makes these package-internal specifiers.

This can give you explicit internal module boundaries without relying on fragile relative-path depth.

---

# 52. Why Package Encapsulation Matters

Without a controlled export map:

```text
consumer
  ↓
private/internal/file.js
```

Then you cannot safely reorganize internals.

With `"exports"`:

```text
consumer
  ↓
public contract
  ↓
internal implementation
```

This makes refactoring safer.

---

# 53. Package Design Example

```text
my-library/
├── package.json
├── src/
│   ├── index.js
│   ├── parser.js
│   ├── formatter.js
│   └── internal/
│       └── token.js
└── test/
```

Public:

```json
{
  "exports": {
    ".": "./src/index.js"
  }
}
```

The internal token module remains implementation detail.

---

# 54. CommonJS Interoperability Preview

A later chapter covers this in depth.

For now:

```text
ESM → CommonJS
```

is supported through ESM import mechanisms.

Example:

```js
import pkg from './legacy.cjs';
```

The CommonJS module's `module.exports` is provided as the default export value, with named-export detection available as a compatibility convenience. Node documents this behavior. citeturn279903search0

---

# 55. `require()` from ESM

ESM does not provide a native local `require`.

When required, Node provides:

```js
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);
```

Then:

```js
const legacy = require('./legacy.cjs');
```

Use this as an interoperability bridge, not as the default ESM style.

---

# 56. `require()` and ESM

Current Node supports `require()` loading synchronous ES modules under specific constraints; an ESM graph containing top-level `await` cannot be synchronously loaded through that path. Node's current documentation explicitly describes this limitation. citeturn279903search0turn279903search1

This is why:

```text
CJS → ESM
```

often uses:

```js
await import(...)
```

rather than trying to force synchronous semantics onto an asynchronous module graph.

---

# 57. Module Resolution vs Loader

Node's ESM architecture separates concepts:

```text
specifier
   ↓
resolution
   ↓
URL + format
   ↓
loading
   ↓
source/module
   ↓
evaluation
```

The default resolver is URL-oriented and differs from CommonJS extension/folder lookup. Node documents that it does not perform extension searching for ESM relative/absolute file specifiers and does not use directory indexes implicitly. citeturn279903search0turn279903search1

---

# 58. No Automatic Extension Searching

CommonJS historically permits patterns such as:

```js
require('./utils');
```

which may resolve through extension lookup.

Node ESM expects:

```js
import './utils.js';
```

This improves explicitness and aligns well with browser module behavior.

---

# 59. No Implicit Directory Index

Do not assume:

```js
import './routes';
```

will resolve to:

```text
routes/index.js
```

under the default ESM resolution model.

Write:

```js
import './routes/index.js';
```

or expose the desired package path explicitly.

---

# 60. Browser Compatibility

ESM is a web standard.

Example:

```html
<script type="module" src="/src/main.js"></script>
```

Browser and Node module systems are not identical hosts, but the core ESM language model is standardized.

Node adds:

- package resolution,
- `node:` built-ins,
- package exports/imports,
- filesystem URLs,
- Node-specific `import.meta` properties,
- loader customization.

---

# 61. Node-Specific vs Standardized

| Feature | Standard ESM | Node-specific |
|---|---:|---:|
| `import` | ✅ | |
| `export` | ✅ | |
| dynamic `import()` | ✅ | |
| top-level await | ✅ | |
| `import.meta` base concept | ✅ | |
| `import.meta.url` | host-dependent | ✅ behavior |
| `import.meta.dirname` | | ✅ |
| `import.meta.filename` | | ✅ |
| `import.meta.main` | | ✅ |
| `node:` | | ✅ |
| package `"exports"` | | ✅ |
| package `"imports"` | | ✅ |
| CommonJS interop | | ✅ |

---

# 62. Specification Semantics

The ECMAScript specification defines modules using concepts such as:

- ParseModule
- Source Text Module Records
- imported/exported names
- module environments
- linking
- instantiation
- evaluation

At a conceptual level:

```text
Parse
 ↓
Create module record
 ↓
Link dependency graph
 ↓
Instantiate
 ↓
Evaluate
```

This is more precise than:

```text
import = execute another file
```

---

# 63. Module Records

A module record represents the specification-level information needed to process a module.

Conceptually it contains knowledge of:

- requested modules,
- import entries,
- local export entries,
- indirect export entries,
- namespace export behavior,
- evaluation state.

Do not equate the spec abstraction directly with a concrete Node JavaScript object.

---

# 64. Linking

Linking resolves relationships between:

```text
imported binding
      ↕
exported binding
```

This is where the dependency graph's binding relationships are established.

The key benefit:

> consumers reference exports rather than receiving arbitrary copies.

---

# 65. Instantiation

Instantiation creates the necessary execution environments and bindings.

For lexical declarations:

```js
const token = ...
```

the binding exists conceptually before its initialization during evaluation.

This is one reason cycles can observe:

```text
binding exists
but value not initialized
```

and trigger a TDZ-like failure.

---

# 66. Evaluation

Evaluation executes module code.

For:

```js
console.log('module evaluated');
```

that output belongs to the evaluation phase.

Importing a module may therefore trigger side effects once evaluation happens for that module identity.

---

# 67. Module Evaluation Order

For:

```text
A
↓
B
↓
C
```

dependencies must be linked and evaluated according to the module graph semantics.

A useful mental model:

```text
dependencies first
then dependent code
```

but do not reduce the formal semantics to a simple depth-first file order, especially with cycles and top-level await.

---

# 68. Live Binding Example

```js
// state.js
export let state = 'idle';

export function activate() {
  state = 'active';
}
```

```js
// app.js
import { state, activate } from './state.js';

console.log(state);
activate();
console.log(state);
```

Expected:

```text
idle
active
```

The consumer observes the export binding after mutation by the exporting module.

---

# 69. Reassignment Direction

This works:

```js
// state.js
export let state = 'idle';

export function activate() {
  state = 'active';
}
```

This does not:

```js
// app.js
import { state } from './state.js';

state = 'active';
```

The exported binding is mutable by its owner, while the imported binding is read-only from the consumer.

---

# 70. Namespace Objects and Identity

For:

```js
import * as ns from './module.js';
```

the namespace exposes exports in a stable module-oriented structure.

Do not use namespace objects as ad hoc mutable bags:

```js
ns.newValue = ...
```

The module namespace is not a normal application state object.

---

# 71. Top-Level Await and Dependency Blocking

Suppose:

```js
// config.js
export const config = await loadConfig();
```

and:

```js
// app.js
import { config } from './config.js';
console.log(config);
```

`app.js` evaluation depends on the asynchronous completion of `config.js`.

This can be valuable for startup but makes module evaluation part of your initialization protocol.

---

# 72. Top-Level Await Failure

If a top-level await rejects:

```js
export const data = await fetchData();
```

and the Promise rejects, the module graph's evaluation can fail.

Therefore:

```text
module initialization
    ↓
is a failure boundary
```

Treat startup errors explicitly.

---

# 73. Dynamic Import Failure

```js
try {
  const module = await import('./optional-feature.js');
} catch (error) {
  // handle unavailable feature
}
```

This is useful when absence is an expected runtime condition.

Do not catch every import error and silently continue; distinguish:

```text
optional feature absent
```

from:

```text
feature exists but is broken
```

---

# 74. Caching Model

Modules are generally evaluated once per module identity within a loader/cache context.

This means:

```js
import './config.js';
import './config.js';
```

does not normally imply:

```text
evaluate config twice
```

But differing URL identities can result in separate module loads.

Node explicitly documents query/fragment differences as creating multiple module loads for ESM URL identities. citeturn279903search0

---

# 75. Module State

A module can hold state:

```js
let cache = new Map();

export function set(key, value) {
  cache.set(key, value);
}
```

Consumers share the same module instance for that module identity.

This makes modules natural singleton-like scopes.

But:

> module-local state is not automatically a good global state architecture.

---

# 76. Module Singleton Hazards

If a module stores mutable application state:

```js
export const sessions = new Map();
```

then all importers may observe the same state.

This can be useful for:

- registries,
- caches,
- configuration snapshots.

But dangerous for:

- per-request state,
- per-user state,
- tenant-specific mutable data.

Connect this concept to Chapter 63's async-context isolation.

---

# 77. Static Import Hoisting Model

Imports are not ordinary executable statements.

You cannot conditionally write:

```js
if (condition) {
  import { x } from './x.js';
}
```

Use dynamic import:

```js
if (condition) {
  const { x } = await import('./x.js');
}
```

This is an important distinction between:

```text
graph declaration
```

and:

```text
runtime loading
```

---

# 78. Export Hoisting Model

This:

```js
export function add() {}
```

declares an export binding.

The module system knows the export structure independently of the function being called by another module.

This supports static tooling and linking.

---

# 79. Re-Export Chains

```text
app
 ↓
index
 ↓
domain
 ↓
implementation
```

A public export can therefore traverse multiple modules.

Avoid chains so deep that ownership becomes unclear.

---

# 80. Barrels: Benefits and Risks

## Benefits

- clean external API,
- fewer deep import paths,
- centralized exports.

## Risks

- accidental cycles,
- eager side effects,
- broader evaluation,
- ambiguous ownership,
- larger dependency fan-in.

Use them at deliberate boundaries.

---

# 81. Package Exports as Architecture

A package can expose:

```text
.
./client
./server
./testing
```

while keeping:

```text
./internal/*
```

private.

This turns module resolution into an architectural policy mechanism.

---

# 82. Conditional Package API

Example:

```json
{
  "exports": {
    ".": {
      "node": "./dist/node.js",
      "default": "./dist/browser.js"
    }
  }
}
```

Condition ordering and supported conditions matter.

Do not create a large conditional matrix without testing every supported environment.

---

# 83. Dual-Publishing Risks

Supporting both:

```text
ESM
CJS
```

can create tricky edge cases:

- two module instances,
- differing `this`,
- different resolution,
- duplicated singleton state,
- different dependency trees,
- confusing stack traces.

A library should have a deliberate dual-package strategy rather than “build everything twice.”

---

# 84. Dual Package Hazard

A consumer can potentially load:

```text
ESM version
+
CJS version
```

of logically the same package.

If each copy owns singleton state:

```js
const registry = new Map();
```

the application may unexpectedly have:

```text
registry A
registry B
```

instead of one shared registry.

This is a serious library-architecture concern.

---

# 85. Dynamic Loading Architecture

Plugin systems often use:

```js
const plugin = await import(pluginPath);
```

Then validate:

```js
if (typeof plugin.start !== 'function') {
  throw new TypeError('Invalid plugin');
}
```

Never treat dynamically imported code as automatically trustworthy.

---

# 86. Security Considerations

## 86.1 Dynamic specifiers

Avoid:

```js
await import(userInput);
```

unless the input is tightly validated.

This can create arbitrary module-loading behavior.

---

## 86.2 Package supply chain

ESM package graphs depend on package resolution.

A compromised dependency can execute during module evaluation.

Treat:

```text
dependency
→ module evaluation
→ code execution
```

as a security path.

---

## 86.3 Side-effect imports

A module may execute immediately simply because it was imported.

Audit side-effect modules carefully.

---

# 87. Path Traversal and Dynamic Imports

Bad:

```js
await import(`./plugins/${userSuppliedName}.js`);
```

depending on validation, this can produce unexpected module targets.

Prefer an allowlist:

```js
const plugins = {
  email: './plugins/email.js',
  sms: './plugins/sms.js',
};

const specifier = plugins[name];

if (!specifier) {
  throw new Error('Unknown plugin');
}

await import(specifier);
```

---

# 88. Import Attributes and Security

Import attributes can make resource interpretation explicit.

For JSON:

```js
import data from './data.json' with { type: 'json' };
```

Explicit type expectations can reduce ambiguity and align with loader semantics. citeturn279903search0turn279903search1

---

# 89. Performance Considerations

## Static imports

Strengths:

- predictable graph,
- tooling-friendly,
- easy bundler analysis.

Costs:

- dependency loading occurs as part of module graph initialization.

## Dynamic imports

Strengths:

- lazy loading,
- optional dependencies,
- plugin architecture.

Costs:

- async boundary,
- runtime resolution,
- possible latency,
- more complex failure handling.

Choose based on application behavior.

---

# 90. `import.meta.resolve()` Performance

Because Node documents potential synchronous filesystem work in `import.meta.resolve()`, avoid placing it repeatedly in hot paths without evidence that it is acceptable. citeturn279903search0

Cache resolved values if they are stable and performance-sensitive.

---

# 91. Memory Considerations

Module instances can remain live for the lifetime of their loader/runtime context.

Module-level caches therefore have process-wide consequences:

```js
const cache = new Map();
```

If entries are never evicted:

```text
module lifetime
  +
unbounded cache
  =
memory growth
```

Use explicit cache policy.

---

# 92. Testing Considerations

ESM changes test architecture because module loading is graph-based and cached.

Test concerns include:

- module isolation,
- cache reuse,
- dynamic import,
- top-level await,
- ESM/CJS boundaries,
- environment-specific exports.

Avoid tests that rely on undocumented module-cache mutation unless your test framework intentionally supports it.

---

# 93. Debugging Module Resolution

When an import fails:

```text
Cannot find module
```

check in this order:

```text
1. Is the file actually present?
2. Is the specifier relative/bare/absolute?
3. Is the extension present?
4. What package "type" applies?
5. Does package "exports" block the path?
6. Is the package installed where Node searches?
7. Is this ESM or CJS resolution?
8. Is a custom loader involved?
9. Is the URL encoded correctly?
10. Is the imported file itself failing during evaluation?
```

Do not confuse:

```text
resolution failure
```

with:

```text
module evaluation failure
```

---

# 94. Debugging `ERR_MODULE_NOT_FOUND`

Example:

```js
import './utils';
```

Possible correction:

```js
import './utils.js';
```

But do not automatically append `.js` if the actual target is:

```text
utils.mjs
```

or a package export.

Resolve the actual module identity first.

---

# 95. Debugging Package `"exports"`

If:

```js
import x from 'pkg/internal.js';
```

fails despite:

```text
node_modules/pkg/internal.js
```

existing, check:

```json
"exports": {
  ".": "./index.js"
}
```

The file may physically exist but be intentionally inaccessible through the package interface.

This is a feature, not necessarily an error.

---

# 96. Debugging Circular Dependencies

Use a graph:

```text
A → B
B → C
C → A
```

Then identify:

```text
which binding
which module
which evaluation phase
which first access
```

A cycle is easier to debug when modeled as:

```text
graph
+
binding initialization
+
evaluation order
```

rather than as a stack trace alone.

---

# 97. Debugging Top-Level Await

Symptoms:

```text
application startup hangs
```

Investigate:

```text
entry
 ↓
module A
 ↓
module B
 ↓
top-level await
 ↓
external dependency
```

Add:

- initialization timing,
- explicit timeouts,
- failure logs,
- module-level boundaries.

Do not put an unbounded network dependency into a critical module import path.

---

# 98. Common Misconceptions

## Misconception 1

> ESM is just syntax for CommonJS.

False.

## Misconception 2

> `import` copies the exported value.

False.

Imports reference export bindings.

## Misconception 3

> All imports execute in written order.

False.

The module graph determines linking/evaluation semantics.

## Misconception 4

> `.js` always means CommonJS in Node.

False.

`package.json` `"type"` changes interpretation. citeturn279903search1turn279903search2

## Misconception 5

> ESM relative imports automatically add `.js`.

False under Node's normal ESM resolver.

## Misconception 6

> `"exports"` only helps documentation.

False.

It controls package entry points and accessible subpaths.

## Misconception 7

> Top-level await is always better than an explicit startup function.

False.

It can couple module evaluation to asynchronous infrastructure.

## Misconception 8

> Dynamic import is just asynchronous `require`.

False.

They participate in different loading/resolution models.

---

# 99. Common Mistakes

## Mistake 1

Mixing ESM and CJS assumptions.

## Mistake 2

Omitting file extensions in Node ESM.

## Mistake 3

Using deep package imports that `"exports"` intentionally blocks.

## Mistake 4

Creating circular dependencies accidentally through barrel files.

## Mistake 5

Putting long network operations into top-level await without deadlines.

## Mistake 6

Using dynamic imports with untrusted specifiers.

## Mistake 7

Treating modules as request-scoped storage.

## Mistake 8

Publishing both CJS and ESM without considering duplicate singleton state.

---

# 100. Comparison With CommonJS

| Dimension | ESM | CommonJS |
|---|---|---|
| Standardized language module system | ✅ | ❌ |
| Static `import`/`export` | ✅ | ❌ |
| `require()` | ❌ native ESM local binding | ✅ |
| Live bindings | ✅ | different object-based model |
| Top-level await | ✅ | not native CJS syntax |
| Browser-native modules | ✅ | ❌ |
| Node package `"type"` | ✅ affects classification | ✅ affects classification |
| Relative extension requirement in Node | usually explicit | extension searching exists |
| Package `"exports"` | ✅ | ✅ |
| Dynamic `import()` | ✅ | ✅ modern Node |
| `__dirname` local | ❌ | ✅ |
| `import.meta` | ✅ | ❌ |
| `module.exports` | ❌ | ✅ |

---

# 101. Comparison With Bundler Modules

Bundlers may transform:

```text
ESM
CJS
asset imports
virtual modules
CSS
```

into a build-specific format.

Do not confuse:

```text
ECMAScript module semantics
```

with:

```text
bundler runtime chunk format
```

The source can be ESM even if the production bundle is a very different artifact.

---

# 102. Production Architecture Pattern

A strong Node ESM application can use:

```text
src/
├── main.js
├── app.js
├── config/
│   └── index.js
├── domain/
│   ├── users.js
│   └── orders.js
├── infrastructure/
│   ├── db.js
│   └── queue.js
└── transport/
    └── http.js
```

Each module exposes a deliberately small contract.

```text
main
 ↓
app
 ├── transport
 ├── domain
 └── infrastructure
```

Avoid arbitrary cross-layer imports.

---

# 103. Entry Point Pattern

```js
// main.js
import { createApp } from './app.js';

const app = await createApp();

await app.start();

if (import.meta.main) {
  console.log('application started');
}
```

The important architectural choice is that startup is still explicit.

ESM does not require all lifecycle logic to move into top-level module execution.

---

# 104. Application Factory

```js
// app.js
import { createServer } from './transport/http.js';
import { createDatabase } from './infrastructure/db.js';

export async function createApp() {
  const db = await createDatabase();
  const server = createServer({ db });

  return {
    async start() {
      await server.start();
    },

    async stop() {
      await server.stop();
      await db.close();
    },
  };
}
```

This pairs naturally with Chapter 62's lifecycle model.

---

# 105. Public API Boundary

For a library:

```js
// index.js
export { createClient } from './client.js';
export { ClientError } from './errors.js';
```

Then:

```json
{
  "type": "module",
  "exports": {
    ".": "./src/index.js"
  }
}
```

This makes the intended public API obvious.

---

# 106. Implementation From Scratch

## Stage A — Guided

Build:

```text
math.js
string.js
index.js
main.js
```

Requirements:

- named exports,
- default export,
- re-export,
- namespace import.

---

## Stage B — Partially Guided

Build a package with:

```json
{
  "type": "module",
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js"
  }
}
```

Verify deep internal paths are blocked.

---

## Stage C — No Reference

Build a plugin system:

```text
core
 ├── plugin-loader
 ├── plugin-contract
 └── plugins/
```

Use dynamic import with an allowlist.

---

## Stage D — Edge-Case Hardened

Add:

- circular dependency detection,
- top-level await timeout,
- JSON import,
- CommonJS compatibility,
- worker boundary,
- test isolation.

---

## Stage E — Production Grade

Add:

- package export contracts,
- versioning policy,
- public API tests,
- dependency graph checks,
- startup telemetry,
- import failure diagnostics,
- ESM/CJS compatibility tests,
- bundler validation.

---

# 107. Implementation Challenge — Package Boundary

Create:

```text
payments/
├── package.json
├── src/
│   ├── index.js
│   ├── client.js
│   ├── errors.js
│   └── internal/
│       └── transport.js
└── test/
```

Public imports:

```js
import { createPaymentClient, PaymentError } from 'payments';
```

Private implementation:

```text
src/internal/transport.js
```

must not be part of the supported public API.

---

# 108. Implementation Challenge — Dynamic Plugin Loader

Implement:

```js
const plugins = {
  slack: './plugins/slack.js',
  email: './plugins/email.js',
};

export async function loadPlugin(name) {
  const specifier = plugins[name];

  if (!specifier) {
    throw new Error(`Unknown plugin: ${name}`);
  }

  const module = await import(specifier);

  if (typeof module.createPlugin !== 'function') {
    throw new TypeError('Invalid plugin contract');
  }

  return module.createPlugin();
}
```

Then test:

- valid plugin,
- unknown plugin,
- broken plugin,
- plugin evaluation failure.

---

# 109. Implementation Challenge — Live Binding

Build:

```js
// state.js
export let status = 'idle';

export function activate() {
  status = 'active';
}
```

Then consume it from another module.

Prove experimentally that the importer observes the exporting module's updated binding.

---

# 110. Implementation Challenge — Circular Dependency

Create:

```text
a.js
b.js
```

with a deliberate cycle.

First produce a failing version.

Then redesign it using:

```text
third dependency module
```

to break the cycle.

Compare both architectures.

---

# 111. Implementation Challenge — Top-Level Await

Create:

```js
// configuration.js
export const config = await loadConfig();
```

Then add:

```text
timeout
error logging
startup failure
```

Compare this with:

```js
async function initialize() {}
```

and defend one design.

---

# 112. Debugging Exercises

## Exercise 1 — Missing Extension

```js
import './utils';
```

Node reports a module resolution error.

### Task

Explain why.

---

## Exercise 2 — Hidden Export

A file exists at:

```text
node_modules/pkg/internal.js
```

but:

```js
import 'pkg/internal.js';
```

fails.

### Task

Inspect `"exports"` and determine whether the package intentionally blocks it.

---

## Exercise 3 — Live Binding

Predict:

```js
// state.js
export let x = 1;

export function change() {
  x = 2;
}
```

```js
// app.js
import { x, change } from './state.js';

console.log(x);
change();
console.log(x);
```

Explain using bindings, not value-copy terminology.

---

## Exercise 4 — Cycle

Design a graph:

```text
A → B
B → C
C → A
```

Determine which binding access can fail before initialization.

---

## Exercise 5 — Top-Level Await

```js
export const config = await new Promise(() => {});
```

### Task

Predict application startup behavior.

---

## Exercise 6 — Dynamic Import

```js
const name = userInput;
await import(`./plugins/${name}.js`);
```

### Task

List security risks and design a safe replacement.

---

# 113. Code Review Exercise

Review:

```js
// package.json
{
  "type": "module"
}
```

```js
// index.js
import config from './config';
import './bootstrap.js';

export * from './internal/private.js';

const cache = new Map();

export async function run(userInput) {
  const plugin = await import(`./plugins/${userInput}.js`);
  return plugin.run();
}
```

Identify at least 12 issues.

Expected topics:

- missing extension,
- side effects,
- private API exposure,
- dynamic import injection,
- unbounded module-level cache,
- package encapsulation,
- default export conventions,
- error handling,
- plugin contract,
- trust boundaries,
- dependency design,
- testability.

---

# 114. Interview Questions

## Foundation

1. What is an ES module?
2. Why were ES modules standardized?
3. What is a module graph?
4. What is a live binding?
5. What is the difference between named and default exports?
6. What is a side-effect import?
7. What is a re-export?
8. What is dynamic import?
9. Why are imports static?
10. Why are ESM modules strict mode?

## Intermediate

11. Explain ESM resolution in Node.
12. Why must Node ESM relative imports commonly include file extensions?
13. What does `"type": "module"` do?
14. What are `.mjs` and `.cjs`?
15. What is `import.meta.url`?
16. What is `import.meta.resolve()`?
17. What are package `"exports"` and `"imports"`?
18. What happens when two ESM specifiers resolve to different URLs?
19. How do circular dependencies work?
20. What does top-level await change?

## Advanced

21. Explain module linking versus evaluation.
22. Why can circular ESM dependencies exist while still failing at runtime?
23. Why are live bindings useful?
24. Why is deep importing discouraged in packages with `"exports"`?
25. How does ESM interoperate with CommonJS?
26. Why can't CommonJS `require()` synchronously load an ESM graph containing top-level await?
27. Why can dual-package publishing duplicate singleton state?
28. How do bundlers use static ESM structure?
29. What is the difference between resolution and loading?
30. Why are URL semantics important to Node ESM?

## Principal Level

31. Design a package export strategy for a library consumed by both Node and browsers.
32. How would you migrate a large CommonJS monorepo to ESM?
33. How would you detect accidental circular dependencies at CI time?
34. When should top-level await be rejected during architecture review?
35. How would you design a plugin system with dynamic import safely?
36. How would you prevent internal package files from becoming accidental public API?
37. How would you support both ESM and CJS without duplicate singleton state?
38. How would you diagnose a module-resolution failure across multiple package boundaries?
39. How do ESM design decisions affect bundling and tree shaking?
40. What module boundary principles would you impose on a 200-person JavaScript platform?

---

# 115. Predict-the-Output Exercises

## Exercise A

```js
// state.js
export let value = 1;

export function change() {
  value = 2;
}
```

```js
// main.js
import { value, change } from './state.js';

console.log(value);
change();
console.log(value);
```

Predict.

---

## Exercise B

```js
// a.js
console.log('A');

export const a = 'a';
```

```js
// b.js
import { a } from './a.js';

console.log('B', a);
```

What prints first?

---

## Exercise C

```js
// config.js
export const value = await Promise.resolve(42);
console.log('config');
```

```js
// main.js
import { value } from './config.js';

console.log('main', value);
```

Predict the output order.

---

## Exercise D

```js
// module.js
console.log('evaluated');
export const value = 10;
```

```js
// main.js
import './module.js';
import './module.js';
```

How many times is the module normally evaluated for the same module identity?

---

## Exercise E

Two imports use:

```js
import './foo.mjs?x=1';
import './foo.mjs?x=2';
```

Predict whether Node treats them as the same module identity.

---

## Exercise F

```js
import { x } from './state.js';

x = 10;
```

What happens, and why?

---

# 116. Mastery Exercises

## Exercise 1 — ESM Architecture

Build a production-style Node service using only ESM:

```text
main
app
domain
infrastructure
transport
```

No arbitrary deep cross-layer imports.

---

## Exercise 2 — Package API

Create a library with:

```text
public API
private internals
conditional exports
internal imports
```

Write tests proving that unsupported paths fail.

---

## Exercise 3 — ESM/CJS Bridge

Create:

```text
modern ESM package
legacy CJS dependency
```

Use:

```text
static ESM import
createRequire()
dynamic import()
```

and document when each is appropriate.

---

## Exercise 4 — Graph Analysis

Generate an import graph and detect cycles.

Classify each cycle:

```text
safe
suspicious
architecturally invalid
```

---

## Exercise 5 — Plugin Security

Implement dynamic import with:

```text
allowlist
contract validation
error boundaries
logging
timeout around plugin initialization
```

---

# 117. Principal Design Review

For every ESM architecture decision ask:

```text
1. What is the public module boundary?
2. What is private?
3. Is the dependency graph acyclic?
4. Are imports static unless runtime loading is necessary?
5. Are top-level effects intentional?
6. Is top-level await justified?
7. Are package exports explicit?
8. Are dynamic specifiers trusted?
9. Could dual module systems duplicate state?
10. Can tooling analyze the graph?
```

---

# 118. Production Failure Modes

## Failure Mode 1 — Deployment only works locally

Cause:

```text
extensionless local resolver
```

but production uses standard Node ESM resolution.

---

## Failure Mode 2 — Package internals break consumers

Cause:

```text
deep imports became accidental API
```

before `"exports"` was introduced.

---

## Failure Mode 3 — Startup hangs

Cause:

```text
top-level await
+
unbounded external dependency
```

---

## Failure Mode 4 — Duplicate singleton

Cause:

```text
ESM package path
+
CJS package path
```

loading separate module instances.

---

## Failure Mode 5 — Plugin loading vulnerability

Cause:

```text
user-controlled dynamic import specifier
```

---

# 119. Production Checklist

```text
[ ] package.json "type" is explicit
[ ] relative ESM imports are fully specified
[ ] public package exports are explicit
[ ] internal paths are not accidental API
[ ] dynamic imports use allowlists or trusted maps
[ ] top-level await has bounded failure behavior
[ ] module-level mutable state is intentional
[ ] circular dependencies are reviewed
[ ] ESM/CJS boundaries are documented
[ ] JSON imports use current syntax
[ ] import.meta usage is environment-aware
[ ] browser/Node differences are documented
[ ] bundler assumptions are tested
[ ] module resolution is covered in CI
[ ] public API tests exist
```

---

# 120. Principal Decision Framework

| Dimension | Question |
|---|---|
| Correctness | Are module dependencies and evaluation order deterministic? |
| Performance | Are imports and dynamic loading appropriately placed? |
| Memory | Is module-level state bounded? |
| Security | Are dynamic imports and package boundaries safe? |
| Reliability | Can startup survive module-load failures? |
| Maintainability | Are dependencies explicit and graph-friendly? |
| Scalability | Is the module boundary workable for a large codebase? |
| Observability | Can load/evaluation failures be diagnosed? |
| Developer Experience | Is module resolution predictable? |
| Operational Complexity | How many module-system variants must production support? |
| Future Change | Can internals evolve without breaking consumers? |

---

# 121. Chapter Connections

## Depends On

- Chapter 10 — Scope
- Chapter 12 — Execution Model
- Chapter 31–36 — Async
- Chapter 41 — Spec Architecture
- Chapter 42 — Abstract Operations
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context and Diagnostics

## Builds Toward

- Chapter 65 — CommonJS and Interoperability
- Chapter 66 — `package.json` and Resolution
- Chapter 67 — Dependency Management and Supply Chain
- Chapter 68 — Transpilation and Compilation
- Chapter 69 — Bundlers and Build Systems
- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 80 — Library Authoring
- Chapter 94 — Compatibility Engineering
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-Scale JavaScript Platform

## Related Concepts

- CommonJS
- package resolution
- bundling
- tree shaking
- dependency graphs
- top-level await
- live bindings
- dynamic import
- module caching
- package encapsulation

## Why This Chapter Matters Later

Module boundaries are architecture boundaries.

A large JavaScript system is easier to maintain when engineers can answer:

```text
What does this module own?
What does it depend on?
What does it expose?
When is its code evaluated?
Can it be loaded independently?
Can its internals change safely?
```

ESM gives the language and host ecosystem stronger tools for those answers.

---

# 122. Spaced Retrieval Plan

## Day 0

Explain:

```text
resolution
→ linking
→ instantiation
→ evaluation
```

without notes.

## Day 2

Implement:

```text
named export
re-export
dynamic import
```

from scratch.

## Day 7

Debug:

```text
ERR_MODULE_NOT_FOUND
```

without changing package configuration blindly.

## Day 14

Design:

```text
package exports + private internals
```

## Day 30

Defend:

> Why should a large organization prefer explicit module boundaries over deep internal imports?

---

# 123. Dependency Graph

```text
ECMAScript module semantics
          │
          ▼
specifier + resolution
          │
          ▼
module graph
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
 linking  ESM   package
         loading boundaries
    │       │      │
    └───────┼──────┘
            ▼
        evaluation
            │
     ┌──────┼──────┐
     ▼      ▼      ▼
 live     TLA    dynamic
bindings  await   import
            │
            ▼
     Node architecture
            │
            ▼
     package / library design
```

---

# 124. Completion Criteria

```text
[ ] Explain what an ES module is
[ ] Explain module graphs
[ ] Explain static imports
[ ] Explain dynamic imports
[ ] Explain live bindings
[ ] Explain namespace imports
[ ] Explain default exports
[ ] Explain re-exports
[ ] Explain side-effect imports
[ ] Explain resolution
[ ] Explain Node ESM extensions
[ ] Explain package "type"
[ ] Explain .mjs and .cjs
[ ] Explain node: specifiers
[ ] Explain import.meta.url
[ ] Explain import.meta.dirname
[ ] Explain import.meta.filename
[ ] Explain import.meta.main
[ ] Explain import.meta.resolve
[ ] Explain package "exports"
[ ] Explain package "imports"
[ ] Explain conditional exports
[ ] Explain top-level await
[ ] Explain circular dependencies
[ ] Explain module identity
[ ] Explain caching
[ ] Explain ESM/CJS interoperability
[ ] Explain tree shaking
[ ] Explain dynamic import security
[ ] Design a public package API
[ ] Build a dynamic plugin loader
[ ] Debug resolution failures
[ ] Pass principal interview questions
```

---

# 125. Mastery Gate

You have mastered this chapter only when you can:

### Understand

Explain ESM from source text to module evaluation.

### Explain

Teach live bindings, linking, resolution, and module identity without reverting to “file copying” metaphors.

### Predict

Predict evaluation order, circular-dependency failures, top-level-await behavior, and import resolution.

### Implement

Build a production ESM package with an intentional public API.

### Debug

Diagnose resolution, linking, evaluation, and interoperability failures.

### Apply

Use ESM appropriately in Node services, libraries, plugins, and browser-compatible code.

### Compare

Defend ESM versus CommonJS for a specific migration or architecture.

### Defend

Explain your package-boundary and module-system choices to a principal engineering review.

---

# 126. Status

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

# 127. Chapter 64 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain module lifecycle | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain live bindings | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Build package exports | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Debug ESM resolution | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Defend ESM architecture | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. How is ESM different from CommonJS?
2. What exactly is a live binding?
3. Why are imports static?
4. What is the difference between resolution and evaluation?
5. Why does Node require extensions for relative ESM imports?
6. How do package exports protect internal files?
7. Why can cycles exist but still fail?
8. What does top-level await change?
9. How does module URL identity affect caching?
10. When should dynamic import be used?
```

---

# 128. Chapter 64 — Canonical References and Source Discipline

## Primary ECMAScript reference

- ECMA-262 Modules  
  https://tc39.es/ecma262/#sec-modules

Use ECMA-262 for language-level semantics including:

- module records,
- imports and exports,
- linking,
- instantiation,
- evaluation,
- live bindings,
- top-level await semantics.

## Primary Node.js references

- ECMAScript modules  
  https://nodejs.org/api/esm.html
- Packages  
  https://nodejs.org/api/packages.html
- CommonJS modules  
  https://nodejs.org/api/modules.html
- `node:module` APIs  
  https://nodejs.org/api/module.html

The current Node.js documentation establishes that Node has two module systems, that `.mjs` and `.cjs` provide explicit module markers, and that package `"type"` participates in `.js` classification. citeturn279903search0turn279903search1turn279903search2

The current ESM documentation also establishes:

- relative ESM specifiers require file extensions,
- ESM resolution uses URL semantics,
- `import()` is asynchronous,
- `import.meta` provides Node-specific metadata,
- current `import.meta.dirname`, `import.meta.filename`, and `import.meta.main` are supported,
- `import.meta.resolve()` returns a module-relative resolved URL string,
- ESM and CommonJS interoperate with specific constraints. citeturn279903search0

## Source-discipline rules

1. Separate ECMAScript semantics from Node host behavior.
2. Verify Node version-sensitive APIs against current Node documentation.
3. Do not generalize bundler behavior into language semantics.
4. Treat package `"exports"`/`"imports"` as Node package-resolution features.
5. Treat loader customization as advanced host behavior.
6. Treat experiments as evidence of implementation behavior, not substitutes for specification reading.
7. When portability matters, explicitly identify browser-only, Node-only, and standardized features.

---

# 129. Chapter 64 — Completion Snapshot

## Core Theory

```text
[ ] Module records
[ ] Dependency graph
[ ] Resolution
[ ] Linking
[ ] Instantiation
[ ] Evaluation
[ ] Live bindings
[ ] Module namespace
[ ] Top-level await
[ ] Module identity
```

## Node.js

```text
[ ] package "type"
[ ] .mjs / .cjs
[ ] ESM resolution
[ ] node: specifiers
[ ] import.meta
[ ] package exports
[ ] package imports
[ ] JSON import attributes
[ ] CommonJS interoperability
```

## Implementation

```text
[ ] ESM application
[ ] Public package API
[ ] Conditional exports
[ ] Dynamic plugin loader
[ ] Circular dependency exercise
[ ] Top-level await exercise
```

## Debugging

```text
[ ] Resolution failures
[ ] Export-map failures
[ ] Circular evaluation
[ ] Top-level-await hangs
[ ] Dynamic import failures
[ ] ESM/CJS boundary issues
```

## Principal Judgment

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

ESM is more than:

```js
import ...
export ...
```

It changes the way JavaScript programs are structured.

The essential model is:

```text
source text
   ↓
module
   ↓
specifier
   ↓
resolution
   ↓
dependency graph
   ↓
linking
   ↓
live bindings
   ↓
evaluation
   ↓
runtime behavior
```

A principal engineer should therefore think about ESM as an **architecture and runtime system**, not a syntax feature.

The right questions are:

```text
What is this module's public contract?
What does it depend on?
Can its dependencies be statically understood?
When is it evaluated?
Can its evaluation fail?
Does it participate in a cycle?
Does it own mutable process-wide state?
Can consumers bypass its public boundary?
Does top-level await belong here?
Does dynamic loading need trust validation?
Could ESM/CJS interoperability create duplicate state?
Will the package remain evolvable?
```

Once you can answer those questions, you are no longer merely using ESM.

You are engineering a **module graph**.