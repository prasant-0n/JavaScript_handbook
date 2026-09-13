\
# Chapter 69 — Bundlers and Build Systems

> **Curriculum position:** Part XII — Modules / Tooling  
> **Previous chapter:** Chapter 68 — Transpilation and Compilation  
> **Next chapter:** Chapter 70 — Source Maps and Production Debugging  
> **Primary environment:** Modern JavaScript applications, libraries, Node.js, browsers, and current build tooling.

---

# Chapter Mission

Master bundlers and build systems as **dependency-graph construction, optimization, packaging, and artifact-generation systems**.

A bundler is not merely:

```bash
npm run build
```

and it is not merely:

```text
combine JavaScript files
```

A modern bundler sits between source code and deployable artifacts:

```text
source
  ↓
module resolution
  ↓
dependency graph
  ↓
parsing
  ↓
transformation
  ↓
optimization
  ↓
chunking
  ↓
asset graph
  ↓
code generation
  ↓
minification
  ↓
source maps
  ↓
deployment artifact
```

A modern build system extends that graph:

```text
source
 ├── JavaScript / TypeScript
 ├── CSS
 ├── images
 ├── fonts
 ├── WebAssembly
 ├── workers
 ├── HTML
 └── generated files
       ↓
   build graph
       ↓
   optimized artifacts
```

You will learn:

- what a bundler actually does,
- why bundling exists,
- entry points,
- dependency graphs,
- module graphs versus asset graphs,
- tree shaking,
- dead-code elimination,
- side effects,
- code splitting,
- dynamic imports,
- chunk graphs,
- lazy loading,
- preloading,
- minification,
- compression boundaries,
- externalization,
- library mode,
- application mode,
- browser builds,
- Node builds,
- server-side rendering,
- environment variables,
- plugins,
- loaders,
- virtual modules,
- build caching,
- incremental builds,
- watch mode,
- development servers,
- Hot Module Replacement,
- Vite,
- Rollup,
- esbuild,
- webpack,
- Rolldown-based workflows,
- build reproducibility,
- bundle analysis,
- artifact validation,
- security risks,
- and production deployment strategy.

The principal-level goal is:

> **Understand a build as a graph transformation system, then choose the smallest, fastest, safest, and most maintainable build architecture that satisfies the target runtime and deployment model.**

---

# 1. Learning Objectives

After completing this chapter, you should be able to:

## Core theory

- Define a bundler.
- Define a build system.
- Explain why bundlers exist.
- Explain module graphs.
- Explain asset graphs.
- Explain entry points.
- Explain output chunks.
- Explain tree shaking.
- Explain dead-code elimination.
- Explain side-effect analysis.
- Explain code splitting.
- Explain dynamic import-driven splitting.
- Explain chunk graphs.
- Explain external dependencies.
- Explain minification.
- Explain source-map generation.
- Explain build plugins.
- Explain loaders/transforms.

## Tooling

- Explain webpack conceptually.
- Explain Rollup conceptually.
- Explain esbuild conceptually.
- Explain Vite's development/build architecture.
- Explain the current Vite production pipeline and its Rolldown-based build.
- Explain when a bundler should be used for Node.
- Explain when bundling should be avoided.
- Explain library bundling versus application bundling.

## Optimization

- Reason about tree-shaking boundaries.
- Diagnose why code is not eliminated.
- Diagnose duplicated dependencies.
- Diagnose oversized chunks.
- Diagnose excessive code splitting.
- Reason about chunk loading.
- Design cache-friendly output.
- Optimize initial JavaScript delivery.
- Balance chunk count against caching and network overhead.

## Production

- Design a build pipeline.
- Define development versus production responsibilities.
- Externalize dependencies appropriately.
- Build libraries safely.
- Build server applications safely.
- Design reproducible artifacts.
- Analyze bundles.
- Validate build outputs.
- Secure build infrastructure.

## Principal judgment

- Choose webpack, Rollup, esbuild, Vite, or another build system based on requirements.
- Decide whether bundling is necessary.
- Decide where transformations belong.
- Decide what should be external.
- Design chunking policy.
- Balance runtime performance against build complexity.
- Govern plugins and build dependencies.
- Defend build architecture in a principal-level review.

---

# 2. Prerequisites

Recommended:

- Chapter 41 — Spec Architecture
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability
- Chapter 66 — `package.json` and Module Resolution
- Chapter 67 — Dependency Management and Supply Chain
- Chapter 68 — Transpilation and Compilation

---

# 3. What Is a Bundler?

A bundler takes a set of modules/assets and constructs deployable output artifacts.

Conceptually:

```text
entry
 ↓
dependency graph
 ↓
optimized graph
 ↓
output chunks
```

For example:

```text
src/main.js
   │
   ├── ui.js
   ├── api.js
   └── utils.js
          │
          └── dependency.js
```

A bundler can produce:

```text
dist/
  index.html
  assets/
    main-ABC.js
    vendor-XYZ.js
```

The exact output depends on the bundler and configuration.

---

# 4. Why Does Bundling Exist?

Historically, browsers needed many individual assets loaded over the network.

Bundlers became valuable because they can:

- combine modules,
- eliminate unused code,
- split code strategically,
- transform syntax,
- optimize assets,
- fingerprint filenames,
- generate preload hints,
- create deployable artifacts.

The goal is not:

```text
fewer files at all costs
```

The goal is:

```text
optimal dependency delivery
```

---

# 5. Bundling Is a Graph Problem

A bundler sees:

```text
A
├── B
│   └── D
└── C
    └── D
```

It builds a graph and can determine:

```text
D is shared
```

Then output might contain:

```text
main.js
shared.js
```

This is fundamentally different from concatenating files in lexical order.

---

# 6. Build Graph vs Module Graph

A module graph:

```text
JavaScript/TypeScript modules
```

A broader build graph:

```text
JS
CSS
images
fonts
workers
WASM
HTML
```

Modern tools frequently operate on both.

---

# 7. Entry Points

An entry point is where the build graph starts.

Example:

```js
// main.js
import './app.js';
```

Configured:

```text
entry = main.js
```

The bundler traverses dependencies from the entry.

Multiple entries:

```text
app.js
admin.js
```

can produce separate entry chunks.

---

# 8. Entry Point Design

Good entry points reflect deployment units:

```text
web application
admin application
worker
CLI
library entry
```

Bad entry points:

```text
every tiny file
```

unless your distribution model requires it.

---

# 9. Static Imports

Static:

```js
import { render } from './render.js';
```

creates a statically visible graph edge.

Bundlers can analyze:

```text
dependency
exports
reachability
```

very effectively.

---

# 10. Dynamic Imports

Dynamic:

```js
const module = await import('./reports.js');
```

creates a runtime-loading opportunity.

A bundler can transform this into:

```text
main chunk
    +
reports chunk
```

The user may download `reports.js` only when needed.

This is one of the most important bridges between language semantics and build optimization.

---

# 11. Code Splitting

Code splitting divides application code into multiple output chunks.

Conceptual:

```text
main.js
   │
   ├── dashboard.js
   ├── settings.js
   └── admin.js
```

The objective is:

```text
initial load
  ↓
minimum necessary code
```

followed by:

```text
lazy feature
  ↓
load chunk
```

---

# 12. Code Splitting Is Not Always Good

Too many chunks can create:

- request overhead,
- scheduling complexity,
- cache fragmentation,
- preload complexity,
- runtime orchestration overhead.

Therefore:

> **More chunks is not automatically better performance.**

---

# 13. Chunking as Cost Optimization

A simplified model:

```text
Initial cost
= bytes initially required
+ parse/compile
+ execution
+ network overhead
```

Later cost:

```text
lazy feature cost
= chunk bytes
+ request latency
+ parse/compile
+ execution
```

Optimize the total user-perceived cost, not only file count.

---

# 14. Tree Shaking

Tree shaking removes exports that are proven unnecessary.

Example:

```js
// math.js
export function add() {}
export function subtract() {}
```

Consumer:

```js
import { add } from './math.js';
```

A capable bundler may eliminate `subtract`.

Rollup describes tree shaking as deep execution-path analysis for dead-code elimination. citeturn822673search3

---

