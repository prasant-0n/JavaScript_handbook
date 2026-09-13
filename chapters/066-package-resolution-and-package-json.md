\
# Chapter 66 — `package.json` and Module Resolution

> **Curriculum position:** Part XII — Modules / Tooling  
> **Previous chapter:** Chapter 65 — CommonJS and Interoperability  
> **Next chapter:** Chapter 67 — Dependency Management and Supply Chain  
> **Primary environment:** Modern Node.js; current Node.js 26.x documentation is the reference for Node-specific behavior.

---

# Chapter Mission

Master **Node.js package metadata and module resolution** as a runtime architecture problem.

A line such as:

```js
import fastify from 'fastify';
```

looks simple.

Underneath it, Node must determine:

```text
What module system is this?
      ↓
What package scope applies?
      ↓
What does "fastify" mean here?
      ↓
Where is the package?
      ↓
Does package.json define "exports"?
      ↓
Which condition applies?
      ↓
Which target is allowed?
      ↓
What file format is it?
      ↓
How should it be loaded?
      ↓
What module identity results?
```

This chapter turns that hidden process into a system you can reason about.

You will master:

- `package.json` as Node package metadata,
- package scope,
- `"name"`,
- `"version"`,
- `"type"`,
- `"main"`,
- `"exports"`,
- `"imports"`,
- conditional exports,
- environment conditions,
- custom conditions,
- subpath exports,
- subpath patterns,
- package self-reference,
- package imports,
- `node_modules` traversal,
- CommonJS resolution,
- ESM resolution,
- file and directory resolution,
- module format determination,
- resolution errors,
- encapsulation,
- monorepos,
- symlinks,
- workspaces,
- dual packages,
- dependency boundaries,
- and resolution debugging.

The principal-level goal is:

> **Be able to predict what Node will load before running the application, and design package metadata that makes those decisions explicit, safe, and maintainable.**

---

# 1. Learning Objectives

By the end of this chapter you should be able to:

## Package fundamentals

- Explain what `package.json` is for.
- Distinguish Node-consumed fields from package-manager and tool-specific fields.
- Explain package scope.
- Explain `"name"`.
- Explain `"version"` conceptually.
- Explain `"type"`.
- Explain `"main"`.
- Explain `"exports"`.
- Explain `"imports"`.

## Resolution

- Explain relative specifiers.
- Explain absolute specifiers.
- Explain bare specifiers.
- Explain package self-reference.
- Explain `node_modules` traversal.
- Explain CommonJS resolution.
- Explain ESM resolution.
- Explain URL-based ESM resolution.
- Explain extension searching differences.
- Explain folder-main differences.
- Explain package entry point selection.
- Explain package subpath resolution.

## Exports

- Define the purpose of `"exports"`.
- Create a single package entry point.
- Create subpath exports.
- Create conditional exports.
- Use environment conditions.
- Use `"default"`.
- Use patterns.
- Explain condition ordering.
- Explain the encapsulation effects of `"exports"`.
- Explain breaking changes caused by introducing `"exports"`.

## Imports

- Define package-internal `"#"` imports.
- Create internal aliases.
- Use conditional package imports.
- Explain why `"imports"` differs from `"exports"`.

## Production engineering

- Design a package boundary.
- Debug `ERR_PACKAGE_PATH_NOT_EXPORTED`.
- Debug module-not-found failures.
- Debug wrong module format errors.
- Design a monorepo package layout.
- Prevent accidental deep imports.
- Design package self-reference.
- Review package metadata in code review.
- Build CI tests for resolution behavior.

## Principal judgment

- Decide whether `"main"` is sufficient.
- Decide when `"exports"` is required.
- Decide whether to expose subpaths.
- Decide how much conditional resolution is appropriate.
- Decide whether aliases belong in `"imports"` or tooling config.
- Design package metadata that remains compatible across Node versions and environments.

---

# 2. Prerequisites

Strongly recommended:

- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — Abstract Operations
- Chapter 58 — Node.js Architecture
- Chapter 59 — Node Core APIs
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability

Especially important:

> **Resolution decides identity before evaluation happens.**

---

# 3. What Is `package.json`?

`package.json` is a JSON metadata file used by package ecosystems and recognized by Node.js for several runtime decisions.

Node's current packages documentation describes Node-specific fields including:

```text
name
main
type
exports
imports
```

Other fields are primarily consumed by npm and other tools rather than directly interpreted by Node.

Source:

https://nodejs.org/api/packages.html

---

# 4. Why Does `package.json` Matter to the Runtime?

It can influence:

```text
module classification
package entry points
public package boundaries
internal aliases
conditional resolution
self-reference
```

Example:

```json
{
  "name": "payments",
  "type": "module",
  "exports": {
    ".": "./src/index.js"
  }
}
```

This tells Node and the package ecosystem:

```text
package name = payments
.js = ESM in this package scope
public root = ./src/index.js
```

That is architecture encoded as metadata.

---

# 5. Package Scope

A package scope is the directory region governed by a `package.json`.

For module classification, Node examines the nearest relevant package metadata.

Consider:

```text
repo/
├── package.json
├── src/
│   ├── app.js
│   └── legacy/
│       ├── package.json
│       └── old.js
```

`app.js` and `old.js` may have different module-system interpretation depending on their nearest package scope.

---

# 6. The `type` Field

Example:

```json
{
  "type": "module"
}
```

Then `.js` files under the package scope are treated as ESM by default.

Example:

```json
{
  "type": "commonjs"
}
```

Then `.js` files under the package scope are CommonJS by default.

Explicit overrides:

```text
.mjs → ESM
.cjs → CommonJS
```

Node's current documentation describes these rules. citeturn570896search0turn570896search1

---

# 7. Package Type Is a Classification Rule

Think:

```text
file extension
+
nearest package metadata
+
entry/loading context
=
module format
```

Do not think:

```text
.js always means JavaScript CommonJS
```

That rule is obsolete for modern Node.

---

# 8. `"name"`

Example:

```json
{
  "name": "my-library"
}
```

The package name is used for:

- package manager identity,
- bare specifiers,
- package self-reference in supported contexts,
- human-readable dependency identity.

Example:

```js
import { createClient } from 'my-library';
```

---

# 9. Package Self-Reference

A package can reference itself by its own package name when `"exports"` permits the target.

Example:

```json
{
  "name": "@acme/payments",
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js"
  }
}
```

Inside the package:

```js
import { PaymentError } from '@acme/payments/errors';
```

This can provide stable package-level paths rather than deep relative paths.

---

# 10. Why Self-Reference Is Valuable

Instead of:

```js
import { PaymentError } from '../../errors.js';
```

you can use:

```js
import { PaymentError } from '@acme/payments/errors';
```

Now the code depends on:

```text
package contract
```

rather than:

```text
physical directory depth
```

This can improve refactoring safety.

---

# 11. `"main"`

Traditional package metadata:

```json
{
  "main": "./index.js"
}
```

Historically this defines the package's main entry point.

It remains supported.

But it is limited:

```text
one main entry
```

whereas `"exports"` can define:

```text
multiple entry points
conditional entries
encapsulated subpaths
```

Node's package documentation currently recommends `"exports"` for new packages targeting supported Node versions. citeturn570896search0

---

# 12. `"main"` vs `"exports"`

| Capability | `main` | `exports` |
|---|---:|---:|
| Root package entry | ✅ | ✅ |
| Multiple subpaths | ❌ | ✅ |
| Conditional resolution | ❌ | ✅ |
| Public API encapsulation | ❌ | ✅ |
| Environment-specific targets | ❌ | ✅ |
| Modern package boundary | limited | strong |
| Legacy compatibility | ✅ | ✅ |

