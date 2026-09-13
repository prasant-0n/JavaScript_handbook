# Chapter 147 — Package `exports`, Conditional Exports & Modern Resolution

> **JavaScript Mastery — Part XXVII: Modules, Packaging, Resolution & Distribution**
>
> **Mission:** Master modern Node.js package resolution at the boundary between JavaScript modules, package metadata, CommonJS, ECMAScript Modules, tooling, bundlers, runtimes, and published package contracts. Understand `exports`, subpath exports, conditional exports, patterns, `null` targets, `imports`, self-referencing, condition ordering, package encapsulation, dual packages, `module-sync`, `node-addons`, custom conditions, resolution algorithms, extension discipline, monorepos, workspaces, TypeScript interop, bundler conditions, browser conditions, dependency hazards, compatibility strategy, package migration, and principal-level distribution decisions.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Module Systems Engineer · Package Author · Runtime Engineer · Build Engineer · Library Maintainer · Supply-Chain Security Engineer · Monorepo Architect · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Core principle:** **Package resolution is part of your public API. A package does not merely contain files; it exposes a deliberate module graph. `exports` controls that boundary, conditional exports make that boundary environment-sensitive, and modern resolution turns package metadata into executable compatibility policy.**

---

# 1. Learning Objectives

```text
[ ] explain package resolution
[ ] explain package entry points
[ ] explain main
[ ] explain exports
[ ] explain imports
[ ] explain type
[ ] explain name
[ ] explain self-referencing
[ ] explain package encapsulation
[ ] explain subpath exports
[ ] explain subpath imports
[ ] explain conditional exports
[ ] explain nested conditions
[ ] explain condition ordering
[ ] explain default condition
[ ] explain node condition
[ ] explain node-addons condition
[ ] explain import condition
[ ] explain require condition
[ ] explain module-sync condition
[ ] explain custom conditions
[ ] explain user conditions
[ ] explain condition matching
[ ] explain exact export keys
[ ] explain wildcard patterns
[ ] explain pattern substitution
[ ] explain null targets
[ ] explain package target restrictions
[ ] explain path traversal restrictions
[ ] explain node_modules restrictions
[ ] explain extensioned subpaths
[ ] explain extensionless subpaths
[ ] understand import maps concepts
[ ] understand package maps
[ ] understand ESM resolution
[ ] understand CommonJS resolution
[ ] understand package self-resolution
[ ] understand bare specifiers
[ ] understand relative specifiers
[ ] understand absolute specifiers
[ ] understand package scope
[ ] understand node_modules lookup
[ ] understand package.json boundary
[ ] understand file extension rules
[ ] understand package type
[ ] understand .js
[ ] understand .mjs
[ ] understand .cjs
[ ] understand directory imports
[ ] understand package imports
[ ] understand package exports
[ ] understand package conditions
[ ] understand resolution context
[ ] understand import vs require selection
[ ] understand condition priority
[ ] understand fallback behavior
[ ] understand ERR_PACKAGE_PATH_NOT_EXPORTED
[ ] understand ERR_PACKAGE_IMPORT_NOT_DEFINED
[ ] understand ERR_MODULE_NOT_FOUND
[ ] understand invalid package target behavior
[ ] understand dual package hazards
[ ] understand module identity duplication
[ ] understand state duplication
[ ] understand singleton duplication
[ ] understand two-loader hazards
[ ] understand ESM/CJS interop
[ ] understand require(ESM)
[ ] understand import(CJS)
[ ] understand module-sync
[ ] understand top-level await constraints
[ ] understand node-addons
[ ] understand native addon fallback
[ ] understand WebAssembly fallback
[ ] understand browser/default strategies
[ ] understand universal defaults
[ ] understand custom environment conditions
[ ] understand condition collisions
[ ] understand condition naming
[ ] understand package ecosystem conventions
[ ] understand why unknown conditions can be ignored
[ ] understand condition portability
[ ] understand nested condition semantics
[ ] understand package target arrays
[ ] understand fallback target arrays
[ ] understand invalid target fallback
[ ] understand export key precedence
[ ] understand exact match vs pattern
[ ] understand wildcard specificity
[ ] understand package metadata as API
[ ] understand semver effects of exports
[ ] identify breaking exports changes
[ ] migrate main to exports
[ ] preserve legacy subpaths
[ ] design public package surface
[ ] hide private internals
[ ] expose feature subpaths
[ ] expose types-related assets safely
[ ] expose package.json intentionally
[ ] design ESM-only packages
[ ] design CJS-only packages
[ ] design dual packages
[ ] design conditional packages
[ ] design native-enhanced packages
[ ] design universal fallback packages
[ ] design environment-aware packages
[ ] design monorepo package boundaries
[ ] design workspace package boundaries
[ ] design internal aliases
[ ] use package imports
[ ] understand # specifiers
[ ] understand internal mappings
[ ] compare imports and exports
[ ] compare main and exports
[ ] compare aliases and relative paths
[ ] compare runtime resolution and bundler resolution
[ ] compare Node conditions and browser conditions
[ ] compare type conditions and runtime conditions
[ ] understand TypeScript resolution interaction
[ ] understand bundler resolution interaction
[ ] understand test-runner resolution interaction
[ ] understand IDE/editor resolution
[ ] maintain cross-tool compatibility
[ ] understand conditional export testing
[ ] test every intended branch
[ ] avoid unreachable branches
[ ] avoid condition shadowing
[ ] avoid default-branch mistakes
[ ] avoid import/require divergence
[ ] avoid state duplication
[ ] avoid deep-import breakage
[ ] avoid accidental public APIs
[ ] avoid accidental package encapsulation breakage
[ ] avoid extension inconsistency
[ ] understand package migration strategies
[ ] build compatibility matrix
[ ] build resolution test matrix
[ ] build package verification scripts
[ ] inspect resolved paths
[ ] use import.meta.resolve where appropriate
[ ] use require.resolve where appropriate
[ ] understand limitations of resolution probes
[ ] test dynamic import
[ ] test createRequire
[ ] test self-reference
[ ] test package imports
[ ] test conditional exports
[ ] test CJS
[ ] test ESM
[ ] test native branches
[ ] test default branches
[ ] test custom conditions
[ ] test unsupported conditions
[ ] test missing subpaths
[ ] test package encapsulation
[ ] test symlink/workspace behavior
[ ] test npm-packed artifact
[ ] test published artifact
[ ] test package tarball
[ ] understand files included in package
[ ] understand source-vs-dist layouts
[ ] understand package exports with dist
[ ] understand package manager effects
[ ] understand peer dependency boundaries
[ ] understand optional native dependencies
[ ] understand optional branches
[ ] understand package supply-chain risk
[ ] understand target validation
[ ] understand path traversal protection
[ ] understand dependency confusion implications
[ ] understand package spoofing risks
[ ] understand custom condition risks
[ ] understand condition injection through tooling
[ ] understand build-time vs runtime conditions
[ ] understand extension policy
[ ] understand semver and public entry points
[ ] understand deprecation paths
[ ] understand compatibility bridges
[ ] understand migration from deep imports
[ ] build a package exports matrix
[ ] build a package resolver mental model
[ ] implement simplified exports resolver
[ ] implement simplified conditional resolver
[ ] implement subpath matcher
[ ] implement condition matcher
[ ] debug resolution failures
[ ] review package metadata
[ ] defend resolution architecture in interviews
[ ] make principal distribution decisions


# 2. Prerequisites

You should already understand:

```text
Chapter 30 — JavaScript Modules
Chapter 49 — ESM / CommonJS
Chapter 126 — Module Linking, Resolution & Async Module Evaluation
Chapter 147 is the packaging/distribution-level continuation of those topics.
```

Also understand:

```text
package.json
npm packages
node_modules
CommonJS
ES Modules
file extensions
dynamic import
require
module caching
TypeScript at a conceptual level
bundlers
monorepos.
```

---

# 3. What Is Package Resolution?

Package resolution answers:

```text
“Given this module specifier and this execution context,
which module file should be loaded?”
```

Examples:

```js
import x from "my-package";
import y from "my-package/utils";
import z from "#internal";
```

The resolver must determine:

```text
package
package boundary
exports/imports map
condition set
target
format
file.
```

---

# 4. What Is an Entry Point?

An entry point is:

```text
supported package interface
```

such as:

```text
my-package
my-package/client
my-package/server
```

The package should treat these as:

```text
public API commitments.
```

---

# 5. `main` vs `exports`

`main`:

```text
one primary package entry point
```

`exports`:

```text
multiple public entry points
conditional entry points
encapsulation
modern package interface.
```

Current Node documentation recommends `exports` for new packages targeting supported Node versions. When both are present, `exports` takes precedence over `main`. citeturn275170search0

---

# 6. Why `exports` Exists

Without `exports`, consumers may reach:

```text
internal files
```

such as:

```text
pkg/dist/internal.js
pkg/src/parser.js
pkg/lib/private.js
```

Those deep imports create:

```text
accidental API contracts.
```

`exports` lets a package say:

```text
“These are the files consumers are allowed to address by package specifier.”
```

---

# 7. Package Encapsulation

With:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

a consumer can use:

```js
import pkg from "my-package";
```

but:

```js
import internal from "my-package/dist/internal.js";
```

fails because the subpath is not exported.

Node reports:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED
```

for an unavailable package subpath. citeturn275170search0

---

# 8. Encapsulation Is API Design

An `exports` map defines:

```text
public boundary.
```

A private file can still physically exist.

The point is:

```text
not every physical file
=
supported package interface.
```

---

# 9. Encapsulation Is Not Filesystem Security

`exports` is not:

```text
cryptographic access control.
```

Node documentation explicitly notes that direct absolute filesystem paths can still load package files despite package export encapsulation. citeturn275170search0

Therefore:

```text
exports
=
module-specifier API boundary

not
=
filesystem sandbox.
```

---

# 10. Minimal `exports`

```json
{
  "name": "my-package",
  "exports": "./dist/index.js"
}
```

This is shorthand for:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

The sugar is supported by Node. citeturn275170search0

---

# 11. Multiple Subpaths

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./client": "./dist/client.js",
    "./server": "./dist/server.js"
  }
}
```

Public surface:

```text
my-package
my-package/client
my-package/server
```

---

# 12. Hidden Internals

Physical files:

```text
dist/
  index.js
  client.js
  server.js
  internal/
    cache.js
    protocol.js