# 15. Tree Shaking Requires Analyzability

Static ESM helps:

```text
known imports
known exports
```

CommonJS patterns can be harder to analyze:

```js
module.exports = somethingDynamic();
```

Therefore:

```text
ESM
→ strong static analysis

dynamic CJS
→ harder optimization
```

---

# 16. Dead-Code Elimination

Tree shaking is a form of dead-code elimination.

Conceptually:

```text
unreachable code
      ↓
remove
      ↓
smaller artifact
```

This can happen at:

- module level,
- export level,
- function level,
- branch level,
- constant-expression level.

---

# 17. Tree Shaking Is Not Runtime Garbage Collection

Do not confuse:

```text
tree shaking
```

with:

```text
GC
```

Tree shaking occurs during build.

GC occurs at runtime.

```text
Build time:
unused code → eliminate

Runtime:
unreachable objects → collect
```

---

# 18. Side Effects

Consider:

```js
// analytics.js
registerAnalytics();
```

Even if no function export is imported, loading the module has an effect.

A bundler cannot safely remove it if that side effect matters.

---

# 19. `sideEffects`

Packages can communicate side-effect information to bundlers.

Example:

```json
{
  "sideEffects": false
}
```

This is a bundler-facing optimization contract.

It is not an ECMAScript feature.

Never claim:

```text
sideEffects: false
```

unless all relevant module evaluation side effects can truly be eliminated safely.

---

# 20. Side-Effect Classification

Potential side effects include:

```text
global mutation
polyfill registration
custom element registration
event listener installation
prototype modification
CSS import
environment initialization
```

Some apparently “unused” modules are intentionally required for their effects.

---

# 21. Tree-Shaking Failure Example

```js
export function a() {
  return 1;
}

export function b() {
  console.log('side effect');
}

export const x = calculate();
```

If `calculate()` runs during module evaluation:

```text
module itself has side effects
```

Even if `x` is unused.

---

# 22. Pure Annotations

Some tools can recognize annotations indicating that a call is pure.

Conceptually:

```js
/* @__PURE__ */ createObject();
```

This can help dead-code elimination.

Use tool-supported annotations carefully.

A false purity claim can change program behavior.

---

# 23. Constant Folding

Bundlers can simplify:

```js
const enabled = true;

if (enabled) {
  run();
}
```

into effectively:

```js
run();
```

and may remove unreachable branches.

This becomes particularly powerful with build-time environment constants.

---

# 24. Environment Replacement

A build can inject constants:

```text
process.env.NODE_ENV
```

or tool-specific compile-time constants.

Then:

```js
if (NODE_ENV === 'production') {
  debug();
}
```

may allow the bundler/minifier to eliminate the development branch.

Do not confuse:

```text
build-time substitution
```

with:

```text runtime environment lookup
```

---

# 25. Security: Environment Variables

Do not expose secrets to browser bundles.

Bad:

```js
const API_SECRET = process.env.SECRET;
```

if the bundler substitutes it into a browser artifact.

The resulting code may literally contain:

```text
secret value
```

Treat browser-exposed environment variables as public configuration.

---

# 26. Minification

Minification reduces artifact size through transformations such as:

- whitespace removal,
- identifier shortening,
- constant folding,
- unreachable-code removal.

Example:

```js
function add(a, b) {
  return a + b;
}
```

may become:

```js
function n(n,o){return n+o}
```

Do not depend on generated identifier names.

---

# 27. Minification Is Not Compression

Minification:

```text
source transformation
```

Compression:

```text
transport encoding
```

Examples:

```text
gzip
Brotli
```

Pipeline:

```text
source
 ↓
bundle
 ↓
minify
 ↓
compress during delivery
```

---

# 28. Compression and Chunking

Chunk boundaries affect compression.

A single giant file may compress efficiently but load too much code.

Many tiny files may increase request/header overhead and reduce compression context.

Use measurement.

---

# 29. Asset Hashing

Production output often uses:

```text
main-8f3a2.js
```

The content hash changes when content changes.

Benefits:

```text
long-lived browser cache
+
cache invalidation by filename
```

This is a build/deployment concern.

---

# 30. Content Hashing and Deployment

A typical strategy:

```text
HTML
  no-cache / short cache

hashed assets
  long cache
```

Vite's production build rewrites asset references and supports configurable public base paths. citeturn822673search2

---

# 31. Stale HTML / Chunk Failures

A client may have:

```text
old HTML
```

referencing:

```text
old chunk
```

while the server has already removed that chunk.

This can cause dynamic import failures.

Vite documents this deployment scenario and provides `vite:preloadError` for handling dynamic import/preload errors. citeturn822673search2

---

# 32. Deployment Strategy for Chunks

Safer deployment:

```text
new HTML
+
new assets
```

while retaining previous assets long enough for active clients to migrate.

Do not immediately delete old hashed chunks if long-lived browser sessions are possible.

---

# 33. Externalization

Bundling every dependency is not always desirable.

For a Node library:

```text
your library
   ↓
externalize node runtime deps
```

For browser applications:

```text
bundle application dependencies
```

Potential externalization:

```text
react
node built-ins
database client
peer dependency
```

depending on artifact purpose.

---

# 34. Library vs Application Bundling

## Application

You control the deployment runtime.

Bundling can:

- optimize,
- reduce files,
- split code,
- inline dependencies.

## Library

Your consumer controls the runtime.

Bundling everything can:

- duplicate dependencies,
- hide peer dependencies,
- make debugging harder,
- increase package size.

Library bundling should be conservative.

---

# 35. Rollup as a Library-Oriented Bundler

Rollup emphasizes:

- ESM,
- tree shaking,
- code splitting,
- multiple output formats,
- plugins.

Its official site describes output support for ESM, CommonJS, UMD, SystemJS and others, along with deep tree shaking and code splitting. citeturn822673search3

This makes it strong for libraries and specialized build flows.

---

# 36. webpack Mental Model

webpack historically models the application as a dependency graph and transforms many resource types through loaders/plugins.

Typical concepts:

```text
entry
module rules
loaders
plugins
chunks
optimization
output
```

It remains a major build system for mature applications with extensive plugin ecosystems.

---

# 37. esbuild Mental Model

esbuild focuses heavily on:

```text
speed
bundling
transformation
minification
```

It is often used as a very fast compiler/bundler and can also serve as the transformation engine inside larger build systems.

---

# 38. Vite Mental Model

Current Vite documentation describes two major pieces:

```text
development:
  dev server + native ESM-oriented workflow

production:
  build command using Rolldown
```

Current Vite 8 documentation states that production builds are bundled with Rolldown and that Vite offers optimized build output with advanced tree shaking and code splitting. citeturn822673search1turn822673search0

This is important:

> Do not mentally model modern Vite as “the old dev server plus Rollup build forever.” Its production build architecture has evolved.

---

# 39. Vite Development Model

In development, Vite can serve source modules close to their native ESM form and transform them on demand.

Conceptually:

```text
browser
  ↓
dev server
  ↓
module request
  ↓
transform / dependency optimization
  ↓
browser ESM
```

This reduces the need for full application bundling on every edit.

Vite describes this as serving source over native ESM with dependency pre-bundling. citeturn822673search0turn822673search1

---

# 40. Vite Production Model

Production:

```text
source
 ↓
Vite build
 ↓
Rolldown
 ↓
optimized chunks/assets
```

Current Vite production documentation states that `vite build` produces optimized static assets and exposes chunking controls through Rolldown options. citeturn822673search2

---

# 41. Development vs Production Build

Development prioritizes:

```text
fast startup
fast feedback
HMR
minimal rebuild
```

Production prioritizes:

```text
tree shaking
chunking
minification
caching
artifact size
startup/runtime performance
```

Do not force development architecture to optimize production bundle size at every moment.

---

# 42. Hot Module Replacement

HMR updates changed modules without full-page reload.

Conceptually:

```text
edit
 ↓
detect change
 ↓
retransform module
 ↓
send update
 ↓
replace module
```

HMR state preservation depends on framework/tooling conventions.

HMR is not a production deployment feature by itself.

---

# 43. HMR and Module Semantics

HMR can create runtime behavior that does not exist in a clean process:

```text
module loaded
 ↓
updated
 ↓
old state retained in framework/runtime
```

Therefore:

> Always test important lifecycle behavior from a clean production build.

---

# 44. Plugins

A bundler plugin can participate in:

```text
resolve
load
transform
analyze
generate
write
```

The exact hooks differ by tool.

A plugin can therefore:

- rewrite imports,
- load virtual modules,
- transform source,
- inject assets,
- modify chunks.

---

# 45. Plugin Power = Plugin Risk

Build plugins often execute with access to:

```text
filesystem
environment
network
process
build graph
```

This makes plugins supply-chain code.

Chapter 67 principles apply directly.

---

# 46. Virtual Modules

A plugin can define a virtual module:

```js
import config from 'virtual:config';
```

The module does not need to exist as a physical file.

This enables:

- generated configuration,
- framework metadata,
- asset manifests,
- environment modules.

It also means runtime filesystem inspection cannot always explain build behavior.

---

# 47. Loaders

Some build systems support loader concepts such as:

```text
.js
.ts
.css
.json
.svg
```

A loader controls how source is interpreted/transformed.

Do not confuse a bundler loader with a Node runtime loader.

---

# 48. Runtime Loader vs Build Loader

Node:

```text
runtime loading
```

Bundler:

```text
build-time loading
```

Both may use the word:

```text
loader
```

but solve different problems.

---

# 49. CSS as Dependency

Modern bundlers often model:

```js
import './styles.css';
```

as a build-graph edge.

Then:

```text
JS module
   ↓
CSS asset
   ↓
output CSS
```

This is why modern build systems are more accurately described as asset graph processors.

---

# 50. Asset Graph

Conceptual:

```text
main.js
 ├── component.js
 │    └── image.png
 └── styles.css
       └── font.woff2
```

A bundler can track all these relationships and emit:

```text
main.js
styles.css
image-hash.png
font-hash.woff2
```

with rewritten references.

---

# 51. URL Rewriting

If source uses:

```css
background: url('./image.png');
```

the build can rewrite the output path to:

```text
/assets/image-A1B2.png
```

Asset hashing and deployment base paths therefore belong to the build graph.

---

# 52. Public Base Path

Applications may be deployed at:

```text
https://example.com/
```

or:

```text
https://example.com/app/
```

The build must know how asset URLs should be generated.

Vite exposes a configurable `base` setting for this and rewrites asset references accordingly. citeturn822673search2

---

# 53. Code Splitting Strategies

Common split dimensions:

```text
route
feature
vendor
framework
locale
worker
```

Do not split blindly by source directory.

Split by user-facing loading boundaries.

---

# 54. Route-Level Splitting

Example:

```js
const AdminPage = () => import('./admin-page.js');
```

Then:

```text
public users
   ↓
main chunk

admin users
   ↓
admin chunk
```

This is usually more valuable than splitting every component.

---

# 55. Vendor Splitting

A bundler may create:

```text
vendor.js
```

for dependencies.

But modern caching and chunking strategies can be more nuanced than “one vendor bundle.”

Dependencies may evolve independently.

Prefer measured strategies over fixed folklore.

---

# 56. Common Dependency Duplication

If:

```text
A → lodash@4
B → lodash@4
```

a bundler should ideally avoid shipping two copies if the graph can safely share them.

But different versions or incompatible package identity can result in duplication.

Bundle analysis can reveal this.

---

# 57. Bundle Analysis

Use a visual or textual analyzer to inspect:

```text
largest modules
largest chunks
duplicate dependencies
unexpected libraries
source map composition
```

The important question is:

> What actually entered production?

---

# 58. Bundle Size Is Not One Number

Separate:

```text
raw size
gzip size
Brotli size
parse size
compile cost
execution cost
```

A smaller gzip file can still be expensive to parse/execute.

---

# 59. Runtime Performance Model

Browser startup:

```text
download
 ↓
decompress
 ↓
parse
 ↓
compile
 ↓
execute
```

Reducing bytes helps.

But:

```text
less code
```

is only one optimization dimension.

---

# 60. Chunk Count Trade-Off

More chunks can:

```text
improve lazy loading
improve caching
```

but can also:

```text
increase request complexity
increase scheduling
increase metadata
```

HTTP/2 and HTTP/3 reduce some historical costs, but they do not make unlimited chunking free.

---

# 61. Cache Boundaries

Good split:

```text
stable dependency
+
frequently changing application
```

separate.

Then application changes do not invalidate stable cached vendor code unnecessarily.

But overly aggressive vendor splitting can create long dependency chains.

Measure.

---

# 62. Build Cache

Large projects can cache:

```text
parsed modules
transforms
dependency analysis
generated outputs
```

Benefits:

```text
faster incremental builds
```

Risks:

```text
stale cache
cache poisoning
incorrect invalidation
```

---

# 63. Incremental Builds

A good build system answers:

```text
What changed?
What depends on it?
What must rebuild?
```

This is graph invalidation.

Conceptually:

```text
changed node
   ↓
dependent closure
   ↓
rebuild only affected outputs
```

---

# 64. Watch Mode

Watch mode:

```text
filesystem change
 ↓
invalidate graph
 ↓
rebuild affected modules
 ↓
update output
```

A good watch system is dependency-aware, not simply:

```text
run entire build again
```

---

# 65. Build Systems at Scale

For large monorepos:

```text
package A
package B
package C
```

the build graph can be:

```text
A → B
A → C
B → D
C → D
```

You can parallelize independent work.

This is why build graph architecture matters.

---

# 66. Parallel Builds

If:

```text
A
B
```

do not depend on each other:

```text
build A ─┐
         ├──→ final
build B ─┘
```

A scalable build system should exploit this.

---

# 67. Cache + Parallelism

Best-case large build:

```text
cache hit
   ↓
skip

cache miss
   ↓
parallel build
```

Build performance is therefore:

```text
graph quality
+
cache quality
+
parallelism
```

not simply compiler speed.

---

# 68. Reproducible Builds

A production build should aim for:

```text
same source
+
same dependency graph
+
same tool versions
+
same config
=
same logical artifact
```

Deterministic chunk naming and stable asset content improve debugging and caching.

---

# 69. Build IDs and Versioning

Useful artifact metadata:

```text
commit SHA
build timestamp
release version
toolchain version
```

Avoid embedding constantly changing timestamps into every asset if they destroy content-hash stability.

---

# 70. Environment Configuration

Separate:

```text
build-time config
```

from:

```text
runtime config
```

Browser bundles frequently compile configuration into static artifacts.

Server applications may instead load configuration at runtime.

Do not force one model everywhere.

---

# 71. Server Bundling

Node server bundling can reduce:

```text
startup filesystem resolution
deployment file count
artifact complexity
```

But it can also:

- break dynamic `require`,
- complicate native addons,
- alter module identity,
- change package behavior,
- complicate source maps.

Server bundling should be justified.

---

# 72. Node Externalization

For a Node application:

```text
bundle application code
externalize selected packages
```

or:

```text
bundle everything that is safely bundleable
```

depending on runtime requirements.

Externalizing native addons and dynamic packages is often necessary.

---

# 73. Native Modules and Bundlers

Native dependencies such as:

```text
.node
```

files cannot simply be treated like ordinary JS.

The build may need:

```text
externalization
copy asset
runtime path
```

A bundler configuration must preserve their runtime expectations.

---

# 74. Dynamic `require`

Bundlers struggle with:

```js
require(`./plugins/${name}.js`);
```

because the full dependency graph cannot always be statically known.

Possible approaches:

```text
explicit allowlist
context modules
plugin APIs
dynamic imports with known patterns
externalize
```

---

# 75. Dynamic Import and Bundle Graph

A predictable dynamic import:

```js
import('./admin.js');
```

is easy to model.

An arbitrary runtime-generated specifier:

```js
import(userInput);
```

is much harder to bundle safely.

---

# 76. Library Externalization

For a library:

```text
peer dependency
```

is often externalized so consumers provide the dependency.

Example:

```text
your-react-library
     ↓
React peer dependency
```

Bundling another React copy can create duplicate runtime identity.

---

# 77. Library Output Formats

A library may publish:

```text
ESM
CJS
```

or other formats where genuinely required.