Important:

> If both are present, supported modern Node versions prioritize `"exports"`.

---

# 13. Introducing `"exports"` Can Be Breaking

Suppose an old package allowed:

```js
import 'pkg'
import 'pkg/lib/parser.js'
import 'pkg/lib/token.js'
```

Then you add:

```json
{
  "exports": {
    ".": "./index.js"
  }
}
```

Now:

```text
pkg
  ✅
pkg/lib/parser.js
  ❌
pkg/lib/token.js
  ❌
```

This can be a breaking change.

Node explicitly warns that adding `"exports"` can prevent previously accessible entry points from being imported. citeturn570896search0

---

# 14. Package Encapsulation

With:

```json
{
  "exports": {
    ".": "./index.js"
  }
}
```

the supported public contract is:

```text
pkg
```

not:

```text
pkg/anything-that-happens-to-exist
```

This is one of the most important package-architecture improvements in modern Node.

---

# 15. Encapsulation Is Not Filesystem Security

Package export encapsulation is a module-resolution contract.

It is not a filesystem access-control system.

Node's documentation notes that direct absolute filesystem loading can bypass the package export map. citeturn570896search0

Therefore:

```text
exports map
≠
security sandbox
```

---

# 16. Subpath Exports

Example:

```json
{
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js",
    "./client": "./src/client.js"
  }
}
```

Consumers:

```js
import { createClient } from 'pkg/client';
import { PackageError } from 'pkg/errors';
```

This is better than exposing:

```text
./src/*
```

because the public API remains deliberate.

---

# 17. Explicit Subpath Design

Prefer:

```json
{
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js"
  }
}
```

over:

```json
{
  "exports": "./src/*.js"
}
```

unless a broad pattern is genuinely needed.

Explicit APIs are easier to document and version.

---

# 18. Subpath Patterns

Node supports export patterns.

Conceptually:

```json
{
  "exports": {
    "./features/*.js": "./src/features/*.js"
  }
}
```

This allows structured exposure.

But broad patterns can become accidental APIs.

Treat every pattern as a contract.

---

# 19. Wildcards Are String Replacement

A pattern such as:

```text
"./features/*.js"
```

should be understood as a package-map pattern, not as arbitrary filesystem globbing.

The resolver substitutes the matching subpath into the target according to package-map rules.

Read the Node packages documentation for the exact current pattern semantics.

---

# 20. Conditional Exports

Example:

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

This lets Node choose different targets depending on how the package is loaded.

Common conditions include:

```text
import
require
node
default
```

Current Node documentation defines condition matching and ordering behavior. citeturn570896search0turn570896search2

---

# 21. Conditions Are Ordered

Example:

```json
{
  "exports": {
    ".": {
      "node": "./node.js",
      "default": "./generic.js"
    }
  }
}
```

The resolver considers condition keys in object order according to the package resolution rules.

Order can therefore change which target wins.

---

# 22. Condition Design

Common conditions:

```text
node
browser
import
require
development
production
default
```

Do not invent many project-specific conditions without documenting exactly which runtime passes them.

A larger condition matrix means:

```text
more combinations
→ more tests
→ more operational complexity
```

---

# 23. `"default"`

A useful fallback:

```json
{
  "exports": {
    ".": {
      "node": "./node.js",
      "default": "./generic.js"
    }
  }
}
```

The `"default"` condition is generally the final fallback target.

Use it intentionally.

---

# 24. `node` Condition

Example:

```json
{
  "exports": {
    ".": {
      "node": "./node.js",
      "default": "./browser.js"
    }
  }
}
```

This is useful when a package truly has environment-specific implementations.

Do not use conditional exports merely to hide poor architecture.

---

# 25. `import` vs `require`

Example:

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

Conceptually:

```text
ESM import → index.js
CJS require → index.cjs
```

This is a major pattern for dual-package support.

---

# 26. Conditional Export Ambiguity

Bad:

```json
{
  "exports": {
    ".": {
      "browser": "./browser.js",
      "node": "./node.js",
      "import": "./esm.js",
      "require": "./cjs.cjs",
      "default": "./fallback.js"
    }
  }
}
```

This creates many possible paths.

You must decide:

```text
What wins if Node uses ESM?
What wins if a bundler sets browser?
What wins under custom conditions?
```

Do not add conditions without a test matrix.

---

# 27. `"imports"`

The `"imports"` field defines package-internal mappings.

Example:

```json
{
  "imports": {
    "#config": "./src/config.js",
    "#db": "./src/db/index.js"
  }
}
```

Then inside the package:

```js
import config from '#config';
```

The `#` prefix distinguishes these internal package specifiers.

Node's current package docs require package import keys to start with `#` and describe conditional mappings. citeturn570896search0

---

# 28. `"imports"` vs `"exports"`

| Field | Scope |
|---|---|
| `exports` | public package entry points |
| `imports` | internal package mappings |

Think:

```text
exports
  = outside → package

imports
  = inside package → internal/external target
```

---

# 29. `"imports"` Can Target External Packages

Unlike `"exports"`, package imports can map to external packages.

Example:

```json
{
  "imports": {
    "#crypto": {
      "node": "node:crypto",
      "default": "./polyfill.js"
    }
  }
}
```

This can let internal code use:

```js
import crypto from '#crypto';
```

without exposing the alias publicly.

---

# 30. Package Import Example

```text
package.json
src/
  app.js
  config.js
  infrastructure/
    db.js
```

Metadata:

```json
{
  "imports": {
    "#config": "./src/config.js",
    "#db": "./src/infrastructure/db.js"
  }
}
```

Now:

```js
import config from '#config';
import db from '#db';
```

This can be easier to refactor than:

```js
../../../config.js
```

---

# 31. Important `"imports"` Rule

`imports` is an internal package mechanism.

It should not be treated as:

```text
public npm alias
```

External consumers do not normally use your `#config` path as a package API.

---

# 32. Package Resolution Layers

When resolving a specifier, think in layers.

```text
specifier
  │
  ├── relative?
  │
  ├── absolute?
  │
  ├── #imports?
  │
  └── bare package?
          │
          ▼
      package resolution
```

Then:

```text
package metadata
      ↓
exports / main / legacy rules
      ↓
target
      ↓
format
      ↓
load
```

---

# 33. Relative Specifiers

Examples:

```js
import './utils.js';
import '../config.js';
```

These resolve relative to the importing module's URL/location.

No package-name traversal is required.

---

# 34. Absolute ESM Specifiers

Example:

```js
import 'file:///opt/app/config.js';
```

Node ESM uses URL semantics for absolute URL specifiers.

Filesystem path handling and URL handling are not identical.

---

# 35. Bare Specifiers

Examples:

```js
import express from 'express';
import { parse } from 'some-package/parser';
```

These require package resolution.

Node searches package locations and then applies package metadata.

---

# 36. Package Search

A simplified mental model:

```text
/app/src/node_modules/pkg
/app/node_modules/pkg
/node_modules/pkg
```

depending on the directory and environment.

Actual package-manager layout can vary, especially in monorepos.

Never assume that physical installation layout exactly matches a traditional `node_modules` tree.

---

# 37. `node_modules` and Package Managers

Modern package managers can implement:

- hoisting,
- isolated dependency trees,
- symlinks,
- virtual stores,
- workspaces.

Node still resolves according to the resulting runtime-visible structure and package metadata.

This is why a package manager's installation strategy and Node's resolver must be understood together.

---

# 38. Package Self-Reference Resolution

