# Chapter 148 — JavaScript Build Artifacts, ESM/CJS Packaging & Distribution

> **JavaScript Mastery — Part XXVII: Modules, Build Systems, Packaging & Release Engineering**
>
> **Mission:** Master the complete path from JavaScript/TypeScript source to a reliable published artifact: build graphs, transpilation, bundling, preservation of ESM semantics, CommonJS compatibility, file extensions, declaration assets, source maps, package metadata, export maps, package contents, npm tarballs, conditional builds, native artifacts, browser distributions, Node distributions, package provenance, reproducible releases, compatibility matrices, and principal-level distribution architecture.
>
> **Role perspective:** Principal JavaScript Engineer · Build Engineer · Package Maintainer · Node.js Runtime Engineer · Library Architect · Release Engineer · Supply-Chain Security Engineer · Monorepo Architect
>
> **Status:** `[ ] Not Started`
>
> **Core principle:** **Source code is not the product artifact. Consumers execute the published module graph, and that graph is created by the build, package metadata, package contents, resolver, and runtime together. A production package is correct only when those layers agree.**

---

# 1. Learning Objectives

```text
[ ] explain build artifacts
[ ] explain source artifacts
[ ] explain distribution artifacts
[ ] explain transpilation
[ ] explain compilation
[ ] explain bundling
[ ] explain minification
[ ] explain tree shaking
[ ] explain code splitting
[ ] explain dead-code elimination
[ ] explain syntax lowering
[ ] explain polyfilling
[ ] distinguish syntax transform from runtime polyfill
[ ] distinguish transpiling from bundling
[ ] distinguish bundling from packaging
[ ] distinguish packaging from publishing
[ ] explain ESM
[ ] explain CommonJS
[ ] explain dual packaging
[ ] explain ESM-only packaging
[ ] explain CJS-only packaging
[ ] explain hybrid packaging
[ ] explain conditional builds
[ ] explain conditional exports
[ ] explain source-vs-dist architecture
[ ] explain package.json
[ ] explain exports
[ ] explain imports
[ ] explain main
[ ] explain type
[ ] explain files
[ ] explain bin
[ ] explain types
[ ] explain typings
[ ] explain sideEffects
[ ] explain engines
[ ] explain optionalDependencies
[ ] explain peerDependencies
[ ] explain bundledDependencies conceptually
[ ] explain publishConfig
[ ] explain package tarballs
[ ] explain npm pack
[ ] explain npm publish
[ ] explain package inclusion rules
[ ] inspect published artifacts
[ ] validate export targets
[ ] validate file existence
[ ] validate package contents
[ ] validate type declarations
[ ] validate source maps
[ ] validate CLI binaries
[ ] validate ESM consumers
[ ] validate CJS consumers
[ ] validate TypeScript consumers
[ ] validate bundler consumers
[ ] validate browser consumers
[ ] understand source maps
[ ] explain inline source maps
[ ] explain external source maps
[ ] explain declaration maps
[ ] explain sourceRoot
[ ] explain sourcesContent
[ ] understand stack trace mapping
[ ] understand debugger mapping
[ ] understand package provenance
[ ] understand integrity metadata
[ ] understand lockfiles
[ ] understand reproducible builds
[ ] understand deterministic archives
[ ] understand build environment pinning
[ ] understand artifact digests
[ ] understand content-addressable storage
[ ] understand release metadata
[ ] understand changelog generation
[ ] understand semantic versioning
[ ] identify public API changes
[ ] identify artifact compatibility changes
[ ] identify module-format changes
[ ] identify export-map changes
[ ] identify source-map changes
[ ] identify declaration changes
[ ] identify native-binary changes
[ ] identify browser-build changes
[ ] identify runtime-target changes
[ ] understand target environments
[ ] understand Node targets
[ ] understand browser targets
[ ] understand edge/runtime targets
[ ] understand worker targets
[ ] understand minimum language level
[ ] understand minimum runtime level
[ ] understand package compatibility
[ ] understand feature detection
[ ] understand transpilation targets
[ ] understand browserslist conceptually
[ ] understand Node target matrices
[ ] understand syntax compatibility
[ ] understand API compatibility
[ ] understand semantic compatibility
[ ] understand runtime compatibility
[ ] understand dependency compatibility
[ ] understand loader compatibility
[ ] understand native ABI compatibility
[ ] understand WASM compatibility
[ ] explain ESM file extensions
[ ] explain .mjs
[ ] explain .cjs
[ ] explain .js under package type
[ ] understand package type
[ ] understand extension discipline
[ ] understand import specifiers
[ ] understand require specifiers
[ ] understand dynamic import
[ ] understand createRequire
[ ] understand module-sync
[ ] understand top-level await impact
[ ] understand CommonJS interop
[ ] understand default exports
[ ] understand named exports
[ ] understand CJS namespace construction
[ ] understand CJS named-export detection
[ ] understand live binding differences
[ ] understand module.exports customization
[ ] understand dual module hazards
[ ] understand duplicated state
[ ] understand class identity duplication
[ ] understand Symbol identity duplication
[ ] design one canonical implementation
[ ] design ESM wrapper
[ ] design CJS wrapper
[ ] design shared internal core
[ ] design conditional export graph
[ ] design browser fallback
[ ] design Node-native optimization
[ ] design portable fallback
[ ] understand native addons in packages
[ ] understand optional native dependencies
[ ] understand prebuilds
[ ] understand source builds
[ ] understand platform matrices
[ ] understand architecture matrices
[ ] understand libc matrices
[ ] understand CPU-feature matrices
[ ] understand native build failures
[ ] understand fallback strategy
[ ] understand install scripts
[ ] understand build-time dependencies
[ ] understand runtime dependencies
[ ] understand dev dependencies
[ ] understand peer dependencies
[ ] understand optional dependencies
[ ] understand dependency bundling trade-offs
[ ] understand package size
[ ] understand startup cost
[ ] understand tree-shaking impact
[ ] understand source code leakage
[ ] understand license files
[ ] understand NOTICE files
[ ] understand README publishing
[ ] understand package metadata exposure
[ ] understand private source maps
[ ] understand source map security
[ ] understand declaration packaging
[ ] understand package file allowlists
[ ] understand package ignore behavior
[ ] understand package manager differences
[ ] understand workspace publishing
[ ] understand monorepo build graphs
[ ] understand incremental builds
[ ] understand caching
[ ] understand remote caching
[ ] understand build invalidation
[ ] understand content hashes
[ ] understand artifact reuse
[ ] understand task pipelines
[ ] understand build dependencies
[ ] understand circular build dependencies
[ ] understand build isolation
[ ] understand generated sources
[ ] understand generated declarations
[ ] understand generated metadata
[ ] understand postbuild validation
[ ] understand release pipelines
[ ] understand canary releases
[ ] understand prereleases
[ ] understand dist-tags
[ ] understand rollback
[ ] understand package unpublishing constraints conceptually
[ ] understand yanked/retracted release strategies
[ ] understand deprecation
[ ] understand security advisories
[ ] understand compromised packages
[ ] understand provenance
[ ] understand signed artifacts conceptually
[ ] understand CI trust boundaries
[ ] understand publish credentials
[ ] understand npm access controls conceptually
[ ] understand secret isolation
[ ] understand package integrity
[ ] implement a source-to-dist pipeline
[ ] implement an ESM build
[ ] implement a CJS build
[ ] implement type declarations
[ ] implement source maps
[ ] implement a dual package
[ ] implement export validation
[ ] implement tarball verification
[ ] implement consumer smoke tests
[ ] implement artifact manifest
[ ] implement API diff
[ ] implement reproducible build checks
[ ] debug missing files
[ ] debug wrong format
[ ] debug source-map failures
[ ] debug CJS/ESM failures
[ ] debug package publish failures
[ ] debug bundler compatibility
[ ] debug TypeScript compatibility
[ ] debug native package failures
[ ] debug conditional build failures
[ ] compare build strategies
[ ] defend packaging choices in interviews
[ ] make principal distribution decisions


# 2. Prerequisites

You should already understand:

```text
Chapter 49 — ESM / CommonJS
Chapter 126 — Module Linking, Resolution & Async Module Evaluation
Chapter 147 — Package exports, Conditional Exports & Modern Resolution
Chapter 143 — Native Addons / N-API / ABI Boundaries
Chapter 145 — Property-Based Testing / Fuzzing
Chapter 146 — Determinism / Reproducibility / Flaky Testing
```

Also understand:

```text
package.json
npm
node_modules
TypeScript at a conceptual level
bundlers
source maps
CI/CD
semantic versioning
monorepos
native binaries.
```

---

# 3. What Is a Build Artifact?

A build artifact is:

```text
output produced from source/build inputs
```

that another tool or runtime can consume.

Examples:

```text
JavaScript files
CommonJS files
ES modules
source maps
.d.ts files
declaration maps
WASM modules
native binaries
CLI wrappers
CSS
metadata.
```

---

# 4. Source vs Artifact

Source:

```text
src/index.ts
```

Artifact:

```text
dist/index.js
dist/index.cjs
dist/index.d.ts
```

Published package:

```text
package.tar.gz
```

These are:

```text
different representations
```

of the same:

```text
product contract.
```

---

# 5. Build Pipeline

A common pipeline:

```text
SOURCE
 ↓
PARSE
 ↓
TRANSFORM
 ↓
TYPE/STATIC ANALYSIS
 ↓
BUNDLE OR EMIT
 ↓
MINIFY
 ↓
SOURCE MAP
 ↓