Do not produce many formats automatically.

Every output increases:

```text
test surface
maintenance
publication complexity
```

---

# 78. Vite Library Mode

Current Vite documentation supports a library build mode and recommends externalizing dependencies that should not be bundled, such as framework dependencies. It can produce configurable output formats for libraries. citeturn822673search2

Use library mode when Vite's packaging model fits your distribution requirements.

---

# 79. Bundler Plugin Ecosystems

A build system can have:

```text
core
plugins
transforms
asset handlers
framework adapters
```

This provides flexibility.

But every plugin is another:

```text
dependency
execution surface
configuration layer
failure mode
```

Govern plugins like production dependencies.

---

# 80. Plugin Ordering

If:

```text
plugin A
```

changes an import before:

```text
plugin B
```

resolves it, order matters.

For large systems, document:

```text
resolution stage
transform stage
asset stage
output stage
```

rather than relying on incidental plugin order.

---

# 81. Plugin Security

Build plugins may read:

```text
filesystem
environment
git metadata
credentials
```

Do not install arbitrary plugins without review.

This is a direct Chapter 67 supply-chain concern.

---

# 82. Configuration as Code

Build configs are executable code.

Example:

```js
export default {
  plugins: [...],
  build: {...},
};
```

Therefore they should receive:

```text
code review
tests
version control
security review
```

---

# 83. Development Server Security

A local dev server can expose:

```text
source files
environment variables
filesystem paths
HMR endpoints
```

Bind/listen intentionally.

Do not expose a development server to untrusted networks by accident.

---

# 84. HMR Security

HMR protocols can provide powerful development capabilities.

Production should not accidentally ship development-only:

```text
HMR clients
debug endpoints
development variables
```

Verify production output independently.

---

# 85. Source Exposure

Bundled browser artifacts are public.

Do not assume:

```text
minification
=
secrecy
```

Anything shipped to a browser should be treated as recoverable by users.

---

# 86. Minification Is Not Obfuscation

Minification:

```text
reduces code size
```

It does not provide:

```text
confidentiality
```

Never ship secrets because you believe minification hides them.

---

# 87. Build-Time Secrets

Danger:

```text
build environment
 ↓
define SECRET
 ↓
bundle
 ↓
public JS
```

The secret is now in the artifact.

Use build-time secret injection only for values genuinely safe to embed.

---

# 88. Build Artifact Integrity

After building:

```text
hash artifacts
sign/attest where appropriate
store release metadata
```

This connects build architecture to software supply-chain security.

---

# 89. Artifact Promotion

A mature pipeline can use:

```text
source
 ↓
build
 ↓
test
 ↓
scan
 ↓
attest
 ↓
artifact registry
 ↓
promote same artifact
 ↓
production
```

Do not rebuild production from a slightly different environment after testing when artifact immutability is a priority.

---

# 90. Build Once, Deploy Many

Strong deployment model:

```text
build once
   ↓
artifact
   ↓
staging
   ↓
production
```

This reduces environment-specific drift.

---

# 91. Environment-Specific Builds

Sometimes you genuinely need:

```text
browser build
node build
edge build
```

Create explicit artifacts.

Do not hide environment divergence in dozens of runtime conditionals.

---

# 92. Browser vs Node Bundling

Browser:

```text
DOM APIs
Web Workers
CSS
assets
```

Node:

```text
filesystem
network
native addons
process
```

Different targets require different externalization/polyfill policies.

---

# 93. Edge/Serverless Bundling

Edge runtimes may have:

```text
small cold-start budget
limited Node APIs
strict module/asset behavior
```

Bundling can be particularly valuable.

But target the actual runtime APIs available.

---

# 94. SSR

Server-side rendering often has:

```text
server bundle
+
client bundle
```

The build graph must distinguish:

```text
server-only dependencies
client-safe dependencies
shared modules
```

Accidental server-only code in browser output can cause build or security problems.

---

# 95. Client/Server Boundary

A clean SSR build models:

```text
shared
 ├── pure domain
 └── safe utilities

server
 ├── DB
 ├── filesystem
 └── secrets

client
 ├── DOM
 └── browser APIs
```

The bundler helps enforce this only when the graph is explicit and tooling is configured properly.

---

# 96. Dead Code from Environment Branches

Example:

```js
if (import.meta.env.SSR) {
  serverOnly();
} else {
  clientOnly();
}
```

A build can replace the environment constant and remove the unreachable branch.

This is a major benefit of environment-aware bundling.

---

# 97. Environment Constants and Safety

Even when build-time constants are used for optimization, do not rely solely on:

```text
dead branch
```

for security if both branches contain sensitive code that can somehow be retained by a different build target.

Architect the source graph so server-only modules are not imported into client entries in the first place.

---

# 98. External Modules in Browser Bundles

If a browser build leaves:

```js
import something from 'some-package';
```

external, then the deployment must provide that module.

Do not accidentally externalize an application dependency unless the runtime knows how to load it.

---

# 99. Bundle vs CDN

Some applications intentionally externalize stable libraries to CDNs.

Benefits:

- browser caching,
- reduced application artifact.

Risks:

- third-party availability,
- CSP complexity,
- integrity,
- version drift,
- privacy/security.

This is an architecture decision, not a default optimization.

---

# 100. Build System Observability

Record:

```text
build duration
cache hit rate
artifact sizes
chunk count
largest modules
transform time
plugin time
```

For large builds:

```text
build bottleneck
=
data
```

not intuition.

---

# 101. Bundle Analysis Metrics

Useful:

```text
initial JS
total JS
largest chunk
largest dependency
duplicated package bytes
lazy chunk count
CSS size
asset count
```

For browser performance, connect these to:

```text
real-user metrics
```

rather than optimizing bundle size in isolation.

---

# 102. Build Performance Metrics

Measure:

```text
cold build
warm build
incremental build
test build
CI build
```

A build that is:

```text
5 min cold
10 sec incremental
```

may be acceptable.

A build that is:

```text
10 sec cold
9 sec every edit
```

may still feel slow to developers.

---

# 103. Build Caching Strategy

Possible layers:

```text
dependency cache
transform cache
package build cache
CI cache
remote artifact cache
```

The more layers, the more invalidation complexity.

Use explicit ownership.

---

# 104. Clean Build

Always maintain a way to perform:

```bash
clean build
```

because incremental caches can hide:

```text
missing dependencies
stale transforms
incorrect configuration
```

---

# 105. Rebuild-from-Scratch Test

Periodically verify:

```text
delete cache
delete dist
fresh install
build
```

The build should still succeed.

---

# 106. Build Reproducibility and Timestamps

Some generated artifacts can contain:

```text
timestamps
absolute paths
machine-specific identifiers
```

These reduce reproducibility.

Prefer stable metadata where feasible.

---

# 107. Absolute Paths in Artifacts

Generated source maps can expose:

```text
/home/developer/project
```

or:

```text
C:\Users\...
```

This can leak internal directory structure.

Configure source paths appropriately.

Chapter 70 covers this more deeply.

---

# 108. Plugin Supply Chain

A build system can fail security review even when application dependencies are safe if:

```text
unknown build plugin
```

has access to:

```text
CI secrets
```

Therefore dependency governance includes:

```text
compiler
bundler
plugin
loader
preset
```

---

# 109. Lockfiles and Bundlers

The build artifact depends on:

```text
package.json
+
lockfile
+
bundler version
+
plugins
```

A lockfile alone is insufficient if:

```text
bundler/plugin versions
```

change outside the expected graph.

Keep build tools under dependency governance.

---

# 110. Build Configuration Drift

Bad:

```text
local config
CI config
production config
```

all manually maintained.

Better:

```text
shared configuration
+
explicit environment overrides
```

Document differences.

---

# 111. Build System Failure Modes

## Failure Mode 1 — Tree shaking does nothing

Possible cause:

```text
CommonJS dynamic exports
side effects
```

## Failure Mode 2 — Bundle too large

Possible cause:

```text
entire library imported
duplicate dependencies
```

## Failure Mode 3 — App fails only in production

Possible cause:

```text
development transform
≠
production transform
```

## Failure Mode 4 — Dynamic import fails after deployment

Possible cause:

```text
stale HTML / deleted old chunk
```

---

# 112. Failure Mode 5 — Native Dependency Broken

Cause:

```text
bundled .node incorrectly
```

## Failure Mode 6 — Library consumer gets two React copies

Cause:

```text
peer dependency bundled
```

## Failure Mode 7 — SSR leaks secrets

Cause:

```text
server module enters client graph
```

---

# 113. Common Misconceptions

## Misconception 1

> A bundler just concatenates files.

False.

It resolves and transforms a graph.

## Misconception 2

> Tree shaking removes anything unused at runtime.

No.

It uses static/build-time analysis.

## Misconception 3

> Bundling always improves Node performance.

No.

It can improve or worsen startup/debugging/module behavior.

## Misconception 4

> Minified code is secure.

False.

## Misconception 5

> Vite is only a development server.

False.

Current Vite provides a production build pipeline using Rolldown. citeturn822673search1turn822673search2

## Misconception 6

> More chunks always mean faster pages.

False.

## Misconception 7

> A library should bundle all its dependencies.

Often false.

## Misconception 8

> `sideEffects: false` is harmless metadata.

False.

It can change whether modules are eliminated.

---

# 114. Common Mistakes

```text
[ ] Bundling without identifying target runtime
[ ] Bundling Node internals unnecessarily
[ ] Bundling peer dependencies
[ ] Using broad dynamic imports
[ ] Marking side effects incorrectly
[ ] Adding too many chunks
[ ] Ignoring duplicate dependencies
[ ] Exposing secrets through build-time replacement
[ ] Shipping dev-only code
[ ] Relying on source maps as security controls
[ ] Ignoring plugin supply-chain risk
[ ] Testing only dev server behavior
[ ] Deleting old hashed chunks too quickly
[ ] Publishing artifacts without direct runtime tests
```

---

# 115. Comparison — webpack vs Rollup vs esbuild vs Vite

| Tool | Core strength | Typical use |
|---|---|---|
| webpack | mature configurable ecosystem | large/mature apps |
| Rollup | library-oriented graph optimization | libraries and specialized builds |
| esbuild | speed | fast transforms/builds |
| Vite | dev experience + modern production build | web applications |
| Rolldown | Rust-based high-performance bundling foundation used by current Vite production builds | high-performance bundling workflows |

Vite's current documentation states that production builds use Rolldown, while Rollup's documentation emphasizes tree shaking, code splitting, and plugin-driven builds. citeturn822673search1turn822673search3

Tool choice must be based on actual project requirements and current release behavior.

---

# 116. Comparison — Bundler vs Transpiler

| Concern | Transpiler | Bundler |
|---|---|---|
| Syntax transformation | ✅ | often |
| Type erasure | ✅ | often via plugin/integration |
| Type checking | separate | separate |
| Dependency graph | limited | ✅ |
| Code splitting | ❌ | ✅ |
| Asset graph | usually ❌ | ✅ |
| Tree shaking | limited | ✅ |
| Minification | sometimes | often |
| Output chunks | ❌ | ✅ |
| Packaging | partial | ✅ |

Modern tools blur boundaries, so evaluate actual capabilities rather than labels.

---

# 117. Comparison — Bundling vs Native ESM

Native ESM:

```text
browser/Node resolves modules at runtime
```

Bundling:

```text
build resolves modules ahead of time
```

Native ESM is attractive when:

- runtime can handle the module graph,
- minimal build step is desired,
- deployment supports many modules.

Bundling is attractive when:

- asset optimization matters,
- code splitting matters,
- legacy compatibility matters,
- deployment wants optimized artifacts.

---

# 118. Comparison — Build Once vs Runtime Compilation

### Build once

```text
source
↓
artifact
↓
deploy
```

### Runtime transform

```text
request/startup
↓
transform
↓
execute
```

Production generally benefits from predictable prebuilt artifacts.

---

# 119. Specification Boundary

Bundlers are not ECMAScript.

They operate above the language/runtime:

```text
ECMAScript semantics
      ↓
Node/browser runtime
      ↑
bundler transforms graph before runtime
```

A bundler can alter packaging without changing the language specification.

---

# 120. Runtime Semantics Must Survive Bundling

A correct bundler should preserve required semantics.

But bundling can expose issues around:

```text
eval
dynamic import
dynamic require
module identity
top-level await
side effects
global scope
worker URLs
native modules
```

Test the output, not just source.

---

# 121. `eval` and Bundlers

Dynamic evaluation complicates static analysis:

```js
eval(code);
```

The bundler cannot fully know what modules/code will be needed.

Avoid dynamic code loading patterns that defeat graph analysis unless required.

---

# 122. `new Function`

Likewise:

```js
new Function(source);
```

creates runtime code outside static analysis.

This can interfere with:

- minification,
- CSP,
- security,
- tree shaking.

Treat carefully.

---

# 123. Top-Level Await and Bundling

Bundlers need to preserve or transform asynchronous module evaluation.

If the target runtime supports top-level await:

```text
preserve
```

may be appropriate.

If not:

```text
transform
```

may be necessary, if semantics can be preserved.

Do not use top-level await merely because the bundler accepts it.

---

# 124. Module Identity After Bundling

Bundling can collapse many modules into one artifact.

This can change assumptions around:

```text
module-level singleton
module.filename
import.meta.url
dynamic loading
```

Server-side bundling should test such assumptions.

---

# 125. Worker Bundling

Workers often need their own output artifact:

```text
main.js
worker.js
```

The bundler must preserve:

```text
worker URL
asset path
module format
```

Do not assume:

```text
worker code inside main bundle
```

can be executed directly.

---

# 126. WebAssembly

A bundler can include:

```text
.wasm
```

as an asset or transform target depending on configuration.

Treat:

```text
runtime fetch
module loading
asset URL
```

as explicit concerns.

---

# 127. Build Output for Node Workers

A Node worker may require:

```js
new Worker(new URL('./worker.js', import.meta.url));
```

After bundling, the asset relationship must remain valid.

This is a classic example of:

```text
source module graph
+
runtime asset identity
```

needing coordinated handling.

---

# 128. Build Output for Dynamic Assets

If code constructs:

```js
new URL(`./icons/${name}.svg`, import.meta.url)
```

the bundler needs enough static information to know which assets exist.

Arbitrary runtime paths can defeat asset analysis.

Use explicit maps where possible.

---

# 129. Plugin-Driven Asset Handling

A plugin might transform:

```text
.svg
```

into:

```js
export default 'data:image/svg+xml,...';
```

or emit:

```text
asset-hash.svg
```

The runtime semantics are different.

Test consumer expectations.

---

# 130. Build System API Design

A mature build system should separate:

```text
configuration
graph construction
transformation
optimization
output
```

This makes it easier to:

- cache,
- test,
- debug,
- parallelize.

---

# 131. Configuration Layering

Conceptually:

```text
base config
   ↓
development config
production config
library config
server config
```

Avoid copying the entire configuration four times.

---

# 132. Build Profiles

Useful profiles:

```text
dev
test
production
analyze
library
```

Each should differ only where the deployment contract differs.

---

# 133. Analyze Mode

A dedicated build mode can emit:

```text
bundle statistics
chunk graph
dependency sizes
timings
```

without changing production behavior.

This makes performance analysis repeatable.

---

# 134. Build Validation Pipeline

A mature CI pipeline:

```text
type check
 ↓
unit tests
 ↓
build
 ↓
artifact validation
 ↓
bundle analysis
 ↓
security scan
 ↓
integration smoke test
 ↓
publish/promote
```

---

# 135. Build Artifact Smoke Tests

For browser:

```text
serve dist
load entry
execute critical flows
```

For Node:

```text
node dist/server.js
```

For libraries:

```text
clean consumer project
install package
import package
require package
```

---

# 136. Clean Consumer Test

A library should be tested from:

```text
fresh directory
```

because monorepo build tooling can hide:

- missing files,
- phantom dependencies,
- export-map problems,
- undeclared runtime dependencies.

---

# 137. Bundle Budget

Set limits:

```text
initial JS < X
largest chunk < Y
total JS < Z
```

A budget is useful because performance regressions otherwise accumulate.

Do not set arbitrary numbers without measuring your product.