When a package uses its own `"name"` as a bare specifier, Node can use the package's own `"exports"` mapping.

Conceptually:

```text
package source
   ↓
import "my-package"
   ↓
self-reference
   ↓
this package.json
   ↓
exports map
```

This provides stable internal package-level APIs.

---

# 39. Why Self-Reference Beats Deep Relative Paths

Deep relative:

```js
import { validate } from '../../../../shared/validate.js';
```

Problems:

- difficult refactoring,
- directory-structure coupling,
- unclear package ownership,
- fragile monorepo movement.

Self-reference:

```js
import { validate } from '@acme/core/validate';
```

Communicates package intent.

---

# 40. Resolution Is Not Loading

Important distinction:

```text
resolution
  = determine target

loading
  = obtain source/module

evaluation
  = execute code
```

A module can resolve correctly but fail to load.

It can load correctly but fail during evaluation.

Do not collapse all errors into:

```text
module not found
```

---

# 41. Resolution Is Not Evaluation

Suppose:

```js
import './broken.js';
```

and `broken.js` contains:

```js
throw new Error('boom');
```

Resolution succeeded.

Loading succeeded.

Evaluation failed.

Your debugging path should reflect this distinction.

---

# 42. Module Format Determination

Once a target is resolved, Node determines whether it is:

```text
ESM
CommonJS
JSON
native addon
```

according to the relevant loader rules.

For `.js`, package `"type"` can be decisive.

---

# 43. CommonJS Resolution vs ESM Resolution

### CommonJS

Historically supports:

```text
extension searching
folder mains
package main
```

### ESM

Uses:

```text
URL-based resolution
explicit relative file extensions
no default directory index
package exports
```

Node's current ESM documentation explicitly lists “no default extensions” and “no folder mains” in the default ESM resolver. citeturn570896search1

---

# 44. Example Difference

CommonJS:

```js
require('./utils');
```

may resolve:

```text
./utils.js
```

ESM:

```js
import './utils';
```

does not normally perform the same extension search.

Use:

```js
import './utils.js';
```

---

# 45. Directory Difference

CommonJS can historically resolve:

```js
require('./routes');
```

through directory/package/index rules.

ESM:

```js
import './routes';
```

does not implicitly resolve to:

```text
routes/index.js
```

under the default resolver. citeturn570896search1

---

# 46. `"exports"` Overrides Legacy Convenience

Without `"exports"`:

```text
package
  ↓
legacy package entry rules
```

With `"exports"`:

```text
package
  ↓
explicit export map
```

This is why introducing `"exports"` can change runtime behavior even when the filesystem is unchanged.

---

# 47. `"exports"` and CommonJS

`"exports"` affects both:

```js
require('pkg')
```

and:

```js
import 'pkg'
```

The active conditions differ according to the loader.

Node's CommonJS documentation shows that CommonJS package resolution can invoke the package export resolver with `require`-relevant conditions. citeturn570896search2

---

# 48. `"exports"` and ESM

For ESM:

```text
bare specifier
   ↓
package resolution
   ↓
exports map
   ↓
target
```

If the subpath is not exported, Node can report:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED
```

---

# 49. `ERR_PACKAGE_PATH_NOT_EXPORTED`

Example:

```js
import internal from 'pkg/internal.js';
```

while:

```json
{
  "exports": {
    ".": "./index.js"
  }
}
```

exists.

The physical file might exist.

But the package contract says:

```text
internal.js is not public
```

The error is therefore a package-boundary failure, not necessarily a missing-file failure.

---

# 50. `ERR_PACKAGE_IMPORT_NOT_DEFINED`

If code uses:

```js
import '#missing';
```

without:

```json
{
  "imports": {
    "#missing": "..."
  }
}
```

Node can reject it because the package import was not defined.

This is a different failure class from missing external packages.

---

# 51. `ERR_MODULE_NOT_FOUND`

This generally indicates that the requested module/package target could not be resolved or found.

Debug:

```text
specifier
package location
package exports
file existence
module format
```

Do not immediately reinstall dependencies.

---

# 52. Invalid Package Configuration

Examples include:

```text
invalid exports shape
invalid imports shape
invalid package metadata
inconsistent package map keys
```

Treat these as configuration contract errors.

---

# 53. Package Conditions and Tooling

Bundlers may pass or interpret conditions such as:

```text
browser
development
production
```

Node itself has a defined set of conditions and supports user conditions through specific runtime mechanisms.

Do not assume:

```text
Node's condition set
=
bundler's condition set
```

---

# 54. Custom Conditions

Node can be launched with user-defined conditions through the appropriate CLI options.

Example concept:

```bash
node --conditions=development app.js
```

Then a package can have:

```json
{
  "exports": {
    ".": {
      "development": "./dev.js",
      "default": "./prod.js"
    }
  }
}
```

Use custom conditions sparingly.

Every condition adds another resolution dimension.

---

# 55. Condition Matrix Explosion

Suppose a package supports:

```text
node
browser
development
production
import
require
```

Theoretical combinations become numerous.

Do not think:

```text
6 conditions = 6 tests
```

A single environment may activate multiple conditions in ordered resolution.

Design a small explicit matrix.

---

# 56. `"exports"` Arrays

Node package maps can use arrays in supported forms for fallback targets.

Conceptually:

```json
{
  "exports": {
    ".": [
      "./preferred.js",
      "./fallback.js"
    ]
  }
}
```

This should be used only when there is a genuine ordered fallback strategy.

Do not use arrays as an excuse to make resolution nondeterministic.

---

# 57. Null Targets

A package export can explicitly block a path with:

```json
{
  "exports": {
    "./internal/*": null
  }
}
```

This can be useful for:

- explicitly excluded areas,
- sealing legacy entry points,
- controlling pattern exposure.

---

# 58. Package Boundary as API Governance

A mature package should answer:

```text
What can consumers import?
What can internal modules import?
Which files are supported?
Which paths are experimental?
What can change without semver breakage?
```

`exports` makes these decisions machine-readable.

---

# 59. Public API vs Physical Files

Never equate:

```text
file exists
```

with:

```text
file is public
```

With modern package maps:

```text
physical implementation
      ≠
public contract
```

That distinction is fundamental to library authoring.

---

# 60. `"main"` and Legacy Consumers

A package may retain:

```json
{
  "main": "./index.js",
  "exports": {
    ".": "./index.js"
  }
}
```

for compatibility with older tooling while using `"exports"` as the modern contract.

The exact compatibility requirements depend on your minimum supported runtime/toolchain.

---

# 61. Package Version Is Not a Resolver Directive

This is a common misconception:

```json
{
  "version": "3.2.0"
}
```

does not tell Node:

```text
load version 3.2.0
```

The package manager installs the version.

Node resolves whatever package tree is actually present.

Keep responsibilities distinct:

```text
package manager
  = dependency selection/install

Node
  = runtime resolution/loading
```

---

# 62. Dependencies vs DevDependencies

Node's runtime resolver sees the installed filesystem.

The semantic distinction:

```json
"dependencies": {}
"devDependencies": {}
```

primarily informs package management/deployment.

Do not assume that declaring a dependency in `devDependencies` makes Node refuse to load it.

The runtime sees what is installed and reachable.

---

# 63. Peer Dependencies and Resolution

Peer dependency behavior is largely coordinated by package managers.

But its runtime consequence matters:

```text
which physical package instance
```

is available at the resolved path.

This can affect:

- singleton state,
- version identity,
- `instanceof`,
- plugin ecosystems.

---

# 64. Monorepos

Example:

```text
repo/
├── packages/
│   ├── api/
│   ├── db/
│   └── shared/
└── package.json
```

Each package may have its own:

```text
name
type
exports
imports
```

and package boundary.

Do not treat a monorepo as one giant package unless that is intentionally the architecture.

---

# 65. Workspace Resolution

Workspace tooling may link packages together.

From Node's perspective:

```text
import '@acme/shared'
```

still needs to resolve to a runtime-visible package location.

The package manager determines installation/linking layout; Node applies package resolution rules to what is available.

---

# 66. Symlinks

Symlinks can influence module identity and package resolution.

Potential consequence:

```text
same logical package
    ↓
