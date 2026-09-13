# Chapter 95 — Legacy JavaScript

> **JavaScript Mastery — Part XVIII: Legacy / Interoperability**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-10
>
> **Core rule:** **Legacy JavaScript is not synonymous with bad JavaScript. It is JavaScript whose semantics, compatibility obligations, tooling, architecture, or deployment assumptions were shaped by an older ecosystem.**

---

# 0. Chapter Mission

Modern JavaScript is easier to use safely when the engineer understands the historical layers underneath it.

Real production systems may contain:

```text
var
function-scoped assumptions
implicit globals
sloppy mode
arguments
with
eval
IIFEs
constructor functions
prototype inheritance
DOM0 event handlers
inline event handlers
XMLHttpRequest
callback APIs
error-first Node callbacks
CommonJS
AMD
UMD
global namespaces
prototype monkey patches
browser sniffing
polyfills
old Babel output
old bundlers
generated compatibility helpers
legacy build systems
```

Some are obsolete.

Some remain fully valid.

Some remain because an ecosystem contract still depends on them.

The correct engineering question is therefore:

> **Why is this code here, what behavior does it provide, what environment requires it, and what evidence says it can or cannot be removed?**

The migration model is:

```text
Inventory
  ↓
Classify
  ↓
Understand historical semantics
  ↓
Characterize current behavior
  ↓
Define target semantics
  ↓
Introduce boundary
  ↓
Migrate incrementally
  ↓
Observe
  ↓
Deprecate
  ↓
Remove
```

---

# 1. Learning Objectives

You should be able to:

- Define legacy JavaScript precisely.
- Distinguish old, legacy, deprecated, obsolete, unsupported, dangerous, and incorrect.
- Explain `var`, `let`, and `const` semantics.
- Explain function scope, block scope, hoisting, and TDZ.
- Explain sloppy mode and strict mode.
- Explain implicit globals.
- Explain `arguments`, including historical parameter aliasing.
- Explain `with`.
- Explain direct and indirect `eval`.
- Explain IIFEs and their historical role.
- Explain constructor functions and prototype inheritance.
- Compare constructor functions with `class`.
- Explain DOM0 events and `addEventListener`.
- Explain XHR and its differences from Fetch.
- Explain callback APIs and Promise migration.
- Explain error-first Node.js callbacks.
- Explain CommonJS, AMD, and UMD.
- Explain global namespace libraries.
- Explain prototype monkey patching.
- Explain browser sniffing and capability detection.
- Explain legacy polyfills.
- Explain transpiler-generated code.
- Explain legacy build systems and artifacts.
- Identify semantic migration hazards.
- Build characterization tests.
- Design compatibility bridges.
- Use codemods safely.
- Design strangler-style modernization.
- Design rollback plans.
- Perform legacy security reviews.
- Perform compatibility and performance reviews.
- Build a legacy-debt register.
- Conduct a principal-level modernization review.

---

# 2. Prerequisites

Recommended prerequisites:

```text
Chapter 01 — JavaScript, ECMAScript, Runtime Landscape
Chapter 05 — Variables, Declarations, Assignment
Chapter 10 — Scope / Lexical Environments
Chapter 11 — Hoisting / TDZ
Chapter 12 — Execution Contexts
Chapter 13 — Closures
Chapter 14 — this / Invocation / Binding
Chapter 17 — Prototypes
Chapter 18 — Classes / OOP
Chapter 19 — Proxy / Reflect
Chapter 29 — Errors
Chapter 31 — Async Fundamentals
Chapter 35 — Promises
Chapter 36 — Async/Await
Chapter 55 — Fetch / HTTP
Chapter 58 — Node Architecture
Chapter 64 — ES Modules
Chapter 65 — CommonJS
Chapter 66 — package.json / Resolution
Chapter 67 — Dependency Management
Chapter 68 — Transpilation
Chapter 69 — Bundlers
Chapter 70 — Source Maps
Chapter 88 — Debugging
Chapter 89 — Code Review / Refactoring
Chapter 94 — Compatibility Engineering
```

---

# 3. What Is Legacy JavaScript?

Legacy JavaScript is code shaped by a historical requirement that may no longer be the dominant requirement.

Examples:

```js
var count = 0;
```

```js
(function () {
  // private scope
})();
```

```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  return "Hello " + this.name;
};
```

```js
button.onclick = handler;
```

```js
const xhr = new XMLHttpRequest();
```

```js
fs.readFile(file, function (err, data) {
  // callback
});
```

These patterns are historically important.

The mistake is to assume:

```text
old syntax
=
bad semantics
```

That equivalence is false.

---

# 4. Old vs Legacy vs Deprecated

| Term | Meaning |
|---|---|
| Old | Introduced long ago |
| Legacy | Historical pattern still present or relevant |
| Deprecated | Discouraged by its owner/standard |
| Obsolete | No longer needed for the supported environment |
| Unsupported | No guarantee of support |
| Dangerous | Has unacceptable risk in the current context |
| Incorrect | Violates the required behavior |

Example:

```js
Array.prototype.map
```

is old.

It is not legacy in the sense of being obsolete.

By contrast:

```js
document.write(...)
```

may be legacy in a particular architecture.

The classification depends on use and environment.

---

# 5. Why Legacy Code Exists

Typical evolution:

```text
business requirement
        ↓
old platform limitation
        ↓
workaround
        ↓
framework abstraction
        ↓
new framework
        ↓
partial migration
        ↓
compatibility layer
        ↓
dependency
        ↓
more migration cost
```

After years, a ten-line workaround can become a platform contract.

That is why removal requires investigation.

---

# 6. Mental Model — Historical Layers

```text
ECMAScript core
      ↓
classic browser scripts
      ↓
global namespaces
      ↓
IIFEs
      ↓
prototype constructors
      ↓
callback APIs
      ↓
AMD / CommonJS / UMD
      ↓
transpiled ES5
      ↓
ES2015+
      ↓
ES modules
      ↓
modern runtimes
```

A real application may contain several layers simultaneously.

Modernization is therefore an archaeology problem before it becomes a coding problem.

---

# 7. `var`

`var` is function-scoped.

```js
function demo() {
  if (true) {
    var value = 10;
  }

  return value;
}

console.log(demo());
```

Result:

```text
10
```

The `if` block does not create a `var` scope.

This behavior is a language semantic, not a style preference.

---

# 8. `var` Hoisting

```js
console.log(value);
var value = 10;
```

Conceptually:

```js
var value;
console.log(value);
value = 10;
```

Result:

```text
undefined
```

The declaration is processed before the assignment.

---

# 9. `let` and `const`

Modern declarations are block-scoped.

```js
{
  let a = 1;
  const b = 2;
}

console.log(typeof a);
```

The block-local bindings are not visible outside the block.

The comparison is:

```text
var
→ function scope

let / const
→ block scope
```

---

# 10. Migration Hazard — `var` to `let`

Consider:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

Typical result:

```text
3
3
3
```

With:

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

the loop iteration bindings behave differently.

Therefore:

> **Replacing `var` is a semantic migration, not a cosmetic rewrite.**

---

# 11. `const` Does Not Freeze Objects

```js
const user = {
  age: 20
};

user.age = 21;
```

This is valid.

`const` protects the binding:

```text
user → object
```

It does not automatically freeze the referenced object.

---

# 12. Implicit Globals

Legacy sloppy-mode code can accidentally assign to undeclared identifiers.

```js
function bad() {
  total = 10;
}
```

The historical result can involve global state.

Modern strict/module code rejects this pattern.

Hidden globals create:

```text
coupling
ordering dependencies
test leakage
security risk
name collisions
```

---

# 13. Strict Mode

Strict mode changes several legacy semantics:

```js
"use strict";
```

ES modules are strict by definition.

Important effects include:

- accidental undeclared assignment throws,
- `with` is not permitted,
- certain duplicate/legacy syntax is restricted,
- `this` semantics differ for plain calls,
- dynamic code interactions are constrained.

Do not treat strict mode as a generic “optimization switch.”

It is a semantic mode.

---

# 14. Sloppy Mode

Classic scripts without strict mode can execute with legacy sloppy semantics.

A system may contain:

```text
classic sloppy script
classic strict script
ES module
CommonJS wrapper
```

These execution environments are not interchangeable.

When debugging a historical system, first determine how the file is loaded.

---

# 15. `arguments`

Legacy functions often use:

```js
function sum() {
  let total = 0;

  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }

  return total;
}
```

Modern code can use:

```js
function sum(...values) {
  return values.reduce(
    (total, value) => total + value,
    0
  );
}
```

The replacement is often clearer.

But the semantics are not universally identical.

---

# 16. `arguments` Is Not an Array

```js
function check() {
  return Array.isArray(arguments);
}

console.log(check());
```

Result:

```text
false
```

It is array-like.

Legacy code may depend on:

```text
length
indexing
parameter aliasing
call-site behavior
```

---

# 17. `arguments` Parameter Aliasing

In historical non-strict function semantics:

```js
function demo(value) {
  value = 20;
  return arguments[0];
}
```

can return:

```text
20
```

because the parameter and corresponding `arguments` property can be linked.

Strict mode changes this relationship.

A migration to rest parameters removes this historical coupling.

---

# 18. Function `.length`

Function metadata can matter.

```js
function legacy(a, b) {}

console.log(legacy.length);
```

Result:

```text
2
```

Changing the declaration to:

```js
function legacy(...args) {}
```

changes the function's `length`.

Frameworks and dependency injection systems can historically use function arity as metadata.

---