---

# 138. Bundle Budget Exceptions

When a feature needs more code:

```text
document reason
measure impact
approve exception
```

Do not allow budgets to become meaningless because every team bypasses them.

---

# 139. Performance Regression

Compare:

```text
previous build
vs
current build
```

Track:

```text
initial bytes
lazy bytes
parse cost
execution cost
cache hit potential
```

---

# 140. Build Failure Diagnostics

When a production bundle breaks:

```text
1. identify artifact
2. identify entry
3. inspect chunk graph
4. inspect source module
5. inspect transformed output
6. inspect environment replacements
7. inspect plugin output
8. reproduce clean build
```

Avoid immediately changing bundler configuration.

---

# 141. Tree-Shaking Debugging

Ask:

```text
Is the import statically analyzable?
Does the module have side effects?
Does package metadata declare sideEffects?
Is CommonJS involved?
Is the exported symbol actually reachable?
Did a plugin hide the graph?
```

---

# 142. Duplicate Dependency Debugging

Look for:

```text
same package multiple versions
same package bundled twice
ESM/CJS variants
symlinked workspaces
peer dependency mismatch
```

---

# 143. Chunk Debugging

If a chunk is huge:

```text
identify top contributors
```

Then ask:

```text
Can it be lazy?
Can unused exports be removed?
Can dependency be externalized?
Is there a duplicate?
Should the feature be split?
```

---

# 144. Dynamic Import Debugging

If:

```js
await import('./feature.js');
```

fails in production:

```text
check generated chunk
check asset path
check base path
check deployment retention
check cache
check runtime support
```

Vite specifically documents stale-deployment dynamic-import/preload errors. citeturn822673search2

---

# 145. Build-Time Environment Debugging

If:

```js
if (DEV) ...
```

behaves unexpectedly:

```text
inspect replacement
inspect build mode
inspect config
inspect emitted artifact
```

Do not assume `process.env` remains dynamic after bundling.

---

# 146. Source Map Preview

Source maps make debugging transformed/bundled output feasible.

A typical chain:

```text
source
 ↓
transform
 ↓
bundle
 ↓
minify
 ↓
output.js
output.js.map
```

Chapter 70 will make source maps a first-class debugging topic.

---

# 147. Security Considerations

Bundlers can accidentally:

- expose secrets,
- expose source paths,
- include server-only modules,
- include debug code,
- ship development dependencies,
- expose internal endpoints,
- execute malicious build plugins.

Treat build artifacts as public unless proven otherwise.

---

# 148. Server-Only Code Leakage

Example:

```js
import { getSecret } from './server/secrets.js';
```

If imported by client code, the bundler may package server logic into browser assets.

Architecture should prevent this at the module-boundary level.

---

# 149. Dependency License Leakage

Bundling dependencies into one artifact can make license obligations less visible.

Preserve:

```text
license files
notices
attribution
```

as required.

---

# 150. Production Decision Framework

When evaluating a bundler/build system, score:

| Dimension | Question |
|---|---|
| Correctness | Does output preserve required runtime behavior? |
| Performance | Are startup/load/build costs acceptable? |
| Memory | Does output avoid unnecessary duplication? |
| Security | Is the build graph trustworthy and least-privileged? |
| Reliability | Are builds deterministic and reproducible? |
| Maintainability | Is configuration understandable? |
| Scalability | Does the graph/cache handle repository growth? |
| Observability | Can bundle/build failures be diagnosed? |
| Developer Experience | Are feedback loops fast? |
| Operational Complexity | How many plugins/configurations are required? |
| Future Change | Can target/tooling evolve safely? |

---

# 151. Build-System Selection Guide

## Choose a highly configurable system when:

```text
large legacy application
many loaders/plugins
complex migration
special asset pipeline
```

## Choose a library-focused bundler when:

```text
library distribution
tree shaking
multiple output formats
minimal runtime
```

## Choose a speed-focused bundler/compiler when:

```text
large build
simple transforms
fast CI
minimal configuration
```

## Choose a modern integrated web build tool when:

```text
browser application
fast dev server
HMR
optimized production output
```

The right answer depends on the project.

---

# 152. When Not to Bundle

Avoid bundling when:

```text
runtime already handles modules efficiently
library consumers need native dependency identity
dynamic loading is central
native addons are complex
debugging simplicity is more valuable
```

A Node backend can be perfectly healthy as:

```text
dist/**/*.js
node_modules/
package.json
```

without a single monolithic bundle.

---

# 153. When Bundling Is Worth It

Strong cases:

```text
browser application
serverless cold-start optimization
edge deployment
CLI single-artifact distribution
complex asset pipeline
legacy browser compatibility
```

Still measure.

---

# 154. Production Build Architecture Example

```text
src/
 ├── app/
 ├── domain/
 ├── infrastructure/
 └── main.ts
       │
       ▼
Type checker
       │
       ▼
Bundler
 ├── resolution
 ├── transforms
 ├── tree shaking
 ├── chunking
 ├── minification
 └── source maps
       │
       ▼
dist/
 ├── index.html
 ├── assets/
 └── chunks/
       │
       ▼
static hosting / CDN
```

---

# 155. Node Production Build Example

```text
src/
    ↓
TypeScript type check
    ↓
transformation/bundling
    ↓
dist/
    ↓
artifact validation
    ↓
container
    ↓
Node
```

Externalize or preserve runtime-sensitive dependencies deliberately.

---

# 156. Library Production Build Example

```text
src
 ↓
type check
 ↓
library bundler
 ├── ESM
 ├── CJS if required
 └── types separately
 ↓
exports map
 ↓
clean consumer tests
 ↓
publish
```

---

# 157. Build Security Checklist

```text
[ ] build dependencies are reviewed
[ ] plugins are approved
[ ] CI secrets are minimized
[ ] browser build contains no secrets
[ ] server-only code is excluded from client graph
[ ] source maps are handled intentionally
[ ] artifacts are scanned
[ ] licenses are preserved
[ ] build provenance is tracked
[ ] production artifact is immutable
```

---

# 158. Production Bundle Checklist

```text
[ ] correct entry
[ ] correct target
[ ] correct module format
[ ] tree shaking effective
[ ] duplicate dependencies checked
[ ] chunk sizes measured
[ ] lazy paths tested
[ ] hashed assets generated
[ ] base paths tested
[ ] stale chunk strategy tested
[ ] source maps validated
[ ] dev-only code excluded
[ ] secrets absent
```

---

# 159. Implementation From Scratch

## Stage A — Guided

Build:

```text
main.js
feature.js
unused.js
```

Use a bundler and determine:

```text
what gets emitted
```

---

# 160. Stage B — Partially Guided

Add:

```text
dynamic import
```

and inspect generated chunks.

---

# 161. Stage C — No Reference

Create:

```text
main
 ├── home
 ├── dashboard
 └── settings
```

Split:

```text
home → initial
dashboard → lazy
settings → lazy
```

Measure the result.

---

# 162. Stage D — Edge-Case Hardened

Add:

- side effects,
- CommonJS package,
- duplicate dependency versions,
- worker,
- dynamic asset,
- environment constant,
- native dependency simulation.

---

# 163. Stage E — Production Grade

Build:

```text
type check
+
bundle
+
source maps
+
artifact validation
+
bundle budget
+
security checks
+
clean deployment test
```

---

# 164. Implementation Challenge — Tree Shaking

Create:

```js
export function used() {}
export function unused() {}
```

Import only:

```js
used
```

Prove whether `unused` reaches the output.

Then add a side-effectful module and compare.

---

# 165. Implementation Challenge — Code Splitting

Create:

```text
main.js
admin.js
```

with:

```js
const loadAdmin = () => import('./admin.js');
```

Verify:

```text
admin code is not part of the initial chunk
```

when the bundler/target configuration supports this split.

---

# 166. Implementation Challenge — Library Externalization

Build a library that uses:

```text
react
```

as a peer dependency.

Verify that the output does not contain a private second React copy.

---

# 167. Implementation Challenge — Bundle Budget

Create CI rules:

```text
initial JS < threshold
largest chunk < threshold
```

Fail builds on regressions.

---

# 168. Implementation Challenge — Secure Environment