DECLARATIONS
 ↓
PACKAGE
 ↓
VALIDATE
 ↓
PUBLISH
```

---

# 6. Transpilation

Transpilation means:

```text
transform source syntax
```

into:

```text
different syntax
```

with the goal of preserving:

```text
intended semantics.
```

Example:

```text
modern syntax
→
older syntax.
```

---

# 7. Transpilation Is Not Polyfilling

Changing:

```text
syntax
```

does not automatically provide:

```text
missing runtime API.
```

Example:

```text
transform async syntax
```

does not necessarily provide:

```text
missing Promise implementation.
```

---

# 8. Polyfill

A polyfill supplies:

```text
runtime capability
```

that the target environment does not natively provide.

Therefore:

```text
syntax compatibility
≠
runtime API compatibility.
```

---

# 9. Bundling

Bundling combines:

```text
multiple modules
```

into:

```text
one or more distribution artifacts.
```

Bundling can affect:

```text
module boundaries
dynamic imports
side effects
tree shaking
debugging.
```

---

# 10. Packaging

Packaging organizes:

```text
build artifacts
+
metadata
+
licenses
+
README
+
files.
```

into:

```text
installable distribution.
```

---

# 11. Publishing

Publishing transfers:

```text
package artifact
```

to:

```text
registry/distribution system.
```

Publishing is:

```text
release operation,
```

not:

```text
build.
```

---

# 12. Product Artifact Mental Model

```text
SOURCE
 ↓
BUILD OUTPUT
 ↓
PACKAGE
 ↓
REGISTRY
 ↓
INSTALL
 ↓
RESOLUTION
 ↓
LOADER
 ↓
RUNTIME
```

A failure can exist at any layer.

---

# 13. Why Artifact Engineering Matters

The code that works in:

```text
repository source
```

can still fail after:

```text
build
```

because of:

```text
wrong extension
missing file
wrong export target
wrong format
missing dependency
bad source map
missing declaration.
```

---

# 14. ESM Distribution

An ESM artifact uses:

```js
import
export
```

and follows:

```text
ES module semantics.
```

Node's current documentation describes ECMAScript Modules as stable and supports explicit ESM markers including `.mjs` and package `"type": "module"`. citeturn378470search1

---

# 15. CommonJS Distribution

A CommonJS artifact can use:

```js
const x = require("x");
module.exports = ...
```

Its loading model differs from:

```text
ESM.
```

---

# 16. Dual Distribution

A dual package supports:

```text
ESM consumer
+
CommonJS consumer.
```

Usually:

```json
{
  "exports": {
    "import": "./dist/index.js",
    "require": "./dist/index.cjs"
  }
}
```

---

# 17. One Source, Two Artifacts

A common architecture:

```text
src/
  index.ts
```

produces:

```text
dist/
  index.js
  index.cjs
  index.d.ts