# 19. `this` Migration Hazard

Legacy:

```js
const obj = {
  value: 10,

  getValue: function () {
    return this.value;
  }
};
```

A naive arrow conversion:

```js
const obj = {
  value: 10,

  getValue: () => {
    return this.value;
  }
};
```

changes the meaning of `this`.

Arrow functions capture lexical `this`.

They do not create their own dynamic `this`.

---

# 20. `with`

Historical JavaScript allowed:

```js
with (user) {
  console.log(name);
}
```

The identifier lookup becomes dynamically dependent on object properties.

This harms:

```text
static reasoning
optimization
linting
security review
refactoring
scope understanding
```

Strict mode prohibits `with`.

---

# 21. Migrating `with`

Legacy:

```js
with (user) {
  render(name, email);
}
```

Modern:

```js
render(user.name, user.email);
```

or:

```js
const { name, email } = user;
render(name, email);
```

The goal is explicit identifier resolution.

---

# 22. `eval`

```js
eval(source);
```

evaluates source code dynamically.

It can affect:

```text
scope
security
optimization
static analysis
debugging
tooling
CSP
```

Use only when dynamic code is a deliberate design requirement.

Never treat arbitrary untrusted text as JavaScript source.

---

# 23. Direct vs Indirect `eval`

Direct:

```js
eval("...");
```

has special access to the current execution context.

Indirect:

```js
const run = eval;
run("...");
```

has different scope behavior.

Legacy debugging often requires identifying which form is present.

---

# 24. `new Function`

Another dynamic-code mechanism:

```js
const fn =
  new Function("a", "b", "return a + b;");
```

It creates executable source dynamically.

Security principle:

```text
untrusted data
≠
trusted code
```

Do not allow user input to become source code.

---

# 25. Why Dynamic Code Complicates Optimization

Consider:

```js
function f() {
  let x = 1;
  eval("x = 2;");
  return x;
}
```

The source text alone no longer completely describes the function's binding behavior.

Modern engines handle dynamic code carefully, but such code weakens assumptions available to optimization and static analysis.

---

# 26. IIFEs

Before standard modules were broadly deployed, IIFEs created private scopes:

```js
(function () {
  const secret = 42;
})();
```

They solved a real problem:

```text
global variables were easy to create
classic scripts lacked module scoping
```

Therefore IIFEs were an important architectural technique.

---

# 27. Global Namespace Pattern

Legacy libraries often used:

```js
window.MyApp = window.MyApp || {};
```

then:

```js
window.MyApp.users = {};
window.MyApp.orders = {};
```

This reduced random global names while still using a global namespace.

The modern direction is:

```text
global namespace
→ module exports
→ explicit imports
```

---

# 28. IIFE to Module Migration

Possible target:

```js
function start() {
  // ...
}

start();
```

or:

```js
export function start() {}
```

But inspect first:

```text
global dependencies
script ordering
exports
side effects
multiple loading
initialization
```

The IIFE may be providing more than “privacy.”

---

# 29. Constructor Functions

Legacy OOP commonly used:

```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  return "Hello " + this.name;
};
```

Usage:

```js
const p = new Person("A");
```

This remains valid JavaScript.

---

# 30. Prototype Inheritance

Historical inheritance:

```js
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function () {
  return this.name;
};

function Dog(name) {
  Animal.call(this, name);
}

Dog.prototype =
  Object.create(Animal.prototype);

Dog.prototype.constructor = Dog;
```

Important semantics:

```text
prototype chain
constructor call
method lookup
instanceof
own vs inherited properties
```

---

# 31. Constructor Functions vs `class`

`class` provides class syntax, but JavaScript remains prototype-based.

Example:

```js
class Person {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return "Hello " + this.name;
  }
}
```

The underlying object model still relies on prototypes.

Therefore:

```text
class syntax
≠
different object model
```

---

# 32. Migration Hazard — Enumerability

Prototype assignment:

```js
Person.prototype.greet = function () {};
```

creates a property with semantics different from a class method declaration.

Class methods are non-enumerable.

A migration can therefore change:

```js
Object.keys(Person.prototype)
```

and related reflection behavior.

---

# 33. `__proto__`

Legacy code may use:

```js
obj.__proto__
```

Prefer standardized object APIs:

```js
Object.getPrototypeOf(obj);
Object.setPrototypeOf(obj, proto);
```

where prototype manipulation is genuinely necessary.

Better still, design the prototype relationship at object creation time.

---

# 34. Monkey Patching

Example:

```js
const original = lib.method;

lib.method = function (...args) {
  audit();
  return original.apply(this, args);
};
```

This can be useful for instrumentation but introduces hidden coupling.

Potential problems:

```text
load order
double patching
identity changes
stack traces
dependency conflicts
future collisions
```

Prefer explicit hooks where possible.

---

# 35. Prototype Pollution vs Prototype Extension

Do not confuse:

```text
intentional prototype extension
```

with:

```text
prototype pollution vulnerability
```

The former can be architectural coupling.

The latter is a security class often involving attacker-controlled property paths such as `__proto__`, `constructor`, or `prototype`.

Security review must inspect data flow rather than only a keyword.

---

# 36. Extending `Object.prototype`

Legacy:

```js
Object.prototype.someHelper = function () {};
```

This can affect:

```text
for...in
property checks
libraries
serialization assumptions
security boundaries
```

Avoid global built-in mutation.

Migrate toward standalone functions or explicit utility modules.

---

# 37. Extending `Array.prototype`

Legacy:

```js
Array.prototype.sum = function () {};
```

Problems include:

```text
name collisions
enumeration changes
future standard collisions
dependency interference
```

An internal utility function is usually safer:

```js
function sum(values) {}
```

---

# 38. `for...in` on Arrays

Legacy:

```js
for (var i in array) {
  use(array[i]);
}
```

`for...in` iterates enumerable property keys, including inherited enumerable properties.

Modern code often uses:

```js
for (const value of array) {}
```

or:

```js
array.forEach(...)
```

But the correct replacement depends on required semantics.

---

# 39. Enumeration Migration

Before changing:

```js
for...in
```

ask:

```text
own or inherited?
keys or values?
symbols?
mutation?
enumeration order?
prototype extensions?
```

The loop construct is part of the program's semantic contract.

---

# 40. DOM0 Events

Legacy browser code:

```js
button.onclick = handler;
```

Multiple assignments replace each other:

```js
button.onclick = first;
button.onclick = second;
```

The second handler replaces the first.

---

# 41. `addEventListener`

Modern browser code:

```js
button.addEventListener(
  "click",
  handler
);
```

This supports multiple listeners and richer listener options.

Migration should consider:

```text
listener removal
duplicate registration
capture
bubbling
this
passive behavior
once
AbortSignal
```

---

# 42. Inline HTML Events

Legacy:

```html
<button onclick="saveOrder()">Save</button>
```

Problems include:

```text
behavior/content coupling
global-name dependence
harder CSP strategy
harder automated testing
```

Migrate carefully when templates or CMS systems generate the markup.

---

# 43. `XMLHttpRequest`

Legacy browser networking:

```js
const xhr =
  new XMLHttpRequest();

xhr.open("GET", "/orders");

xhr.onload = function () {
  console.log(xhr.responseText);
};

xhr.send();
```

XHR is historically important and still exists.

Its API model differs from Fetch.

---

# 44. XHR vs Fetch

Fetch:

```js
const response =
  await fetch("/orders");

const data =
  await response.json();
```

But not all XHR behavior maps directly.

Review:

```text
HTTP status handling
network error model
abort
upload progress
download progress
streaming
credentials
response types
timeouts
```

A blanket rewrite is unsafe.

---

# 45. Fetch Status Semantics

Fetch does not normally reject merely because the HTTP response status is 404 or 500.

Modern code may need:

```js
const response = await fetch(url);

if (!response.ok) {
  throw new Error(
    `HTTP ${response.status}`
  );
}
```

Migration must preserve the old application's error contract.

---

# 46. Callback APIs

Legacy async:

```js
doWork(function (err, value) {
  if (err) {
    return handleError(err);
  }

  use(value);
});
```

This style was a practical solution to asynchronous I/O.

It is not inherently “bad.”

Problems emerge when callback structure makes:

```text
composition
error flow
cleanup
testing
cancellation
```

hard to reason about.

---

# 47. Callback Hell

Example:

```js
step1((err, a) => {
  if (err) return fail(err);

  step2(a, (err, b) => {
    if (err) return fail(err);

    step3(b, (err, c) => {
      if (err) return fail(err);

      finish(c);
    });
  });
});
```

Promises can flatten the control flow.

But the migration must preserve semantics.

---

# 48. Callback Timing

Legacy:

```js
function legacy(cb) {
  cb();
}
```

does not necessarily have the same observable timing as:

```js
function modern() {
  return Promise.resolve();
}
```

Promise reactions execute through the job/microtask model.

Timing changes can break tests and application logic.

---

# 49. Error-First Callbacks

A common Node convention:

```js
callback(error, result);
```

But actual APIs can be more complex:

```js
callback(null, valueA, valueB);
```

An adapter to Promises must define how multiple callback values are represented.

Do not write a universal promisifier without understanding the API.

---

# 50. Callback-to-Promise Migration

Legacy:

```js
function getValue(done) {
  oldApi((err, value) => {
    if (err) return done(err);
    done(null, value);
  });
}
```

Modern:

```js
async function getValue() {
  return await newApi();
}
```

Check:

```text
errors
timing
multiple resolutions
cleanup
cancellation
ordering
resource lifetime
```