Create two values:

```text
PUBLIC_API_URL
PRIVATE_API_SECRET
```

Build a browser bundle.

Prove:

```text
PUBLIC_API_URL → allowed
PRIVATE_API_SECRET → absent
```

---

# 169. Implementation Challenge — Artifact Promotion

Build:

```text
artifact-v1
```

deploy to:

```text
staging
production
```

without rebuilding.

Record:

```text
hash
commit
toolchain
```

---

# 170. Debugging Exercises

## Exercise 1

A 3 MB dependency enters a 100 KB feature bundle.

Find why.

---

## Exercise 2

Tree shaking leaves a supposedly unused library.

Investigate:

```text
side effects
CJS
package metadata
```

---

## Exercise 3

A production dynamic import fails after deployment.

Investigate:

```text
old HTML
deleted chunk
base path
cache
```

---

## Exercise 4

Two React copies are in the browser bundle.

Find:

```text
dependency graph
peer dependency
version mismatch
```

---

## Exercise 5

Server secret appears in client artifact.

Trace:

```text
client entry
 ↓
import
 ↓
server module
 ↓
constant replacement
```

Remove the graph edge.

---

## Exercise 6

Production works locally but fails after `npm pack`.

Inspect:

```text
published artifacts
exports
assets
```

---

# 171. Code Review Exercise

Review:

```js
export default {
  build: {
    minify: true,
    sourcemap: true,
  },

  define: {
    SECRET: JSON.stringify(process.env.SECRET),
  },

  plugins: [
    someRandomPlugin(),
  ],

  resolve: {
    alias: {
      '@': '/home/developer/project/src',
    },
  },
};
```

Identify at least 15 production concerns.

Expected topics:

- secret injection,
- source-map exposure,
- absolute paths,
- plugin trust,
- environment handling,
- reproducibility,
- portability,
- artifact leakage,
- configuration validation.

---

# 172. Interview Questions

## Foundation

1. What is a bundler?
2. Why do bundlers exist?
3. What is an entry point?
4. What is a module graph?
5. What is tree shaking?
6. What is dead-code elimination?
7. What is code splitting?
8. What is a chunk?
9. What is minification?
10. What is externalization?

## Intermediate

11. Why does ESM help tree shaking?
12. Why can CommonJS make static analysis harder?
13. What is a side effect?
14. How does dynamic import enable code splitting?
15. Why can too many chunks be harmful?
16. What is a build graph?
17. What is a plugin?
18. What is a virtual module?
19. What is a build cache?
20. Why should libraries often externalize peer dependencies?

## Advanced

21. Explain tree shaking and side-effect analysis.
22. How does a bundler determine dependency reachability?
23. Why can two copies of a package appear in one bundle?
24. How can bundling alter module identity assumptions?
25. Why can dynamic `require()` be hard to bundle?
26. Why should server bundling be treated differently from browser bundling?
27. How do source maps interact with minification?
28. Why can environment replacement leak secrets?
29. Why can stale browser chunks break after deployment?
30. How would you diagnose an unexpectedly large chunk?

## Principal Level

31. Design a build architecture for a 500-package monorepo.
32. When should a Node backend not be bundled?
33. When should a library be bundled?
34. How would you choose between webpack, Rollup, esbuild, and Vite?
35. How would you design a chunking strategy around product routes?
36. How would you secure build plugins?
37. How would you make builds reproducible?
38. How would you design build caches for a monorepo?
39. How would you prevent server-only dependencies from entering browser bundles?
40. How would you govern build configuration across hundreds of packages?

---

# 173. Predict-the-Outcome Exercises

## Exercise A

```js
// math.js
export const used = 1;
export const unused = 2;
```

```js
// main.js
import { used } from './math.js';
console.log(used);
```

Will a capable production bundler necessarily emit both exports?

Explain the difference between source and output.

---

## Exercise B

```js
import('./admin.js');
```

What build opportunity does this create?

---

## Exercise C

```js
const production = false;

if (production) {
  loadDebugTools();
}
```

What optimization might a build/minifier perform?

---

## Exercise D

```js
import './register-global.js';
```

Can the bundler safely remove the module merely because no export is consumed?

---

## Exercise E

A library marks:

```json
{
  "sideEffects": false
}
```

but one module performs:

```js
customElements.define(...);
```

What could go wrong?

---

## Exercise F

A browser bundle contains:

```text
PRIVATE_DATABASE_PASSWORD
```

Where should you investigate first?

---

## Exercise G

Two chunks each contain a copy of the same dependency.

What graph conditions might explain this?

---

# 174. Mastery Exercises

## Exercise 1 — Build Graph

Draw the module/asset graph for a real application.

Label:

```text
entry
static
dynamic
asset
external
```

---

## Exercise 2 — Tree-Shaking Laboratory

Create modules with:

```text
pure exports
side effects
CommonJS
dynamic exports
```

Measure elimination.

---

## Exercise 3 — Chunking Laboratory

Compare:

```text
one bundle
route split
vendor split
feature split
```

Measure:

```text
initial bytes
lazy bytes
requests
cacheability
```

---

## Exercise 4 — Tool Comparison

Build the same small application with:

```text
webpack
Rollup
esbuild
Vite
```

Compare:

```text
build time
output size
chunking
configuration
source maps
```

---

## Exercise 5 — Server Bundle

Bundle a small Node API.

Then test:

```text
filesystem
native addon
dynamic import
worker
module-relative paths
```

Document what works and what breaks.

---

## Exercise 6 — Library Bundle

Build a library with:

```text
ESM
types
peer dependency
sideEffects metadata
```

Test from a clean consumer project.

---

# 175. Principal Build Decision

Before introducing a bundler, ask:

```text
What problem are we solving?
What target runtime do we have?
Do we need code splitting?
Do we need asset processing?
Do we need legacy compatibility?
Do we need single-file deployment?
Do we need tree shaking?
Can native runtime modules handle the graph?
What debugging costs will bundling introduce?
What plugins will we trust?
```

If the answer is:

```text
we need none of these
```

do not add a bundler out of habit.

---

# 176. Build System Anti-Pattern

```text
framework
+
bundler
+
transpiler
+
5 plugins
+
3 alias systems
+
2 environment systems
+
custom post-build script
+
custom deploy rewrite
```

when the project only requires:

```text
modern Node + ESM
```

Tooling complexity itself becomes an operational liability.

---

# 177. Minimal Tooling Principle

Prefer:

```text
few tools
clear ownership
explicit boundaries
```

over:

```text
maximum configurability
```

The ideal build system is not the most powerful.

It is the least complex system that satisfies the real requirements.

---

# 178. Build-System Governance

For a large organization:

```text
Approved bundlers
Approved plugins
Supported targets
Build config standards
Artifact validation
Bundle budgets
Security review
Upgrade policy
```

This reduces ecosystem fragmentation.

---

# 179. Build Upgrade Strategy

When upgrading a bundler:

```text
1. lock versions
2. run baseline build
3. compare graph/output
4. compare performance
5. run runtime tests
6. inspect chunk changes
7. inspect source maps
8. deploy canary
```

Do not upgrade build infrastructure solely because a new version exists.

---

# 180. Production Artifact Ownership

Assign ownership of:

```text
application build
library build
shared build presets
plugins
CI
artifact repository
```

Build systems without clear ownership become fragile.

---

# 181. Chapter Connections

## Depends On

- Chapter 41 — Spec Architecture
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability
- Chapter 66 — `package.json` and Module Resolution
- Chapter 67 — Dependency Management and Supply Chain
- Chapter 68 — Transpilation and Compilation

## Builds Toward

- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 80 — Library Authoring
- Chapter 85 — Performance
- Chapter 94 — Compatibility Engineering
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 101 — Real-World Production Scenarios
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-Scale JavaScript Platform

## Related Concepts

- ASTs
- module graphs
- dependency resolution
- tree shaking
- dead-code elimination
- code splitting
- chunking
- minification
- source maps
- caching
- plugins
- asset pipelines
- CI/CD

## Why This Chapter Matters Later

A production application is not only source code.

It is:

```text
source
→ dependency graph
→ transformed graph
→ optimized graph
→ artifacts
→ runtime
```