```

can remain:

```text
internal.
```

Do not export:

```text
./internal/cache.js
```

unless it is truly:

```text
public API.
```

---

# 13. Breaking Change: Adding `exports`

An existing package may have historically supported:

```text
pkg
pkg/lib
pkg/lib/utils
pkg/package.json
```

Adding:

```text
exports
```

without preserving those entry points can be:

```text
breaking.
```

Node documentation warns that adding `exports` can prevent previously reachable subpaths from working. citeturn275170search0

---

# 14. Migration Strategy

Before adding `exports`, inventory:

```text
documented imports
popular deep imports
internal package consumers
tests
examples
ecosystem usage.
```

Then:

```text
export every intentionally supported path
```

before:

```text
restricting the surface.
```

---

# 15. Public API Inventory

Create:

```text
root
subpaths
conditions
formats
types
```

matrix.

Example:

```text
.             ESM + CJS
./client      ESM
./server      ESM
./package.json explicit
```

---

# 16. Exact Subpath Export

```json
{
  "exports": {
    "./client": "./dist/client.js"
  }
}
```

Consumers use:

```js
import "my-package/client";
```

not:

```js
import "my-package/dist/client.js";
```

---

# 17. Extensioned Subpaths

You can publish:

```text
my-package/client.js
```

instead of:

```text
my-package/client
```

Example:

```json
{
  "exports": {
    "./client.js": "./dist/client.js"
  }
}
```

Extension discipline is an ecosystem design choice.

---

# 18. Extensionless vs Extensioned

Extensionless:

```text
pkg/client
```

Pros:

```text
short
clean
```

Cons:

```text
less explicit
tooling behavior can vary
future coexistence with similarly named assets can be harder.
```

Extensioned:

```text
pkg/client.js
```

Pros:

```text
explicit
closer to URL-like specifiers
can reduce ambiguity.
```

Choose a consistent policy.

---

# 19. Node's Export Targets

Export target paths must:

```text
start with ./
```

and target files within the package boundary. Node enforces restrictions against path traversal and `node_modules` segments in targets. citeturn275170search0

---

# 20. Invalid Export Target

Conceptually invalid:

```json
{
  "exports": {
    ".": "../shared/index.js"
  }
}
```

because:

```text
target leaves package root.
```

---

# 21. Why Target Restrictions Exist

They provide:

```text
predictability
encapsulation
package-local resolution
security against arbitrary target escapes.
```

---

# 22. `node_modules` Restriction

Export targets cannot be used to:

```text
escape through node_modules paths
```

inside the target mapping.

This keeps:

```text
package export graph
```

bounded to:

```text
package contents.
```

---

# 23. Pattern Exports

Patterns support wildcard mappings.

Example:

```json
{
  "exports": {
    "./features/*.js": "./src/features/*.js"
  }
}
```

This lets:

```text
feature-a.js
feature-b.js
```

map through:

```text
same wildcard segment.
```

Node supports export patterns. citeturn275170search0

---

# 24. Pattern Substitution

Conceptually:

```text
specifier:
./features/a.js