---

# 51. CommonJS

CommonJS:

```js
const fs = require("node:fs");

module.exports = {
  run
};
```

It became deeply embedded in Node.js and the npm ecosystem.

It remains relevant for:

```text
legacy packages
Node interoperability
tooling
published modules
```

---

# 52. CommonJS vs ESM

ESM:

```js
import fs from "node:fs";

export function run() {}
```

CommonJS and ESM differ in:

```text
loading
resolution
evaluation
exports
cycles
top-level behavior
interop
host integration
```

Do not treat syntax replacement as a complete migration.

---

# 53. CJS → ESM Migration

Before migration inventory:

```text
entry points
require() calls
dynamic require()
module.exports
exports.x
side-effect imports
cycles
package.json
test runner
bundler
consumers
```

Then migrate incrementally.

---

# 54. Circular Dependency Risk

CommonJS cycles can expose partially initialized exports.

ESM has its own live-binding and module-linking semantics.

Therefore:

```text
CJS cycle
→ ESM cycle
```

is not automatically semantically equivalent.

Characterize circular dependencies.

---

# 55. AMD

AMD examples:

```js
define(
  ["dep"],
  function (dep) {
    return {};
  }
);
```

It addressed asynchronous browser module loading before native modules were broadly available.

Modern migration:

```text
AMD
→ ESM
```

but loader and plugin semantics must be analyzed.

---

# 56. UMD

UMD packages attempted interoperability among:

```text
CommonJS
AMD
globals
```

A typical pattern branches on the available loader.

UMD is an important historical interoperability pattern.

Modern packages can often publish explicit module formats instead.

---

# 57. Global Script Loading

Legacy HTML may use:

```html
<script src="library.js"></script>
<script src="plugin.js"></script>
<script src="app.js"></script>
```

The order itself can be a dependency graph.

For example:

```text
library.js
  ↓
plugin.js
  ↓
app.js
```

Replacing this with arbitrary asynchronous loading can break the system.

---

# 58. Module Migration as Graph Extraction

A useful migration process:

```text
discover global dependencies
       ↓
construct dependency graph
       ↓
make dependencies explicit
       ↓
introduce modules
       ↓
remove globals
```

This is more reliable than file-by-file syntax replacement.

---

# 59. Legacy Browser Detection

Historical code:

```js
if (navigator.userAgent.includes("MSIE")) {
  useOldPath();
}
```

The branch may exist because of:

```text
browser bug
missing API
parser limitation
rendering behavior
```

Before deleting it, discover the original reason.

---

# 60. Feature Detection

Modern capability checks:

```js
if (typeof globalThis.SomeAPI === "function") {
  useModernPath();
} else {
  useFallback();
}
```

Capability checks are generally more robust than identifying a browser brand.

But the check should match the behavior required.

---

# 61. Behavioral Detection

Existence may not be enough.

```js
typeof api.method === "function"
```

only proves:

```text
a callable property appears to exist
```

It does not prove:

```text
correct semantics
required options
bug-free behavior
performance
security
```

Use behavioral tests when known implementation differences matter.

---

# 62. Polyfills

A polyfill supplies runtime behavior using existing platform primitives.

It can help with:

```text
missing standard API
```

It usually cannot make an old parser understand arbitrary new syntax.

Thus:

```text
transpilation
+
polyfill
```

may be required for some legacy targets.

---

# 63. Transpiled Artifacts

Old bundles may contain helpers:

```js
function _inherits(...) {}
function _extends(...) {}
function _classCallCheck(...) {}
```

These may be generated by Babel or another compiler.

Do not manually edit generated output.

Find the original source and transformation pipeline.

---

# 64. Source-to-Artifact Pipeline

```text
source
 ↓
parser
 ↓
transform
 ↓
bundle
 ↓
minify
 ↓
artifact
 ↓
runtime
```

Debugging should identify which layer introduced the behavior.

---

# 65. Legacy Build Systems

Examples:

```text
Grunt
Gulp
RequireJS
Browserify
older Webpack configurations
custom shell pipelines
```

Age alone is not a reason to rewrite.

Measure:

```text
build reliability
security
maintenance cost
build duration
bundle quality
developer experience
```

A stable old build system can be safer than a rushed migration.

---

# 66. Legacy Babel Configurations

Audit:

```text
presets
plugins
targets
polyfill strategy
runtime helpers
module transform
```

Remove obsolete transformations only after the supported environment changes.

---

# 67. Legacy Tests

Old test suites may assert implementation details.

Example:

```js
expect(object._value).toBe(10);
```

Before refactoring, classify tests:

```text
public contract
integration
implementation detail
snapshot
generated artifact
```

Preserve meaningful contract tests.

Retire tests that encode intentionally removed internals.

---

# 68. Characterization Tests

Before modifying legacy behavior:

```js
test("legacy behavior", () => {
  expect(legacyFunction(input))
    .toEqual(expected);
});
```

The purpose is to record:

```text
what the system currently does
```

not:

```text
what it ideally should do
```

This distinction is essential during migration.

---

# 69. Golden Tests

Golden tests capture artifacts:

```text
JSON output
HTML
generated code
configuration
serialized messages
CLI output
```

They can be valuable for legacy systems with large externally visible outputs.

Review changes carefully rather than blindly updating snapshots.

---

# 70. Semantic Diffing

Compare:

```text
legacy result
new result
```

Classify differences:

```text
intentional
bug fix
regression
undefined legacy behavior
environmental difference
```

A difference is not automatically a bug.

---

# 71. Migration Invariants

Define what must not change.

Examples:

```text
public API
error codes
serialization
ordering
security guarantees
performance budget
supported environments
```

Migration tests should enforce those invariants.

---

# 72. Compatibility Bridge

Example:

```js
export function oldApi(...args) {
  return newApi(...args);
}
```

Then:

```text
legacy caller
    ↓
compatibility bridge
    ↓
modern implementation
```

This separates:

```text
implementation migration
from
consumer migration
```

---

# 73. Strangler Migration

```text
legacy system
    │
    ├── old path
    │
    └── new path
          ↓
      migrate boundary
          ↓
      shrink old path
          ↓
      remove
```

This reduces blast radius.

---

# 74. Codemods

Good codemod candidates:

```text
simple var patterns
deprecated method calls
known import rewrites
API renames
mechanical syntax changes
```

Bad candidates:

```text
semantically ambiguous this rewrites
complex callback migration
dynamic require conversion
behavior-dependent control flow
```

Run:

```text
codemod
→ tests
→ lint
→ type checks
→ semantic review
```

---

# 75. Security Review

Prioritize:

```text
eval
new Function
innerHTML
document.write
inline scripts
prototype mutation
prototype pollution paths
dynamic script insertion
unsafe URL construction
global mutable state
```

Do not classify solely from names.

Follow:

```text
input
→ transformation
→ sink
```

---

# 76. `innerHTML`

`innerHTML` is not inherently insecure.

Risk depends on:

```text
untrusted input
+
HTML interpretation
```

A modernization should replace unsafe data flow, not mechanically ban the API.

---

# 77. `document.write`

Legacy:

```js
document.write(
  "<script src='legacy.js'></script>"
);
```

This couples script behavior to parsing.

Modern approaches may use:

```text
static modules
dynamic import
explicit script elements
loader infrastructure
```

depending on requirements.

---

# 78. Inline Script and CSP

Legacy:

```html
<script>
  boot();
</script>
```

or:

```html
<button onclick="save()">
```

can complicate Content Security Policy strategy.

Migration should understand:

```text
nonces
hashes
external/module scripts
event listeners
```

and the site's existing security architecture.

---

# 79. Legacy Global State

```js
window.AppState = {
  user: null
};
```

Global mutable state causes:

```text
ordering bugs
hidden dependencies
test contamination
concurrency assumptions
```

Modernize toward explicit state ownership and dependency injection.

---

# 80. Singleton Migration

Legacy:

```js
var instance;

function getInstance() {
  if (!instance) {
    instance = create();
  }

  return instance;
}
```

Before replacing it with module state, ask:

```text
What scope is intended?
What lifecycle?
What isolation?
What test behavior?
What process/worker boundaries?
```

A singleton is a design decision, not merely a syntax pattern.

---

# 81. Legacy Object Dictionaries

Old:

```js
const cache = {};
cache[key] = value;
```

Potential issues:

```text
prototype keys
coercion of keys
enumeration behavior
collision
```

Modern choices include:

```js
new Map()
```

or:

```js
Object.create(null)
```

depending on the contract.

---

# 82. `Map` Migration

Do not replace every object dictionary with `Map`.

Compare:

```text
key types
lookup semantics
iteration
serialization
prototype behavior
memory
performance
API surface
```

A migration should serve the data model.

---

# 83. Legacy String Concatenation

Old:

```js
"Hello " + name + "!"
```

may become:

```js
`Hello ${name}!`
```

But review:

```text
coercion
null/undefined
localization
encoding
security
```

Template literals do not sanitize output.

---

# 84. Legacy Defaulting

Old:

```js
const value = input || fallback;
```

Modern nullish behavior:

```js
const value = input ?? fallback;
```

These differ for:

```text
0
""
false
NaN
```

A migration can therefore introduce an intentional behavior change or an accidental regression.

---

# 85. Legacy Optional Access

Old:

```js
if (obj && obj.user && obj.user.address) {
  return obj.user.address.city;
}
```

Modern:

```js
return obj?.user?.address?.city;
```

This is often clearer.

Still inspect:

```text
getter side effects
truthiness semantics
defaulting
evaluation order
```