different resolved paths
    ↓
different module instances
```

This matters for:

- singleton modules,
- class constructors,
- registries,
- caches.

Be explicit about symlink-related runtime flags and tooling.

---

# 67. Realpath and Identity

Node's loaders have documented behaviors involving filesystem paths and real paths.

The exact identity model differs across CommonJS and ESM.

When debugging duplicate packages:

```text
do not compare package names only
```

compare:

```text
resolved module path / URL
```

and the loader involved.

---

# 68. `require.resolve()`

CommonJS:

```js
const resolved = require.resolve('pkg');
console.log(resolved);
```

Useful for discovering:

```text
what file/package CommonJS would load
```

It does not execute the target module itself.

---

# 69. `import.meta.resolve()`

ESM:

```js
console.log(import.meta.resolve('pkg'));
```

Useful for asking:

```text
what URL would ESM resolution produce here?
```

The two APIs belong to different resolution systems.

---

# 70. Compare the Two

| Concern | CJS | ESM |
|---|---|---|
| Resolve package | `require.resolve()` | `import.meta.resolve()` |
| Main loader | `require()` | `import` / `import()` |
| Extension search | historically yes | no default |
| Folder main | historically yes | no default |
| URL-based | not primary model | primary |
| Package exports | supported | supported |
| Package imports | supported in modern Node contexts | supported |
| Error surface | CommonJS-specific errors | ESM-specific errors |

---

# 71. Relative Path Pitfall

Consider:

```js
// src/a.js
import './b.js';
```

Resolution is:

```text
a.js location
   ↓
./b.js
```

not:

```text
process.cwd()
   ↓
./b.js
```

This is why module-relative resource loading differs from working-directory-relative resource loading.

---

# 72. Package Root Pitfall

Do not assume:

```js
import 'pkg';
```

means:

```text
node_modules/pkg/index.js
```

Modern packages may define:

```json
{
  "exports": {
    ".": "./dist/server.js"
  }
}
```

The package contract decides.

---

# 73. Self-Reference Pitfall

This:

```js
import '@acme/pkg/internal.js';
```

may fail because `"exports"` does not expose it.

Self-reference is still constrained by the package's public export map.

---

# 74. `"imports"` Pitfall

This:

```js
import '#config';
```

works only if the current package scope defines that mapping.

It is not globally available.

---

# 75. Nested Package Scope Pitfall

Suppose:

```text
repo/
  package.json type=module
  legacy/
    package.json type=commonjs
    plugin.js
```

Then:

```text
repo/app.js
  → ESM

repo/legacy/plugin.js
  → CJS
```

A refactor that moves `plugin.js` can silently change its module interpretation if the package scope changes.

---

# 76. Directory Boundary as Semantic Boundary

A directory can change runtime meaning because of:

```text
package.json
```

Therefore moving files across package boundaries can be a semantic change even when source text is unchanged.

This is a crucial monorepo insight.

---

# 77. Resolution and Build Output

Suppose source:

```text
src/index.ts
```

builds to:

```text
dist/index.js
```

Your package:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

Node resolves the published artifact, not your source tree.

Therefore build and package metadata must agree.

---

# 78. Build Artifact Mismatch

Bad:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

but build only creates:

```text
dist/main.js
```

The package may install successfully but fail at runtime.

This is a packaging failure.

---

# 79. `files` Field Is Not a Node Resolver Rule

Package manifests often include:

```json
{
  "files": [
    "dist"
  ]
}
```

This influences package publishing/packing.

Node does not use `"files"` to decide what imports are public.

Public import access is controlled by runtime package resolution rules such as `"exports"`.

---

# 80. `bin`

A `package.json` can define:

```json
{
  "bin": {
    "my-cli": "./bin/cli.js"
  }
}
```

This is primarily package-manager/CLI integration.

But the resulting executable still launches a Node module whose classification depends on:

```text
extension
package type
entry point
```

Keep package-manager metadata and Node loader metadata conceptually separate.

---

# 81. `type`, `exports`, and CLI Design

A package can use:

```json
{
  "type": "module",
  "exports": {
    ".": "./src/index.js"
  },
  "bin": {
    "my-cli": "./src/cli.js"
  }
}
```

Now:

```text
library API
+
CLI entry
```

must both be tested as ESM entry points.

---

# 82. Environment-Specific Exports

Possible structure:

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

Questions:

```text
Does browser tooling honor node/default?
Does bundler condition ordering match Node?
Does the node build accidentally get bundled for browser?
```

Test actual consumers.

---

# 83. Browser Field vs Exports

Some ecosystems historically use package metadata such as:

```json
{
  "browser": {
    "./node.js": "./browser.js"
  }
}
```

This is generally bundler/tooling behavior rather than a universal Node runtime rule.

Prefer modern explicit `"exports"` conditions when your supported toolchains understand them.

---

# 84. `imports` for Polyfills

Example:

```json
{
  "imports": {
    "#storage": {
      "node": "./src/node-storage.js",
      "default": "./src/memory-storage.js"
    }
  }
}
```

Then:

```js
import storage from '#storage';
```

This can keep environment switching internal.

---

# 85. Avoid Path Alias Fragmentation

Do not simultaneously maintain:

```text
tsconfig paths
bundler aliases
Jest aliases
ESLint aliases
package imports
```

with slightly different meanings.

For Node runtime semantics, `"imports"` gives you an actual runtime-aware package mechanism.

Tooling configurations may still be needed, but they should mirror one source of truth where possible.

---

# 86. TypeScript and Runtime Resolution

TypeScript can resolve:

```text
paths
baseUrl
moduleResolution
```

in ways that differ from Node.

A source file can type-check successfully while Node cannot execute it.

This is one of the most common package-resolution failures in TypeScript projects.

Principal rule:

> **Type-checking resolution is not automatically runtime resolution.**

---

# 87. Build Tool Resolution

Bundlers may implement:

```text
extension inference
aliases
virtual modules
conditions
package maps
```

which differ from native Node.

Do not infer production Node behavior from a successful bundle build.

---

# 88. Test Runtime Resolution Directly

For packages that must run directly in Node, include tests that execute:

```bash
node ...
```

rather than testing only through the bundler or transpiler.

---

# 89. Resolution Contract Tests

Useful automated checks:

```text
[ ] package root resolves
[ ] public subpaths resolve
[ ] private subpaths fail
[ ] CJS condition resolves correctly
[ ] ESM condition resolves correctly
[ ] Node condition resolves correctly
[ ] default fallback resolves correctly
[ ] internal #imports resolve
```

---

# 90. Debugging Module Resolution

Use a disciplined order.

### Step 1

Identify loader:

```text
ESM?
CJS?
bundler?
test runner?
```

### Step 2

Identify specifier:

```text
./
../
/
#
bare package
URL
```

### Step 3

Identify package scope.

### Step 4

Inspect package metadata.

### Step 5

Inspect `"exports"` / `"imports"`.

### Step 6

Inspect actual installed package location.

### Step 7

Determine selected condition.

### Step 8

Verify target exists.

### Step 9

Verify target module format.

### Step 10

Only then inspect evaluation errors.

---

# 91. Debugging `ERR_PACKAGE_PATH_NOT_EXPORTED`

Checklist:

```text
[ ] Is the subpath defined in exports?
[ ] Is the package self-reference involved?
[ ] Is a condition selecting another branch?
[ ] Is the consumer using require or import?
[ ] Was exports recently introduced?
[ ] Is the target intentionally private?
```

Do not “fix” it by removing `"exports"` without deciding whether that breaks the intended package boundary.

---

# 92. Debugging `Cannot find package`

Checklist:

```text
[ ] package installed?
[ ] package name correct?
[ ] dependency declared?
[ ] package manager layout expected?
[ ] workspace linked?
[ ] current package scope?
[ ] package exported root?
[ ] correct loader?
```

---

# 93. Debugging `Cannot find module`

This message can be misleading because it may arise from different resolution failures.

Ask:

```text
What loader emitted it?
What specifier failed?
What absolute target was expected?
Was package exports consulted?
```

Then reproduce the smallest case.

---

# 94. Debugging Wrong Entry Point

You expected:

```text
dist/index.js
```

but Node loads:

```text
dist/legacy.cjs
```

Inspect:

```json
{
  "exports": {
    ".": {
      "require": "./dist/legacy.cjs",
      "import": "./dist/index.js"
    }
  }
}
```

The consumer's loader determines the condition.

---

# 95. Debugging Wrong Format

Symptom:

```text
ReferenceError: require is not defined
```

Potential cause:

```text
Node loaded a file as ESM
```

Symptom:

```text
Cannot use import statement outside a module
```

Potential cause:

```text
Node loaded code as CommonJS
```

Inspect:

```text
type
extension
package scope
entry target
```

---

# 96. Debugging Build Artifacts

A package may contain:

```text
src/
dist/
```

but `"exports"` points to:

```text
dist/
```

Always debug the published/installed artifact, not just source.

Useful verification:

```bash
npm pack
```

then inspect the tarball.

This checks what consumers actually receive.

---

# 97. Package Publish Verification

A production package workflow should verify:

```text
source
  ↓