Build systems determine what code actually reaches users and servers.

Therefore build engineering is part of runtime engineering.

---

# 182. Spaced Retrieval Plan

## Day 0

Explain:

```text
module graph
build graph
tree shaking
chunk
```

without notes.

## Day 2

Build one application with a dynamic import.

## Day 7

Diagnose an oversized bundle.

## Day 14

Compare two bundling strategies with measured data.

## Day 30

Defend:

> Why should a Node backend sometimes avoid bundling entirely?

---

# 183. Dependency Graph

```text
source
  │
  ▼
resolution
  │
  ▼
module graph
  │
  ├──────────────┐
  ▼              ▼
transforms      assets
  │              │
  └──────┬───────┘
         ▼
      build graph
         │
   ┌─────┼────────────┐
   ▼     ▼            ▼
tree   chunking     minify
shake
   │     │            │
   └─────┼────────────┘
         ▼
      artifacts
         │
         ▼
       runtime
```

---

# 184. Completion Criteria

```text
[ ] Define bundler
[ ] Define build system
[ ] Explain module graph
[ ] Explain build graph
[ ] Explain entry points
[ ] Explain chunks
[ ] Explain code splitting
[ ] Explain dynamic-import splitting
[ ] Explain tree shaking
[ ] Explain dead-code elimination
[ ] Explain side effects
[ ] Explain sideEffects metadata
[ ] Explain pure annotations
[ ] Explain constant folding
[ ] Explain environment replacement
[ ] Explain minification
[ ] Explain compression distinction
[ ] Explain asset hashing
[ ] Explain stale chunk deployment
[ ] Explain externalization
[ ] Explain library bundling
[ ] Explain application bundling
[ ] Explain webpack conceptually
[ ] Explain Rollup conceptually
[ ] Explain esbuild conceptually
[ ] Explain Vite development model
[ ] Explain Vite production model
[ ] Explain Rolldown's current Vite role
[ ] Explain plugins
[ ] Explain loaders
[ ] Explain virtual modules
[ ] Explain build caching
[ ] Explain incremental builds
[ ] Explain watch mode
[ ] Explain HMR
[ ] Explain build reproducibility
[ ] Explain artifact promotion
[ ] Explain server bundling
[ ] Explain browser bundling
[ ] Explain SSR build boundaries
[ ] Explain worker bundling
[ ] Explain dynamic require limitations
[ ] Explain bundler security
[ ] Debug tree shaking
[ ] Debug chunk size
[ ] Debug dynamic imports
[ ] Debug dependency duplication
[ ] Build a production pipeline
[ ] Pass principal interview questions
```

---

# 185. Mastery Gate

You have mastered this chapter only when you can:

### Understand

Explain a bundler as a graph transformation system.

### Explain

Teach tree shaking, chunking, externalization, and artifact generation without reducing them to file concatenation.

### Predict

Predict what source modules and assets should appear in output artifacts.

### Implement

Build applications and libraries with intentional bundle boundaries.

### Debug

Find why code is retained, duplicated, split incorrectly, or missing.

### Apply

Choose and configure build systems based on actual runtime/deployment requirements.

### Compare

Defend webpack, Rollup, esbuild, Vite, or native runtime execution for a specific project.

### Defend

Explain build architecture to a principal review including performance, security, maintainability, and operational cost.

---

# 186. Status

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

# 187. Chapter 69 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain module graph | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain tree shaking | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design code splitting | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Compare bundlers | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Debug large bundle | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal build review | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. Why do bundlers exist?
2. What is a build graph?
3. How does tree shaking work?
4. Why do side effects matter?
5. How does dynamic import create a chunk boundary?
6. Why can too many chunks hurt?
7. Why do libraries often externalize peer dependencies?
8. Why can bundling change runtime assumptions?
9. Why can Vite's production architecture differ from its dev architecture?
10. When should a Node service avoid bundling?
```

---

# 188. Chapter 69 — Canonical References and Source Discipline

## Primary Node.js references

- Node.js Packages  
  https://nodejs.org/api/packages.html
- Node.js ECMAScript Modules  
  https://nodejs.org/api/esm.html

Use these for runtime module semantics that build tools must preserve.

## Primary Rollup reference

- Rollup  
  https://rollupjs.org/

Rollup documents:

- module bundling,
- tree shaking,
- code splitting,
- plugin architecture,
- multiple output formats. citeturn822673search3

## Primary Vite references

- Vite  
  https://vite.dev/
- Vite Getting Started  
  https://vite.dev/guide/
- Vite Production Build  
  https://vite.dev/guide/build
- Vite CLI  
  https://vite.dev/guide/cli

Current Vite documentation states that:

- development uses a dev server with native ESM-oriented serving and dependency pre-bundling,
- production `vite build` generates optimized assets,
- the current production build is powered by Rolldown,
- Vite supports advanced tree shaking,
- chunking can be configured through Rolldown build options,
- library mode supports externalizing dependencies. citeturn822673search0turn822673search1turn822673search2

## Primary esbuild reference

- esbuild  
  https://esbuild.github.io/

Use esbuild documentation for its exact bundling, transformation, target, splitting, plugin, and minification behavior.

## Primary webpack reference

- webpack  
  https://webpack.js.org/

Use webpack documentation for:

- entry/output,
- loaders,
- plugins,
- code splitting,
- optimization,
- caching.

---

# 189. Source Discipline

1. Distinguish bundling from transpilation.
2. Distinguish build-time behavior from runtime behavior.
3. Distinguish Node module resolution from bundler resolution.
4. Distinguish development-server behavior from production-build behavior.
5. Verify current tool behavior against the tool's current documentation.
6. Do not generalize one bundler's optimization behavior to another.
7. Treat plugin behavior as tool-specific.
8. Treat package `sideEffects` metadata as an optimization contract.
9. Test output artifacts directly.
10. Validate build behavior from clean installations.
11. Treat source maps as debugging infrastructure, not security boundaries.
12. Treat build plugins as supply-chain dependencies.
13. Measure bundle performance using real artifacts, not theoretical file counts.
14. Record tool versions for reproducibility.

---

# 190. Chapter 69 — Completion Snapshot

## Graph Theory

```text
[ ] module graph
[ ] build graph
[ ] entry points
[ ] static edges
[ ] dynamic edges
[ ] assets
```

## Optimization

```text
[ ] tree shaking
[ ] dead-code elimination
[ ] side-effect analysis
[ ] constant folding
[ ] chunking
[ ] code splitting
[ ] minification
[ ] hashing
```

## Tooling

```text
[ ] webpack
[ ] Rollup
[ ] esbuild
[ ] Vite
[ ] Rolldown
[ ] plugins
[ ] loaders
[ ] virtual modules
```

## Production

```text
[ ] externalization
[ ] library builds
[ ] Node builds
[ ] browser builds
[ ] SSR
[ ] workers
[ ] asset URLs
[ ] stale chunk strategy
[ ] artifact validation
[ ] security review
```

## Performance

```text
[ ] build time
[ ] incremental time
[ ] bundle size
[ ] parse cost
[ ] execution cost
[ ] chunk cost
[ ] cacheability
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

A bundler is not a file compressor.

It is a graph compiler.

The real pipeline is:

```text
source
   ↓
resolve
   ↓
construct graph
   ↓
transform
   ↓
analyze reachability
   ↓
remove what is provably unnecessary
   ↓
split what should load later
   ↓
generate optimized artifacts
   ↓
serve/deploy artifacts
```

A principal engineer should therefore ask:

```text
What is the actual runtime target?
What code is reachable from each entry?
What can be removed safely?
What must remain because of side effects?
Where should lazy boundaries exist?
Which dependencies should be external?
Could bundling duplicate runtime identity?
Could environment substitution leak secrets?
Can a plugin compromise CI?
Are build outputs reproducible?
How does deployment handle stale chunks?
Can the emitted artifact be debugged?
```

The deepest lesson is:

> **Build tooling is part of the program.**

It determines what code exists at runtime, what code does not exist, how assets are loaded, how dependencies are packaged, and what the production artifact actually contains.

Once you understand bundlers as graph transformations rather than configuration files, modern JavaScript build systems become much easier to reason about.