---

# 86. Legacy Date

Old systems commonly use:

```js
new Date()
date.setMonth(...)
date.getTime()
```

Date/time semantics can hide:

```text
local zone
UTC
mutation
precision
calendar arithmetic
```

Chapter 92 provides the modern Temporal model.

Do not globally replace every Date with `Temporal.Instant`.

First classify the domain:

```text
Instant
PlainDate
PlainTime
PlainDateTime
ZonedDateTime
Duration
```

---

# 87. Legacy Unicode Assumptions

Older code may treat:

```text
UTF-16 code unit
=
character
```

This is not generally correct.

Modern text handling may require:

```text
code units
code points
grapheme clusters
bytes
```

Migration from string indexing requires domain analysis.

---

# 88. Legacy JSON Assumptions

JSON may:

```text
drop undefined properties
convert Date through toJSON
reject BigInt
lose special object identity
```

When modernizing serialization, define the wire contract explicitly.

---

# 89. Legacy Error Values

Historical code may:

```js
throw "failed";
```

or:

```js
callback("TIMEOUT");
```

Modern APIs generally benefit from `Error` instances and structured error contracts.

But migration must preserve:

```text
error code
retry classification
logging
transport mapping
```

---

# 90. Error Contract Migration

Legacy:

```js
throw "TIMEOUT";
```

Target:

```js
class TimeoutError extends Error {
  constructor(message = "Timeout") {
    super(message);
    this.name = "TimeoutError";
    this.code = "TIMEOUT";
  }
}
```

The new shape must be introduced with consumer migration.

---

# 91. Legacy Promise Libraries

Historical systems may use:

```text
custom Deferred
jQuery Deferred
Q
Bluebird
```

A thenable is not necessarily identical to native Promise semantics.

Check:

```text
scheduling
assimilation
error handling
cancellation
multiple handlers
```

before migrating.

---

# 92. Event-Driven Legacy APIs

Old event systems may use:

```js
emitter.on("data", handler);
```

Migration to streams/async iterators can be beneficial, but inspect:

```text
ordering
backpressure
listener cleanup
error events
resource lifetime
```

---

# 93. Legacy Node Environment Assumptions

A shared library may assume:

```js
process.env.NODE_ENV
```

or:

```js
process
```

exists.

That assumption can fail in:

```text
browser
worker
SSR
edge runtime
bundled package
```

Host assumptions must be explicit.

---

# 94. Legacy Configuration

Global:

```js
window.Config = {};
```

can become:

```text
validated startup configuration
dependency injection
immutable configuration object
```

Do not allow arbitrary modules to mutate configuration after startup unless explicitly designed.

---

# 95. Legacy Module Resolution

Historical packages may depend on:

```text
main
browser
module
custom resolver behavior
```

Modern `exports` and conditional exports can alter consumer resolution.

Test:

```text
Node CJS
Node ESM
browser bundler
direct browser
```

---

# 96. Side-Effect Imports

Legacy:

```js
require("./register");
```

may intentionally load side effects.

A migration to ESM may be:

```js
import "./register.js";
```

Do not delete an import just because its value is unused.

Unused can mean:

```text
side effect
```

---

# 97. Legacy Initialization Order

Older systems can depend on:

```text
global setup
↓
plugin registration
↓
configuration
↓
app boot
```

ESM makes dependency structure more explicit but changes module loading/evaluation semantics.

Initialization must be modeled deliberately.

---

# 98. Dependency Cycles

Legacy modules can contain cycles:

```text
A → B → A
```

Migration can expose previously hidden partial initialization.

Before changing loaders, inventory cycles and characterize them.

---

# 99. Legacy Compatibility Hacks

Examples:

```text
browser-specific branches
feature flags with expired values
manual DOM workarounds
old parser guards
polyfills
global shims
```

Each should eventually have:

```text
purpose
affected environment
owner
test
removal condition
```

---

# 100. Compatibility Debt Register

Create:

```md
# Compatibility Debt

## Item
-

## Historical reason
-

## Current requirement
-

## Affected environments
-

## Security risk
-

## Performance risk
-

## Migration cost
-

## Replacement
-

## Owner
-

## Removal trigger
-
```

This turns “old code” into managed work.



---

# 101. How to Find Legacy Code

Search for signals:

```text
var
with (
eval(
new Function(
onclick=
onload=
document.write
document.all
navigator.userAgent
XMLHttpRequest
__proto__
Object.prototype.
Array.prototype.
arguments
require(
module.exports
define(
IIFE
global namespace
polyfill
```

These are indicators, not automatic violations.

Classify findings after discovery.

---

# 102. Static Inventory

Create a report:

```text
pattern
file
line
count
owner
risk
replacement
```

Then group by:

```text
security
compatibility
correctness
performance
maintainability
```

Do not prioritize only by occurrence count.

---

# 103. Legacy Risk Score

Score:

```text
Blast radius
Business criticality
Semantic complexity
Security impact
Runtime coupling
Test coverage
Migration cost
Rollback difficulty
```

A useful output:

```text
Low
Medium
High
Critical
```

Keep the scoring model transparent.

---

# 104. Migration Prioritization

Prefer areas that have:

```text
high business value
high maintenance cost
good test coverage
clear replacement
small boundary
```

Avoid starting with:

```text
highest complexity
lowest test coverage
widest shared dependency
```

unless security or operational risk demands it.

---

# 105. Characterization Before Refactoring

Order:

```text
observe
→ test
→ refactor
```

not:

```text
rewrite
→ discover behavior
→ emergency repair
```

Characterization tests are your behavioral safety net.

---

# 106. Golden Corpus

For parsers, formatters, serializers, and compatibility utilities, build:

```text
known input
known legacy output
```

Then compare the new implementation against the corpus.

Add edge cases continuously.

---

# 107. Semantic Equivalence

Ask what must remain equal.

Possible dimensions:

```text
return value
throws/rejects
error identity
timing
ordering
serialization
side effects
resource lifetime
```

Different migrations require different equivalence definitions.

---

# 108. Migration Invariants Example

For an API migration:

```text
same inputs
→
same public outputs

same failure category
same error code
same ordering
same externally visible side effects
```

Performance may be allowed to change within a defined budget.

---

# 109. Dual Run

Where safe:

```js
const oldResult = legacy(input);
const newResult = modern(input);

assertEquivalent(oldResult, newResult);
```

Use dual execution only when:

```text
side effects are controlled
cost is acceptable
determinism is sufficient
```

Do not execute external payments twice.

---

# 110. Shadow Traffic

For services:

```text
real request
      ↓
legacy path
      +
shadow modern path
      ↓
compare
```

Shadowing must avoid duplicate side effects.

---

# 111. Canary Rollout

Example:

```text
1%
5%
25%
50%
100%
```

At each phase observe:

```text
error rate
latency
memory
CPU
fallback rate
compatibility failures
business KPIs
```

Define rollback thresholds before rollout.

---

# 112. Rollback

Before migrating ask:

```text
Can old code run?
Can new data be read by old code?
Can old clients call the new API?
Can the bridge remain?
Can deployment be reversed?
```

Data migrations often make rollback harder than code migrations.

---

# 113. Database Compatibility

Modernizing JavaScript can alter:

```text
serialization
timestamps
numbers
undefined behavior
property names
error payloads
```

Run data compatibility tests before rollout.

---

# 114. Public Library Migration

For npm libraries, consumers may have:

```text
older Node
older bundler
CJS
ESM
different TypeScript
browser builds
```

Document:

```text
supported runtime
supported module formats
syntax level
engine constraints
migration path
```

---

# 115. Library API Bridge

Possible:

```text
old API
 ↓
deprecated wrapper
 ↓
new implementation
```

Then:

```text
measure usage
notify users
migrate docs
remove when policy allows
```

Do not hide breaking changes under a minor release.

---

# 116. Deprecation Design

A good deprecation states:

```text
What?
Why?
Replacement?
Affected users?
Removal release?
Migration steps?
Detection?
```

Use:

```text
warnings
lint rules
codemods
telemetry
```

where appropriate.

---

# 117. Compatibility Debt Lifecycle

```text
introduced
   ↓
documented
   ↓
monitored
   ↓
baseline changes
   ↓
candidate for removal
   ↓
deprecated
   ↓
removed
```

A compatibility workaround should have a life cycle.

---

# 118. Legacy Security Audit

High-value search patterns:

```text
eval
new Function
innerHTML
outerHTML
insertAdjacentHTML
document.write
script.src from user data
prototype assignment
dynamic property path
global configuration mutation
```

Then perform data-flow analysis.

---

# 119. Prototype Pollution Review

Risk pattern:

```js
target[userKey] = value;
```

with attacker-controlled nested keys can create prototype-related vulnerabilities in some architectures.

Review:

```text
input validation
prototype-sensitive keys
merge utilities
recursive assignment
object construction
```

A search for `__proto__` alone is insufficient.

---

# 120. CSP Migration

When moving away from:

```html
onclick="..."
```

or inline script, coordinate with:

```text
Content Security Policy
nonces
hashes
module scripts
external assets
```

Security policy and application architecture must move together.

---

# 121. Legacy Authentication Code

Historical systems may use:

```text
custom token parsing
localStorage assumptions
global auth state
manual cookie parsing
```

Do not modernize only for style.

Verify:

```text
session semantics
CSRF
XSS
cookie attributes
expiry
clock behavior
```

---

# 122. Legacy Crypto Wrappers

An old wrapper around crypto APIs may exist because:

```text
browser compatibility
different implementation
old library contract
```

Before replacing it, compare:

```text
algorithm
parameters
key handling
randomness
error behavior
encoding
```

Never replace cryptographic code with an insecure fallback for compatibility convenience.

---

# 123. Legacy Performance Hacks

Historical JavaScript contains advice like:

```text
avoid closures
cache array length
avoid function creation
use strings instead of arrays
manually pool objects
```

Some may have been justified in old engines.

Modern engines optimize differently.

Measure before removing or preserving.

---

# 124. Legacy Memory Tricks

Example:

```js
largeValue = null;
```

Sometimes useful, sometimes cargo cult.

Reason from:

```text
reachability
lifetime
retention
closure capture
GC behavior
```

Chapter 45 provides the deeper memory model.

---

# 125. Legacy DOM Performance

Older browser code may manually batch DOM changes.

Some techniques remain useful because layout and rendering costs still exist.

Modernization should benchmark:

```text
style recalculation
layout
paint
script
```

Do not remove batching solely because the code is old.

---

# 126. Legacy Error Timing

A refactor can preserve values but change timing:

```text
sync throw
→
async rejection
```

or:

```text
same callback
→
microtask callback
```

This can break consumers.

Timing is part of an API contract when observable.

---

# 127. Legacy Event Ordering

Event-driven code can depend on:

```text
listener order
capture/bubble phase
synchronous dispatch
microtask scheduling
```

Migration must test ordering.

---

# 128. Legacy Browser APIs

Historical browser code may use:

```text
document.all
attachEvent
window.event
event.srcElement
```

These patterns indicate older event/platform assumptions.

Do not remove before determining current support obligations.

---

# 129. `attachEvent` / Early Event Models

Very old browser code may contain:

```js
element.attachEvent("onclick", handler);
```

This is historically associated with old IE event models.

If such code remains, check whether it is actually reachable in the current deployment.

---

# 130. Legacy `window.event`

Old handlers may use:

```js
window.event
```

rather than receiving:

```js
function handler(event) {}
```

Modern event listeners should prefer the explicit event parameter.

---

# 131. Legacy Forms

Old applications may rely on:

```text
form.submit()
onclick validation
DOM mutation
global form names
```

Migration should inspect:

```text
validation
default action
submit event
preventDefault
browser built-ins
```

---

# 132. Global Element IDs

Historic browsers exposed elements by `id` as global variables in some environments:

```js
console.log(myElement);
```

Do not depend on this behavior.

Use explicit DOM lookup:

```js
document.getElementById("myElement");
```

or a well-defined reference.

---

# 133. Legacy HTML Injection

Patterns such as:

```js
container.innerHTML += content;
```

can replace/recreate descendant nodes in ways that disrupt:

```text
event listeners
references
state
selection
```

A migration should understand DOM mutation behavior before changing rendering code.

---

# 134. Legacy jQuery Patterns

A historical application may contain:

```js
$(".button").click(handler);
$.ajax(...);
```

jQuery often abstracted browser inconsistencies.

Migration to native APIs requires comparing:

```text
event normalization
Ajax defaults
serialization
promise/deferred behavior
DOM traversal
selector semantics
```

Do not assume the native replacement is one line for one line.

---

# 135. Legacy Framework Migration

Older framework code may encode hidden assumptions around:

```text
scope
this
digest/update cycles
events
dependency injection
templates
lifecycle
```

The migration target should preserve the user-visible contract and system lifecycle, not just API names.

---

# 136. Legacy State Management

Old systems may store state on:

```text
window
DOM nodes
module globals
singleton objects
```

Modern architecture may use explicit state stores.

The migration should establish:

```text
ownership
lifecycle
serialization
persistence
cross-tab behavior
```

---

# 137. Legacy DOM Data Storage

Code may use:

```js
element._state = {...};
```

or expando properties.

This can work but creates host/object coupling.

Modern alternatives may include:

```text
WeakMap
dataset
explicit state maps
component state
```

Choose based on ownership and lifetime.

---

# 138. WeakMap Migration

`WeakMap` can associate state with object lifetimes without exposing properties.

Example:

```js
const state = new WeakMap();

state.set(element, {
  active: true
});
```

This can replace some legacy expando patterns.

But it is not a drop-in replacement if code needs enumeration or serialization.

---

# 139. Legacy Timers

Old code may rely on:

```js
setTimeout(fn, 0);
```

as a synchronization mechanism.

Modern event-loop understanding is required before changing this.

A zero-delay timer is not:

```text
immediate
```

It schedules work through the host event loop.

---

# 140. Legacy `setInterval`

Repeated polling:

```js
setInterval(poll, 1000);
```

can cause:

```text
overlap
drift
work after shutdown
uncontrolled concurrency
```

Modern designs may use recursive scheduling or task control.

But do not rewrite mechanically without preserving expected timing.

---

# 141. Legacy Polling to Event-Driven Design

Possible evolution:

```text
polling
→
long polling
→
SSE/WebSocket/events
```

But this changes:

```text
failure model
connection lifetime
backpressure
server load
reconnect behavior
```

Architecture migration is required, not only API migration.

---

# 142. Legacy AJAX Abstraction

Legacy:

```js
function api(url, cb) {
  // wraps XHR
}
```

Before replacing it, document:

```text
headers
credentials
JSON parsing
status handling
retry
timeouts
abort
logging
```

The wrapper may encode important policy.

---

# 143. Legacy Utility Libraries

A project may depend on:

```text
lodash
underscore
custom helpers
```

Many built-ins now cover common functionality.

But removing a utility library requires checking:

```text
deep cloning
equality
path access
collection semantics
security
bundle size
```

Do not replace by name alone.

---

# 144. Utility Library Migration

For each helper:

```text
current semantics
native equivalent
performance
edge cases
browser baseline
```

Then migrate one helper family at a time.

---

# 145. Legacy Deep Clone Helpers

Code may use:

```js
JSON.parse(JSON.stringify(value))
```

as a “deep clone.”

This loses:

```text
undefined
BigInt
Map
Set
Date semantics
cycles
custom prototypes
```

Modern `structuredClone()` has different capabilities and failure behavior.

Migration should be domain-specific.

---

# 146. Legacy Equality Helpers

Old utility libraries often implement deep equality.

Native equality:

```js
a === b
```

is not deep equality.

Migration should define:

```text
same identity
same structure
same serialized value
```

before replacing a helper.

---

# 147. Legacy URL Parsing

Manual parsing:

```js
const [host, path] = value.split("/");
```

is fragile.

Modern platform URL APIs provide structured parsing.

But migration must preserve:

```text
encoding
validation
relative URL behavior
origin handling
```

---

# 148. Legacy Query String Parsing

Old code manually handles:

```text
?x=1&y=2
```

Modern URL APIs can parse query parameters.

Still check:

```text
duplicate keys
encoding
ordering
empty values
array conventions
```

---

# 149. Legacy Module Loaders and Testing

Test runners may use:

```text
require hooks
transpiler registration
custom globals
```

Migration can fail only in tests.

Always test:

```text
source
unit
integration
production build
```

not only local development.

---

# 150. Legacy Environment Mocking

Old tests may set:

```js
global.window = {};
global.document = {};
```

This can hide true host assumptions.

Prefer explicit dependency boundaries and realistic test environments where practical.


---

# 151. Legacy Modernization Drill — Var-To-Const Migration

## Scenario

A production system contains a historical implementation for **var-to-const migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 152. Legacy Modernization Drill — Strict-Mode Introduction

## Scenario

A production system contains a historical implementation for **strict-mode introduction**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 153. Legacy Modernization Drill — Iife-To-Esm Migration

## Scenario

A production system contains a historical implementation for **IIFE-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 154. Legacy Modernization Drill — Global-Namespace Removal

## Scenario

A production system contains a historical implementation for **global-namespace removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 155. Legacy Modernization Drill — Constructor-To-Class Migration

## Scenario

A production system contains a historical implementation for **constructor-to-class migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 156. Legacy Modernization Drill — Prototype Inheritance Review

## Scenario

A production system contains a historical implementation for **prototype inheritance review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 157. Legacy Modernization Drill — Dom0-To-Addeventlistener Migration

## Scenario

A production system contains a historical implementation for **DOM0-to-addEventListener migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 158. Legacy Modernization Drill — Xhr-To-Fetch Migration

## Scenario

A production system contains a historical implementation for **XHR-to-Fetch migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 159. Legacy Modernization Drill — Callback-To-Promise Migration

## Scenario

A production system contains a historical implementation for **callback-to-Promise migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 160. Legacy Modernization Drill — Commonjs-To-Esm Migration

## Scenario

A production system contains a historical implementation for **CommonJS-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 161. Legacy Modernization Drill — Amd-To-Esm Migration

## Scenario

A production system contains a historical implementation for **AMD-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 162. Legacy Modernization Drill — Umd Removal

## Scenario

A production system contains a historical implementation for **UMD removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 163. Legacy Modernization Drill — Browser-Sniffing Removal

## Scenario

A production system contains a historical implementation for **browser-sniffing removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 164. Legacy Modernization Drill — Polyfill Inventory

## Scenario

A production system contains a historical implementation for **polyfill inventory**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 165. Legacy Modernization Drill — Transpiler Target Audit

## Scenario

A production system contains a historical implementation for **transpiler target audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 166. Legacy Modernization Drill — Legacy Babel Cleanup

## Scenario

A production system contains a historical implementation for **legacy Babel cleanup**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 167. Legacy Modernization Drill — Legacy Bundler Migration