build
  ↓
package
  ↓
npm pack / publish artifact
  ↓
fresh install
  ↓
CJS consumer
  ↓
ESM consumer
```

This catches errors invisible inside the monorepo.

---

# 98. `npm pack` as a Boundary Test

Local source may resolve because:

```text
workspace symlink
```

or:

```text
repository-local path
```

exists.

The packed artifact may omit:

```text
dist/
exports target
metadata file
```

Test from a clean temporary directory.

---

# 99. Monorepo Resolution Trap

Inside the monorepo:

```js
import '@acme/shared';
```

works.

After publishing:

```text
consumer → @acme/shared
```

fails.

Possible cause:

```text
workspace tooling provided a path
that the published package does not expose.
```

Always test package boundaries independently.

---

# 100. Dependency Graph and Resolution

Resolution constructs a graph edge:

```text
A
 ↓
specifier
 ↓
target B
```

Package maps can transform the apparent graph:

```text
A → package public entry → implementation B
```

This is why changing `"exports"` can change architectural dependency relationships without moving source files.

---

# 101. Resolution Graph vs Filesystem Graph

Filesystem:

```text
src/
  a.js
  b.js
```

Runtime graph:

```text
public package API
   ↓
dist/index.js
   ↓
dist/internal.js
```

They do not have to match.

A package boundary intentionally decouples them.

---

# 102. Encapsulation and Semver

If:

```text
exports = public API
```

then semver should be based on:

```text
supported exported paths
```

not:

```text
every file physically present
```

This dramatically improves maintainability.

---

# 103. Public API Stability

A package should document:

```text
pkg
pkg/client
pkg/errors
```

rather than:

```text
pkg/src/client.js
pkg/dist/client.js
```

The former are contracts.

The latter are implementation details.

---

# 104. Versioning Rule

If you remove:

```json
{
  "exports": {
    "./client": "./src/client.js"
  }
}
```

you have removed a public package entry point.

That should be evaluated as an API compatibility change.

---

# 105. Conditional Export Semver

Changing:

```json
"import": "./v1.js"
```

to:

```json
"import": "./v2.js"
```

can be a behavior change even if:

```text
public path remains the same
```

The implementation behind the same package path is still part of the API contract.

---

# 106. Security Considerations

## Package boundary

Use `"exports"` to minimize accidental API exposure.

## Dynamic specifiers

Never allow arbitrary user strings to become package names or subpaths without validation.

## Dependency confusion

Package names are resolved through the package ecosystem.

Use scoped package names where appropriate:

```text
@company/internal-package
```

and ensure internal registry/package-manager policy is correct.

## Malicious package metadata

A package can execute code during module evaluation.

Resolution itself is not the final security boundary.

---

# 107. Dependency Confusion Model

Imagine:

```text
internal code:
import 'payment-client';
```

If the intended private package is not correctly scoped/published and a public package with the same name exists, package-manager configuration can become a security issue.

Prefer explicit organizational naming and registry controls.

---

# 108. Package Metadata Injection

Treat:

```text
package.json
```

as executable architecture metadata.

A malicious package can manipulate:

- entry points,
- conditions,
- installed file structure,
- lifecycle scripts through package managers.

Node runtime concerns and package-manager script execution are separate but related threat surfaces.

---

# 109. Performance Considerations

Resolution costs can include:

```text
filesystem lookups
package.json reads
exports/imports matching
directory traversal
module-format determination
```

Do not prematurely optimize.

But avoid unnecessary runtime resolution in hot loops.

---

# 110. `import.meta.resolve()` Performance

Current Node documentation warns that `import.meta.resolve()` may entail synchronous filesystem work.

Therefore:

```js
for (const item of items) {
  import.meta.resolve('./foo.js');
}
```

is not an innocent zero-cost operation.

Cache stable resolutions where appropriate.

---

# 111. Package Map Complexity

A giant `"exports"` object can increase:

```text
reasoning cost
test matrix
maintenance cost
```

Prefer the smallest public API map that satisfies real consumers.

---

# 112. Memory Considerations

Resolution itself usually has much lower memory impact than long-lived module state, but package architecture affects:

```text
duplicate module instances
```

If different conditions or package paths load separate artifacts:

```text
one logical library
   ↓
two module graphs
   ↓
duplicated state
```

This connects directly to Chapter 65.

---

# 113. Duplicate Dependency Identity

Suppose:

```text
app
 ├── package A → dependency v1
 └── package B → dependency v2