pattern:
./features/*.js

target:
./src/features/*.js

*
=
a
```

Then target becomes:

```text
./src/features/a.js
```

---

# 25. Patterns Are Not Directory Wildcards Alone

The wildcard is:

```text
string replacement
```

within the pattern semantics.

It is not:

```text
general filesystem glob execution
```

in the ordinary shell sense.

---

# 26. Statically Enumerable Exports

Node documents package exports patterns as statically enumerable, allowing export sets to be reasoned about from package contents. citeturn275170search0

This supports:

```text
tooling
analysis
API discovery.
```

---

# 27. Excluding Patterned Internals

You can use:

```json
{
  "exports": {
    "./features/*.js": "./src/features/*.js",
    "./features/private/*": null
  }
}
```

to exclude a private region from a broader pattern. citeturn275170search0

---

# 28. `null` Export Targets

`null` means:

```text
intentionally unavailable.
```

Useful for:

```text
pattern exclusion
deprecated private branches
blocked internals.
```

---

# 29. Public + Private Pattern

```text
./features/*.js
        +
./features/private/* → null
```

Mental model:

```text
allow broad class
→ explicitly deny sensitive subset.
```

---

# 30. Conditionals

Conditional exports choose targets based on:

```text
resolution context.
```

Example:

```json
{
  "exports": {
    "import": "./dist/index.js",
    "require": "./dist/index.cjs"
  }
}
```

Node supports conditional exports for both ESM and CommonJS entry points. citeturn275170search0

---

# 31. Why Conditional Exports Exist

Different environments may need:

```text
ESM
CJS
native addon
universal implementation
development build
production build
```

Conditional exports allow:

```text
one package specifier
```

to map to:

```text
different implementations.
```

---

# 32. Condition Ordering

Condition object order matters.

Earlier conditions:

```text
higher priority.
```

Later conditions:

```text
lower priority.
```

Therefore:

```json
{
  "exports": {
    "node": "...",
    "default": "..."
  }
}
```

is not equivalent in behavior to:

```json
{
  "exports": {
    "default": "...",
    "node": "..."
  }
}
```

Node documents condition ordering as significant. citeturn275170search0

---

# 33. Most Specific to Least Specific

A practical ordering principle:

```text
more specific
    ↓
less specific
    ↓
default
```

---

# 34. `"default"`

`default` is the generic fallback.

Node recommends putting:

```text
default
```

last.

This allows unknown environments to consume:

```text
universal fallback
```

instead of:

```text
pretending to be a known environment.
```

citeturn275170search0

---

# 35. `"node"`

`node` matches:

```text
Node.js environments.
```

But do not use it automatically when:

```text
a universal default
```

is better.

---

# 36. `"import"`

`import` matches resolution through:

```text
import
import()
```

and related ESM loader resolution.

It is mutually exclusive with:

```text
require.
```

Node documents these conditions and their context. citeturn275170search0

---

# 37. `"require"`

`require` matches:

```text
require()
```

resolution.

It is mutually exclusive with:

```text
import.
```

The referenced target needs to be compatible with the applicable loading path. citeturn275170search0

---

# 38. `"node-addons"`

`node-addons` targets Node environments where:

```text
native C++ addons
```

are intended.

It can be disabled with:

```bash
--no-addons
```

Node recommends using `default` as a more universal fallback when native addons are an enhancement. citeturn275170search0

---

# 39. Native Enhancement Pattern

```json
{
  "exports": {
    "node-addons": "./dist/native.js",
    "default": "./dist/portable.js"
  }
}
```

Mental model:

```text
native fast path
↓
portable fallback.
```

---

# 40. `"module-sync"`

Current Node supports:

```text
module-sync
```

which matches regardless of whether loading occurs through:

```text
import
import()
require
```

provided the targeted ESM graph contains no top-level await.

If a `require()` path reaches a graph with top-level await, Node can throw:

```text
ERR_REQUIRE_ASYNC_MODULE.
```

citeturn275170search0

---

# 41. Why `module-sync` Matters

It can represent:

```text
one synchronous ESM-compatible implementation
```

across:

```text
import
require.
```

But the:

```text
no top-level await
```

constraint is part of the contract.

---

# 42. Top-Level Await Hazard

Suppose:

```text
exports
→ module-sync
→ module A
→ module B
→ top-level await
```

Then:

```text
require()
```

cannot safely synchronously load the graph.

Therefore:

```text
module-sync
```

requires:

```text
synchronous module graph.
```

---

# 43. Nested Conditions

Node supports:

```json
{
  "exports": {
    "node": {
      "import": "./node.mjs",
      "require": "./node.cjs"
    },
    "default": "./universal.mjs"
  }
}
```

Conditions behave conceptually like:

```text
nested if statements.
```

citeturn275170search0

---

# 44. Nested Condition Trace

```text
node?
 ├─ yes
 │   ├─ import?
 │   │   └─ node.mjs
 │   └─ require?
 │       └─ node.cjs
 └─ no
     └─ default
```

---

# 45. Condition Tree Mental Model

Think:

```text
specifier
   ↓
exact/pattern match
   ↓
condition tree
   ↓
first valid target
   ↓
target path
```

---

# 46. Condition Arrays

Exports targets can use fallback arrays in supported package-map forms.

Conceptually:

```json
{
  "exports": {
    ".": [
      "./dist/optional.js",
      "./dist/fallback.js"
    ]
  }
}
```

A resolver can move through fallback targets according to package resolution semantics.

Always verify exact behavior for the Node version and loader involved.

---

# 47. Conditions Are Not Booleans

The resolver is not evaluating:

```js
if ("node") ...
```

in JavaScript.

It is performing:

```text
ordered condition matching
```

against:

```text
active condition set.
```

---

# 48. Custom Conditions

Node supports:

```bash
node --conditions=development app.js
```

which activates:

```text
development
```

during package condition resolution. Multiple `--conditions` flags may be used. citeturn275170search0

---

# 49. Custom Condition Example

```json
{
  "exports": {
    "development": "./dist/dev.js",
    "default": "./dist/prod.js"
  }
}
```

Then:

```bash
node --conditions=development app.js
```

can select:

```text
dev.js.
```

---

# 50. Condition Naming

Node documents restrictions for custom conditions, including avoiding:

```text
leading .
comma
integer-only property keys
```

because these can create compatibility problems with tooling and property ordering. citeturn275170search0

---

# 51. Community Conditions

Unknown non-core conditions are ignored by default.

Therefore:

```text
condition strings
```

must be designed with:

```text
ecosystem compatibility
```

in mind.

Node's core conditions include:

```text
node
node-addons
import
require
module-sync
default.
```

citeturn275170search0

---

# 52. Condition Ecosystem Risk

A package can define:

```text
development
browser
react-server
worker
edge
types
```

but different tools may:

```text
activate
ignore
or interpret
```

such conditions differently.

Therefore:

```text
document exact resolver expectations.
```

---

# 53. Unknown Condition Rule

An unknown condition usually means:

```text
that condition does not match.
```

The resolver continues:

```text
to later conditions.
```

This is why:

```text
default
```

is important.

---

# 54. `"browser"` vs `"default"`

Avoid blindly writing:

```json
{
  "exports": {
    "node": "./node.js",
    "browser": "./browser.js"
  }
}
```

without:

```text
default.
```

Node recommends a universal `default` fallback where possible, because other environments should not need to masquerade as `node` or `browser`. citeturn275170search0

---

# 55. `node` + `default` Pattern

Often:

```json
{
  "exports": {
    "node": "./node.js",
    "default": "./universal.js"
  }
}
```

is more portable than:

```text
node + browser only.
```

---

# 56. Condition Branch Explosion

Too many conditions produce:

```text
N-dimensional compatibility matrix.
```

Example:

```text
import
require
node
browser
development
production
native
edge
worker
types
```

can create:

```text
complex interaction space.
```

Use conditions only when they solve:

```text
real distribution problems.
```

---

# 57. Conditional Export Review Rule

For every condition:

```text
What problem does this branch solve?
Who activates it?
Which tools recognize it?
What is the fallback?
How is it tested?
```

---

# 58. `exports` and Self-Reference

A package can reference itself by:

```text
its package name
```

from inside its own files.

Example:

```json
{
  "name": "a-package",
  "exports": {
    ".": "./index.mjs"
  }
}
```

Then:

```js
import "a-package";
```

can resolve through its own exports map.

Node supports self-referencing when `exports` is defined. citeturn275170search0

---

# 59. Why Self-Reference Helps

Self-reference ensures internal code consumes:

```text
public package API
```

rather than:

```text
private relative internals.
```

This helps test:

```text
actual published boundaries.
```

---

# 60. Self-Reference Benefit in Monorepos

Instead of:

```js
import { x } from "../src/x.js";
```

a package can use:

```js
import { x } from "my-package/x";
```

when the public subpath is intended.

This catches:

```text
unexported API assumptions.
```

---

# 61. Package `imports`

`imports` provides:

```text
private package-local mappings.
```

Keys:

```text
must start with #.
```

Example:

```json
{
  "imports": {
    "#internal": "./src/internal.js"
  }
}
```

Then:

```js
import x from "#internal";
```

Node supports package imports for internal modules. citeturn275170search0

---

# 62. `imports` vs `exports`

`exports`:

```text
external consumers
public package surface.
```

`imports`:

```text
inside-package consumers
private aliases.
```

---

# 63. `imports` Can Map External Packages

Unlike `exports`, `imports` may map:

```text
to external packages
```

as well as:

```text
local files.
```

Node documents this difference explicitly. citeturn275170search0

---

# 64. Internal Conditional Dependency

Example:

```json
{
  "imports": {
    "#dep": {
      "node": "dep-node-native",
      "default": "./dep-polyfill.js"
    }
  }
}
```

This lets internal code use:

```js
import dep from "#dep";
```

while:

```text
resolution changes by condition.
```

citeturn275170search0

---

# 65. Why `imports` Is Valuable

It provides:

```text
stable internal names
```

without exposing:

```text
internal path structure
```

to package consumers.

---

# 66. Internal Alias vs Relative Import

Relative:

```js
import x from "../internal/x.js";
```

couples code to:

```text
directory structure.
```

Internal alias:

```js
import x from "#internal/x";
```

couples code to:

```text
logical internal interface.
```

---

# 67. Monorepo Benefits

`imports` can reduce:

```text
relative-path complexity
```

inside:

```text
large package trees.
```

But do not use:

```text
aliases
```

to hide:

```text
poor package boundaries.
```

---

# 68. Package Name

`name` matters for:

```text
publishing
package self-reference
package identity.
```

It is the identifier used in:

```text
package specifiers.
```

---

# 69. `type`

Package `type` controls how `.js` files are interpreted:

```text
module
```

vs:

```text
commonjs
```

for that package scope.

Use explicit:

```text
.mjs
.cjs
```

when you need unambiguous format boundaries.

---

# 70. `.mjs`

`.mjs` means:

```text
ES module.
```

---

# 71. `.cjs`

`.cjs` means:

```text
CommonJS.
```

---

# 72. `.js`

`.js` interpretation depends on:

```text
nearest package scope
+
package.json "type".
```

---

# 73. Package Scope

Package scope is determined by:

```text
package.json boundaries.
```

A nested:

```text
package.json
```

can change:

```text
interpretation
```

for files below it.

---

# 74. Resolution Layers

A simplified model:

```text
specifier
 ↓
is it relative/absolute?
 ↓ no
bare/package specifier
 ↓
find package
 ↓
package.json
 ↓
exports?
 ├─ yes → exports resolution
 └─ no  → legacy main/package rules
```

---

# 75. `exports` Takes Precedence

If package has:

```json
{
  "main": "./legacy.js",
  "exports": {
    ".": "./modern.js"
  }
}
```

package-name resolution uses:

```text
exports
```

instead of:

```text
main.
```

citeturn275170search0

---

# 76. Bare Specifier

```js
import x from "package-name";
```

is:

```text
bare specifier.
```

The resolver searches:

```text
package locations
```

rather than:

```text
relative file path.
```

---

# 77. Relative Specifier

```js
import x from "./x.js";
```

uses:

```text
relative URL-like resolution
```

from the current module.

`exports` does not rewrite:

```text
arbitrary relative specifiers.
```

---

# 78. Absolute Specifier

An absolute filesystem URL/path follows:

```text
filesystem resolution semantics
```

and is not the same API boundary as:

```text
package-name imports.
```

---

# 79. Package Resolution vs Filesystem Resolution

Important distinction:

```text
package API
≠
filesystem layout.
```

A package can have:

```text
100 files
```

and export:

```text
3 specifiers.
```

---

# 80. Deep Imports

Before `exports`:

```js
import x from "pkg/dist/internal.js";
```

may work.

After `exports`:

```text
may fail.
```

This is intentionally:

```text
encapsulation.
```

---

# 81. Deep Import Migration

Replace:

```text
pkg/dist/internal.js
```

with:

```text
pkg/internal-feature
```

only after:

```text
package author
```

defines:

```text
supported subpath.
```

---

# 82. Public Subpath Design

Good:

```text
pkg/parser
pkg/client
pkg/server
```

Bad:

```text
pkg/dist/a/b/c
```

because it exposes:

```text
build layout.
```

---

# 83. API Naming

Choose names based on:

```text
concept
domain
capability
```

not:

```text
directory hierarchy.
```

---

# 84. Feature-Focused Exports

Prefer:

```text
./parser
./transport
./crypto
```

rather than:

```text
./src/parser
./src/internal/transport
```

---

# 85. Exporting `package.json`

If consumers need:

```js
import pkg from "my-package/package.json";
```

you must explicitly export it under `exports`.

Otherwise:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED.
```

Node documentation gives this as a consequence of export encapsulation. citeturn275170search0

---

# 86. Should You Export `package.json`?

Only if consumers have a:

```text
real contract
```

to use:

```text
package metadata.
```

Otherwise prefer:

```text
explicit runtime API.
```

---

# 87. Metadata API Alternative

Instead of:

```js
import pkg from "package/package.json";
```

consider:

```js
import { version } from "package";
```

when the package can provide:

```text
stable semantic metadata.
```

---

# 88. Conditional Exports for ESM/CJS

A common dual-package map:

```json
{
  "exports": {
    "import": "./dist/index.js",
    "require": "./dist/index.cjs"
  }
}
```

---

# 89. Dual Package Hazard

If:

```text
import
```

and:

```text
require
```

load:

```text
different module instances,
```

stateful libraries can accidentally create:

```text
duplicate singleton state.
```

---

# 90. Duplicate Module State

Suppose both branches define:

```js
export const registry = new Map();
```

Then:

```text
ESM registry
≠
CJS registry.
```

A user may:

```text
register in one
read from another
```

and see:

```text
missing state.
```

---

# 91. Cache Identity

Different loaders can maintain:

```text
different module identities
```

even when:

```text
source conceptually represents the same package.
```

This is one reason:

```text
dual packages
```

require deliberate design.

---

# 92. Strategies for Dual Packages

Options include:

```text
ESM-only
CJS-only
single shared implementation
state-free design
bridge one format to the other
careful dual entry points.
```

---

# 93. State-Free Dual Package

If exports are:

```text
pure functions
```

duplicate module state may:

```text
matter less.
```

Still consider:

```text
caches
symbols
class identity
instanceof
events.
```

---

# 94. Class Identity Hazard

ESM and CJS branches can produce:

```text
two copies of a class definition.
```

Then:

```js
value instanceof ClassA
```

can differ from:

```text
expected.
```

---

# 95. Symbol Identity Hazard

If each branch creates:

```js
const TOKEN = Symbol("token");
```

then:

```text
TOKEN_ESM !== TOKEN_CJS.
```

This can break:

```text
cross-format protocols.
```

---

# 96. Singleton Hazard

Libraries often contain:

```text
global event bus
registry
connection pool
cache
metrics collector.
```

Dual branches can duplicate them.

---

# 97. Better Dual Strategy

Where practical:

```text
one canonical implementation
```

with:

```text
thin format-specific wrappers.
```

This can reduce:

```text
behavior drift
```

and:

```text
state duplication.
```

---

# 98. CJS Wrapper Around ESM

Be cautious because:

```text
require
```

must obey:

```text
synchronous loading constraints.
```

Current Node supports `require()` of synchronous ESM in appropriate cases, including through the `module-sync` condition, but graphs with top-level await cannot be synchronously required. citeturn275170search0turn275170search1

---

# 99. `createRequire`

ESM can create a CommonJS-style resolver:

```js
import { createRequire } from "node:module";

const require =
  createRequire(import.meta.url);
```

Use when:

```text
ESM code
```

needs:

```text
CJS resolution semantics
```

for a specific dependency.

---

# 100. `require.resolve`

`require.resolve()` answers:

```text
which path CommonJS resolution would resolve.
```

It should not be treated as:

```text
universal ESM resolver.
```

---

# 101. `import.meta.resolve`

Modern ESM provides:

```js
import.meta.resolve("package");
```

for resolving module specifiers in the ESM context.

Use carefully:

```text
resolution result
```

is still:

```textcontext-specific.
```

It does not mean:

```text
every bundler resolves the same way.
```

---

# 102. Resolution Context Matters

Resolution can differ by:

```text
import
require
custom conditions
package location
Node version
runtime/tool.
```

Therefore:

```text
“it resolves on my machine”
```

is incomplete evidence.

---

# 103. Bundler Resolution

Bundlers may use:

```text
exports
imports
conditions
browser fields
custom condition sets.
```

But:

```text
bundler resolution
```

can differ from:

```text
Node runtime resolution.
```

---

# 104. Tooling Compatibility

For each public export, test:

```text
Node ESM
Node CJS where supported
bundler
TypeScript
test runner
package manager.
```

---

# 105. TypeScript Interaction

TypeScript can interpret package exports according to its own module resolution modes.

Therefore:

```text
runtime success
```

does not automatically mean:

```text
type-checker success.
```

Verify:

```text
type declarations
exports
types conditions/paths
module resolution configuration.
```

---

# 106. Type Declarations

A package may need:

```text
runtime target
+
type declaration target.
```

Ensure:

```text
public runtime subpaths
```

have:

```text
corresponding type declarations.
```

---

# 107. Runtime/Type Surface Divergence

Bad:

```text
runtime exports ./client
```

but:

```text
types only expose root.
```

This creates:

```text
editor/type-checker failure
```

for:

```text
valid runtime API.
```

---

# 108. Conditions for Types

Some ecosystems support:

```text
types
```

conditions in package maps/tooling.

But this is:

```text
tooling ecosystem behavior
```

and should not be assumed identical:

```text
across all resolvers.
```

---

# 109. Browser Builds

A browser-focused package may use:

```text
browser
default
```

but must consider:

```text
Node
bundlers
other runtimes.
```

---

# 110. Universal Fallback

The safest broad strategy often includes:

```text
specific condition
→
default fallback.
```

Example:

```json
{
  "exports": {
    "node": "./node.js",
    "default": "./universal.js"
  }
}
```

---

# 111. Environment-Dependent Package

```text
node
→ native/Node implementation

default
→ portable implementation.
```

This avoids:

```text
unknown runtime
```

pretending to be:

```text
browser
```

just to get a compatible branch.

---

# 112. Custom Development Builds

A development condition can be useful:

```json
{
  "exports": {
    "development": "./dev.js",
    "default": "./prod.js"
  }
}
```

But ensure:

```text
every supported environment
```

has a safe default.

---

# 113. Production Build Conditions

Do not rely on:

```text
process.env.NODE_ENV
```

alone for:

```text
package resolution.
```

Package conditions and runtime environment variables are:

```text
different mechanisms.
```

---

# 114. Build-Time vs Runtime Conditions

Build tool:

```text
resolves branch
```

at:

```text
bundle time.
```

Node:

```text
resolves branch
```

at:

```text
runtime load time.
```

They may therefore:

```text
select different targets.
```

---

# 115. Condition Drift

Example:

```text
bundler sees browser
Node sees default
TypeScript sees types
```

Now:

```text
three tools
```

may use:

```text
three different graphs.
```

This is a major source of:

```text
works-in-build-fails-in-runtime
```

bugs.

---

# 116. Resolution Compatibility Matrix

Create:

```text
                import   require   bundler   TS
root               ✓        ✓         ✓       ✓
client             ✓        —         ✓       ✓
server             ✓        ✓         ✓       ✓
native              ✓       ✓         ?       ?
```

Track:

```text
intended
actual.
```

---

# 117. Package Matrix Testing

For each supported Node version test:

```text
import root
require root
import subpath
dynamic import
self-reference
imports alias
conditional branch.
```

---

# 118. Package Tarball Testing

Never test only:

```text
repository checkout.
```

Also test:

```bash
npm pack
```

or equivalent published artifact.

Why?

Because published package can differ in:

```text
files
paths
dist output
package.json
exports
```

from:

```text
source tree.
```

---

# 119. Published Artifact as Contract

The real package is:

```text
what the package manager publishes.
```

Not:

```text
what exists in the Git repository.
```

---

# 120. `files` and Build Artifacts

Ensure:

```text
every export target
```

actually exists in:

```text
published tarball.
```

---

# 121. Missing Dist File

You can have:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

while:

```text
dist/index.js
```

is not included in the package.

Result:

```text
published package fails.
```

---

# 122. Package Verification

Automate:

```text
npm pack
→ inspect tarball
→ install tarball
→ import/require public entry points
```

---

# 123. Resolution Smoke Test

A package smoke test should test:

```text
public specifier
→ actual load.
```

not merely:

```text
file exists.
```

---

# 124. Export Contract Test

For each export:

```text
expected target
```

must be:

```text
loadable
```

from:

```text
supported contexts.
```

---

# 125. Import/Require Matrix

Example:

```text
root:
  import ✓
  require ✓

client:
  import ✓
  require ✗ intentionally

server:
  import ✓
  require ✓
```

Make this explicit.

---

# 126. Conditional Branch Test

Run:

```bash
node --conditions=development ...
```

and verify:

```text
development branch.
```

Then run:

```text
without custom condition
```

and verify:

```text
default branch.
```

---

# 127. Condition Branch Reachability

A branch is dead if:

```text
earlier condition always matches.
```

Example:

```json
{
  "exports": {
    "node": "./node.js",
    "development": "./dev.js",
    "default": "./default.js"
  }
}
```

If your development environment already always matches:

```text
node
```

then:

```text
development
```

may never be reached unless nested appropriately.

---

# 128. Condition Shadowing

Bad ordering:

```json
{
  "exports": {
    "default": "./default.js",
    "node": "./node.js"
  }
}
```

Once:

```text
default
```

matches, later:

```text
node
```

is unreachable.

---

# 129. Correct Ordering

Prefer:

```json
{
  "exports": {
    "node": "./node.js",
    "default": "./default.js"
  }
}
```

or:

```text
more-specific
→
less-specific
→
default.
```

---

# 130. Nested Condition Design

Prefer:

```json
{
  "exports": {
    "node": {
      "import": "./node.mjs",
      "require": "./node.cjs"
    },
    "default": "./universal.mjs"
  }
}
```

when the conceptual hierarchy is:

```text
environment
→
module format.
```

---

# 131. Flat vs Nested

Flat:

```json
{
  "import": "...",
  "require": "...",
  "default": "..."
}
```

Nested:

```json
{
  "node": {
    "import": "...",
    "require": "..."
  },
  "default": "..."
}
```

Use nesting when:

```text
conditions form meaningful hierarchy.
```

---

# 132. Conditional Semantics as Policy

Each branch answers:

```text
Which environment?
Which loader?
Which capability?
Which implementation?
```

Document:

```text
why branch exists.
```

---

# 133. Target Arrays

Fallback arrays can represent:

```text
preferred
then alternative
```

targets.

But every fallback branch still needs:

```text
valid target semantics
```

and:

```text
actual test coverage.
```

---

# 134. Error During Conditional Resolution

A missing target may:

```text
fail
```

or:

```text
allow fallback
```

depending on the exact exports-map semantics.

Do not infer behavior from:

```text
ordinary JavaScript array fallback.
```

Use the Node package resolution rules.

---

# 135. `exports` Is Declarative

Package metadata is:

```text
declarative configuration
```

not:

```text
arbitrary executable code.
```

This enables:

```text
static analysis
```

and:

```text
predictable packaging.
```

---

# 136. Package Map Staticness

Node documents package maps as:

```text
single static files.
```

Dynamic package configuration is not part of the package map model. citeturn275170search0

---

# 137. Circular Package Map Detection

Do not assume the package map resolver performs:

```text
general circular dependency detection.
```

Node documentation notes:

```text
circular dependency detection
```

is not performed by the package map resolver. citeturn275170search0

---

# 138. Package Maps Loaded at Startup

Node documents package map files as being:

```text
loaded synchronously at startup.
```

This contributes to the expectation that:

```text
package metadata
```

is:

```text
static configuration.
```

citeturn275170search0

---

# 139. Package Resolution Security

Restrictions on targets help prevent:

```text
package map escaping root
```

and:

```text
arbitrary target paths.
```

This is:

```text
defense-in-depth
```

not:

```text
complete package security.
```

---

# 140. Supply Chain Implication

A malicious package can still:

```text
execute code when loaded
```

through:

```text
its exported entry point.
```

Therefore:

```text
exports
```

does not eliminate:

```text
supply-chain risk.
```

---

# 141. Dependency Confusion

Package names participate in:

```text
package-manager identity.
```

A wrong package source can still expose:

```text
malicious package
```

under a familiar import specifier.

Use:

```text
trusted registries
lockfiles
provenance
scoped naming
verification.
```

---

# 142. Optional Native Dependency

A package can expose:

```text
node-addons
```

for:

```text
native fast path
```

and:

```text
default
```

for:

```text
portable fallback.
```

But verify:

```text
native binary availability
ABI/runtime compatibility
installation behavior.
```

---

# 143. Native Branch Failure

Do not assume:

```text
node-addons branch
```

means:

```text
native addon always installed.
```

Your packaging strategy must define:

```text
build/install path
fallback.
```

---

# 144. `--no-addons`

Node allows disabling native addon support with:

```bash
--no-addons
```

This can be useful for:

```text
portable testing
security hardening
fallback verification.
```

Node documents its relationship to the `node-addons` condition. citeturn275170search0

---

# 145. Native vs Portable Strategy

```text
node-addons
→ performance/capability

default
→ portability.
```

This is often cleaner than:

```text
platform-specific package names
```

for every environment.

---

# 146. Package Exports and Native Addon Chapters

See:

```text
Chapter 143 — Native Addons / N-API / FFI / ABI Boundaries.
```

Resolution determines:

```text
which implementation enters the runtime.
```

Chapter 143 determines:

```text
what happens after native code is loaded.
```

---

# 147. `module-sync` and Native

Do not confuse:

```text
module-sync
```

with:

```text
node-addons.
```

They answer different questions:

```text
module-sync
→ synchronous ESM-compatible loading

node-addons
→ native addon capability.
```

---

# 148. Condition Specificity

Suppose:

```text
node
node-addons
default
```

Which should come first?

Because:

```text
node-addons
```

is more specific:

```text
node-addons
→
node
→
default
```

may better represent intent.

Node's documented condition ordering lists `node-addons` before `node`. citeturn275170search0

---

# 149. `import` vs `require`

These conditions are:

```text
mutually exclusive.
```

This makes:

```json
{
  "import": "./esm.js",
  "require": "./cjs.cjs"
}
```

a direct split:

```text
ESM loader
vs
CommonJS loader.
```

---

# 150. Dynamic Import Is Still `import`

```js
await import("pkg");
```

uses:

```text
import
```

condition semantics.

It does not mean:

```text
require.
```

Node documents dynamic `import()` as part of the ESM-side context for conditional exports. citeturn275170search0

---

# 151. `require()` and ESM Targets

Current Node can synchronously require certain ESM graphs when they contain no top-level await.

This capability is version-sensitive and should be tested against the target Node versions.

---

# 152. CJS Import from ESM

An ESM import of a CommonJS package generally uses:

```text
CommonJS loading
```

plus:

```text
interop namespace behavior.
```

This chapter focuses on:

```text
which package branch resolves,
```

while Chapter 126 covers deeper linking/evaluation.

---

# 153. Modern Resolution vs Legacy Resolution

Legacy:

```text
main
index.js
directory conventions
deep imports.
```

Modern:

```text
exports
imports
conditions
explicit subpaths
encapsulation.
```

---

# 154. Resolution Is a Compatibility Contract

Changing:

```text
./client
```

from:

```text
./dist/client.js
```

to:

```text
./dist/new-client.js
```

is often safe:

```text
if the public specifier remains the same
```

and:

```text
behavior/semantics are compatible.
```

Changing:

```text
remove ./client
```

is:

```text
API-breaking.
```

---

# 155. Semver and Exports

Removing:

```text
exports["./feature"]
```

is a likely:

```text
breaking change.
```

Adding:

```text
new public subpath
```

is generally:

```text
additive.
```

But document:

```text
runtime constraints.
```

---

# 156. Changing Condition Behavior

Suppose:

```text
import
```

used:

```text
implementation A
```

and a minor release changes it to:

```text
implementation B.
```

Even if:

```text
specifier unchanged,
```

behavior may change.

Therefore:

```text
conditional target changes
```

can affect:

```text
semver expectations
```

when observable behavior differs.

---

# 157. Stable Public Specifier

A package should encourage consumers to depend on:

```text
package specifier
```

not:

```text
resolved filesystem path.
```

---

# 158. Package API Layers

A strong library can expose:

```text
root
→ high-level stable API

subpaths
→ specialized stable APIs

private source files
→ not public.
```

---

# 159. Root Export Minimalism

Do not put:

```text
entire internal library
```

at:

```text
root.
```

A root export should be:

```text
deliberate.
```

---

# 160. Subpath Stability

Every public subpath becomes:

```text
maintenance responsibility.
```

Therefore:

```text
more exports
=
more API surface
=
more compatibility cost.
```

---

# 161. Export Surface Budget

A principal package review should ask:

```text
Can this subpath be removed later?
Does a consumer really need it?
Is its naming stable?
Does it expose internal abstractions?
```

---

# 162. Encapsulation and Refactoring

Without `exports`:

```text
refactoring dist layout
```

can break:

```text
deep imports.
```

With `exports`:

```text
internal layout
```

can change while:

```text
public specifier
```

remains stable.

---

# 163. Refactoring Example

Version 1:

```text
dist/
  client.js
```

Version 2:

```text
dist/
  client/
    index.js
```

If:

```json
"exports": {
  "./client": "./dist/client.js"
}
```

changes to:

```json
"exports": {
  "./client": "./dist/client/index.js"
}
```

consumers can continue using:

```text
pkg/client.
```

---

# 164. Without Encapsulation

Consumer might import:

```text
pkg/dist/client.js
```

and break when:

```text
layout changes.
```

This is exactly:

```text
accidental public API.
```

---

# 165. Package Boundary as Abstraction

Think:

```text
physical implementation
        ↓
exports map
        ↓
logical public API
```

This is:

```text
abstraction boundary.
```

---

# 166. Monorepo Package Boundaries

In a monorepo:

```text
package A
package B
package C
```

should consume:

```text
public exports
```

rather than:

```text
../../package-A/src/private.
```

This makes package architecture:

```text
visible.
```

---

# 167. Internal Package API Testing

Self-reference lets:

```text
package A
```

test:

```text
its public specifiers
```

before publishing.

---

# 168. Workspace Symlinks

Package managers can use:

```text
symlinks/workspaces.
```

Resolution can therefore involve:

```text
workspace topology
```

in addition to:

```text
package metadata.
```

Do not confuse:

```text
development workspace resolution
```

with:

```text
published tarball layout.
```

---

# 169. Workspace vs Published Package

A source tree may contain:

```text
../shared
```

while the package tarball does not.

Therefore:

```text
packaged-artifact testing
```

is mandatory.

---

# 170. Monorepo Anti-Pattern

```text
internal package directly imports src files of another package.
```

This bypasses:

```text
exports contract.
```

Instead:

```text
consume package specifier.
```

---

# 171. Package-Level Contract Tests

Build tests that:

```text
install package tarball
→ import public root
→ import public subpaths
→ attempt blocked deep import
→ verify expected error.
```

---

# 172. Verifying Blocked Deep Imports

Expected:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED
```

when consumer requests:

```text
unexported package subpath.
```

This protects:

```text
encapsulation.
```

---

# 173. Package Resolution Test Harness

Build:

```text
fixtures/
  esm-consumer/
  cjs-consumer/
  condition-consumer/
  blocked-subpath/
```

Each fixture contains:

```text
minimal package
```

and:

```text
one resolution assertion.
```

---

# 174. Consumer Fixtures

Test from the consumer's perspective:

```js
import pkg from "target-package";
```

not merely:

```js
import "./dist/index.js";
```

---

# 175. Why Consumer Perspective Matters

The real API is:

```text
specifier
+
resolver
+
package metadata.
```

Not:

```text
source file import.
```

---

# 176. `npm pack` Workflow

```bash
npm pack
```

then:

```bash
mkdir /tmp/package-test
cd /tmp/package-test
npm init -y
npm install /path/to/package.tgz
```

then test:

```text
public imports.
```

---

# 177. Tarball Verification Script

A production package pipeline should verify:

```text
tarball exists
expected dist files exist
package.json exports correct
all export targets exist
public imports succeed
private imports fail
```

---

# 178. Static Export Verification

Because exports are declarative and enumerable, build tooling can inspect:

```text
export key set
target set
```

and compare:

```text
expected API manifest.
```

Node documents the static-enumerability property of exports patterns. citeturn275170search0

---

# 179. Export Manifest

Example:

```json
{
  "root": "./dist/index.js",
  "client": "./dist/client.js",
  "server": "./dist/server.js"
}
```

Use as:

```text
human/API review artifact.
```

---

# 180. Public API Diff

Compare:

```text
previous release
vs
current release.
```

Report:

```text
added exports
removed exports
changed targets
changed conditions.
```

---

# 181. Export Changes as API Diff

A CI job can fail on:

```text
unexpected removed export.
```

This makes:

```text
package API compatibility
```

machine-verifiable.

---

# 182. Conditional API Diff

Track:

```text
import target
require target
node target
default target
native target.
```

A change in:

```text
branch mapping
```

should be reviewed like:

```text
API code change.
```

---

# 183. Test Custom Conditions

If package documents:

```text
development
```

then CI should explicitly test:

```bash
node --conditions=development ...
```

and:

```bash
node ...
```

---

# 184. Unsupported Condition

Test:

```text
unknown condition
```

to ensure:

```text
default fallback
```

still works.

---

# 185. Condition Collision

Suppose tooling activates:

```text
development
```

unexpectedly.

The package can select:

```text
debug build
```

in production.

Therefore:

```text
custom conditions
```

must be treated as:

```text
configuration input.
```

---

# 186. Custom Condition Security

Do not put:

```text
security-critical behavior
```

behind a condition that:

```text
an untrusted toolchain
```

can easily activate.

---

# 187. Build Tool Condition Sets

Different tools may activate:

```text
node
browser
development
production
import
require
```

and possibly:

```text
custom conditions.
```

Verify each supported tool.

---

# 188. Condition Documentation

Document:

```text
condition
purpose
who activates it
target
fallback.
```

---

# 189. Example Condition Table

| Condition | Purpose | Target | Fallback |
|---|---|---|---|
| `node-addons` | native enhancement | native build | `default` |
| `node` | Node-specific implementation | Node build | `default` |
| `import` | ESM path | ESM file | `default` |
| `require` | CommonJS path | CJS file | `default` |
| `development` | debug build | dev | `default` |
| `default` | universal path | portable | — |

Node core's recognized condition ordering includes `node-addons`, `node`, `import`, `require`, `module-sync`, and `default`. citeturn275170search0

---

# 190. `exports` and Tests

A package's tests should import through:

```text
public package specifiers
```

where practical.

This prevents:

```text
tests passing through private paths
```

while:

```text
consumers fail.
```

---

# 191. Testing Internal Code

Internal implementation tests can still use:

```text
relative source imports
```

when:

```text
white-box coverage
```

is valuable.

But also maintain:

```text
black-box public API tests.
```

---

# 192. White-Box + Black-Box

White-box:

```text
implementation details
```

Black-box:

```text
published contract.
```

A mature package needs:

```text
both.
```

---

# 193. Black-Box Export Tests

These should answer:

```text
Can a consumer load every documented public specifier?
```

---

# 194. White-Box Module Tests

These answer:

```text
Does this internal algorithm work?
```

They should not define:

```text
public API accidentally.
```

---

# 195. Consumer-Driven Tests

For popular packages:

```text
real consumer fixtures
```

can detect:

```text
ecosystem incompatibility.
```

---

# 196. Package Entrypoint Performance

Every exported root can introduce:

```text
startup cost
```

depending on:

```text
module graph.
```

Do not create:

```text
massive root barrel
```

if users only need:

```text
specialized subpaths.
```

---

# 197. Subpath Performance

Subpaths can reduce:

```text
initial module graph
```

when consumers load:

```text
specific functionality.
```

But bundlers may:

```text
tree-shake
```

other designs.

Measure before assuming.

---

# 198. Conditional Target Performance

Native branch:

```text
faster startup/runtime
```

may have:

```text
installation
build
distribution
ABI
platform
```

costs.

Portable branch:

```text
lower installation complexity
```

may have:

```text
runtime cost.
```

---

# 199. Distribution Trade-Off

Every target increases:

```text
build complexity
testing matrix
release risk.
```

Do not create:

```text
one build per platform
```

without:

```text
ROI.
```

---

# 200. Package Size

Export maps do not automatically reduce:

```text
tarball size.
```

They reduce:

```text
supported module-address surface.
```

Bundle size reduction depends on:

```text
consumer/bundler behavior.
```

---

# 201. Tree Shaking

ESM exports can support:

```text
tree shaking
```

in compatible bundlers.

But:

```text
package exports map
```

and:

```text
ESM syntax
```

are separate concepts.

---

# 202. `exports` vs Tree Shaking

`exports` answers:

```text
what subpaths exist.
```

Tree shaking answers:

```text
what code can be removed from a loaded graph.
```

Do not conflate them.

---

# 203. Package Conditions and Tree Shaking

Different condition branches can produce:

```text
different dependency graphs.
```

So:

```text
bundle output
```

can vary with:

```text
condition set.
```

---

# 204. Default Fallback Quality

A default branch should ideally be:

```text
portable
supported
tested
semantically equivalent
```

to the specialized branch.

---

# 205. Default Branch Is Not “Whatever”

Do not use:

```text
default
```

as:

```text
unmaintained fallback.
```

Treat it as:

```text
first-class implementation.
```

---

# 206. Condition Equivalence Testing

For equivalent contracts:

```text
node branch
default branch
```

should pass:

```text
same behavioral test suite
```

where behavior is intended to match.

---

# 207. Branch-Specific Tests

Native branch may need:

```text
performance
ABI
resource
```

tests.

Portable branch may need:

```text
compatibility
availability
```

tests.

---

# 208. Branch Selection Tests

Verify:

```text
which file
```

is actually loaded.

Do not only assert:

```text
behavior passed.
```

because:

```text
wrong branch
```

can accidentally produce:

```text
same behavior.
```

---

# 209. Resolution Probe

A small consumer can print:

```js
import.meta.url
```

or use:

```text
resolution APIs
```

to verify:

```text
selected entry.
```

Use the appropriate loader-specific mechanism.

---

# 210. `require.resolve` vs `import.meta.resolve`

These represent:

```text
different resolution contexts.
```

Use:

```text
the resolver matching the consumer mode.
```

---

# 211. Dynamic `import()` Test

Test:

```js
const mod = await import("package");
```

to verify:

```text
import condition
```

and:

```text
async loading.
```

---

# 212. CommonJS Consumer Test

Test:

```js
const mod = require("package");
```

to verify:

```text
require condition
```

when package promises CJS support.

---

# 213. Self-Reference Test

Inside the package:

```js
import { feature } from "my-package";
```

should resolve:

```text
through exports.
```

This validates:

```text
package public boundary.
```

---

# 214. Internal `imports` Test

Inside package:

```js
import x from "#internal";
```

should resolve:

```text
through imports.
```

A consumer outside the package should:

```text
not
```

be able to use:

```text
#internal.
```

---

# 215. `imports` Privacy

`#` specifiers are:

```text
package-internal names.
```

They are not:

```text
public package subpaths.
```

---

# 216. `imports` and External Targets

Because `imports` can map external packages:

```text
internal logical API
```

can switch between:

```text
dependency A
dependency B
local fallback.
```

---

# 217. Internal Polyfill Pattern

```json
{
  "imports": {
    "#crypto": {
      "node": "node:crypto",
      "default": "./polyfills/crypto.js"
    }
  }
}
```

Then:

```js
import crypto from "#crypto";
```

---

# 218. Polyfill Architecture

The package sees:

```text
logical dependency
```

while resolver chooses:

```text
native implementation
or
fallback.
```

This centralizes:

```text
compatibility policy.
```

---

# 219. Alias Stability

Internal aliases let you refactor:

```text
src/
```

without changing:

```text
logical import names.
```

But:

```text
alias
```

should map to:

```text
stable concept
```

not:

```text
temporary implementation name.
```

---

# 220. Export Pattern Design

Broad:

```json
{
  "exports": {
    "./*.js": "./dist/*.js"
  }
}
```

is convenient but can expose:

```text
future files accidentally.
```

Prefer:

```text
explicit public subpaths
```

for:

```text
small stable APIs.
```

---

# 221. Pattern Exposure Risk

Adding a new internal file:

```text
dist/private.js
```

might accidentally make it:

```text
public
```

under a broad pattern.

Therefore:

```text
patterns trade brevity for future exposure risk.
```

---

# 222. Explicit Exports

Explicit:

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./client": "./dist/client.js"
  }
}
```

is easier to audit.

---

# 223. Pattern vs Explicit Decision

Use pattern when:

```text
large stable family
```

needs:

```text
systematic exposure.
```

Use explicit keys when:

```text
public API is curated.
```

---

# 224. Versioned APIs

You can expose:

```text
./v1
./v2
```

but this creates:

```text
long-term maintenance
```

responsibility.

Prefer:

```text
semantic versioning
```

unless:

```text
parallel API generations
```

are genuinely necessary.

---

# 225. Deprecating an Export

Safe path:

```text
keep specifier
→
document deprecated
→
warn where appropriate
→
remove in breaking release.
```

---

# 226. Deprecation and `exports`

You cannot:

```text
safely deprecate
```

an export by:

```text
silently removing it.
```

That changes:

```text
resolution semantics
```

into:

```text
runtime failure.
```

---

# 227. Export Tombstone

A `null` target can intentionally block:

```text
former pattern path
```

but this is still:

```text
behavioral breakage
```

for consumers that used it.

---

# 228. Package Exports and Semver

Track:

```text
added key
removed key
changed key
changed condition
changed target
changed format.
```

All can influence:

```text
consumer compatibility.
```

---

# 229. Export Map Review

Code review should ask:

```text
Does every public key have documentation?
Does every target exist?
Are conditions ordered correctly?
Is default last?
Are private files accidentally exposed?
Does this break deep imports?
Do CJS/ESM branches share state safely?
```

---

# 230. Package Map Linter

Build an internal linter that checks:

```text
target starts with ./
target exists
no forbidden segments
default placement
condition naming
duplicate keys
shadowed conditions
public API manifest.
```

---

# 231. Export Map Validator

A validator can:

```text
parse package.json
walk export tree
resolve target paths
detect missing targets
emit warnings.
```

It should not attempt to become:

```text
full Node resolver
```

unless necessary.

---

# 232. Simplified Resolver — Step 1

Input:

```text
package exports
specifier subpath
conditions
```

Return:

```text
target.
```

---

# 233. Simplified Resolver — Step 2

Algorithm:

```text
if exact key exists:
    resolve exact mapping

else if pattern applies:
    perform substitution

else:
    error.
```

Then:

```text
if target is conditional:
    walk conditions in order
```

---

# 234. Simplified Resolver — Step 3

Validate:

```text
target begins ./
target stays inside package root
target exists
```

---

# 235. Simplified Resolver — Step 4

If:

```text
target = null
```

return:

```text
blocked.
```

---

# 236. Simplified Condition Resolver

```js
function resolveCondition(node, conditions) {
  if (typeof node === "string") {
    return node;
  }

  if (node === null) {
    return null;
  }

  for (const [condition, target] of Object.entries(node)) {
    if (conditions.has(condition)) {
      const result =
        resolveCondition(target, conditions);

      if (result !== undefined) {
        return result;
      }
    }
  }

  return undefined;
}
```

This is a:

```text
learning model
```

not:

```text
complete Node implementation.
```

---

# 237. Condition Resolver Caveat

Real Node package resolution handles:

```text
patterns
arrays
fallback behavior
target validation
exact semantics
loader context
```

in more detail.

Use the real Node implementation/docs as authority. citeturn275170search0turn275170search1

---

# 238. Simplified Pattern Resolver

```js
function matchPattern(key, subpath) {
  const star = key.indexOf("*");

  if (star === -1) return null;

  const prefix = key.slice(0, star);
  const suffix = key.slice(star + 1);

  if (!subpath.startsWith(prefix)) {
    return null;
  }

  if (!subpath.endsWith(suffix)) {
    return null;
  }

  return subpath.slice(
    prefix.length,
    subpath.length - suffix.length
  );
}
```

---

# 239. Pattern Resolver Hardening

Then validate:

```text
empty wildcard
path traversal
encoded separators
node_modules segments
target boundaries.
```

---

# 240. Simplified Package Resolver

```text
request
 ↓
package name
 ↓
locate package
 ↓
read package.json
 ↓
exports?
 ├─ yes
 │   ↓
 │ match subpath
 │   ↓
 │ evaluate conditions
 │   ↓
 │ validate target
 │   ↓
 │ load target
 │
 └─ no
     ↓
   legacy resolution.
```

---

# 241. Why Not Reimplement Node Resolution?

Because Node's resolver incorporates:

```text
ESM semantics
CJS semantics
package scopes
conditions
extensions
URL rules
version-specific behavior.
```

A simplified resolver is:

```text
educational
```

unless you are explicitly building:

```text
tooling infrastructure.
```

---

# 242. Resolution Debugging Strategy

When package import fails:

```text
1. What exact specifier?
2. Which package?
3. Which Node version?
4. Import or require?
5. Which conditions?
6. Which package.json?
7. Does exports exist?
8. Exact or pattern key?
9. Does target exist?
10. Is target packaged?
```

---

# 243. `ERR_PACKAGE_PATH_NOT_EXPORTED`

Meaning:

```text
consumer requested
a package subpath
not exposed by exports.
```

Fix:

```text
use public subpath
or
package author exports intended subpath.
```

---

# 244. `ERR_PACKAGE_IMPORT_NOT_DEFINED`

Meaning:

```text
internal #specifier
```

was not defined in:

```text
package imports.
```

Fix:

```text
define # mapping
or
use an existing internal specifier.
```

---

# 245. `ERR_MODULE_NOT_FOUND`

Can mean:

```text
target/file/dependency
not found
```

after resolution.

Distinguish from:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED.
```

The latter is:

```text
encapsulation.
```

---

# 246. `ERR_REQUIRE_ASYNC_MODULE`

Relevant when:

```text
require()
```

reaches:

```text
ESM graph
```

containing:

```text
top-level await.
```

Current Node `module-sync` semantics explicitly account for this boundary. citeturn275170search0

---

# 247. Resolution Error Triage

Classify first:

```text
package lookup
subpath export
condition branch
target path
file existence
format loading
dependency loading.
```

---

# 248. Resolution Debugging Example

Request:

```js
import "pkg/internal";
```

Error:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED.
```

Do not debug:

```text
file path
```

first.

Debug:

```text
exports map
```

first.

---

# 249. Resolution Debugging Example — Missing Tarball

Repository works:

```text
import "pkg/client";
```

Published package fails.

Likely suspects:

```text
dist missing from tarball
package.json changed
exports target wrong
build step skipped.
```

---

# 250. Resolution Debugging Example — Works in Bundler

Bundler:

```text
works.
```

Node:

```text
fails.
```

Investigate:

```text
condition sets
browser/default behavior
extension rules
CJS/ESM format
tool-specific aliases.
```

---

# 251. Resolution Debugging Example — Works in Node

Node:

```text
works.
```

TypeScript:

```text
cannot find module.
```

Investigate:

```text
type declarations
TS module resolution mode
package metadata interpretation
types mapping.
```

---

# 252. Resolution Debugging Example — ESM Works, CJS Fails

Check:

```text
require condition
CJS target
synchronous dependency graph
extension format
top-level await.
```

---

# 253. Resolution Debugging Example — Browser Fails

Check:

```text
browser condition
default fallback
Node-only APIs
bundler configuration
```

---

# 254. Modern Resolution and Build Artifacts

A package's:

```text
source
build
exports
types
package files
```

must form:

```text
one consistent graph.
```

---

# 255. Source Layout Example

```text
src/
  index.ts

dist/
  index.js
  index.cjs
  index.d.ts
```

Exports:

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

---

# 256. Source-vs-Dist Mismatch

If:

```text
src/client.ts
```

exists but:

```text
dist/client.js
```

does not, then:

```text
exports ./client
```

must not target the missing dist artifact.

---

# 257. Build Before Publish

CI order:

```text
install
→ build
→ validate export targets
→ pack
→ install tarball
→ consumer tests.
```

---

# 258. Package Verification in CI

Fail if:

```text
export target missing
```

or:

```text
public import fails
```

or:

```text
blocked private import unexpectedly works.
```

---

# 259. Package Locking

The package itself should pin:

```text
build dependencies
```

through:

```text
lockfile
```

where workflow requires reproducibility.

---

# 260. Conditional Exports and Lockfiles

Lockfiles control:

```text
dependency graph.
```

Exports control:

```text
package branch selection.
```

Both matter for:

```text
reproducibility.
```

---

# 261. Release Reproducibility

A release should be reproducible from:

```text
source commit
lockfile
build tool versions
Node version
package metadata.
```

---

# 262. Package Integrity

For security:

```text
package tarball
```

should be:

```text
reviewable
hashable
verifiable.
```

---

# 263. Artifact Provenance

Track:

```text
source commit
build environment
Node version
compiler
package manager
```

when shipping:

```text
native or generated artifacts.
```

---

# 264. Package Exports and Security

`exports` reduces:

```text
accidental API surface.
```

It does not prevent:

```text
malicious code execution.
```

---

# 265. Public Surface and Attack Surface

Fewer exported APIs can reduce:

```text
unsupported entry-point exposure
```

but:

```text
the root export
```

still executes:

```text
package code.
```

---

# 266. Conditional Security Drift

Suppose:

```text
node
```

branch has:

```text
security hardening
```

but:

```text
default
```

does not.

Then:

```text
browser/edge consumers
```

may behave differently.

Branch parity review is:

```text
security review.
```

---

# 267. Native Security Branch

If:

```text
node-addons
```

uses native code:

```text
security boundary
```

differs from:

```text
default JS/WASM.
```

Review:

```text
input handling
memory safety
permissions
supply chain.
```

---

# 268. Condition Confusion

A package that assumes:

```text
development
```

means:

```text
safe local environment
```

may be wrong.

Conditions are:

```text
resolution inputs
```

not:

```text
security assertions.
```

---

# 269. Package Resolution Attack Surface

Potential risk areas:

```text
malicious package substitution
unexpected condition activation
wrong package target
dependency confusion
native branch loading
tool/runtime mismatch.
```

---

# 270. Principal Security Rule

Never make:

```text
security-critical authorization
```

depend only on:

```text
which package condition happened to match.
```

Security policy belongs in:

```text
explicit application/runtime controls.
```

---

# 271. Debugging Labs

## Lab A — Add `exports`

Start with:

```text
main
```

only.

Add:

```text
exports root.
```

Verify:

```text
deep import
```

fails.

---

## Lab B — Preserve Legacy Paths

Inventory:

```text
pkg
pkg/client
pkg/server
```

and encode:

```text
all intentional public paths.
```

---

## Lab C — Conditional ESM/CJS

Build:

```text
index.js
index.cjs
```

and export:

```text
import
require.
```

Test both.

---

## Lab D — Nested Conditions

Build:

```text
node
  import
  require
default.
```

Verify:

```text
each branch.
```

---

## Lab E — Custom Condition

Create:

```text
development
default
```

and run:

```bash
node --conditions=development ...
```

---

## Lab F — `imports`

Create:

```text
#internal
```

and verify:

```text
internal package usage
```

works while:

```text
external usage
```

fails.

---

## Lab G — Pattern + Null

Create:

```text
features/*.js
features/private/* → null
```

Verify:

```text
public feature
private feature.
```

---

## Lab H — Native Fallback

Create:

```text
node-addons
default
```

and test:

```bash
node --no-addons ...
```

---

## Lab I — Tarball Test

Run:

```bash
npm pack
```

install the tarball, then test:

```text
public exports.
```

---

## Lab J — API Diff

Create:

```text
export-manifest.json
```

and fail CI if:

```text
public subpath removed unexpectedly.
```

---

# 272. Code Review Exercise — Broad Pattern

```json
{
  "exports": {
    "./*": "./dist/*"
  }
}
```

Find:

```text
accidental exposure of future internal files
```

and design:

```text
explicit public surface
```

or:

```text
narrow pattern.
```

---

# 273. Code Review Exercise — Default First

```json
{
  "exports": {
    "default": "./default.js",
    "node": "./node.js"
  }
}
```

Find:

```text
shadowed node branch.
```

---

# 274. Code Review Exercise — No Fallback

```json
{
  "exports": {
    "node-addons": "./native.js"
  }
}
```

Find:

```text
no universal fallback
```

when:

```text
portable environments
```

need support.

---

# 275. Code Review Exercise — Deep Import Test

```js
import "pkg/dist/internal.js";
```

Find:

```text
test bypasses public contract.
```

---

# 276. Code Review Exercise — Tarball Blindness

CI runs:

```text
npm test
```

but never:

```text
npm pack
```

or:

```text
consumer install.
```

Find:

```text
published artifact blind spot.
```

---

# 277. Code Review Exercise — State Duplication

```json
{
  "exports": {
    "import": "./esm.js",
    "require": "./cjs.cjs"
  }
}
```

Both define:

```js
const registry = new Map();
```

Find:

```text
potential duplicate singleton state.
```

---

# 278. Predict-the-Behavior Exercises

### Exercise 1

```json
{
  "exports": {
    ".": "./index.js",
    "./client": "./client.js"
  }
}
```

Predict:

```text
import "pkg/client"
```

succeeds.

---

### Exercise 2

Same package.

Predict:

```text
import "pkg/internal"
```

if no export exists.

Expected:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED.
```

citeturn275170search0

---

### Exercise 3

```json
{
  "exports": {
    "default": "./default.js",
    "node": "./node.js"
  }
}
```

Predict:

```text
which target wins in Node.
```

`default` because:

```text
condition order is significant
```

and it appears first.

---

### Exercise 4

```json
{
  "exports": {
    "node": {
      "import": "./node.mjs",
      "require": "./node.cjs"
    },
    "default": "./universal.mjs"
  }
}
```

Predict:

```text
Node ESM import.
```

Result:

```text
node.mjs.
```

---

### Exercise 5

Same map.

Predict:

```text
Node require.
```

Result:

```text
node.cjs.
```

---

### Exercise 6

```bash
node --conditions=development app.js
```

with:

```json
{
  "exports": {
    "development": "./dev.js",
    "default": "./prod.js"
  }
}
```

Predict:

```text
dev.js.
```

---

### Exercise 7

```text
node --no-addons
```

with:

```json
{
  "exports": {
    "node-addons": "./native.js",
    "default": "./portable.js"
  }
}
```

Predict:

```text
portable.js.
```

---

### Exercise 8

`module-sync` targets an ESM graph containing top-level await.

Predict:

```text
what can happen under require.
```

Potential result:

```text
ERR_REQUIRE_ASYNC_MODULE.
```

citeturn275170search0

---

### Exercise 9

Package defines:

```text
imports: "#internal"
```

outside consumer package:

```js
import "#internal";
```

Predict:

```text
unavailable outside package context.
```

---

### Exercise 10

A package has both:

```text
main
exports.
```

Predict which is used for package-name import in modern Node.

```text
exports.
```

citeturn275170search0

---

# 279. Interview Questions

### Fundamentals

```text
1. Why does exports exist?
2. How is exports different from main?
3. What does package encapsulation mean?
4. What is a package subpath?
5. What is a bare specifier?
```

### Conditional Exports

```text
6. What is a conditional export?
7. Why does key ordering matter?
8. Why should default usually be last?
9. What is import?
10. What is require?
11. What is node?
12. What is node-addons?
13. What is module-sync?
```

### Imports

```text
14. What is package imports?
15. Why must imports keys start with #?
16. How is imports different from exports?
17. Can imports target external packages?
```

### Dual Packages

```text
18. What is the dual package hazard?
19. How can singleton state be duplicated?
20. How can class identity break?
21. How would you design a safe dual package?
22. When would you choose ESM-only?
```

### Compatibility

```text
23. Why is adding exports potentially breaking?
24. How would you migrate a package safely?
25. How do you test published tarballs?
26. How do you maintain TypeScript compatibility?
27. How do you test bundler and Node resolution differences?
```

### Principal

```text
28. Design a package with native and portable implementations.
29. Design a monorepo package boundary.
30. Design an exports matrix for ESM/CJS/browser/Node.
31. How would you detect shadowed conditions?
32. How would you prevent accidental public APIs?
33. How would you verify package exports in CI?
34. How would you handle a breaking deep-import ecosystem?
35. How would you review conditional branches for security?
36. How would you debug “works in bundler, fails in Node”?
37. How would you design package self-reference?
38. How would you decide between explicit exports and patterns?
```

---

# 280. Mastery Exercises

### Exercise 1 — Package Surface

Create:

```text
root
client
server
parser
```

and explicitly expose only:

```text
supported APIs.
```

### Exercise 2 — Dual Package

Provide:

```text
ESM
CJS
```

branches.

Then verify:

```text
class identity
singleton behavior
events
errors.
```

### Exercise 3 — Native Fallback

Create:

```text
node-addons
default
```

branches and test:

```text
normal Node
--no-addons.
```

### Exercise 4 — Custom Conditions

Create:

```text
development
production-like default.
```

Test:

```text
condition enabled
condition disabled
unknown condition.
```

### Exercise 5 — Internal Imports

Design:

```text
#config
#logger
#platform
```

without exposing them publicly.

### Exercise 6 — Export Pattern Audit

Create:

```text
pattern exports
```

then intentionally add:

```text
private file.
```

Observe:

```text
accidental exposure.
```

### Exercise 7 — Tarball Gate

Automate:

```text
build
pack
install
consumer tests.
```

### Exercise 8 — API Diff

Write a tool that compares:

```text
old exports
new exports.
```

### Exercise 9 — Resolver Model

Implement:

```text
exact matching
pattern matching
conditional matching
null targets.
```

### Exercise 10 — Compatibility Matrix

Test:

```text
Node ESM
Node CJS
TypeScript
bundler
published tarball.
```

---

# 281. Track A — Core Theory

Master:

```text
package scope
package.json
main
exports
imports
subpaths
conditions
condition ordering
nested conditions
patterns
null targets
self-reference
dual packages
module-sync
node-addons
resolution contexts
encapsulation
semver
published artifacts.
```

Deliverable:

```text
Explain exactly how a package specifier
becomes a concrete runtime module.
```

---

# 282. Track B — Implementation

Build:

```text
package with explicit exports
dual package
conditional package
native fallback
internal imports map
export validator
consumer fixture suite
tarball verification pipeline
public API diff tool
simplified resolver.
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

# 283. Track C — Interview / Reasoning

Practice:

```text
“Why is adding exports potentially breaking?”

“Why does condition ordering matter?”

“What happens when default comes first?”

“How do import and require branches differ?”

“What is the dual package hazard?”

“Why use package imports?”

“Why should CI test the packed tarball?”

“How do you design node-addons + default?”

“When are patterns dangerous?”

“How do you debug a Node/bundler resolution mismatch?”
```

Answer using:

```text
specifier
package map
conditions
target
loader
format
artifact
compatibility
trade-off.
```

---

# 284. Principal Decision Framework

For every package-resolution architecture ask:

```text
1. What is the intended public API?
2. Which subpaths are truly supported?
3. Which physical files are intentionally private?
4. Should the package be ESM-only, CJS-only, or dual?
5. Does a dual package duplicate state?
6. Which conditions are required?
7. Why does each condition exist?
8. Which tools activate each condition?
9. Is default a real universal fallback?
10. Are conditions ordered most-specific to least-specific?
11. Are any branches shadowed?
12. Are patterns necessary?
13. Could patterns expose future internals?
14. Should private pattern regions be null-blocked?
15. Does the package need internal imports?
16. Could aliases hide architectural problems?
17. Are runtime and type surfaces aligned?
18. Are Node and bundler resolution compatible?
19. Are all export targets in the published tarball?
20. Are public specifiers tested from a consumer project?
21. Are deep-import compatibility changes documented?
22. Is semver impact understood?
23. Does native branching justify its operational cost?
24. What happens with --no-addons?
25. Does module-sync introduce top-level-await constraints?
26. Are custom conditions documented?
27. Could unknown environments fall back safely?
28. Are security assumptions independent of condition selection?
29. Is package resolution observable during debugging?
30. Can CI detect export-map regressions?
31. What is the long-term API maintenance cost?
```

---

# 285. Production Package Exports Checklist

```text
[ ] exports documented
[ ] root entry intentional
[ ] public subpaths intentional
[ ] private files not accidentally exposed
[ ] target files exist
[ ] targets remain inside package root
[ ] forbidden path segments absent
[ ] conditions justified
[ ] conditions ordered correctly
[ ] default last
[ ] default is maintained
[ ] import branch tested
[ ] require branch tested
[ ] node branch tested
[ ] node-addons branch tested if used
[ ] module-sync branch tested if used
[ ] custom conditions documented
[ ] imports aliases documented
[ ] self-reference tested
[ ] deep-import migration reviewed
[ ] ESM/CJS state duplication reviewed
[ ] tarball tested
[ ] TypeScript surface tested
[ ] bundler compatibility tested
[ ] public API diff automated
```

---

# 286. Conditional Exports Checklist

```text
[ ] every condition has purpose
[ ] condition activation documented
[ ] ordering reviewed
[ ] default fallback present
[ ] branch reachability tested
[ ] branch behavior tested
[ ] unknown condition tested
[ ] custom condition names portable
[ ] no accidental condition shadowing
[ ] runtime/tooling condition divergence tested
```

---

# 287. Dual Package Checklist

```text
[ ] import branch
[ ] require branch
[ ] same semantic contract
[ ] state duplication reviewed
[ ] singleton duplication reviewed
[ ] class identity reviewed
[ ] Symbol identity reviewed
[ ] error identity reviewed
[ ] event bus reviewed
[ ] cache reviewed
[ ] documentation explicit
```

---

# 288. Published Artifact Checklist

```text
[ ] build succeeds
[ ] npm pack succeeds
[ ] tarball inspected
[ ] package.json correct
[ ] dist exists
[ ] export targets exist
[ ] type targets exist
[ ] root import works
[ ] root require works where supported
[ ] public subpaths work
[ ] blocked subpaths fail
[ ] tarball install works
[ ] consumer fixture succeeds
```

---

# 289. Resolution Debugging Checklist

```text
[ ] exact specifier
[ ] consumer module format
[ ] Node version
[ ] package name
[ ] package location
[ ] package.json selected
[ ] exports present?
[ ] exact key?
[ ] pattern?
[ ] condition set
[ ] condition order
[ ] target path
[ ] target existence
[ ] packed tarball
[ ] loader compatibility
[ ] error class.
```

---

# 290. Current Node Platform Notes

As of September 11, 2026, the current official Node.js documentation baseline is v26.8.2.

Node documents:

```text
exports
imports
conditional exports
subpath exports
subpath imports
self-reference
export patterns
null targets
node
node-addons
import
require
module-sync
default
custom conditions.
```

Current documentation recommends `exports` for new packages targeting supported Node versions, with `exports` taking precedence over `main` when both are defined. Export target paths must be package-relative and Node enforces restrictions against traversal and `node_modules` target segments. citeturn275170search0

Current Node CommonJS documentation shows that package-name `require()` resolution consults the package `exports` map and calls the ESM package resolver with the appropriate condition set. citeturn275170search1

---

# 291. Specification / Runtime Source Discipline

Primary sources:

```text
Node.js Packages documentation
Node.js ESM documentation
Node.js CommonJS Modules documentation
ECMAScript Modules specification
package manager documentation
TypeScript module resolution documentation
bundler resolution documentation
```

For every claim distinguish:

```text
ECMAScript guarantee
Node package-resolution behavior
CommonJS loader behavior
bundler behavior
TypeScript behavior
package-manager behavior
application convention.
```

Never turn:

```text
bundler behavior
```

into:

```text
Node guarantee.
```

---

# 292. Performance Considerations

Exports maps can improve:

```text
API discipline
```

but conditional branches increase:

```text
testing matrix
tooling complexity
release complexity.
```

Subpaths can improve:

```text
consumer granularity
```

while a huge root graph can increase:

```text
startup cost.
```

Measure:

```text
real package performance.
```

Do not assume:

```text
more subpaths
=
smaller bundle.
```

---

# 293. Memory Considerations

Different condition branches can create:

```text
different module graphs
```

and dual package designs can create:

```text
duplicate state.
```

Watch for:

```text
multiple singleton registries
connection pools
caches
event buses.
```

---

# 294. Security Considerations

Use `exports` to:

```text
reduce accidental public surface
```

but do not treat it as:

```text
sandbox.
```

Review:

```text
native branches
custom conditions
package provenance
published artifacts
dependency confusion.
```

---

# 295. Common Misconceptions

### Misconception 1

```text
“exports is just a nicer main.”
```

Reality:

```text
exports defines a multi-entry, conditional, encapsulated package interface.
```

### Misconception 2

```text
“Not exported means physically inaccessible.”
```

Reality:

```text
package-specifier access is restricted; direct filesystem access is different.
```

citeturn275170search0

### Misconception 3

```text
“default can go anywhere.”
```

Reality:

```text
condition ordering matters.
```

### Misconception 4

```text
“import and require always load the same module instance.”
```

Reality:

```text
dual designs can produce separate module state.
```

### Misconception 5

```text
“works in a repository means package is published correctly.”
```

Reality:

```text
tarball contents are part of the contract.
```

### Misconception 6

```text
“Node and bundlers resolve packages identically.”
```

Reality:

```text
their condition sets and resolution policies can differ.
```

### Misconception 7

```text
“custom conditions are environment variables.”
```

Reality:

```text
they are resolver conditions, activated through loader/tool configuration.
```

---

# 296. Common Mistakes

```text
[ ] adding exports without migration
[ ] default before specific conditions
[ ] missing default fallback
[ ] overly broad patterns
[ ] accidentally exporting internals
[ ] missing null exclusions
[ ] target outside package
[ ] target not included in tarball
[ ] deep imports in tests
[ ] no CJS consumer test
[ ] no ESM consumer test
[ ] no condition branch test
[ ] dual package singleton duplication
[ ] custom conditions undocumented
[ ] runtime/type surface mismatch
[ ] bundler/Node resolution mismatch
[ ] assuming import.meta.resolve = universal resolver
[ ] assuming require.resolve = ESM resolver
[ ] source tree tested but tarball not tested
```

---

# 297. Final Package Resolution Mental Model

```text
CONSUMER SPECIFIER
        ↓
RESOLUTION CONTEXT
        ↓
PACKAGE LOCATION
        ↓
PACKAGE.JSON
        ↓
exports?
 ├─ yes
 │   ↓
 │ subpath match
 │   ↓
 │ condition match
 │   ↓
 │ target
 │   ↓
 │ module format/load
 │
 └─ no
     ↓
 legacy package resolution
```

---

# 298. Conditional Resolution Mental Model

```text
PACKAGE MAP
    ↓
SUBPATH
    ↓
CONDITION TREE
    ↓
MOST SPECIFIC MATCH
    ↓
FALLBACK
    ↓
TARGET
```

---

# 299. Public API Mental Model

```text
FILES
 ↓
EXPORT MAP
 ↓
PUBLIC SPECIFIERS
 ↓
CONSUMER CODE
```

The API is:

```text
specifier
```

not:

```text
physical file path.
```

---

# 300. Dual Package Mental Model

```text
        package
          ↓
    ┌─────┴─────┐
 import       require
    ↓             ↓
  ESM           CJS
    ↓             ↓
state A        state B
```

Ask:

```text
Should A and B share state?
```

If yes:

```text
design explicitly.
```

---

# 301. Condition Mental Model

```text
specific
   ↓
less specific
   ↓
default
```

Never:

```text
default
   ↓
specific.
```

---

# 302. Package Migration Mental Model

```text
LEGACY PACKAGE
 ↓
INVENTORY DEEP IMPORTS
 ↓
DEFINE PUBLIC API
 ↓
ADD EXPORTS FOR SUPPORTED PATHS
 ↓
TEST CONSUMERS
 ↓
PACK
 ↓
INSTALL TARBALL
 ↓
VERIFY
 ↓
RELEASE
```

---

# 303. Resolver Debugging Mental Model

```text
WHAT WAS REQUESTED?
        ↓
WHO REQUESTED IT?
        ↓
WHICH LOADER?
        ↓
WHICH PACKAGE.JSON?
        ↓
WHICH CONDITIONS?
        ↓
WHICH EXPORT KEY?
        ↓
WHICH TARGET?
        ↓
DOES TARGET EXIST?
        ↓
CAN TARGET LOAD?
```

---

# 304. Package Platform Mental Model

```text
SOURCE
 ↓
BUILD
 ↓
PACKAGE METADATA
 ↓
PACKED ARTIFACT
 ↓
NODE/BUNDLER/TS RESOLUTION
 ↓
RUNTIME MODULE
```

Every layer must:

```text
agree.
```

---

# 305. Dependency Graph

```text
Chapter 30
JavaScript Modules
        ↓
Chapter 49
ESM / CommonJS
        ↓
Chapter 126
Module Linking / Resolution / Async Evaluation
        ↓
Chapter 143
Native Addons / ABI
        ↓
Chapter 148
Packaging / ESM-CJS Artifacts
        ↓
Chapter 147
Package Exports / Conditional Exports / Modern Resolution
```

Cross-cutting:

```text
Node runtime
package managers
TypeScript
bundlers
monorepos
native addons
security
semver
CI
distribution.
```

---

# 306. Concept Connections

## Depends On

```text
ES modules
CommonJS
package.json
Node package resolution
module loading
file formats
Node runtime.
```

## Builds Toward

```text
package engineering
distribution
ESM/CJS packaging
build artifacts
library publishing
monorepo architecture
runtime portability.
```

## Related Concepts

```text
main
exports
imports
conditions
patterns
subpaths
self-reference
dual packages
module-sync
node-addons
default.
```

## Concepts Revisited

```text
module resolution
module identity
package boundaries
native addons
security
semver
testing
build systems.
```

## Why This Chapter Matters

A library is not merely:

```text
source code
```

or:

```text
npm tarball.
```

It is:

```text
public specifiers
+
resolution rules
+
conditions
+
module formats
+
published artifacts.
```

That makes:

```text
package resolution
```

a first-class engineering discipline.

---

# 307. Revision / Retrieval Record

```md
# Chapter 147 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## exports
-

## main
-

## imports
-

## subpaths
-

## patterns
-

## null targets
-

## conditionals
-

## condition ordering
-

## default
-

## node
-

## node-addons
-

## import
-

## require
-

## module-sync
-

## custom conditions
-

## self-reference
-

## package encapsulation
-

## dual package hazards
-

## ESM/CJS
-

## monorepos
-

## TypeScript compatibility
-

## bundler compatibility
-

## tarball testing
-

## CI verification
-

## security
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

# 308. Spaced Retrieval Schedule

### Day 0

Explain:

```text
main
exports
imports
conditions
self-reference.
```

### Day 1

Draw:

```text
specifier → package.json → exports → condition → target.
```

### Day 3

Build:

```text
ESM/CJS dual package.
```

### Day 7

Build:

```text
node-addons/default package.
```

### Day 14

Build:

```text
conditional monorepo package
+
consumer fixtures.
```

### Day 21

Build:

```text
export-map validator
+
API diff.
```

### Day 30

Design:

```text
library distribution architecture
```

for:

```text
Node
browser
TypeScript
bundlers
native platforms.
```

---

# 309. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
read and write basic exports maps.
```

Mark:

```text
[?] Needs Revision
```

when you:

```text
misorder conditions
break CJS/ESM consumers
expose internals accidentally
forget tarball testing
cannot explain deep-import failures.
```

Mark:

```text
[+] Completed
```

when you can:

```text
design and test explicit public package surfaces,
conditional branches, ESM/CJS compatibility,
and published artifacts.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
design a package-resolution strategy across Node,
bundlers, TypeScript, monorepos, native fallbacks,
dual modules, conditional branches, and semver
without accidentally creating incompatible module graphs.
```

Reading alone does not mark mastery.

---

# 310. Final Principal Principle

> **Package metadata is executable compatibility policy. `exports` defines what consumers are allowed to address, conditional exports define which implementation they receive, and the published artifact determines whether that contract actually works in the real ecosystem. Treat every export key, condition, target, and format boundary as production API.**

The principal workflow is:

```text
DEFINE PUBLIC API
→
DEFINE PACKAGE BOUNDARY
→
CHOOSE MODULE STRATEGY
→
DESIGN CONDITIONS
→
ORDER CONDITIONS
→
DEFINE FALLBACK
→
VALIDATE TARGETS
→
BUILD ARTIFACTS
→
PACK
→
TEST CONSUMER CONTEXTS
→
DIFF PUBLIC API
→
RELEASE
```

Remember:

```text
exports ≠ main

exports ≠ filesystem sandbox

imports ≠ exports

condition ≠ environment variable

default ≠ “put anywhere”

pattern ≠ harmless wildcard

self-reference ≠ deep import

dual package ≠ automatically shared state

Node resolution ≠ every bundler's resolution

repository tree ≠ published package

working build ≠ working tarball

specifier stability > physical file stability

public subpath = long-term API responsibility

conditional branch = compatibility surface

module-sync ≠ node-addons

package encapsulation ≠ supply-chain security.
```

The principal question is:

```text
“What exact public specifiers do we promise, which resolution
contexts must they support, which implementation should each
context receive, what state and compatibility hazards exist
between those branches, and how will we prove that the published
artifact resolves exactly as our consumers expect?”
```

That is modern JavaScript package-resolution engineering.


# Implementation From Scratch

Build, in progression:

```text
package manifest parser
exact export matcher
pattern matcher
conditional resolver
null-target handling
target validator
consumer resolution fixture
tarball verifier
public export diff tool.
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```