## Scenario

A production system contains a historical implementation for **legacy bundler migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 168. Legacy Modernization Drill — Dependency Engine Audit

## Scenario

A production system contains a historical implementation for **dependency engine audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 169. Legacy Modernization Drill — Legacy Test Characterization

## Scenario

A production system contains a historical implementation for **legacy test characterization**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 170. Legacy Modernization Drill — Golden-Output Testing

## Scenario

A production system contains a historical implementation for **golden-output testing**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 171. Legacy Modernization Drill — Semantic Diffing

## Scenario

A production system contains a historical implementation for **semantic diffing**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 172. Legacy Modernization Drill — Codemod Safety

## Scenario

A production system contains a historical implementation for **codemod safety**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 173. Legacy Modernization Drill — Compatibility Bridge Design

## Scenario

A production system contains a historical implementation for **compatibility bridge design**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 174. Legacy Modernization Drill — Canary Rollout

## Scenario

A production system contains a historical implementation for **canary rollout**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 175. Legacy Modernization Drill — Rollback Design

## Scenario

A production system contains a historical implementation for **rollback design**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 176. Legacy Modernization Drill — Security Audit

## Scenario

A production system contains a historical implementation for **security audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 177. Legacy Modernization Drill — Prototype-Pollution Review

## Scenario

A production system contains a historical implementation for **prototype-pollution review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 178. Legacy Modernization Drill — Global-State Removal

## Scenario

A production system contains a historical implementation for **global-state removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 179. Legacy Modernization Drill — Singleton Migration

## Scenario

A production system contains a historical implementation for **singleton migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 180. Legacy Modernization Drill — Date-To-Temporal Classification

## Scenario

A production system contains a historical implementation for **Date-to-Temporal classification**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 181. Legacy Modernization Drill — Serialization Audit

## Scenario

A production system contains a historical implementation for **serialization audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 182. Legacy Modernization Drill — Error-Contract Migration

## Scenario

A production system contains a historical implementation for **error-contract migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 183. Legacy Modernization Drill — Event-Ordering Audit

## Scenario

A production system contains a historical implementation for **event-ordering audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 184. Legacy Modernization Drill — Timer Migration

## Scenario

A production system contains a historical implementation for **timer migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 185. Legacy Modernization Drill — Polling Migration

## Scenario

A production system contains a historical implementation for **polling migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 186. Legacy Modernization Drill — Utility-Library Replacement

## Scenario

A production system contains a historical implementation for **utility-library replacement**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 187. Legacy Modernization Drill — Deep-Clone Replacement

## Scenario

A production system contains a historical implementation for **deep-clone replacement**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 188. Legacy Modernization Drill — Legacy Url Parsing

## Scenario

A production system contains a historical implementation for **legacy URL parsing**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 189. Legacy Modernization Drill — Module-Cycle Analysis

## Scenario

A production system contains a historical implementation for **module-cycle analysis**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 190. Legacy Modernization Drill — Side-Effect Import Analysis

## Scenario

A production system contains a historical implementation for **side-effect import analysis**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 191. Legacy Modernization Drill — Generated-Artifact Debugging

## Scenario

A production system contains a historical implementation for **generated-artifact debugging**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 192. Legacy Modernization Drill — Source-Map Debugging

## Scenario

A production system contains a historical implementation for **source-map debugging**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 193. Legacy Modernization Drill — Browser-Baseline Review

## Scenario

A production system contains a historical implementation for **browser-baseline review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 194. Legacy Modernization Drill — Node-Baseline Review

## Scenario

A production system contains a historical implementation for **Node-baseline review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 195. Legacy Modernization Drill — Library Compatibility

## Scenario

A production system contains a historical implementation for **library compatibility**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 196. Legacy Modernization Drill — Package Exports Migration

## Scenario

A production system contains a historical implementation for **package exports migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 197. Legacy Modernization Drill — Conditional Exports

## Scenario

A production system contains a historical implementation for **conditional exports**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 198. Legacy Modernization Drill — Dual-Build Strategy

## Scenario

A production system contains a historical implementation for **dual-build strategy**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 199. Legacy Modernization Drill — Legacy Browser Exception

## Scenario

A production system contains a historical implementation for **legacy browser exception**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 200. Legacy Modernization Drill — Compatibility-Debt Removal

## Scenario

A production system contains a historical implementation for **compatibility-debt removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 201. Legacy Modernization Drill — Platform Ownership

## Scenario

A production system contains a historical implementation for **platform ownership**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 202. Legacy Modernization Drill — Migration Adr

## Scenario

A production system contains a historical implementation for **migration ADR**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 203. Legacy Modernization Drill — Principal Architecture Review

## Scenario

A production system contains a historical implementation for **principal architecture review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 204. Legacy Modernization Drill — Post-Migration Verification

## Scenario

A production system contains a historical implementation for **post-migration verification**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 205. Legacy Modernization Drill — Legacy Incident Response

## Scenario

A production system contains a historical implementation for **legacy incident response**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 206. Legacy Modernization Drill — Var-To-Const Migration

## Scenario

A production system contains a historical implementation for **var-to-const migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 207. Legacy Modernization Drill — Strict-Mode Introduction

## Scenario

A production system contains a historical implementation for **strict-mode introduction**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 208. Legacy Modernization Drill — Iife-To-Esm Migration

## Scenario

A production system contains a historical implementation for **IIFE-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 209. Legacy Modernization Drill — Global-Namespace Removal

## Scenario

A production system contains a historical implementation for **global-namespace removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 210. Legacy Modernization Drill — Constructor-To-Class Migration

## Scenario

A production system contains a historical implementation for **constructor-to-class migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 211. Legacy Modernization Drill — Prototype Inheritance Review

## Scenario

A production system contains a historical implementation for **prototype inheritance review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 212. Legacy Modernization Drill — Dom0-To-Addeventlistener Migration

## Scenario

A production system contains a historical implementation for **DOM0-to-addEventListener migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 213. Legacy Modernization Drill — Xhr-To-Fetch Migration

## Scenario

A production system contains a historical implementation for **XHR-to-Fetch migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 214. Legacy Modernization Drill — Callback-To-Promise Migration

## Scenario

A production system contains a historical implementation for **callback-to-Promise migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 215. Legacy Modernization Drill — Commonjs-To-Esm Migration

## Scenario

A production system contains a historical implementation for **CommonJS-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 216. Legacy Modernization Drill — Amd-To-Esm Migration

## Scenario

A production system contains a historical implementation for **AMD-to-ESM migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 217. Legacy Modernization Drill — Umd Removal

## Scenario

A production system contains a historical implementation for **UMD removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 218. Legacy Modernization Drill — Browser-Sniffing Removal

## Scenario

A production system contains a historical implementation for **browser-sniffing removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 219. Legacy Modernization Drill — Polyfill Inventory

## Scenario

A production system contains a historical implementation for **polyfill inventory**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 220. Legacy Modernization Drill — Transpiler Target Audit

## Scenario

A production system contains a historical implementation for **transpiler target audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 221. Legacy Modernization Drill — Legacy Babel Cleanup

## Scenario

A production system contains a historical implementation for **legacy Babel cleanup**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 222. Legacy Modernization Drill — Legacy Bundler Migration

## Scenario

A production system contains a historical implementation for **legacy bundler migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 223. Legacy Modernization Drill — Dependency Engine Audit

## Scenario

A production system contains a historical implementation for **dependency engine audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 224. Legacy Modernization Drill — Legacy Test Characterization

## Scenario

A production system contains a historical implementation for **legacy test characterization**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 225. Legacy Modernization Drill — Golden-Output Testing

## Scenario

A production system contains a historical implementation for **golden-output testing**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 226. Legacy Modernization Drill — Semantic Diffing

## Scenario

A production system contains a historical implementation for **semantic diffing**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 227. Legacy Modernization Drill — Codemod Safety

## Scenario

A production system contains a historical implementation for **codemod safety**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 228. Legacy Modernization Drill — Compatibility Bridge Design

## Scenario

A production system contains a historical implementation for **compatibility bridge design**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 229. Legacy Modernization Drill — Canary Rollout

## Scenario

A production system contains a historical implementation for **canary rollout**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 230. Legacy Modernization Drill — Rollback Design

## Scenario

A production system contains a historical implementation for **rollback design**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 231. Legacy Modernization Drill — Security Audit

## Scenario

A production system contains a historical implementation for **security audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 232. Legacy Modernization Drill — Prototype-Pollution Review

## Scenario

A production system contains a historical implementation for **prototype-pollution review**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 233. Legacy Modernization Drill — Global-State Removal

## Scenario

A production system contains a historical implementation for **global-state removal**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 234. Legacy Modernization Drill — Singleton Migration

## Scenario

A production system contains a historical implementation for **singleton migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 235. Legacy Modernization Drill — Date-To-Temporal Classification

## Scenario

A production system contains a historical implementation for **Date-to-Temporal classification**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 236. Legacy Modernization Drill — Serialization Audit

## Scenario

A production system contains a historical implementation for **serialization audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 237. Legacy Modernization Drill — Error-Contract Migration

## Scenario

A production system contains a historical implementation for **error-contract migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 238. Legacy Modernization Drill — Event-Ordering Audit

## Scenario

A production system contains a historical implementation for **event-ordering audit**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 239. Legacy Modernization Drill — Timer Migration

## Scenario

A production system contains a historical implementation for **timer migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 240. Legacy Modernization Drill — Polling Migration