```

Node can legitimately have two copies.

If the dependency exports:

```js
class Token {}
```

then:

```text
Token from v1
≠
Token from v2
```

This can break `instanceof`, singleton assumptions, and plugin registries.

Resolution is therefore also an identity problem.

---

# 114. Production Architecture Pattern

A strong package:

```text
package.json
├── name
├── version
├── type
├── exports
└── imports
```

Example:

```json
{
  "name": "@acme/payments",
  "version": "4.0.0",
  "type": "module",
  "exports": {
    ".": "./dist/index.js",
    "./errors": "./dist/errors.js"
  },
  "imports": {
    "#config": "./dist/config.js",
    "#internal/*": "./dist/internal/*.js"
  }
}
```

This makes package architecture explicit.

---

# 115. Production Package Layout

Recommended conceptual layout:

```text
package/
├── package.json
├── README.md
├── LICENSE
├── dist/
│   ├── index.js
│   ├── errors.js
│   └── internal/
└── test/
```

Consumers see:

```text
package
package/errors
```

They should not need to know:

```text
dist/internal/*
```

---

# 116. Internal Imports Pattern

Use:

```json
{
  "imports": {
    "#domain/*": "./src/domain/*.js",
    "#infra/*": "./src/infrastructure/*.js"
  }
}
```

Then:

```js
import { createOrder } from '#domain/orders';
```

This can make internal architecture more readable.

Be careful to keep the alias system understandable.

---

# 117. Layered Package Design

For a large platform:

```text
@acme/core
@acme/domain
@acme/db
@acme/http
@acme/observability
```

Each package can expose only:

```text
approved public subpaths
```

This creates architecture through package resolution.

---

# 118. Package Boundary Enforcement

CI can reject:

```text
../../other-package/src/private.js
```

when the intended dependency is:

```text
@acme/other-package
```

Prefer package-level imports that respect `"exports"`.

This reduces physical-layout coupling.

---

# 119. Private Paths and Test Access

A common temptation:

```js
import privateModule from 'pkg/internal/private.js';
```

because a unit test wants direct access.

Better options:

- test the public contract,
- expose a documented testing entry point,
- test internal source inside the package itself.

Do not weaken package API solely for tests.

---

# 120. Test Exports Separately

Example:

```bash
node -e "import('pkg').then(m => console.log(Object.keys(m)))"
node -e "import('pkg/errors').then(m => console.log(Object.keys(m)))"
node -e "import('pkg/internal/private').catch(e => console.log(e.code))"
```

This validates the package contract directly.

---

# 121. Production Resolution Checklist

```text
[ ] name is stable
[ ] type is explicit
[ ] main/exports policy is intentional
[ ] exports is minimal
[ ] public subpaths are explicit
[ ] internal files are encapsulated
[ ] imports aliases are documented
[ ] conditions are justified
[ ] condition ordering is tested
[ ] CJS/ESM paths are tested
[ ] clean packed artifact is tested
[ ] monorepo workspace behavior is tested
[ ] Node direct-runtime behavior is tested
[ ] bundler behavior is tested where supported
```

---

# 122. Implementation From Scratch

## Stage A — Guided

Create:

```text
pkg/
  package.json
  src/
    index.js
    errors.js
    internal.js
```

Add:

```json
{
  "type": "module",
  "exports": {
    ".": "./src/index.js",
    "./errors": "./src/errors.js"
  }
}
```

Test:

```text
pkg
pkg/errors
pkg/internal
```

---

# 123. Stage B — Partially Guided

Add:

```text
imports:
  #config
  #db
```

Refactor internal imports to use them.

---

# 124. Stage C — No Reference

Design a package with:

```text
ESM root
CJS root
errors subpath
browser condition
node condition
private internal aliases
```

Write the package map yourself.

---

# 125. Stage D — Edge-Case Hardened

Add tests for:

- unsupported deep import,
- condition ordering,
- package self-reference,
- missing target,
- malformed package map,
- CJS consumer,
- ESM consumer,
- packed artifact.

---

# 126. Stage E — Production Grade

Build a package publish pipeline:

```text
build
 ↓
lint
 ↓
unit tests
 ↓
package pack
 ↓
fresh temp install
 ↓
ESM test
 ↓
CJS test
 ↓
subpath contract test
 ↓
private path rejection test
```

---

# 127. Implementation Challenge — Public API Map

Design:

```text
@acme/http
```

with public:

```text
.
./client
./errors
./testing
```

and private:

```text
./internal/*
```

Then enforce the contract.

---

# 128. Implementation Challenge — Conditions

Design:

```json
{
  "exports": {
    ".": {
      "node": {
        "import": "./dist/node-esm.js",
        "require": "./dist/node-cjs.cjs"
      },
      "default": "./dist/browser.js"
    }
  }
}
```

Then test every intended path.

---

# 129. Implementation Challenge — Self Reference

Inside:

```text
@acme/domain
```

replace:

```js
import { User } from '../domain/user.js';
```

with package-level self-reference where appropriate.

Then ensure `"exports"` makes the referenced path public.

---

# 130. Implementation Challenge — Private Imports

Create:

```json
{
  "imports": {
    "#db": {
      "node": "./src/db/node.js",
      "default": "./src/db/memory.js"
    }
  }
}
```

Then use:

```js
import db from '#db';
```

from internal code.

---

# 131. Debugging Exercises

## Exercise 1

```js
import './utils';
```

works under a bundler but fails under Node ESM.

Explain.

---

## Exercise 2

`pkg/internal.js` exists physically but:

```js
import 'pkg/internal.js';
```

fails.

Inspect `"exports"`.

---

## Exercise 3

A package has:

```json
{
  "type": "module"
}
```

but:

```text
legacy.cjs
```

still works.

Explain why.

---

## Exercise 4

A nested package contains:

```json
{
  "type": "commonjs"
}
```

inside a parent ESM package.

Explain which package type controls the nested `.js` files.

---

## Exercise 5

`require('pkg')` loads:

```text
index.cjs
```

but:

```js
import 'pkg';
```

loads:

```text
index.js
```

Explain using conditional exports.

---

## Exercise 6

A workspace package works locally but fails after `npm pack`.

Find likely causes.

---

## Exercise 7

`import '#db'` fails.

Determine whether:

```text
imports missing
wrong package scope
wrong key
```

is responsible.

---

## Exercise 8

A package resolves but crashes with an ESM/CJS format error.

Determine where resolution ends and evaluation begins.

---

# 132. Code Review Exercise

Review:

```json
{
  "name": "my-pkg",
  "main": "./src/index.js",
  "type": "module",
  "exports": {
    ".": "./src/index.js",
    "./*": "./src/*.js"
  },
  "imports": {
    "config": "./src/config.js"
  }
}
```

Find at least 12 issues/questions.

Expected areas:

- `main` policy,
- broad pattern,
- private files,
- missing `#` in imports,
- source-vs-dist packaging,
- semver exposure,
- package API contract,
- build output,
- condition strategy,
- publish contents,
- tests.

---

# 133. Interview Questions

## Foundation

1. What is `package.json`?
2. What does the `"type"` field do?
3. What is `"main"`?
4. What is `"exports"`?
5. What is `"imports"`?
6. What is package scope?
7. What is a bare specifier?
8. What is a relative specifier?
9. What is package self-reference?
10. Why does Node need package metadata?

## Intermediate

11. What is the difference between `main` and `exports`?
12. Why can introducing `exports` be breaking?
13. What is a subpath export?
14. What is a conditional export?
15. What is `"default"`?
16. Why is condition ordering important?
17. What is the difference between `exports` and `imports`?
18. Why do package imports start with `#`?
19. How does Node traverse packages?
20. Why does ESM not use extension searching by default?

## Advanced

21. Explain CommonJS package resolution.
22. Explain ESM package resolution.
23. Why can a package resolve differently under `import` and `require`?
24. What is `ERR_PACKAGE_PATH_NOT_EXPORTED`?
25. What is package encapsulation?
26. Why is package encapsulation not a security sandbox?
27. How can symlinks cause duplicate module instances?
28. Why can monorepo resolution differ from published resolution?
29. Why can TypeScript resolve something Node cannot?
30. How do conditional exports increase test complexity?

## Principal Level

31. Design a package API using `"exports"` for a 200-module library.
32. How would you migrate a legacy package from `"main"` to `"exports"` without breaking consumers?
33. How would you design a dual ESM/CJS package without duplicate state?
34. How would you create package boundaries across a monorepo?
35. When should `"imports"` aliases be preferred over relative paths?
36. How would you prevent accidental internal imports in CI?
37. How would you test package behavior after publishing?
38. How would you diagnose a workspace-only resolution success?
39. How would you design environment-specific exports without condition explosion?
40. How would you explain Node resolution to a principal architecture review?

---

# 134. Predict-the-Output / Resolution Exercises

These are intentionally prediction-first.

## Exercise A — Type

Given:

```text
package.json:
{
  "type": "module"
}
```

What module system should:

```text
src/app.js
```

use by default?

---

## Exercise B — Nested Type

```text
root/package.json:
{
  "type": "module"
}

root/legacy/package.json:
{
  "type": "commonjs"
}
```

What is the classification of:

```text
root/legacy/app.js
```

?

---

## Exercise C — Exports

```json
{
  "exports": {
    ".": "./index.js",
    "./errors": "./errors.js"
  }
}
```

Which of these should be public?

```text
pkg
pkg/errors
pkg/internal
```

---

## Exercise D — Conditions

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

What should:

```js
import 'pkg';
```

select?

What should:

```js
require('pkg');
```

select?

---

## Exercise E — Imports

```json
{
  "imports": {
    "#config": "./config.js"
  }
}
```

Does:

```js
import '#config';
```

work from another unrelated package?

Explain.

---

## Exercise F — Extension

Predict whether:

```js
import './utils';
```

and:

```js
import './utils.js';
```

behave the same in Node ESM.

---

## Exercise G — Main vs Exports

Given:

```json
{
  "main": "./legacy.js",
  "exports": "./modern.js"
}
```

Which field governs supported modern package resolution?

---

# 135. Mastery Exercises

## Exercise 1 — Package Contract

Build a package with:

```text
root
client
errors
testing
private internals
```

and expose only the intended contract.

---

## Exercise 2 — Monorepo

Create:

```text
packages/core
packages/http
packages/cli
```

with explicit package names and exports.

Prevent source-level cross-package deep imports.

---

## Exercise 3 — Resolution Matrix

For every package entry test:

```text
ESM import
CJS require
self-reference
subpath import
unsupported private path
```

Record the expected target.

---

## Exercise 4 — Publish Artifact

Run:

```text
build → pack → fresh install → test
```

and prove that local workspace behavior matches the published package.

---

## Exercise 5 — Condition Governance

Design a package using only the minimum necessary conditions.

Document why each exists.

---

## Exercise 6 — Resolution Debugging

Create five deliberate package-resolution failures and diagnose each from first principles without trial-and-error edits.

---

# 136. Principal Decision Framework

For package and resolution design, evaluate:

| Dimension | Question |
|---|---|
| Correctness | Does every supported specifier resolve to the intended artifact? |
| Performance | Is runtime resolution acceptable? |
| Memory | Could conditional/duplicate paths create duplicate module state? |
| Security | Are package boundaries and dependency names controlled? |
| Reliability | Does packaging reproduce cleanly outside the monorepo? |
| Maintainability | Is the export map small and understandable? |
| Scalability | Can the package architecture grow without path chaos? |
| Observability | Can resolution failures be diagnosed quickly? |
| Developer Experience | Are imports easy to write and refactor? |
| Operational Complexity | How many loader/condition combinations exist? |
| Future Change | Can internals evolve without breaking consumers? |

---

# 137. Production Scenario

You own:

```text
@acme/platform
```

with:

```text
300 source modules
20 public concepts
```

Bad public model:

```text
every source file importable
```

Better:

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./auth": "./dist/auth.js",
    "./errors": "./dist/errors.js",
    "./testing": "./dist/testing.js"
  }
}
```

Now the package contract is:

```text
root
auth
errors
testing
```

Everything else can change internally.

---

# 138. Large-Scale Package Governance

For a large organization, establish rules such as:

```text
1. New packages must define "exports".
2. Public entry points require review.
3. Deep imports across package boundaries are prohibited.
4. New conditions require architecture justification.
5. "imports" aliases require runtime/tooling parity tests.
6. Published artifacts are tested from clean installs.
7. CJS/ESM behavior is tested separately.
8. Package boundaries are documented.
```

---

# 139. Failure Modes

## Failure Mode 1 — Hidden API

Cause:

```text
consumers deep-import implementation files
```

## Failure Mode 2 — Publish break

Cause:

```text
exports target points to missing build artifact
```

## Failure Mode 3 — Monorepo illusion

Cause:

```text
workspace resolution differs from published resolution
```

## Failure Mode 4 — Wrong condition

Cause:

```text
condition ordering or consumer loader mismatch
```

## Failure Mode 5 — Alias works only in TypeScript

Cause:

```text
tsconfig path ≠ runtime Node resolution
```

## Failure Mode 6 — Duplicate package state

Cause:

```text
multiple resolved instances / CJS+ESM dual paths
```

---

# 140. Common Misconceptions

## Misconception 1

> `package.json` is only for npm.

False.

Node itself uses several package fields for runtime behavior.

## Misconception 2

> `"main"` defines every public file.

False.

It mainly defines the primary entry.

## Misconception 3

> `"exports"` just documents the package.

False.

It controls package entry resolution.

## Misconception 4

> `imports` is a public alias system.

False.

It is package-internal.

## Misconception 5

> If a file exists, importers can load it.

False when package exports encapsulate it.

## Misconception 6

> Bundler resolution equals Node resolution.

False.

## Misconception 7

> TypeScript `paths` automatically work at runtime.

False.

## Misconception 8

> Every package condition is a harmless optimization.

False.

Conditions increase complexity and can alter API behavior.

---

# 141. Common Mistakes

```text
[ ] Using main without an explicit export policy
[ ] Introducing exports without a migration audit
[ ] Exporting every source file
[ ] Creating too many conditions
[ ] Using imports aliases unsupported by runtime
[ ] Ignoring package scope
[ ] Testing only inside the monorepo
[ ] Forgetting build output paths
[ ] Relying on bundler-only resolution
[ ] Treating package maps as security boundaries
[ ] Allowing arbitrary dynamic package names
```

---

# 142. Deep Mental Model

Keep this permanently:

```text
Specifier
   ↓
Classification
   ↓
Package scope
   ↓
Resolution algorithm
   ↓
package.json
   ↓
exports / imports / main / type
   ↓
Target identity
   ↓
Module format
   ↓
Loading
   ↓
Evaluation
```

The crucial boundary is:

> **Resolution decides what code the runtime means before that code executes.**

---

# 143. Specification / Runtime Boundary

`package.json`, `"exports"`, `"imports"`, `node_modules` traversal, and CommonJS/ESM package-resolution rules are Node host/runtime behavior.

ECMAScript specifies:

```text
language-level ESM
```

but not:

```text
npm package trees
package.json
node_modules
exports maps
```

Always cite Node documentation for package-resolution claims.

---

# 144. Canonical Resolution References

Primary sources:

- Node.js Packages  
  https://nodejs.org/api/packages.html
- Node.js ECMAScript modules  
  https://nodejs.org/api/esm.html
- Node.js CommonJS modules  
  https://nodejs.org/api/modules.html
- Node.js Modules: `node:module`  
  https://nodejs.org/api/module.html

Current Node.js documentation states:

- package `"type"` determines `.js` interpretation,
- `"exports"` is the modern package entry-point mechanism,
- `"imports"` provides internal package mappings,
- exports can define conditional entries,
- package paths not listed in `"exports"` can be blocked,
- package maps are resolved synchronously and are static,
- CommonJS package resolution integrates with the ESM package-export resolver for package maps. citeturn570896search0turn570896search2

---

# 145. Source Discipline

When studying Node module resolution:

1. Separate package-manager behavior from Node runtime behavior.
2. Separate Node ESM resolution from CommonJS resolution.
3. Treat bundler aliases as separate unless they map to real Node runtime semantics.
4. Verify package behavior from the published artifact.
5. Treat `"exports"` changes as API changes.
6. Verify condition order rather than guessing.
7. Treat `package.json` fields according to the runtime/tool that consumes them.
8. Prefer official Node documentation for current behavior.
9. Use experiments to validate a concrete environment, not to replace the resolution model.
10. Record the Node version when resolution behavior is version-sensitive.

---

# 146. Chapter Connections

## Depends On

- Chapter 41 — Spec Architecture
- Chapter 42 — Abstract Operations
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability

## Builds Toward

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

- module identity
- package boundaries
- npm/package managers
- workspaces
- symlinks
- bundlers
- TypeScript resolution
- dependency graph
- dual packages
- semver

## Why This Chapter Matters Later

A mature JavaScript platform treats module resolution as architecture.

It determines:

```text
who depends on whom
what is public
what is private
which artifact runs
which environment receives which implementation
and whether local development matches production
```

Once you understand this layer, package management and build tooling stop looking like unrelated configuration files.

They become visible parts of the runtime dependency system.

---

# 147. Spaced Retrieval Plan

## Day 0

Explain:

```text
type
main
exports
imports
```

without notes.

## Day 2

Design a package export map from scratch.

## Day 7

Debug:

```text
ERR_PACKAGE_PATH_NOT_EXPORTED
```

using only the package manifest and runtime rules.

## Day 14

Build and test a clean packed artifact.

## Day 30

Defend:

> Why is `"exports"` an architectural contract rather than a convenience field?

---

# 148. Dependency Graph

```text
package.json
    │
    ├── type
    ├── main
    ├── exports
    └── imports
          │
          ▼
     package resolver
          │
    ┌─────┴──────┐
    ▼            ▼
 CommonJS       ESM
 resolver       resolver
    │            │
    └─────┬──────┘
          ▼
     target identity
          │
          ▼
       loading
          │
          ▼
      evaluation
```

---

# 149. Completion Criteria

```text
[ ] Explain package.json's Node-relevant fields
[ ] Explain package scope
[ ] Explain type
[ ] Explain main
[ ] Explain exports
[ ] Explain imports
[ ] Explain package self-reference
[ ] Explain bare specifiers
[ ] Explain relative specifiers
[ ] Explain package resolution
[ ] Explain node_modules traversal
[ ] Explain CJS resolution
[ ] Explain ESM resolution
[ ] Explain extension differences
[ ] Explain folder-main differences
[ ] Explain subpath exports
[ ] Explain export patterns
[ ] Explain conditional exports
[ ] Explain condition ordering
[ ] Explain default condition
[ ] Explain package encapsulation
[ ] Explain why exports can be a breaking change
[ ] Explain package imports
[ ] Explain custom conditions
[ ] Explain published-artifact testing
[ ] Explain monorepo resolution
[ ] Explain symlink identity issues
[ ] Debug package-path errors
[ ] Debug wrong entry points
[ ] Debug module format mismatch
[ ] Design production package metadata
[ ] Pass principal interview questions
```

---

# 150. Mastery Gate

You have mastered this chapter only when you can:

### Understand

Explain Node package resolution from specifier to loaded module.

### Explain

Teach `"exports"` and `"imports"` without treating them as ordinary aliases.

### Predict

Predict which package target Node will choose for a given loader and condition set.

### Implement

Build and publish a package with a deliberate public API.

### Debug

Diagnose resolution failures without randomly changing extensions or deleting package metadata.

### Apply

Use package metadata to enforce architecture across a real project.

### Compare

Defend `main`, `exports`, relative paths, and package-internal `imports`.

### Defend

Explain a package-resolution strategy to a principal engineering review with correctness, compatibility, security, and maintenance reasoning.

---

# 151. Status

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

# 152. Chapter 66 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain exports | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain imports | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Predict condition selection | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Debug path-not-exported | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Test published artifact | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal package review | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. What does package scope mean?
2. How does type affect .js?
3. Why is exports stronger than main?
4. Why can adding exports break consumers?
5. What is the difference between exports and imports?
6. How does self-reference work?
7. How do conditions affect target selection?
8. Why is condition ordering important?
9. Why can monorepo resolution differ from published resolution?
10. Why can TypeScript resolution disagree with Node runtime resolution?
```

---

# 153. Chapter 66 — Canonical References and Source Discipline

## Primary Node.js documentation

- Node.js Packages  
  https://nodejs.org/api/packages.html
- Node.js ECMAScript Modules  
  https://nodejs.org/api/esm.html
- Node.js CommonJS Modules  
  https://nodejs.org/api/modules.html
- Node.js Modules API  
  https://nodejs.org/api/module.html

The current Node.js 26.x packages documentation describes:

- package `"type"`,
- `"main"`,
- `"exports"`,
- `"imports"`,
- package self-reference,
- conditional exports,
- export subpaths,
- package encapsulation,
- and package map rules.  
Source: https://nodejs.org/api/packages.html citeturn570896search0

The current ESM documentation describes:

- relative and bare specifiers,
- mandatory file extensions for relative/absolute ESM imports,
- URL-based ESM resolution,
- package resolution,
- and the ESM/CJS interoperability boundary.  
Source: https://nodejs.org/api/esm.html citeturn570896search1

The current CommonJS documentation shows that package resolution can invoke the package export resolver with CommonJS-relevant conditions.  
Source: https://nodejs.org/api/modules.html citeturn570896search2

---

# 154. Chapter 66 — Completion Snapshot

## Package Metadata

```text
[ ] name
[ ] version
[ ] type
[ ] main
[ ] exports
[ ] imports
```

## Resolution

```text
[ ] relative specifiers
[ ] bare specifiers
[ ] package scope
[ ] package self-reference
[ ] node_modules traversal
[ ] CJS resolver
[ ] ESM resolver
[ ] module format
```

## Package API

```text
[ ] root export
[ ] subpath export
[ ] patterns
[ ] conditional exports
[ ] condition ordering
[ ] default fallback
[ ] encapsulation
```

## Production

```text
[ ] clean publish test
[ ] CJS consumer test
[ ] ESM consumer test
[ ] private-path rejection
[ ] monorepo test
[ ] resolution diagnostics
[ ] package API governance
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

A package import is not merely a path lookup.

When you write:

```js
import { Client } from '@acme/client';
```

you are invoking a system that considers:

```text
current module
   ↓
module format
   ↓
package scope
   ↓
package identity
   ↓
package metadata
   ↓
exports/imports
   ↓
conditions
   ↓
target URL/path
   ↓
module format
   ↓
loader
   ↓
module identity
```

That system determines the real architecture of the application.

The deepest lesson is:

> **The filesystem is an implementation detail; the package-resolution contract is the architecture boundary.**

A principal JavaScript engineer should therefore be able to answer:

```text
What is public?
What is private?
Which package entry is loaded?
Why?
Which condition selected it?
What happens under CJS?
What happens under ESM?
What changes in a browser build?
What happens after packaging?
What happens in a clean install?
Could two paths create duplicate state?
Can a consumer bypass this contract?
How will we evolve the package without breaking consumers?
```

Once you can answer those questions, `package.json` stops being “configuration.”

It becomes what it truly is:

> **a machine-readable part of the JavaScript runtime's dependency and architecture model.**