```

This requires:

```text
semantic parity
```

between:

```text
ESM
and
CJS.
```

---

# 18. Semantic Parity

If:

```text
ESM API
```

returns:

```text
A
```

while:

```text
CJS API
```

returns:

```text
B
```

the package is:

```text
conditionally inconsistent.
```

---

# 19. Dual Package Hazard

Two formats can create:

```text
two module instances
```

with:

```text
duplicated state.
```

See:

```text
Chapter 147.
```

---

# 20. Shared Core Strategy

Prefer:

```text
src/core/*
       ↓
   shared logic
   ↙        ↘
ESM build   CJS wrapper
```

rather than:

```text
two independent implementations.
```

---

# 21. CJS Wrapper Strategy

Conceptually:

```js
module.exports =
  createPublicAPI(core);
```

The exact design depends on:

```text
source format
build tool
API shape
interop.
```

---

# 22. ESM Wrapper Strategy

Conceptually:

```js
export {
  foo,
  bar
};
```

from:

```text
shared compiled logic.
```

---

# 23. Wrapper Principle

Keep format-specific wrappers:

```text
thin.
```

The more logic they contain:

```text
greater divergence risk.
```

---

# 24. CJS Interop

Node allows ESM code to import CommonJS. The default export corresponds to the CommonJS `module.exports` value, while named exports are provided through static analysis heuristics and are not guaranteed to represent live updates. citeturn378470search1

---

# 25. CJS Named Export Detection

This behavior is:

```text
best effort
```

rather than:

```text
ESM live-binding semantics.
```

Library authors should prefer:

```text
stable default/object API
```

when interoperability must be robust.

---

# 26. `module.exports` Interop

Current Node also exposes a:

```text
'module.exports'
```

named marker in CommonJS namespaces and supports explicitly exporting a value from ESM under the string name `"module.exports"` for `require(esm)` interoperability. citeturn378470search1turn378470search2

Treat these as:

```text
advanced interoperability behavior
```

and test exact supported Node versions.

---

# 27. `require()` of ESM

Current Node CommonJS `require()` supports:

```text
synchronous ES modules
```

that do not use:

```text
top-level await.
```

Node documents this restriction explicitly. citeturn378470search1turn378470search2

---

# 28. Top-Level Await Packaging Hazard

An ESM build can introduce:

```text
top-level await
```

and accidentally make:

```text
require()
```

incompatible.

Therefore:

```text
build transformations
```

can change:

```text
loader compatibility.
```

---

# 29. Build Must Preserve Loader Contract

Before shipping:

```text
ESM import
CJS require
dynamic import
```

must be tested according to:

```text
documented support.
```

---

# 30. `module-sync`

Node's package condition:

```text
module-sync
```

represents an ESM graph that can be synchronously loaded, including through `require()`, as long as the graph contains no top-level await. citeturn378470search0

This can be useful for:

```text
single sync ESM implementation.
```

---

# 31. `.mjs`, `.cjs`, `.js`

`.mjs`:

```text
ESM
```

`.cjs`:

```text
CommonJS
```

`.js`:

```text
depends on package type/scope
```

Node documents these explicit module-format controls. citeturn378470search1

---

# 32. Package `"type"`

```json
{
  "type": "module"
}
```

means:

```text
.js
```

files in that package scope are interpreted as:

```text
ESM
```

unless explicitly overridden by:

```text
.cjs.
```

---

# 33. Explicit Extension Strategy

A robust dual package can use:

```text
index.js   → ESM under type=module
index.cjs  → CommonJS
```

This makes:

```text
format
```

obvious.

---

# 34. Extension Discipline

Avoid producing:

```text
ambiguous artifacts.
```

Choose:

```text
consistent file-format policy
```

and encode it in:

```text
package.json
exports.
```

---

# 35. Build Entry Points

A build should explicitly define:

```text
entry files
format
target runtime
output directory
source map policy
declaration policy.
```

---

# 36. Multiple Entry Points

```text
root
client
server
cli
worker
```

may require:

```text
separate output artifacts.
```

Do not accidentally:

```text
bundle all functionality into every entry.
```

---

# 37. Entry-Point Graph

```text
root
 ├─ shared core
 ├─ feature A
 └─ feature B

client
 ├─ shared core
 └─ browser adapter

server
 ├─ shared core
 └─ Node adapter
```

This improves:

```text
distribution clarity.
```

---

# 38. Barrel Files

A root barrel can:

```text
re-export many modules.
```

Useful for:

```text
API ergonomics.
```

Potential downside:

```text
large dependency graph
eager evaluation
tree-shaking complexity.
```

---

# 39. Tree Shaking

Tree shaking removes:

```text
unused statically analyzable code
```

in compatible bundlers.

It depends on:

```text
module structure
side-effect semantics
bundler analysis.
```

---

# 40. ESM and Tree Shaking

ESM's static structure generally makes:

```text
import/export relationships
```

more analyzable than:

```text
dynamic CommonJS patterns.
```

But:

```text
ESM syntax
```

does not guarantee:

```text
tree shaking.
```

---

# 41. `sideEffects`

Some package ecosystems use:

```json
{
  "sideEffects": false
}
```

as bundler metadata indicating:

```text
modules can be removed when unused.
```

This is not:

```text
a JavaScript language feature.
```

Misdeclaring it can remove:

```text
required side-effect code.
```

---

# 42. Side-Effect Safety

Potential side effects:

```text
polyfill installation
global mutation
CSS import
registration
event listener installation
telemetry.
```

Review before declaring:

```text
sideEffects: false.
```

---

# 43. Dead Code Elimination

DCE removes:

```text
provably unused code.
```

A bad build assumption can remove:

```text
dynamically discovered behavior.
```

---

# 44. Dynamic Imports

Preserve:

```js
import("./feature.js");
```

when packaging requires:

```text
runtime code splitting.
```

A bundler may instead create:

```text
separate chunk.
```

---

# 45. Dynamic Import Artifact Contract

After bundling verify:

```text
chunk exists
runtime knows chunk URL/path
package contains chunk
server/browser can load it.
```

---

# 46. Code Splitting

Code splitting creates:

```text
multiple runtime artifacts
```

from one source graph.

This affects:

```text
hosting
package distribution
cache strategy
path resolution.
```

---

# 47. Library vs Application Bundling

Applications can often:

```text
bundle aggressively.
```

Libraries often need:

```text
preserve module boundaries
```

so consumers can:

```text
tree-shake
choose environment
avoid duplicate dependencies.
```

---

# 48. Library Bundling Anti-Pattern

Bundling every dependency into:

```text
one giant file
```

can cause:

```text
duplicate dependencies
large package
broken singleton identity
license complexity.
```

---

# 49. External Dependencies

Library builds may mark dependencies as:

```text
external
```

so consumers load:

```text
their own copy.
```

But some dependencies may need:

```text
bundling
```

for:

```text
self-contained distribution
```

or:

```text
runtime compatibility.
```

---

# 50. Dependency Bundling Decision

Ask:

```text
Is dependency part of public runtime?
Is singleton identity important?
Does consumer need one shared instance?
Does dependency have platform assumptions?
Does license permit bundling?
Does package size justify bundling?
```

---

# 51. Peer Dependencies

Peer dependencies express:

```text
consumer-provided compatibility dependency.
```

Useful for:

```text
framework integrations
plugins
host APIs.
```

Bundling a peer can break:

```text
identity/compatibility assumptions.
```

---

# 52. Optional Dependencies

Optional dependencies can represent:

```text
enhanced capability
```

with:

```text
portable fallback.
```

Example:

```text
native accelerator
```

with:

```text
pure JS fallback.
```

---

# 53. Native Distribution

Native packages can distribute:

```text
prebuilt binaries
```

or:

```text
build from source.
```

Common matrix:

```text
OS
architecture
libc
Node ABI/N-API
CPU feature.
```

---

# 54. N-API Benefit

Node-API reduces direct dependence on:

```text
V8 ABI
```

for native addons.

But package distribution still requires:

```text
platform artifact
```

and:

```text
installation strategy.
```

See:

```text
Chapter 143.
```

---

# 55. Native Prebuild Strategy

```text
install
 ↓
detect platform
 ↓
download/select prebuilt
 ↓
fallback to source build
```

or:

```text
build locally.
```

---

# 56. Native Packaging Risks

```text
binary size
platform coverage
download reliability
ABI compatibility
supply chain
compiler availability
permission/security.
```

---

# 57. Native + Default

A package can use:

```json
{
  "exports": {
    "node-addons": "./dist/native.js",
    "default": "./dist/portable.js"
  }
}
```

with portable fallback.

Node documents `node-addons` as the condition for native C++ addon entry points. citeturn378470search0

---

# 58. Browser Artifact

Browser distribution may require:

```text
no Node built-ins
web-compatible APIs
bundler-friendly format
```

and:

```text
correct global/runtime assumptions.
```

---

# 59. Browser vs Node Build

Do not merely:

```text
replace process with browser.
```

Review:

```text
fs
net
tls
Buffer
child_process
crypto
URL
streams
timers.
```

---

# 60. Universal Build

A universal build targets:

```text
common standards
```

with minimal:

```text
runtime assumptions.
```

Pros:

```text
portability.
```

Cons:

```text
less specialized optimization.
```

---

# 61. Specialized Build

A Node build can use:

```text
native APIs
Node built-ins
native addon
```

for:

```text
better integration
performance.
```

Cost:

```text
less portability
```

---

# 62. Target Strategy

Choose:

```text
universal
vs
specialized
```

based on:

```text
consumer population
performance
maintenance
compatibility.
```

---

# 63. Syntax Targeting

Target:

```text
lowest runtime syntax
```

that must be supported.

Do not transpile:

```text
everything to oldest syntax
```

without:

```text
reason.
```

Older syntax can increase:

```text
code size
helper overhead
debug complexity.
```

---

# 64. Runtime API Targeting

Separate:

```text
syntax target
```

from:

```text
API target.
```

A runtime can understand:

```text
modern syntax
```

but lack:

```text
new API.
```

---

# 65. Target Matrix

Example:

| Consumer | Syntax | APIs | Format |
|---|---|---|---|
| Node 26 | modern | Node 26 | ESM/CJS |
| older Node | lowered | polyfilled/limited | CJS |
| modern browsers | modern | Web APIs | ESM |
| legacy browser | lowered | polyfills | bundled |
| edge runtime | conservative | runtime-specific | ESM |

The exact matrix belongs to:

```text
supported product versions.
```

---

# 66. Build Matrix Explosion

Every combination of:

```text
runtime
format
condition
platform
architecture
language target
```

adds:

```text
testing cost.
```

Keep the matrix:

```text
minimal but sufficient.
```

---

# 67. Source Maps

Source maps connect:

```text
generated code
```

back to:

```text
source code.
```

Useful for:

```text
debugging
stack traces
production diagnostics.
```

---

# 68. External Source Maps

Example:

```text
dist/index.js
dist/index.js.map
```

Pros:

```text
separate artifact
smaller runtime JS
debugging support.
```

Cons:

```text
must publish/deploy map
may expose source.
```

---

# 69. Inline Source Maps

Source map data is embedded into:

```text
generated file.
```

Pros:

```text
self-contained.
```

Cons:

```text
larger artifact
source exposure
memory/transfer overhead.
```

---

# 70. `sourcesContent`

Source maps may embed:

```text
original source content.
```

This makes debugging easier but can expose:

```text
private source
comments
paths
internal implementation.
```

---

# 71. Source Map Security

Never assume:

```text
source map
=
harmless debug metadata.
```

Review:

```text
customer names
internal paths
comments
secrets accidentally present in source.
```

---

# 72. Declaration Files

TypeScript-compatible packages may publish:

```text
.d.ts
```

files describing:

```text
public types.
```

---

# 73. Declaration Generation

Build pipeline:

```text
source
→
type analysis
→
declaration emit
```

The declarations must correspond to:

```text
published runtime API.
```

---

# 74. Declaration Mismatch

Bad:

```text
runtime exports:
createClient()

types:
createServer()
```

This creates:

```text
runtime/type divergence.
```

---

# 75. Declaration Maps

Declaration maps connect:

```text
.d.ts
```

back to:

```text
source.
```

Useful for:

```text
IDE navigation
library debugging
source jump.
```

---

# 76. Generated Declaration Paths

Ensure:

```text
types
```

point to:

```text
published declaration files.
```

not:

```text
repository source.
```

---

# 77. `types` / `typings`

Package metadata can point consumers to:

```text
type declaration entry.
```

The exact runtime/export and type-resolution story must be tested across:

```text
TypeScript versions
```

you support.

---

# 78. Runtime vs Type Package Surface

A package can have:

```text
runtime public API
```

and:

```text
type public API.
```

They must:

```text
align.
```

---

# 79. Generated Types and Conditions

Conditional runtime branches may need:

```text
matching type declarations.
```

Otherwise:

```text
correct runtime branch
```

can still produce:

```text
wrong developer experience.
```

---

# 80. Build Reproducibility

A reproducible build means:

```text
same source
+
same dependencies
+
same build configuration
+
same relevant environment
→
equivalent artifact.
```

Equivalent may mean:

```text
byte-identical
```

or:

```text
semantically identical
```

depending on release requirements.

---

# 81. Byte-Reproducible Build

Harder than:

```text
semantic reproducibility.
```

Potential sources of nondeterminism:

```text
timestamps
file order
absolute paths
random IDs
hostnames
build machine
archive metadata
```

---

# 82. Deterministic File Ordering

Package archives should use:

```text
stable file ordering
```

when reproducibility requires:

```text
byte identity.
```

---

# 83. Deterministic Timestamps

Generated artifacts can include:

```text
timestamps.
```

Use:

```text
controlled timestamps
```

if byte-identical release artifacts are required.

---

# 84. Build Path Leakage

A source map can contain:

```text
/home/runner/project/src/index.ts
```

while another machine has:

```text
/build/workspace/src/index.ts.
```

Normalize:

```text
source roots
```

where appropriate.

---

# 85. Build Environment Pinning

Pin:

```text
Node
package manager
lockfile
build tool
compiler
OS
architecture
native toolchain
```

for strict reproducibility.

---

# 86. Artifact Hash

Compute:

```text
SHA-256
```

for:

```text
important artifacts.
```

Store:

```text
manifest.
```

---

# 87. Artifact Manifest

Example:

```json
{
  "package": "my-package",
  "version": "1.2.3",
  "node": "26.x",
  "files": {
    "dist/index.js": "sha256:...",
    "dist/index.cjs": "sha256:..."
  }
}
```

Use:

```text
release verification.
```

---

# 88. Build Cache

Build systems can cache:

```text
transformed modules
bundles
type outputs
tests.
```

Cache keys must include:

```text
all relevant inputs.
```

---

# 89. Cache Invalidation

A stale artifact can result when cache key ignores:

```text
compiler version
Node version
config
environment
dependency version.
```

---

# 90. Content Hashing

A useful cache key can include:

```text
source hash
dependency lock hash
toolchain hash
configuration hash.
```

---

# 91. Incremental Builds

Only rebuild:

```text
affected packages/files.
```

This reduces:

```text
monorepo CI time.
```

---

# 92. Incremental Build Risk

If dependency graph is wrong:

```text
changed shared library
```

may fail to rebuild:

```text
dependent package.
```

Then:

```text
stale artifact.
```

---

# 93. Build Graph

```text
package-a
   ↓
package-b
   ↓
package-c
```

If A changes:

```text
B/C
```

may need rebuild.

---

# 94. Circular Build Dependencies

```text
A → B
B → A
```

can make:

```text
incremental build
```

ambiguous.

Prefer:

```text
acyclic build graph.
```

---

# 95. Build vs Runtime Dependency

Build dependency:

```text
TypeScript
bundler
minifier
test tools
```

Runtime dependency:

```text
library consumer actually needs.
```

Do not accidentally ship:

```text
entire build toolchain.
```

---

# 96. Dev Dependency Leakage

A package can work in monorepo because:

```text
devDependency
```

exists at root.

Published consumer does not have it.

This creates:

```text
phantom dependency.
```

---

# 97. Packaging Verification

A packed consumer test exposes:

```text
undeclared dependency
missing file
bad export
wrong build path.
```

---

# 98. `npm pack`

Use:

```bash
npm pack
```

to create:

```text
publish-like tarball.
```

Then:

```text
install tarball
```

in:

```text
clean consumer project.
```

---

# 99. Tarball as Golden Artifact

Treat:

```text
tarball
```

as:

```text
release candidate.
```

Run:

```text
consumer tests
```

against it.

---

# 100. Package Contents

Check:

```text
dist
package.json
README
LICENSE
NOTICE
types
maps
bin
native binaries.
```

---

# 101. Missing Artifact

A package can contain:

```text
source
```

but omit:

```text
dist.
```

Consumers then fail even though:

```text
repository tests pass.
```

---

# 102. Excess Artifact

Publishing:

```text
test fixtures
source files
benchmarks
large maps
private data
```

can increase:

```text
package size
```

or:

```text
source exposure.
```

---

# 103. Allowlist Packaging

Prefer:

```text
explicit package contents
```

for sensitive libraries.

Review:

```text
files
ignore rules
generated artifacts.
```

---

# 104. License Files

If a dependency/license requires notice:

```text
include required files.
```

Do not remove:

```text
licenses
notices
```

just to make packages smaller.

---

# 105. README

A published library README should explain:

```text
installation
supported runtimes
imports
require
types
subpaths
security/support.
```

---

# 106. CLI Packages

A package can expose:

```json
{
  "bin": {
    "mytool": "./dist/cli.js"
  }
}
```

Verify:

```text
file exists
executable semantics
shebang where needed
platform behavior.
```

---

# 107. CLI Artifact

Example:

```js
#!/usr/bin/env node
```

The build pipeline must preserve:

```text
shebang
```

when required.

---

# 108. CLI Distribution Matrix

Test:

```text
Linux
macOS
Windows
```

and:

```text
Node supported versions.
```

---

# 109. Source vs Dist CLI Bug

A CLI may work:

```text
tsx src/cli.ts
```

but fail:

```text
node dist/cli.js
```

because:

```text
transpilation
paths
extensions
assets
```

changed.

---

# 110. Runtime Asset Packaging

If code loads:

```text
templates
schemas
WASM
certificates
```

ensure they are:

```text
published.
```

---

# 111. Relative Asset Paths

A build can change:

```text
__dirname
import.meta.url
```

assumptions.

Test:

```text
installed package
```

rather than:

```text
repository path.
```

---

# 112. ESM Asset Resolution

ESM often uses:

```js
new URL("./asset.json", import.meta.url);
```

rather than:

```text
__dirname.
```

Build transformations must preserve:

```text
correct artifact location.
```

---

# 113. CommonJS Asset Resolution

CJS may use:

```js
path.join(__dirname, "asset.json")
```

A dual package must verify:

```text
both branches.
```

---

# 114. Package Root Assumptions

Do not assume:

```text
process.cwd()
```

is:

```text
package root.
```

Consumer applications choose their own:

```text
cwd.
```

---

# 115. Installed Path Testing

Tests should install package into:

```text
external consumer fixture
```

and run from:

```text
different cwd.
```

---

# 116. Absolute Path Leakage

Generated code may contain:

```text
build-machine paths.
```

This can break:

```text
runtime
source maps
snapshots.
```

---

# 117. Source Maps and Published Package

Decide:

```text
publish source maps?
```

based on:

```text
debugging need
source exposure
package size
support policy.
```

---

# 118. Private Maps

Some organizations keep:

```text
private maps
```

in:

```text
symbol server
artifact store.
```

This can preserve:

```text
production debugging
```

without publishing:

```text
full source.
```

---

# 119. Build Failure Taxonomy

```text
syntax transform
dependency graph
format
path
asset
type
source map
package metadata
package contents
native binary
runtime target
publishing.
```

---

# 120. Artifact Validation Order

```text
1. build
2. validate files
3. validate metadata
4. pack
5. inspect tarball
6. install tarball
7. import/require
8. type-check consumer
9. run smoke tests
10. publish.
```

---

# 121. Consumer Fixture

Create:

```text
fixtures/consumer-esm
fixtures/consumer-cjs
fixtures/consumer-ts
fixtures/consumer-bundler
```

Each uses:

```text
published tarball.
```

---

# 122. Consumer Fixture Purpose

It tests:

```text
real package contract
```

instead of:

```text
monorepo source topology.
```

---

# 123. ESM Consumer Fixture

```js
import pkg from "package";
console.log(pkg);
```

Run:

```bash
node consumer.mjs
```

---

# 124. CJS Consumer Fixture

```js
const pkg = require("package");
console.log(pkg);
```

Run:

```bash
node consumer.cjs
```

---

# 125. Dynamic Import Fixture

```js
const pkg =
  await import("package");
```

Verify:

```text
import condition.
```

---

# 126. Subpath Fixture

```js
import client from "package/client";
```

Verify:

```text
public subpath.
```

---

# 127. Blocked Deep Import Fixture

```js
import "package/dist/internal.js";
```

Expect:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED
```

when the subpath is intentionally private.

---

# 128. Conditional Fixture

Run:

```bash
node --conditions=development consumer.mjs
```

Verify:

```text
development branch.
```

---

# 129. `--no-addons` Fixture

For a native package:

```bash
node --no-addons consumer.mjs
```

Verify:

```text
portable fallback
```

where designed.

---

# 130. TypeScript Consumer Fixture

Verify:

```text
runtime imports
+
type declarations
```

resolve the:

```text
same public API.
```

---

# 131. Bundler Consumer Fixture

Build a tiny application using:

```text
package
```

and verify:

```text
bundler resolves expected condition
```

and:

```text
runtime executes.
```

---

# 132. Browser Consumer Fixture

Use:

```text
minimal browser build
```

and verify:

```text
browser/default branch
```

when supported.

---

# 133. API Surface Test

Compare:

```text
documented public exports
```

with:

```text
actual exports map.
```

---

# 134. Export Manifest

Maintain:

```json
{
  ".": true,
  "./client": true,
  "./server": true
}
```

and diff against:

```text
previous release.
```

---

# 135. Public API Diff

Detect:

```text
removed root
removed subpath
new condition
changed format
changed target.
```

---

# 136. Semver Review

Ask:

```text
Did public import paths change?
Did runtime behavior change?
Did module format support change?
Did minimum Node version change?
Did type surface change?
Did native platform support change?
```

---

# 137. Minimum Node Version

Package metadata can communicate:

```text
engine expectations
```

but actual CI must test:

```text
minimum supported version.
```

---

# 138. Build Target vs Engine Range

Do not publish:

```text
engines >= 18
```

while:

```text
build output
```

requires:

```text
Node 22.
```

The declared contract and artifact must agree.

---

# 139. Dependency Engine Constraints

A package may support:

```text
Node 18
```

while a dependency now requires:

```text
Node 22.
```

Your effective compatibility becomes:

```text
intersection of constraints.
```

---

# 140. Dependency Graph Compatibility

For every release:

```text
runtime
+
dependencies
+
build
```

must share:

```text
compatible platform assumptions.
```

---

# 141. Lockfile

Lockfiles capture:

```text
resolved dependency graph
```

and are essential for:

```text
reproducible builds.
```

---

# 142. Registry Metadata vs Tarball

Registry metadata describes:

```text
package version
dependencies
dist integrity
tags.
```

The tarball contains:

```text
actual files.
```

Both matter.

---

# 143. Integrity

Published package metadata commonly includes:

```text
integrity digest.
```

Consumers/package managers can use this to verify:

```text
content integrity.
```

---

# 144. Provenance

Modern supply-chain practice can attach:

```text
build provenance
```

to published artifacts.

The important idea:

```text
who/what built this artifact
from which source
under which workflow?
```

---

# 145. Build Trust Chain

```text
SOURCE COMMIT
 ↓
CI WORKFLOW
 ↓
BUILD ENVIRONMENT
 ↓
ARTIFACT
 ↓
PACKAGE
 ↓
REGISTRY
 ↓
CONSUMER
```

Each transition should be:

```text
auditable.
```

---

# 146. Publish Credentials

Publishing requires:

```text
high-value credentials.
```

Use:

```text
short-lived credentials
least privilege
protected CI environment.
```

Avoid:

```text
long-lived publish tokens
in source repositories.
```

---

# 147. Release Branch Protection

Publishing should require:

```text
green CI
artifact validation
approved version
release provenance.
```

---

# 148. Two-Phase Release

Conceptual:

```text
build artifact
→ validate
→ sign/provenance
→ publish.
```

Do not:

```text
publish first
→ discover artifact failure.
```

---

# 149. Canary Release

Use:

```text
prerelease
```

or:

```text
restricted dist-tag
```

to validate:

```text
real consumer ecosystem
```

before:

```text
stable tag.
```

---

# 150. Prerelease Builds

Use:

```text
1.3.0-beta.1
```

for:

```text
ecosystem validation.
```

---

# 151. Dist Tags

Dist tags can direct:

```text
latest
next
beta
```

to:

```text
different releases.
```

This supports:

```text
controlled rollout.
```

---

# 152. Rollback

If a release is bad:

```text
publish fixed version
```

rather than assuming:

```text
registry history can be erased safely.
```

Understand:

```text
package-manager caching
```

and:

```text
consumer lockfiles.
```

---

# 153. Deprecation

A broken API can be:

```text
deprecated
```

while:

```text
replacement
```

is introduced.

Deprecation needs:

```text
documentation
migration path
timeline.
```

---

# 154. Package Compatibility Contract

Document:

```text
Node versions
browser support
module formats
TypeScript support
native platforms
ES syntax level
```

---

# 155. Compatibility Layers

A package can ship:

```text
portable core
+
specialized adapters.
```

This often reduces:

```text
condition complexity.
```

---

# 156. Build Strategy Comparison

| Strategy | Portability | Size | Debugging | Complexity | Tree-shaking |
|---|---|---|---|---|---|
| Source-like ESM | high | low | high | low | strong |
| ESM + CJS | high | medium | medium | medium | good |
| Single bundle | medium | often larger | lower | medium | consumer-dependent |
| Multi-target bundles | high | larger | lower | high | variable |
| Native + fallback | broad | high | complex | high | variable |

Treat this as:

```text
decision aid, not universal ranking.
```

---

# 157. Library Build Principle

Libraries should usually preserve:

```text
consumer choice
```

where feasible.

Applications usually optimize:

```text
end-user delivery.
```

---

# 158. Build Tool Selection

Evaluate:

```text
ESM fidelity
CJS output
types
source maps
dynamic imports
tree shaking
code splitting
plugin ecosystem
Node targets
browser targets
build speed
```

---

# 159. Tool Lock-In

A build tool becomes:

```text
critical infrastructure
```

when:

```text
package format
source maps
plugin semantics
```

depend heavily on it.

Pin:

```text
tool version
configuration.
```

---

# 160. Build Config as Code

Review:

```text
build.config.*
```

like:

```text
production application code.
```

It controls:

```text
runtime behavior.
```

---

# 161. Build Configuration Drift

Potential drift:

```text
local config
CI config
release config
```

select:

```text
different targets.
```

Prefer:

```text
single canonical configuration
```

with:

```text
explicit environment overrides.
```

---

# 162. Environment-Specific Build

If builds differ by:

```text
development
production
browser
node
```

record:

```text
condition/configuration
```

in:

```text
artifact metadata.
```

---

# 163. Development vs Production Artifact

Development may contain:

```text
debug checks
verbose diagnostics
source maps
unminified code.
```

Production may:

```text
minify
remove diagnostics
optimize
```

But library consumers should receive:

```text
stable production semantics.
```

---

# 164. Production Build Reproducibility

Avoid:

```text
machine-specific optimization
```

that changes:

```text
semantics.
```

Optimization must preserve:

```text
contract.
```

---

# 165. Build Regression Tests

Test:

```text
source behavior
```

and:

```text
built artifact behavior.
```

A compiler/bundler regression can create:

```text
artifact-only bug.
```

---

# 166. Artifact-Level Tests

Run:

```text
node dist/index.js
```

not only:

```text
test source.
```

---

# 167. ESM Artifact Test

Verify:

```text
module syntax
import resolution
export names
dynamic import
TLA policy.
```

---

# 168. CJS Artifact Test

Verify:

```text
require
module.exports
default semantics
named API
dependency loading.
```

---

# 169. Type Artifact Test

Verify:

```text
declaration load
public types
module paths
editor resolution.
```

---

# 170. Source Map Test

Generate an error:

```text
throw new Error("test");
```

from transformed code.

Verify:

```text
stack points to intended source.
```

---

# 171. Source Map Failure

If stack points to:

```text
dist/index.js:1
```

instead of:

```text
src/service.ts:42
```

check:

```text
source map file
sourceRoot
sources
map path
deployment.
```

---

# 172. Artifact Size Budget

Track:

```text
root JS
CJS
ESM
types
maps
native binaries.
```

Set:

```text
budget
```

for critical packages.

---

# 173. Bundle Size Regression

A dependency can accidentally change:

```text
tree-shaking
```

and increase:

```text
bundle.
```

Run:

```text
size regression CI.
```

---

# 174. Startup Cost

Large library roots can increase:

```text
module evaluation
parse
compile
memory.
```

Subpath APIs can:

```text
reduce loaded graph.
```

depending on consumer/runtime.

---

# 175. Package Size vs Runtime Cost

Smaller package:

```text
less download/install cost.
```

but aggressive bundling can:

```text
increase duplicated runtime dependencies.
```

Optimize:

```text
total lifecycle cost.
```

---

# 176. Memory Duplication

If a library bundles its own copy of a dependency:

```text
consumer copy
+
library copy
```

can create:

```text
duplicate state
```

and:

```text
higher memory.
```

---

# 177. Singleton Dependency

Examples:

```text
React-like runtime
logging registry
event bus
validation type registry
metrics SDK.
```

Bundling duplicates can break:

```text
identity semantics.
```

---

# 178. Dependency Version Strategy

For critical shared dependencies:

```text
peerDependency
or
external dependency
```

may be preferable to:

```text
bundled copy.
```

---

# 179. Package Security Scanning

Scan:

```text
source
dependencies
tarball
native binaries
generated artifacts.
```

---

# 180. Tarball Security Review

Check for:

```text
.env
private keys
tokens
internal source
debug dumps
credentials
test data.
```

---

# 181. Ignore Rules

Review:

```text
.npmignore
files
.gitignore
publish configuration.
```

Do not assume:

```text
Git ignore
=
package publish ignore
```

in every situation.

---

# 182. Generated Asset Leak

Build might generate:

```text
coverage
snapshots
test-output
```

inside:

```text
dist.
```

Then publishing can unintentionally include them.

---

# 183. Artifact Directory Hygiene

Separate:

```text
dist
coverage
tmp
test-results
```

to simplify:

```text
package inclusion.
```

---

# 184. Clean Build

Always validate with:

```bash
rm -rf dist
```

before:

```text
release build.
```

A dirty `dist` can hide:

```text
missing generated files.
```

---

# 185. Clean Install

Test from:

```text
clean node_modules.
```

to detect:

```text
phantom dependencies.
```

---

# 186. Clean Consumer

Best:

```text
temporary directory
+
fresh package.json
+
tarball install.
```

---

# 187. Build Artifact Manifest

Generate:

```text
files
sizes
hashes
formats
entrypoints.
```

This becomes:

```text
release evidence.
```

---

# 188. Artifact Manifest Example

```text
dist/index.js      ESM   14 KB
dist/index.cjs     CJS   15 KB
dist/index.d.ts    types  5 KB
dist/index.js.map  map   40 KB
```

---

# 189. Artifact Diff

Compare release:

```text
v1
```

against:

```text
v2.
```

Detect:

```text
unexpected file removal
size growth
format change
hash changes.
```

---

# 190. Generated File Ownership

Every generated artifact should have:

```text
source generator
build command
owner
cleanup policy.
```

---

# 191. Build Reproducibility Test

Run:

```text
build A
build B
```

from equivalent clean environments.

Compare:

```text
artifact hashes
```

or:

```text
semantic manifest.
```

---

# 192. Semantic Reproducibility

When byte equality is too strict, compare:

```text
export surface
runtime behavior
file set
normalized source maps
```

---

# 193. Reproducibility Failure

If:

```text
hash A ≠ hash B
```

debug:

```text
timestamps
ordering
paths
randomness
tool versions
generated IDs.
```

---

# 194. Release Evidence Bundle

Keep:

```text
source commit
lockfile digest
Node version
build tool versions
artifact manifest
tarball digest
test results
consumer fixture results.
```

---

# 195. Rollback Evidence

A rollback needs:

```text
previous known-good artifact
digest
version
compatibility data.
```

---

# 196. Release Candidate

Before publish:

```text
build clean
pack
install
test
scan
diff
approve.
```

---

# 197. Release Pipeline

```text
COMMIT
 ↓
CI
 ↓
BUILD
 ↓
ARTIFACT VALIDATION
 ↓
TARBALL
 ↓
CONSUMER TESTS
 ↓
SECURITY SCAN
 ↓
API DIFF
 ↓
RELEASE APPROVAL
 ↓
PUBLISH
 ↓
POST-PUBLISH SMOKE
```

---

# 198. Post-Publish Smoke

After publish:

```text
fresh consumer
```

installs:

```text
published version
```

and verifies:

```text
root
subpaths
ESM
CJS
types
CLI.
```

---

# 199. Registry Propagation

A publish can be:

```text
accepted
```

before every consumer environment immediately sees:

```text
same metadata.
```

Keep release checks:

```text
bounded and retry-aware.
```

---

# 200. Post-Publish Diagnosis

If consumer fails after publish:

```text
local build passed
tarball passed?
registry metadata?
registry tarball?
consumer cache?
```

---

# 201. Package Manager Behavior

npm-compatible ecosystems can differ in:

```text
workspace handling
scripts
resolution
install
publish.
```

Test with:

```text
package managers
```

you explicitly support.

---

# 202. Workspace Build

A monorepo might build:

```text
package A
→
package B
→
package C.
```

But the published tarball should not depend on:

```text
workspace-only path.
```

---

# 203. Workspace Protocols

Development package managers can use:

```text
workspace relationships.
```

Ensure published metadata contains:

```text
publishable dependency ranges
```

rather than:

```text
uninstallable workspace references.
```

---

# 204. Build Graph vs Dependency Graph

Build graph:

```text
what must build first.
```

Dependency graph:

```text
what must exist at runtime/install time.
```

They are related:

```text
not identical.
```

---

# 205. Generated Code Dependency

A generated artifact can depend on:

```text
build-time package
```

but emitted output may or may not require it at runtime.

Verify:

```text
runtime dependency closure.
```

---

# 206. Runtime Dependency Closure

After packaging ask:

```text
Can a clean consumer execute the artifact
using only declared runtime dependencies?
```

---

# 207. Phantom Dependency Detection

Run:

```text
package in isolated consumer
```

and intentionally remove:

```text
workspace root dependencies.
```

This catches:

```text
undeclared runtime imports.
```

---

# 208. Package Install Scripts

Install scripts can:

```text
compile native modules
download binaries
generate files.
```

They add:

```text
security
reproducibility
platform
CI complexity.
```

---

# 209. Install Script Principle

Minimize:

```text
network downloads
arbitrary execution
environment assumptions.
```

Use:

```text
prebuilt verified artifacts
```

when appropriate.

---

# 210. Native Binary Verification

A downloaded binary should be checked for:

```text
expected version
integrity
signature/provenance
platform
architecture.
```

---

# 211. Build Artifact Trust

Generated output should come from:

```text
known toolchain
known source
known dependencies.
```

---

# 212. Supply-Chain Boundary

Publishing credentials are:

```text
more privileged
```

than ordinary CI credentials.

Separate:

```text
test CI
build CI
release CI.
```

---

# 213. Release Environment

The release environment should be:

```text
minimal
pinned
auditable
ephemeral where practical.
```

---

# 214. Build Container

A container can standardize:

```text
OS
Node
toolchain
environment.
```

It does not automatically guarantee:

```text
byte-identical output.
```

---

# 215. Artifact Signing

Where your organization supports signing:

```text
sign artifact
verify signature
publish verification metadata.
```

Signing complements:

```text
hash/integrity.
```

---

# 216. Integrity vs Authenticity

Hash/integrity says:

```text
artifact was not modified relative to expected digest.
```

Authenticity says:

```text
trusted party produced/approved it.
```

They are:

```text
different properties.
```

---

# 217. Compatibility Test Grid

For a dual package:

```text
Node min
Node current
Node latest
ESM
CJS
TypeScript
bundler
tarball
```

---

# 218. Artifact Regression Matrix

Track:

```text
version
format
entrypoint
target
size
hash
consumer result.
```

---

# 219. Build Artifact Bug Example

Source:

```js
export function add(a, b) {
  return a + b;
}
```

CJS artifact accidentally exports:

```js
module.exports = {
  default: add
};
```

Consumer expected:

```js
require("pkg").add
```

Result:

```text
API mismatch.
```

The source is correct:

```text
artifact contract is wrong.
```

---

# 220. Build Artifact Bug Example — Wrong Extension

Artifact:

```text
dist/index.js
```

contains:

```text
CommonJS
```

while package:

```json
{ "type": "module" }
```

causes Node to interpret:

```text
index.js as ESM.
```

Result:

```text
format failure.
```

Fix with:

```text
correct extension
or
correct package type.
```

---

# 221. Build Artifact Bug Example — Missing File

Exports:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

Tarball omits:

```text
dist/index.js.
```

Consumer:

```text
fails.
```

---

# 222. Build Artifact Bug Example — Wrong Condition

Exports:

```json
{
  "import": "./dist/browser.js",
  "node": "./dist/node.js",
  "default": "./dist/default.js"
}
```

If ordering is inappropriate:

```text
Node can receive wrong branch.
```

Test actual:

```text
condition order.
```

See:

```text
Chapter 147.
```

---

# 223. Build Artifact Bug Example — Missing Dynamic Chunk

Code:

```js
await import("./feature.js");
```

Bundler creates:

```text
feature-ABC.js
```

but package omits:

```text
feature-ABC.js.
```

Runtime fails:

```text
only after dynamic import.
```

---

# 224. Build Artifact Bug Example — Source Map Path

Generated file references:

```text
../../src/index.ts
```

but package structure differs.

Debugger:

```text
cannot map source.
```

---

# 225. Build Artifact Bug Example — Declaration Drift

Runtime removes:

```text
parse()
```

but:

```text
index.d.ts
```

still exports it.

TypeScript users see:

```text
false API.
```

---

# 226. Build Artifact Bug Example — Native Binary

Package selects:

```text
linux-x64
```

but consumer is:

```text
linux-arm64.
```

No:

```text
fallback.
```

Install fails.

---

# 227. Build Artifact Bug Example — Clean Consumer

Monorepo test:

```text
passes.
```

Clean tarball consumer:

```text
Cannot find dependency.
```

Root cause:

```text
phantom dependency.
```

---

# 228. Build Artifact Bug Example — Platform Branch

Node build uses:

```text
node:fs
```

Browser build accidentally includes:

```text
fs.
```

Bundler:

```text
fails/polyfills unexpectedly.
```

---

# 229. Build Artifact Testing Principle

Every release should be tested:

```text
from the consumer side.
```

---

# 230. Implementation From Scratch

Build:

```text
source-to-dist pipeline
ESM emitter
CJS emitter
declaration emitter
source-map generator
package manifest
export validator
tarball inspector
consumer fixtures
artifact manifest
API diff
reproducibility checker.
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

# 231. Milestone 1 — Source to ESM

Create:

```text
src/index.js
```

and emit:

```text
dist/index.js.
```

Add:

```text
source map.
```

---

# 232. Milestone 2 — Source to CJS

Emit:

```text
dist/index.cjs.
```

Verify:

```text
require().
```

---

# 233. Milestone 3 — Package Map

Add:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

---

# 234. Milestone 4 — Types

Emit:

```text
dist/index.d.ts.
```

Add:

```text
types metadata.
```

---

# 235. Milestone 5 — Tarball

Run:

```bash
npm pack
```

Then:

```text
install tarball
```

in:

```text
clean consumer.
```

---

# 236. Milestone 6 — Consumer Matrix

Create:

```text
ESM
CJS
TS
bundler
```

consumer fixtures.

---

# 237. Milestone 7 — Artifact Manifest

Generate:

```text
file
size
hash
format.
```

---

# 238. Milestone 8 — API Diff

Compare:

```text
exports map
```

between:

```text
release N
release N+1.
```

---

# 239. Milestone 9 — Reproducible Build

Run build twice:

```text
clean A
clean B
```

and compare:

```text
normalized artifacts.
```

---

# 240. Milestone 10 — Native Optional Branch

Add:

```text
native
portable
```

artifacts.

Test:

```text
native available
native unavailable.
```

---

# 241. Milestone 11 — Dynamic Import

Add:

```text
feature chunk.
```

Verify:

```text
published artifact includes it.
```

---

# 242. Milestone 12 — CLI

Expose:

```text
bin
```

and test:

```text
installed command.
```

---

# 243. Milestone 13 — Security Scan

Scan tarball for:

```text
secrets
large unexpected files
private paths
debug artifacts.
```

---

# 244. Milestone 14 — Release Evidence

Produce:

```text
commit
Node
lockfile digest
manifest
tarball hash
consumer results.
```

---

# 245. Debugging Exercises

## Exercise A — ESM/CJS Mismatch

Make:

```text
CJS file
```

under:

```text
type=module.
```

Observe:

```text
failure.
```

Fix:

```text
.cjs.
```

---

## Exercise B — Missing Tarball File

Remove:

```text
dist/index.js
```

from package contents.

Run:

```text
consumer.
```

---

## Exercise C — Deep Import Break

Add:

```text
exports
```

without:

```text
legacy subpath.
```

Find:

```text
consumer failure.
```

---

## Exercise D — Condition Order

Put:

```text
default
```

before:

```text
node.
```

Observe:

```text
wrong branch.
```

---

## Exercise E — Type Drift

Remove:

```text
runtime export
```

but retain:

```text
declaration.
```

Run:

```text
TypeScript consumer.
```

---

## Exercise F — Source Map Drift

Change:

```text
sourceRoot.
```

and inspect:

```text
stack trace.
```

---

## Exercise G — Phantom Dependency

Remove:

```text
root node_modules.
```

and install only:

```text
packed package.
```

Find:

```text
undeclared runtime dependency.
```

---

## Exercise H — Dirty Build

Leave stale:

```text
dist/old.js.
```

Run:

```text
package build.
```

Observe:

```text
unexpected published file.
```

---

## Exercise I — Dynamic Chunk Missing

Delete:

```text
chunk
```

from tarball.

Trigger:

```text
dynamic import.
```

---

## Exercise J — Reproducibility

Build twice under:

```text
different workspace paths.
```

Compare:

```text
hashes
source maps.
```

---

# 246. Code Review Exercise — One Giant Bundle

Review:

```text
library
→ bundles all dependencies
→ publishes one 2 MB file.
```

Ask:

```text
tree shaking?
duplicate dependencies?
singleton identity?
debugging?
package size?
```

---

# 247. Code Review Exercise — No Tarball Test

CI:

```text
npm test
```

but:

```text
never npm pack.
```

Find:

```text
release artifact blind spot.
```

---

# 248. Code Review Exercise — Build in Publish

Publishing executes:

```text
arbitrary build chain
```

without:

```text
clean environment
lockfile
artifact validation.
```

Find:

```text
release reproducibility risk.
```

---

# 249. Code Review Exercise — `sideEffects: false`

Package contains:

```text
module that registers global behavior.
```

Metadata:

```json
{
  "sideEffects": false
}
```

Find:

```text
possible tree-shaking semantic break.
```

---

# 250. Code Review Exercise — Dual Logic

ESM build and CJS build contain:

```text
separate business logic implementations.
```

Find:

```text
semantic drift.
```

---

# 251. Code Review Exercise — Published Secrets

Tarball contains:

```text
.env.production
```

Find:

```text
critical release security failure.
```

---

# 252. Predict-the-Behavior Exercises

### Exercise 1

Package:

```json
{
  "type": "module"
}
```

File:

```text
index.js
```

contains:

```js
module.exports = 1;
```

Predict:

```text
which module system Node interprets.
```

ESM.

---

### Exercise 2

Same package.

File:

```text
index.cjs
```

contains:

```js
module.exports = 1;
```

Predict:

```text
module system.
```

CommonJS.

---

### Exercise 3

Package:

```text
main
exports
```

Predict which modern Node package entry is preferred.

```text
exports.
```

citeturn378470search0

---

### Exercise 4

CJS consumer:

```js
require("pkg");
```

Package exposes:

```text
require
```

condition.

Predict:

```text
which branch.
```

`require`.

---

### Exercise 5

ESM consumer:

```js
import "pkg";
```

Package exposes:

```text
import
```

condition.

Predict:

```text
which branch.
```

`import`.

---

### Exercise 6

A package is under:

```text
type=module
```

but exposes:

```text
./dist/index.cjs
```

through:

```text
require.
```

Predict:

```text
why explicit CJS extension is useful.
```

---

### Exercise 7

A dynamic import chunk exists in repository but is omitted from tarball.

Predict:

```text
source tests pass?
consumer runtime?
```

---

### Exercise 8

A source map contains absolute CI paths.

Predict:

```text
what may differ between build machines.
```

---

### Exercise 9

A package declares:

```text
Node >=18
```

but artifact uses:

```text
runtime feature only available in Node 22.
```

Predict:

```text
contract correctness.
```

---

### Exercise 10

A library bundles its own copy of a singleton dependency.

Predict:

```text
what identity problems can occur.
```

---

# 253. Interview Questions

### Fundamentals

```text
1. What is a build artifact?
2. What is the difference between transpiling and bundling?
3. What is packaging?
4. What is publishing?
5. Why test the packed tarball?
```

### ESM/CJS

```text
6. How do you build one source into ESM and CJS?
7. What is the dual package hazard?
8. Why use .cjs?
9. What does package type do?
10. What is module-sync?
11. Why does top-level await matter for require?
```

### Source Maps / Types

```text
12. Why publish source maps?
13. What security risk do source maps create?
14. What are .d.ts files?
15. How can runtime and type APIs drift?
```

### Dependencies

```text
16. When should a library externalize a dependency?
17. What is a peer dependency?
18. What is a phantom dependency?
19. What are optional native dependencies?
20. How do you avoid duplicate singleton state?
```

### Distribution

```text
21. How do you design a browser and Node package?
22. How do you test conditional branches?
23. How do you publish native binaries safely?
24. How do you make builds reproducible?
25. How do you validate package contents?
```

### Principal

```text
26. Design a multi-runtime package distribution system.
27. How would you support ESM, CJS, browser, and native Node?
28. How would you minimize condition matrix explosion?
29. How would you guarantee runtime/type surface parity?
30. How would you detect accidental package-file leaks?
31. How would you build a tarball consumer test platform?
32. How would you make a monorepo release reproducible?
33. How would you design release provenance?
34. How would you safely roll out a breaking artifact change?
35. How would you debug an artifact-only production failure?
```

---

# 254. Mastery Exercises

### Exercise 1 — Dual Package

Build:

```text
ESM
CJS
types
```

from:

```text
one source.
```

### Exercise 2 — Tarball Gate

Automate:

```text
build
pack
install
consumer tests.
```

### Exercise 3 — Conditional Distribution

Build:

```text
node
browser/default
native
portable
```

branches.

### Exercise 4 — Artifact Validator

Validate:

```text
exports
files
types
bin
maps.
```

### Exercise 5 — Reproducibility Checker

Build twice and compare:

```text
hash
file list
normalized map metadata.
```

### Exercise 6 — API Diff

Detect:

```text
removed exports
changed conditions
format changes.
```

### Exercise 7 — Dependency Closure

Install package in:

```text
empty consumer project
```

and prove:

```text
no undeclared runtime dependency.
```

### Exercise 8 — Native Matrix

Create:

```text
platform/architecture matrix
```

and verify:

```text
fallback.
```

### Exercise 9 — Source Map Audit

Verify:

```text
runtime stack
→
source file
```

without:

```text
private source leakage.
```

### Exercise 10 — Release Platform

Design:

```text
build
artifact store
validation
provenance
publish
post-publish smoke
rollback.
```

---

# 255. Track A — Core Theory

Master:

```text
artifact
transpile
bundle
package
publish
ESM
CJS
dual package
exports
imports
type
extensions
source maps
declarations
tree shaking
side effects
dependencies
native binaries
tarballs
reproducibility
provenance
release pipelines.
```

Deliverable:

```text
Explain how source code becomes the exact module graph
consumed by a user.
```

---

# 256. Track B — Implementation

Build:

```text
ESM output
CJS output
types
source maps
exports map
artifact manifest
tarball verifier
consumer fixtures
API diff
reproducibility checker
release pipeline.
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

# 257. Track C — Interview / Reasoning

Practice:

```text
“Why is source code correctness insufficient?”

“Why can a package work in the monorepo but fail after npm install?”

“What is the dual package hazard?”

“Why are source maps part of release engineering?”

“How do you keep types aligned with runtime exports?”

“When should a dependency be externalized?”

“How do you make a release reproducible?”

“How do you test a package after packing?”

“How would you safely support a native optimization?”
```

Answer with:

```text
source
→ build
→ package
→ resolver
→ loader
→ runtime
→ consumer contract
```

---

# 258. Principal Decision Framework

For every build/distribution architecture ask:

```text
1. Who consumes the artifact?
2. Which runtimes are supported?
3. Which module formats are supported?
4. Is ESM-only sufficient?
5. Is CJS support required?
6. If dual, can state be duplicated?
7. What is the canonical implementation?
8. Which transformations are actually required?
9. What is the syntax target?
10. What is the runtime API target?
11. Which dependencies stay external?
12. Which dependencies are peers?
13. Which dependencies are optional?
14. Are native artifacts required?
15. What is the native compatibility matrix?
16. Is there a portable fallback?
17. Does conditional branching justify its complexity?
18. Does every branch have tests?
19. Are source maps required?
20. Do source maps expose sensitive source?
21. Are type declarations aligned?
22. Are dynamic chunks packaged?
23. Are runtime assets packaged?
24. Is CLI metadata correct?
25. Is the package file set intentional?
26. Does a clean tarball consumer pass?
27. Are build dependencies separated from runtime dependencies?
28. Is the build reproducible?
29. Are artifact hashes recorded?
30. Is provenance recorded?
31. Is publishing credential isolation adequate?
32. Is API diff automated?
33. Are minimum runtime claims verified?
34. Are package-manager compatibility assumptions documented?
35. What happens when a release is bad?
```

---

# 259. Production Artifact Checklist

```text
[ ] clean build
[ ] pinned Node
[ ] pinned package manager
[ ] lockfile
[ ] build tool versions
[ ] ESM output validated
[ ] CJS output validated
[ ] types validated
[ ] source maps validated
[ ] export map validated
[ ] all targets exist
[ ] all runtime assets exist
[ ] CLI validated
[ ] package contents inspected
[ ] tarball installed cleanly
[ ] ESM consumer passed
[ ] CJS consumer passed
[ ] TS consumer passed
[ ] bundler consumer passed
[ ] browser consumer passed where supported
[ ] native matrix passed where supported
[ ] no phantom dependencies
[ ] no secret files
[ ] API diff passed
[ ] size budget passed
[ ] integrity manifest produced
[ ] provenance recorded
[ ] post-publish smoke planned.
```

---

# 260. ESM/CJS Checklist

```text
[ ] canonical implementation
[ ] ESM entry
[ ] CJS entry
[ ] explicit extensions
[ ] package type reviewed
[ ] import consumer tested
[ ] require consumer tested
[ ] dynamic import tested
[ ] TLA compatibility reviewed
[ ] state duplication reviewed
[ ] class identity reviewed
[ ] Symbol identity reviewed
[ ] error identity reviewed
[ ] singleton dependencies reviewed
```

---

# 261. Tarball Verification Checklist

```text
[ ] npm pack
[ ] inspect package.json
[ ] inspect file list
[ ] inspect dist
[ ] inspect types
[ ] inspect bin
[ ] inspect maps
[ ] inspect native files
[ ] secret scan
[ ] clean install
[ ] root import
[ ] root require
[ ] subpaths
[ ] blocked deep imports
[ ] dynamic import
[ ] CLI
[ ] TypeScript
```

---

# 262. Reproducible Release Checklist

```text
[ ] source commit fixed
[ ] lockfile fixed
[ ] Node fixed
[ ] toolchain fixed
[ ] environment fixed
[ ] timestamps controlled where required
[ ] file ordering controlled
[ ] paths normalized
[ ] generated IDs controlled
[ ] artifact hashes recorded
[ ] manifest recorded
[ ] tarball digest recorded
[ ] release evidence stored.
```

---

# 263. Security Checklist

```text
[ ] least-privilege publishing
[ ] short-lived credentials
[ ] protected release environment
[ ] secret scanning
[ ] provenance
[ ] integrity
[ ] native binary verification
[ ] dependency scanning
[ ] tarball scanning
[ ] source-map review
[ ] no private test data
[ ] no debug dumps
[ ] no CI credentials in artifact.
```

---

# 264. Current Node / Ecosystem Notes

As of September 11, 2026, the official Node.js documentation baseline is v26.8.2.

Relevant current Node behavior includes:

```text
ESM implementation is stable.

Exports and imports are established package mechanisms.

Conditional exports are established.

node-addons is a core package condition.

module-sync is available for synchronous ESM graphs.

require(ESM) supports synchronous ESM only; top-level await
makes the graph unsuitable for synchronous require().

CommonJS imported from ESM receives a default export representing
module.exports, with heuristic named-export detection for compatibility.

Current Node also supports modern CommonJS/ESM interoperability
markers and behavior described by the v26 module documentation.
```

citeturn378470search0turn378470search1turn378470search2

Node's package documentation recommends `exports` for new packages targeting currently supported Node versions, and states that `exports` takes precedence over `main` when both are present. citeturn378470search0

---

# 265. Specification / Runtime Source Discipline

Primary sources:

```text
Node.js Packages documentation
Node.js ESM documentation
Node.js CommonJS documentation
Node.js module API documentation
npm/package-manager documentation
TypeScript module-resolution documentation
bundler documentation
ECMAScript specification
```

Distinguish:

```text
language semantics
runtime loader behavior
package metadata behavior
bundler behavior
type-checker behavior
package-manager behavior
release-process policy.
```

Never describe:

```text
a bundler transform
```

as:

```text
ECMAScript semantics.
```

---

# 266. Performance Considerations

Build/distribution optimization should consider:

```text
package size
startup time
parse time
compile time
runtime execution
memory
download/install cost
CI build time.
```

Do not optimize:

```text
bundle size
```

by:

```text
breaking consumer tree shaking
```

or:

```text
duplicating dependencies.
```

---

# 267. Memory Considerations

Watch for:

```text
duplicate dependencies
multiple module graphs
native binary copies
large source maps
large generated artifacts.
```

Dual packages can increase:

```text
memory
```

when both formats are loaded into:

```text
one process.
```

---

# 268. Common Misconceptions

### Misconception 1

```text
“Build output is just source code with different syntax.”
```

Reality:

```text
build output can change module graph, dependencies, paths,
assets, formats, source maps, and runtime behavior.
```

### Misconception 2

```text
“Transpilation provides old runtime APIs.”
```

Reality:

```text
syntax transformation and runtime polyfills are different.
```

### Misconception 3

```text
“If source tests pass, published package works.”
```

Reality:

```text
the tarball can differ materially from the repository.
```

### Misconception 4

```text
“ESM and CJS builds are automatically equivalent.”
```

Reality:

```text
format conversion can change exports, identity, timing, and state.
```

### Misconception 5

```text
“Source maps are harmless.”
```

Reality:

```text
they can expose internal source and paths.
```

### Misconception 6

```text
“Bundling all dependencies is always faster.”
```

Reality:

```text
it can increase size, duplication, and identity problems.
```

---

# 269. Common Mistakes

```text
[ ] no clean build
[ ] stale dist
[ ] source-only tests
[ ] no tarball tests
[ ] wrong .js/.cjs/.mjs
[ ] missing export target
[ ] missing dynamic chunk
[ ] missing runtime asset
[ ] type/runtime drift
[ ] source-map drift
[ ] broad dependency bundling
[ ] phantom dependency
[ ] missing native fallback
[ ] build target mismatch
[ ] Node engine mismatch
[ ] condition branch untested
[ ] release credentials in CI broadly
[ ] no artifact manifest
[ ] no API diff
[ ] no post-publish smoke.
```

---

# 270. Final Build Artifact Mental Model

```text
SOURCE
 ↓
TRANSFORM
 ↓
MODULE GRAPH
 ↓
RUNTIME TARGET
 ↓
ARTIFACTS
 ├─ ESM
 ├─ CJS
 ├─ TYPES
 ├─ MAPS
 ├─ ASSETS
 └─ NATIVE/WASM
 ↓
PACKAGE METADATA
 ↓
TARBALL
 ↓
REGISTRY
 ↓
CONSUMER RESOLUTION
 ↓
LOADER
 ↓
RUNTIME
```

---

# 271. Distribution Mental Model

```text
PUBLIC SPECIFIER
      ↓
EXPORT MAP
      ↓
TARGET ARTIFACT
      ↓
MODULE FORMAT
      ↓
DEPENDENCY GRAPH
      ↓
RUNTIME
```

Every layer must agree.

---

# 272. Release Mental Model

```text
SOURCE COMMIT
 ↓
CLEAN BUILD
 ↓
ARTIFACT MANIFEST
 ↓
PACKAGE
 ↓
CONSUMER TEST
 ↓
SECURITY SCAN
 ↓
API DIFF
 ↓
PROVENANCE
 ↓
PUBLISH
 ↓
POST-PUBLISH VERIFY
```

---

# 273. Artifact Trust Model

```text
correct source
+
correct build
+
correct metadata
+
correct package contents
+
correct resolution
+
correct loader
=
credible release.
```

---

# 274. Dependency Graph

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
Native Addons / N-API / FFI
        ↓
Chapter 147
Package exports / conditional exports
        ↓
Chapter 148
Build Artifacts / ESM-CJS Packaging / Distribution
```

Cross-cutting:

```text
Node
bundlers
TypeScript
npm
monorepos
CI
security
semver
source maps
native binaries
release engineering.
```

---

# 275. Concept Connections

## Depends On

```text
modules
ESM
CommonJS
package resolution
exports
imports
Node runtime
build systems
CI.
```

## Builds Toward

```text
release engineering
package platform engineering
ecosystem compatibility
runtime portability
observability of artifacts
supply-chain security
distribution strategy.
```

## Related Concepts

```text
transpiler
bundler
package
tarball
exports
source map
declaration
dual package
native addon
provenance
artifact manifest.
```

## Concepts Revisited

```text
module identity
conditional resolution
runtime compatibility
testing
reproducibility
security
performance
monorepos.
```

## Why This Chapter Matters

The code in:

```text
src/
```

is not what your users execute.

They execute:

```text
a published artifact
```

selected by:

```text
package metadata
+
resolver
+
loader
+
runtime.
```

Therefore:

```text
build correctness
=
product correctness
```

at the distribution boundary.

---

# 276. Revision / Retrieval Record

```md
# Chapter 148 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Build Artifacts
-

## Transpilation
-

## Bundling
-

## ESM
-

## CommonJS
-

## Dual Packaging
-

## Exports
-

## Imports
-

## package type
-

## Extensions
-

## Source Maps
-

## Types
-

## Runtime Assets
-

## Dependencies
-

## Native Artifacts
-

## Tarballs
-

## Consumer Fixtures
-

## Reproducibility
-

## Provenance
-

## API Diff
-

## Release
-

## Security
-

## Performance
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

# 277. Spaced Retrieval Schedule

### Day 0

Explain:

```text
source
→ build
→ package
→ tarball
→ resolver
→ runtime.
```

### Day 1

Draw:

```text
ESM
CJS
types
maps
```

and explain how one source can produce each.

### Day 3

Build:

```text
dual package
```

from:

```text
one implementation.
```

### Day 7

Implement:

```text
tarball verification
+
consumer matrix.
```

### Day 14

Build:

```text
artifact manifest
+
API diff
+
reproducibility check.
```

### Day 21

Design:

```text
native + portable distribution.
```

### Day 30

Design:

```text
enterprise release platform
```

without notes.

---

# 278. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
produce basic ESM/CJS artifacts.
```

Mark:

```text
[?] Needs Revision
```

when you:

```text
test source but not package
forget tarball contents
break types
break exports
mis-handle file extensions
cannot explain dual-package identity.
```

Mark:

```text
[+] Completed
```

when you can:

```text
build
package
validate
publish
and consumer-test
a multi-format JavaScript package.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
design a production distribution platform across Node,
browser, TypeScript, bundlers, native artifacts, source maps,
conditional exports, reproducible builds, provenance, and
rollback while keeping runtime and public API behavior coherent.
```

Reading alone does not mark mastery.

---

# 279. Final Principal Principle

> **The published artifact is the product. Source code is only the starting point. A production-quality JavaScript package requires agreement between source semantics, build output, module format, package metadata, dependency closure, published files, resolver behavior, loader behavior, runtime capabilities, and release controls.**

The principal workflow is:

```text
DESIGN PUBLIC API
→
CHOOSE RUNTIME TARGETS
→
CHOOSE MODULE FORMATS
→
BUILD
→
GENERATE TYPES/MAPS
→
VALIDATE EXPORTS
→
VALIDATE ARTIFACTS
→
PACK
→
INSTALL CLEANLY
→
TEST REAL CONSUMERS
→
SCAN SECURITY
→
COMPARE API
→
RECORD PROVENANCE
→
PUBLISH
→
POST-PUBLISH VERIFY
```

Remember:

```text
source correctness ≠ artifact correctness

transpile ≠ polyfill

bundle ≠ package

package ≠ publish

repository ≠ tarball

ESM ≠ CJS

dual package ≠ shared state automatically

exports ≠ file existence

types ≠ runtime API

source map ≠ harmless metadata

clean build ≠ source test pass

monorepo success ≠ consumer success

dependency installed in workspace ≠ declared runtime dependency

native binary availability ≠ native support

hash integrity ≠ authenticity

reproducible source ≠ reproducible artifact

publish success ≠ consumer success.
```

The principal question is:

```text
“What exact artifact will the consumer install, which specifier will
their resolver select, which module format will their loader execute,
which dependencies and assets will actually exist at runtime, and how
can we prove that the published package—not merely the source tree—
matches the API, compatibility, security, and reliability contract?”
```

That is JavaScript build artifact and distribution engineering.