## Scenario

A production system contains a historical implementation for **polling migration**.

Your task is to modernize it without breaking supported behavior.

## Required analysis

```text
1. Why did the legacy pattern exist?
2. Is it still needed?
3. What exact semantics does it provide?
4. Which environments depend on it?
5. What public behavior must remain stable?
6. What hidden side effects exist?
7. What tests currently protect it?
8. What characterization tests are missing?
9. What is the safest replacement?
10. What compatibility bridge is needed?
11. What can be automated?
12. What requires manual review?
13. What security risks exist?
14. What performance risks exist?
15. What memory risks exist?
16. How will rollout happen?
17. How will rollback happen?
18. How will usage be observed?
19. When can the compatibility layer be removed?
20. Who owns the removal?
```

## Required output

```text
Historical reason:
Current dependency:
Risk:
Behavioral contract:
Characterization tests:
Target design:
Migration boundary:
Compatibility bridge:
Rollout:
Rollback:
Observability:
Removal trigger:
Owner:
```

## Principal challenge

Defend both sides:

```text
Case A — modernize now
Case B — retain temporarily
```

Then identify the evidence that decides between them.

---

# 241. Implementation — Guided Legacy Inventory

Build a Node.js scanner.

Input:

```text
repository/
```

Output:

```text
signal
file
line
count
```

Search for:

```text
var
eval
with
XMLHttpRequest
onclick
require
module.exports
document.write
navigator.userAgent
Object.prototype
Array.prototype
```

Do not count occurrences inside comments and string literals without understanding the parsing problem.

---

# 242. Implementation — Characterization Harness

Create:

```js
function capture(fn, input) {
  try {
    return {
      ok: true,
      value: fn(input)
    };
  } catch (error) {
    return {
      ok: false,
      error: {
        name: error?.name,
        message: error?.message
      }
    };
  }
}
```

Extend it to capture:

```text
return
throw
duration
side-effect summary
serialization
```

Use this to compare old and new implementations.

---

# 243. Implementation — Semantic Comparator

Implement:

```js
function compareResults(oldResult, newResult) {
  // classify:
  // equal
  // intentional difference
  // regression
  // unsupported legacy behavior
}
```

The comparator must not assume that:

```text
different output
=
bug
```

Some migrations intentionally correct undefined or unsafe behavior.

---

# 244. Implementation — Compatibility Bridge

Create:

```js
export function legacyApi(...args) {
  return modernApi(...args);
}
```

Add:

```text
usage count
error count
caller information where appropriate
deprecation notice
```

The bridge should be removable.

---

# 245. Implementation — Codemod Plan

Design codemods for:

```text
simple var declarations
deprecated API renames
require/import conversions
legacy property access
known utility replacements
```

For each transformation specify:

```text
safe case
unsafe case
manual review case
test requirement
rollback
```

---

# 246. Implementation — No-Reference Migration

Select one 300–1000 line legacy module.

Perform:

```text
inventory
characterization
modernization
compatibility bridge
tests
canary
deprecation
```

Produce a migration report.

---

# 247. Implementation — Edge-Case Hardened

Your migration tooling must handle:

```text
comments
strings
generated files
vendored files
minified code
dynamic require
circular imports
side effects
conditional loading
host globals
```

Do not accidentally modify generated artifacts or third-party code.

---

# 248. Implementation — Production Grade

Design:

```text
legacy-inventory/
compatibility/
migration/
codemods/
tests/
reports/
```

Every migrated item should have:

```text
owner
baseline
target
risk
test
rollout
rollback
removal date
```

---

# 249. Debugging Exercise — `var` Loop

Predict:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

Explain:

```text
binding creation
closure capture
event-loop timing
```

Then modernize with `let`.

---

# 250. Debugging Exercise — Arrow Function

Given:

```js
const obj = {
  value: 42,

  getValue: function () {
    return this.value;
  }
};
```

Explain why changing only the function token to:

```js
getValue: () => this.value
```

can break behavior.

---

# 251. Debugging Exercise — Callback Timing

Compare:

```js
function a(cb) {
  cb();
}

function b() {
  return Promise.resolve();
}
```

Explain why converting one to the other can change observable ordering.

---

# 252. Debugging Exercise — Module Side Effects

Given:

```js
require("./register");
```

and:

```js
import { register } from "./register.js";
```

explain how a migration could accidentally remove initialization side effects.

---

# 253. Debugging Exercise — Circular Dependency

Given:

```text
A → B → A
```

identify why CJS→ESM migration requires cycle analysis.

---

# 254. Code Review Exercise — Blanket Arrow Conversion

Reject:

```text
replace all function expressions with arrows
```

because this can change:

```text
this
arguments
constructability
prototype
call semantics
```

---

# 255. Code Review Exercise — Blanket `var` Replacement

Reject:

```text
replace every var with const
```

because this ignores:

```text
reassignment
scope
closure
hoisting
redeclaration
loop behavior
global behavior
```

---

# 256. Code Review Exercise — Blanket XHR Replacement

Reject:

```text
replace every XHR with fetch
```

until these are classified:

```text
progress
abort
timeout
credentials
status handling
response type
streaming
```

---

# 257. Code Review Exercise — Delete Browser Hack

Required evidence:

```text
support baseline
actual telemetry
historical reason
target environment test
customer contract
```

Only then remove.

---

# 258. Interview Questions — Fundamentals

1. What is legacy JavaScript?
2. Is old syntax necessarily bad?
3. Why did IIFEs exist?
4. Why was CommonJS important?
5. What did UMD solve?
6. Why were global namespaces common?
7. Why is `var` still valid?
8. What does strict mode change?
9. What is `arguments`?
10. Why is `with` problematic?
11. Why is `eval` risky?
12. What is prototype inheritance?

---

# 259. Interview Questions — Senior

1. How would you migrate `var`?
2. How would you migrate constructor functions?
3. How would you migrate XHR?
4. How would you migrate callbacks?
5. How would you migrate CJS to ESM?
6. How would you remove a global namespace?
7. How would you remove a browser-detection workaround?
8. How would you remove a polyfill?
9. How would you modernize a ten-year-old build?
10. How would you preserve public API compatibility?

---

# 260. Interview Questions — Principal

1. How would you modernize a ten-year-old frontend without a rewrite?
2. How do you prove behavioral equivalence?
3. How do you prioritize legacy debt?
4. How do you distinguish technical debt from intentional compatibility?
5. How do you migrate a popular npm package?
6. How do you migrate CJS/AMD/UMD consumers?
7. How do you design rollback?
8. How do you handle compatibility exceptions?
9. How do you measure modernization ROI?
10. When should a legacy system intentionally remain legacy?

---

# 261. Predict-the-Output Exercise

```js
function demo() {
  if (true) {
    var x = 10;
  }

  return x;
}

console.log(demo());
```

Predict first.

Expected:

```text
10
```

---

# 262. Predict-the-Output Exercise

```js
function demo(value) {
  value = 20;
  return arguments[0];
}

console.log(demo(10));
```

Predict first.

The result depends on strictness/legacy argument semantics.

Explain the mode before predicting.

---

# 263. Predict-the-Output Exercise

```js
const jobs = [];

for (var i = 0; i < 3; i++) {
  jobs.push(() => i);
}

console.log(jobs.map(fn => fn()));
```

Expected:

```text
[3, 3, 3]
```

---

# 264. Predict-the-Output Exercise

```js
const jobs = [];

for (let i = 0; i < 3; i++) {
  jobs.push(() => i);
}

console.log(jobs.map(fn => fn()));
```

Expected:

```text
[0, 1, 2]
```

---

# 265. Mastery Exercise — Legacy Classification

Select 20 legacy constructs and classify each:

```text
Old but healthy
Legacy but necessary
Legacy and risky
Deprecated
Obsolete
Security-sensitive
Migration candidate
```

Defend every classification.

---

# 266. Mastery Exercise — `Date` Migration

Take 20 date usages and classify:

```text
Instant
PlainDate
PlainTime
PlainDateTime
ZonedDateTime
Duration
not actually date-domain logic
```

Then migrate five with tests.

---

# 267. Mastery Exercise — Module Migration

Choose a CommonJS package.

Document:

```text
entry points
exports
cycles
side effects
dynamic requires
package.json
consumers
```

Then build an incremental ESM migration.

---

# 268. Mastery Exercise — Browser Modernization

Choose a legacy browser application.

Inventory:

```text
global variables
inline handlers
browser sniffing
XHR
polyfills
script ordering
```

Design a migration plan.

---

# 269. Mastery Exercise — Security

Perform a legacy security audit.

Find:

```text
dynamic code
unsafe HTML
global mutation
prototype mutation
dynamic URL creation
inline script
```

Produce:

```text
risk
data flow
mitigation
test
owner
```

---

# 270. Mastery Exercise — Build Modernization

Choose a legacy build pipeline.

Measure:

```text
build duration
bundle size
cache hit rate
source-map quality
failure rate
developer pain
security risk
```

Then compare:

```text
keep
refactor
replace
```

---

# 271. Mastery Exercise — Compatibility Debt

Create:

```text
compatibility debt register
```

For each entry include:

```text
reason
environment
risk
owner
test
fallback
review date
removal trigger
```

---

# 272. Mastery Exercise — Principal Migration Memo

Write:

```text
Context
Current architecture
Historical constraints
Current risks
Target architecture
Migration sequence
Compatibility strategy
Security strategy
Performance strategy
Test strategy
Observability
Rollback
Cost
Timeline
Decision
```

Your final recommendation must explain:

```text
why now
why this sequence
why not a rewrite
```

---

# 273. Spaced Retrieval Schedule

### Day 0

Explain:

```text
old
legacy
deprecated
obsolete
dangerous
```

### Day 1

Explain:

```text
var
arguments
strict/sloppy
this
```

### Day 3

Explain:

```text
IIFE
prototype constructors
global namespaces
```

### Day 7

Explain:

```text
XHR
callbacks
CommonJS
AMD
UMD
```

### Day 14

Perform a legacy inventory.

### Day 30

Perform a characterization-driven refactor.

### Day 60

Defend a modernization architecture.

### Day 90

Design a legacy retirement program for a large organization.

---

# 274. Retrieval Prompts

Answer without notes:

```text
Why does legacy JavaScript persist?
Why is old code not automatically bad?
What changes when var becomes let?
What changes with arrow functions?
Why is arguments different?
Why is with problematic?
Why is eval dangerous?
Why were IIFEs useful?
Why were prototype constructors common?
Why was CommonJS important?
What did UMD solve?
Why can XHR→fetch change behavior?
Why can callback→Promise change timing?
Why can CJS→ESM change cycles?
Why are characterization tests important?
What is a compatibility bridge?
What is a strangler migration?
How do you retire compatibility debt?
```

---

# 275. Dependency Graph

```text
Chapter 05 — Variables
        ↓
Chapter 10 — Scope
        ↓
Chapter 11 — Hoisting / TDZ
        ↓
Chapter 14 — this
        ↓
Chapter 17 — Prototypes
        ↓
Chapter 18 — Classes
        ↓
Chapter 31/35/36 — Async / Promise / Async Await
        ↓
Chapter 55 — Fetch
        ↓
Chapter 64/65 — ESM / CommonJS
        ↓
Chapter 68/69/70 — Tooling
        ↓
Chapter 88/89 — Debugging / Refactoring
        ↓
Chapter 94 — Compatibility Engineering
        ↓
Chapter 95 — Legacy JavaScript
        ↓
Chapter 96 — WebAssembly / Native Interoperability
```

---

# 276. Concept Connections

## Depends On

- scope
- hoisting
- closures
- `this`
- prototypes
- modules
- async execution
- browser APIs
- Node APIs
- transpilation
- bundling
- compatibility
- debugging
- refactoring

## Builds Toward

- native interoperability
- edge/runtime differences
- production migration
- large-scale platform modernization

## Related Concepts

- technical debt
- backwards compatibility
- deprecation
- codemods
- strangler architecture
- canary release
- characterization testing
- semantic equivalence

## Concepts Revisited

- language vs host
- standard vs implementation
- runtime behavior
- compatibility
- performance
- security
- error handling

## Why This Chapter Matters Later

Principal engineers inherit systems.

They rarely start from a blank repository.

Modernization skill means understanding the old contract well enough to replace it safely.

---

# 277. Principal Decision Framework

For every modernization candidate, evaluate:

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

Then ask:

```text
What does the legacy pattern buy us?
What does it cost us?
Who depends on it?
What happens if we remove it?
What is the smallest safe migration boundary?
What evidence is required before rollout?
What is the rollback?
When does the compatibility layer disappear?
```

---

# 278. Production Checklist

```text
[ ] Historical reason understood
[ ] Current dependency understood
[ ] Supported environments verified
[ ] Public contract documented
[ ] Characterization tests added
[ ] Target semantics defined
[ ] Security reviewed
[ ] Performance measured
[ ] Memory impact reviewed
[ ] Compatibility path defined
[ ] Rollout plan defined
[ ] Rollback plan defined
[ ] Observability added
[ ] Owner assigned
[ ] Deprecation plan defined
[ ] Removal trigger defined
```

---

# 279. Legacy Migration ADR Template

```md
# ADR — Legacy JavaScript Migration

## Context
-

## Historical Constraint
-

## Current Problem
-

## Existing Contract
-

## Options
-

## Decision
-

## Compatibility Impact
-

## Security Impact
-

## Performance Impact
-

## Testing
-

## Rollout
-

## Rollback
-

## Deprecation
-

## Removal
-
```

---

# 280. Final Mental Model

```text
Legacy code
    ↓
historical constraint
    ↓
observable semantics
    ↓
current requirements
    ↓
risk analysis
    ↓
characterization
    ↓
target design
    ↓
compatibility bridge
    ↓
incremental rollout
    ↓
observability
    ↓
deprecation
    ↓
removal
```

The central rule is:

> **Modernize behavior deliberately, not syntax mechanically.**

---

# 281. Final Principal Rule

A junior asks:

> “How do I make this code modern?”

A senior asks:

> “What is the safer replacement?”

A principal asks:

> **“Which historical assumptions still matter, what contract must remain stable, what evidence proves the new system is equivalent enough, and how do we retire the old path without creating a larger operational risk?”**

That is legacy engineering.

---

# Chapter 95 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I define legacy precisely? [ ]
- Could I distinguish old from deprecated? [ ]
- Could I explain var/let/const? [ ]
- Could I explain strict/sloppy mode? [ ]
- Could I explain arguments? [ ]
- Could I explain this migration hazards? [ ]
- Could I explain IIFE/module history? [ ]
- Could I explain prototype constructors? [ ]
- Could I explain XHR/callback migration? [ ]
- Could I explain CJS/AMD/UMD? [ ]
- Could I design characterization tests? [ ]
- Could I design a compatibility bridge? [ ]
- Could I design a strangler migration? [ ]
- Could I defend a principal migration decision? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 95 — Canonical References and Source Discipline

Primary references:

1. **ECMAScript Language Specification**
   https://tc39.es/ecma262/

2. **ECMAScript Strict Mode**
   https://tc39.es/ecma262/#sec-strict-mode-code

3. **MDN — `var`**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var

4. **MDN — `let`**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let

5. **MDN — `const`**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const

6. **MDN — `arguments`**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments

7. **MDN — Strict mode**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode

8. **MDN — `eval()`**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval

9. **MDN — `Function()` constructor**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/Function

10. **MDN — Fetch API**
    https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

11. **MDN — XMLHttpRequest**
    https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest

12. **Node.js Modules**
    https://nodejs.org/api/modules.html

13. **Node.js ECMAScript Modules**
    https://nodejs.org/api/esm.html

14. **Node.js Documentation**
    https://nodejs.org/docs/

15. **TC39 Process**
    https://tc39.es/process-document/

16. **MDN JavaScript**
    https://developer.mozilla.org/en-US/docs/Web/JavaScript

Source discipline:

```text
Historical language semantics
→ ECMAScript specification

Browser behavior
→ Web platform documentation + actual target-browser tests

Node behavior
→ Node.js documentation + actual target-runtime tests

Migration equivalence
→ characterization tests + production evidence

Security
→ data-flow analysis + threat model

Performance
→ benchmark/profile evidence
```

---

# Chapter 95 — Completion Snapshot

```text
Part XVIII — Legacy / Interoperability

Chapter 95 — Legacy JavaScript
[ ] Not Started

Track A — Core Theory
[ ] Historical JavaScript semantics
[ ] var / let / const
[ ] strict / sloppy mode
[ ] arguments
[ ] this
[ ] with
[ ] eval
[ ] IIFEs
[ ] prototypes
[ ] DOM0 events
[ ] XHR
[ ] callbacks
[ ] CommonJS
[ ] AMD
[ ] UMD
[ ] globals
[ ] polyfills
[ ] legacy toolchains

Track B — Implementation
[ ] Legacy inventory
[ ] Characterization tests
[ ] Semantic comparator
[ ] Compatibility bridge
[ ] Codemod strategy
[ ] Incremental migration
[ ] Canary
[ ] Rollback
[ ] Security audit
[ ] Production modernization dashboard

Track C — Interview / Reasoning
[ ] Explain legacy accurately
[ ] Defend a migration sequence
[ ] Explain semantic hazards
[ ] Design compatibility boundaries
[ ] Design public-library migration
[ ] Design browser modernization
[ ] Design runtime modernization
[ ] Defend rollback
[ ] Defend keeping legacy temporarily

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

---

# Completion Criteria

Do not mark this chapter mastered because you know historical syntax.

You are ready to move forward when you can independently:

1. Define legacy JavaScript precisely.
2. Explain why legacy code persists.
3. Distinguish old from deprecated and dangerous.
4. Explain `var`, `let`, and `const`.
5. Explain strict/sloppy semantics.
6. Explain `arguments`.
7. Explain `this` migration hazards.
8. Explain IIFEs and module migration.
9. Explain constructor/prototype/class differences.
10. Explain DOM0 and event-listener migration.
11. Explain XHR versus Fetch.
12. Explain callback versus Promise migration.
13. Explain CommonJS/ESM interoperability.
14. Explain AMD and UMD historically.
15. Identify browser-detection hacks.
16. Build characterization tests.
17. Build compatibility bridges.
18. Plan an incremental migration.
19. Design rollback.
20. Measure migration impact.
21. Audit legacy code for security.
22. Audit legacy code for performance.
23. Manage compatibility debt.
24. Defend a modernization program at principal level.

---

# Current-Context Note — 2026-09-10

This chapter is primarily about historical and semantic engineering, so most concepts are stable language/runtime knowledge.

For time-sensitive migration decisions, always re-check:

```text
ECMAScript specification
Node.js supported versions
browser support
package engine requirements
toolchain versions
dependency support
security advisories
```

Never use the presence of an old pattern as proof that its original environment remains supported.