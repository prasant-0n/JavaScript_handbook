\
# Chapter 68 — Transpilation and Compilation

> **Curriculum position:** Part XII — Modules / Tooling  
> **Previous chapter:** Chapter 67 — Dependency Management and Supply Chain  
> **Next chapter:** Chapter 69 — Bundlers and Build Systems  
> **Primary environment:** Modern JavaScript/Node.js/TypeScript tooling. Version-sensitive behavior is identified explicitly.

---

# Chapter Mission

Master the transformation pipeline that turns JavaScript-family source code into executable artifacts.

The goal is not merely to learn:

```bash
tsc
babel
```

The goal is to understand the complete chain:

```text
source language
    ↓
parse
    ↓
AST / intermediate representation
    ↓
analysis
    ↓
transformation
    ↓
lowering
    ↓
module transformation
    ↓
polyfill strategy
    ↓
emit
    ↓
source maps
    ↓
runtime
```

For TypeScript:

```text
.ts
 ↓
parse
 ↓
type analysis
 ↓
transform / erase
 ↓
JavaScript
 ↓
runtime
```

For Babel:

```text
modern JavaScript
 ↓
parse
 ↓
plugins / presets
 ↓
AST transformations
 ↓
generated JavaScript
 ↓
runtime / bundler
```

For Node's current built-in TypeScript support:

```text
.ts
 ↓
type stripping
 ↓
JavaScript-like source
 ↓
Node runtime
```

but importantly:

```text
Node type stripping
≠
full TypeScript compilation
```

Current Node documentation states that built-in type stripping is stable in Node 24.12.0 / 25.2.0 and later, is enabled by default in recent Node releases, performs no type checking, ignores `tsconfig.json`, and supports only erasable TypeScript syntax. citeturn236935search0turn236935search5

This chapter teaches you to distinguish:

- parsing,
- type checking,
- transpilation,
- compilation,
- lowering,
- polyfilling,
- bundling,
- minification,
- code generation,
- runtime loading,
- and execution.

The principal-level goal is:

> **Know exactly which stage changes code, which stage changes semantics, which stage only removes types, which stage supplies missing runtime features, and which artifact production actually executes.**

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

## Concepts

- Define source transformation.
- Define transpilation.
- Define compilation.
- Define code generation.
- Explain ASTs.
- Explain intermediate representations.
- Explain parsing.
- Explain semantic analysis.
- Explain type checking.
- Explain lowering.
- Explain polyfills.
- Explain syntax transforms.
- Explain runtime transforms.
- Explain module-format transforms.
- Explain source maps.
- Explain declaration-file generation.

## JavaScript

- Explain modern syntax lowering.
- Explain class transformation.
- Explain async transformation.
- Explain generator transformation.
- Explain optional chaining transformation.
- Explain nullish coalescing transformation.
- Explain destructuring transformation.
- Explain module transformations.
- Explain why some language features cannot be reproduced by syntax rewriting alone.

## TypeScript

- Explain TypeScript's type erasure model.
- Explain `tsc`.
- Explain `target`.
- Explain `module`.
- Explain `moduleResolution`.
- Explain `declaration`.
- Explain `sourceMap`.
- Explain `verbatimModuleSyntax`.
- Explain `rewriteRelativeImportExtensions`.
- Explain Node-oriented `nodenext` behavior.
- Explain why TypeScript type checking and JavaScript emit are separate concerns.
- Explain Node built-in type stripping.
- Explain what Node type stripping intentionally does not support.
- Explain why Node type stripping ignores `tsconfig.json`.
- Explain `import type`.
- Explain erasable syntax.

## Babel

- Explain Babel's parser/transformer/generator pipeline.
- Explain plugins.
- Explain presets.
- Explain target environments.
- Explain syntax transformation.
- Explain polyfill strategy.
- Explain `@babel/preset-env`.
- Explain runtime vs compile-time behavior.
- Explain Babel configuration scope.
- Explain build-time versus runtime transforms.

## Production

- Design a source-to-artifact pipeline.
- Choose between `tsc`, Babel, SWC, esbuild, runtime TypeScript support, or combinations.
- Keep source maps correct.
- Avoid semantic changes caused by transforms.
- Validate emitted module formats.
- Make builds reproducible.
- Detect source/artifact mismatches.
- Test generated artifacts rather than only source.

## Principal judgment

- Decide whether a feature should be transformed or left to the runtime.
- Decide minimum supported runtime and target.
- Design a compilation strategy for libraries versus applications.
- Decide where type checking belongs.
- Decide when full compilation is unnecessary.
- Defend a build pipeline in architecture review.

---

# 2. Prerequisites

Recommended:

- Chapter 41 — Specification Architecture
- Chapter 42 — Abstract Operations
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability
- Chapter 66 — `package.json` and Module Resolution
- Chapter 67 — Dependency Management and Supply Chain
- Chapter 70 — Source Maps and Production Debugging

---

# 3. What Is Transpilation?

Transpilation generally means transforming source code from one language level or closely related source representation to another representation.

Examples:

```text
ES2024 syntax
      ↓
older JavaScript syntax
```

or:

```text
TypeScript
      ↓
JavaScript
```

The term is informal.

A more precise engineering vocabulary is often:

```text
parse
transform
lower
emit
compile
```

Do not spend time arguing whether a specific tool is “really a transpiler.”

Understand what transformations it actually performs.

---

# 4. What Is Compilation?

Compilation is the broader process of transforming a program into another executable or intermediate representation.

For JavaScript applications:

```text
source
 ↓
parser
 ↓
AST
 ↓
transform
 ↓
JavaScript artifact
 ↓
runtime
```

A JS compiler may stop at JavaScript output.

Or:

```text
JavaScript
 ↓
bytecode / machine code
```

can then happen inside the runtime/engine.

This is why:

> source-to-JavaScript compilation and JIT compilation are different stages.

---

# 5. Transpilation vs JIT Compilation

## Build-time transformation

```text
TypeScript
 ↓
JavaScript
```

## Runtime engine compilation

```text
JavaScript
 ↓
engine parser
 ↓
bytecode / baseline
 ↓
optimized machine code
```

These happen at different layers.

A compiler like Babel or TypeScript generally does not perform V8's runtime optimization.

V8 later analyzes and optimizes the emitted JavaScript.

---

# 6. Mental Model

Use this seven-stage model:

```text
1. Input
2. Parse
3. Analyze
4. Transform
5. Generate
6. Map
7. Run
```

Expanded:

```text
source file
    │
    ▼
parser
    │
    ▼
AST
    │
    ▼
semantic analysis
    │
    ▼
transforms
    │
    ▼
code generator
    │
    ├── JavaScript
    ├── declarations
    └── source maps
            │
            ▼
        runtime loader
            │
            ▼
         JS engine
```

---

# 7. The Most Important Distinction

These are not the same:

```text
Type checking
Transformation
Polyfilling
Bundling
Minification
Execution
```

For example:

```text
TypeScript
```

may perform:

```text
type checking + JavaScript emit
```

while:

```text
Node type stripping
```

performs:

```text
syntax erasure
```

without type checking.

Node's current TypeScript docs explicitly state that built-in type stripping performs no type checking and ignores `tsconfig.json`. citeturn236935search0

---

# 8. Basic Example

Input:

```ts
function add(a: number, b: number): number {
  return a + b;
}
```

TypeScript can emit:

```js
function add(a, b) {
  return a + b;
}
```

The types disappear because JavaScript runtime semantics do not contain TypeScript's static type annotations.

This is primarily **type erasure**.

---

# 9. Syntax Transformation

Consider:

```js
const value = obj?.user?.name;
```

A transformer targeting older syntax may rewrite it into a longer equivalent-ish form.

The exact generated code depends on the compiler.

Conceptually:

```text
optional chaining
       ↓
explicit nullish checks
```

This changes syntax representation while attempting to preserve behavior.

---

# 10. Why Transformation Exists

JavaScript runtimes do not all support the same language features at the same time.

Suppose:

```text
source uses feature F
target runtime does not support F
```

Then:

```text
source
  ↓
transform
  ↓
older syntax
  ↓
target runtime
```

This lets developers write newer language syntax while supporting older environments.

The tradeoff is:

```text
build complexity
+
generated code
+
possible semantic differences
```

---

# 11. Target Environment

A transform should be driven by a target contract:

```text
Node 22
Node 24
modern Chrome
Safari 17
older browsers
embedded runtime
```

The target determines:

```text
what can remain native
what must be transformed
what needs a polyfill
```

A common mistake is:

```text
always transpile everything
```

This can create unnecessarily large or slower output.

---

# 12. TypeScript `target`

TypeScript's `target` controls the JavaScript language level of emitted code.

Conceptually:

```json
{
  "compilerOptions": {
    "target": "ES2022"
  }
}
```

means:

> emit JavaScript suitable for an ES2022-level target.

TypeScript documentation recommends setting a library's target to the lowest ECMAScript version the library intends to support. citeturn982887search0turn982887search5

---

# 13. Target Is Not Browser Support

This:

```json
{
  "target": "ES2020"
}
```

does not guarantee:

```text
all ES2020 APIs exist
```

Syntax target and runtime API availability are separate.

Example:

```text
syntax support
≠
built-in API support
```

This is why polyfills may still be required.

---

# 14. Syntax Transform vs Polyfill

Suppose:

```js
Array.prototype.someNewMethod()
```

does not exist in an older runtime.

Changing syntax does not create the missing method.

You need:

```text
polyfill
```

or:

```text
alternative implementation
```

Babel explicitly separates syntax transformation from polyfill support and uses tools such as `core-js` for missing built-ins. citeturn236935search2turn236935search3

---

# 15. Semantic Transformation

Some features require more than syntax replacement.

Example:

```js
async function f() {
  await g();
}
```

For older targets, a transformer may generate:

```text
Promise / generator / helper runtime
```

This changes the emitted control-flow implementation.

The behavior aims to preserve language semantics, but:

```text
performance
stack traces
microtask behavior
helper size
```

may differ subtly.

---

# 16. Not Every Feature Can Be Transformed Perfectly

Some modern runtime features rely on host or engine support.

Examples conceptually:

```text
WeakRef
FinalizationRegistry
some new host APIs
new engine intrinsics
```

A syntax transform cannot magically create the underlying engine capability.

Therefore:

```text
syntax feature
```

may be transformable, while:

```text
runtime capability
```

may require a newer runtime or polyfill.

---

# 17. Type Erasure

TypeScript syntax often has no JavaScript runtime meaning:

```ts
const count: number = 1;
```

The annotation:

```ts
: number
```

can be erased.

That is why Node's built-in type stripping is lightweight.

Current Node documentation says it replaces TypeScript syntax with whitespace and intentionally avoids source maps because line positions remain correct for this kind of erasure. citeturn236935search0

---

# 18. Type Erasure Does Not Mean Compilation Is Free

Even if types are removed:

```text
parse
+
analyze syntax
+
rewrite
+
load
```

still costs time.

Node's built-in stripping is designed to be lightweight, but it is still a transformation step.

---

# 19. Node Built-In Type Stripping

Modern Node supports lightweight execution of `.ts` source containing erasable TypeScript syntax.

Current Node documentation says:

- type stripping is stable,
- it is enabled by default in recent Node versions,
- it performs no type checking,
- it ignores `tsconfig.json`,
- only erasable TypeScript syntax is supported,
- `.tsx` is unsupported,
- `.mts` is ESM,
- `.cts` is CommonJS,
- and import extensions are required. citeturn236935search0

This is a major modern Node capability.

---

# 20. Node Type Stripping vs `tsc`

| Capability | Node type stripping | `tsc` |
|---|---:|---:|
| Run `.ts` directly | ✅ supported subset | emits or type-checks |
| Type checking | ❌ | ✅ |
| `tsconfig.json` | ❌ ignored | ✅ |
| Type erasure | ✅ | ✅ |
| Full TS syntax transformations | ❌ | ✅ where supported |
| Older JS target lowering | ❌ | ✅ |
| Declaration emit | ❌ | ✅ |
| Source maps | no generated maps needed for stripping | ✅ |
| Path alias transformation | ❌ | compiler-aware; runtime still needs appropriate resolution |
| Production package compilation | ❌ | ✅ |

---

# 21. Node Type Stripping Is Not TypeScript

This distinction is critical.

Node is not implementing the TypeScript type system.

Conceptually:

```text
Node
  = parser + eraser + runtime

TypeScript compiler
  = type system + emitter + tooling
```

Node's docs explicitly recommend third-party tooling for full TypeScript support. citeturn236935search0

---

# 22. Unsupported TypeScript Features in Node Stripping

Features requiring generated JavaScript are outside lightweight type stripping.

Examples include:

```text
enum
parameter properties
certain namespaces
other TS syntax requiring runtime transformation
```

Node documents that TypeScript features requiring transformation are unsupported by type stripping. citeturn236935search0

---

# 23. Why `enum` Is Different

Type annotation:

```ts
const x: number = 1;
```

has no runtime implementation.

Enum:

```ts
enum Direction {
  Up,
  Down,
}
```

requires generated runtime JavaScript.

Therefore:

```text
type annotation
→ erase

enum
→ generate runtime object/code
```

Node's lightweight stripping intentionally avoids the latter.

---

# 24. Type-Only Imports

Correct under type stripping:

```ts
import type { User } from './user.ts';
```

Because the import is clearly type-only.

Current Node documentation warns that a non-`type` import of a type-only name can become a runtime error after type stripping. citeturn236935search0

---

# 25. Why `import type` Matters

Input:

```ts
import { User } from './user.ts';
```

If `User` exists only in the type system:

```text
runtime still sees import
```

and may try to load it.

Correct:

```ts
import type { User } from './user.ts';
```

Now the import can disappear safely.

---

# 26. `erasableSyntaxOnly`

TypeScript provides:

```json
{
  "compilerOptions": {
    "erasableSyntaxOnly": true
  }
}
```

This can help enforce a source style compatible with syntax-erasure-only execution.

Node's current documentation recommends this alongside modern Node-oriented TypeScript settings. citeturn236935search0

---

# 27. `verbatimModuleSyntax`

With:

```json
{
  "compilerOptions": {
    "verbatimModuleSyntax": true
  }
}
```

TypeScript preserves import/export intent more explicitly.

This is particularly important when module-system behavior must match runtime behavior.

TypeScript's current documentation recommends it for Node and library configurations because it prevents ambiguous import forms and keeps module syntax aligned with emitted module semantics. citeturn982887search0turn982887search5

---

# 28. `rewriteRelativeImportExtensions`

Modern TypeScript supports:

```json
{
  "compilerOptions": {
    "rewriteRelativeImportExtensions": true
  }
}
```

This helps when TypeScript source imports use TypeScript extensions but emitted JavaScript needs runtime-compatible extensions.

Node's current type-stripping guidance recommends this option for TypeScript projects targeting direct Node execution. citeturn236935search0

---

# 29. Node + TypeScript Module Classification

Node uses the same broad module classification rules for `.ts` as `.js`.

Current documentation states:

```text
.ts
 → determined by package "type"

.mts
 → always ESM

.cts
 → always CommonJS
```

`tsx` is currently unsupported by Node's built-in type-stripping mechanism. citeturn236935search0

---

# 30. `tsconfig.json` Boundary

Node's built-in stripping intentionally ignores `tsconfig.json`.

Therefore settings such as:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

do not automatically create runtime aliases.

Current Node documentation explicitly notes that `tsconfig` `paths` are not transformed by type stripping and recommends subpath imports as the closest Node-native alternative. citeturn236935search0

---

# 31. TypeScript's `module`

Modern TypeScript has module settings including:

```text
node16
node18
node20
nodenext
preserve
esnext
commonjs
```

The Node-oriented modes describe the behavior of Node's dual module system rather than merely choosing output syntax.

TypeScript documentation explains that `node16`, `node18`, `node20`, and `nodenext` model Node's supported CommonJS/ESM rules. citeturn982887search2turn982887search4

---

# 32. `nodenext`

For a Node application compiled by TypeScript:

```json
{
  "compilerOptions": {
    "module": "nodenext"
  }
}
```

is a modern way to model Node's module behavior.

TypeScript's current documentation recommends `nodenext` for projects compiling and running outputs in Node. citeturn982887search0turn982887search4

---

# 33. `moduleResolution`

This controls how TypeScript resolves imports during analysis.

Important values:

```text
node10
node16
nodenext
bundler
```

They are not interchangeable.

TypeScript's current compiler documentation states that `moduleResolution` defines how TypeScript looks up files from module specifiers. citeturn982887search3

---

# 34. Runtime Resolution vs TypeScript Resolution

This is one of the most important traps.

```text
TypeScript resolves
      ↓
type-check passes
```

does not necessarily mean:

```text
Node resolves
      ↓
runtime succeeds
```

For example:

```text
tsconfig paths
```

can work in TypeScript while native Node cannot resolve the same alias.

---

# 35. Bundler Resolution vs Node Resolution

TypeScript's current documentation explicitly distinguishes:

```text
moduleResolution: bundler
```

from:

```text
moduleResolution: nodenext
```

Bundler resolution can allow import patterns that work because a bundler rewrites them, while Node runtime resolution may reject them. citeturn982887search0turn982887search5

---

# 36. The `bundler` Trap

Suppose:

```ts
import './utils';
```

TypeScript with bundler-oriented resolution may accept it.

A direct Node ESM runtime may require:

```ts
import './utils.js';
```

Your code can therefore:

```text
type-check
✅

bundle
✅

direct Node execution
❌
```

This is why runtime target must be part of compiler configuration.

---

# 37. AST

An Abstract Syntax Tree represents source structure.

Source:

```js
const x = a + b;
```

Conceptual AST:

```text
VariableDeclaration
└── VariableDeclarator
    ├── Identifier(x)
    └── BinaryExpression(+)
        ├── Identifier(a)
        └── Identifier(b)
```

Transformers operate on structures like this rather than raw text whenever possible.

---

# 38. Why AST Transformation Beats Text Replacement

Bad conceptual approach:

```text
replace "=>" with "function"
```

This cannot correctly understand:

```text
strings
comments
nested syntax
scope
parentheses
operator precedence
```

AST transformation understands the grammar structure.

---

# 39. Parse → Transform → Generate

A Babel-style pipeline:

```text
source
 ↓
parser
 ↓
AST
 ↓
plugin transforms
 ↓
generator
 ↓
output source
```

Babel documents itself as a JavaScript compiler that transforms syntax and can integrate polyfill support. citeturn236935search2

---

# 40. Babel Plugins

Plugins define transformations.

Conceptually:

```text
AST
 ↓
plugin A
 ↓
plugin B
 ↓
plugin C
 ↓
generated AST
```

Examples can transform:

```text
arrow functions
classes
optional chaining
modules
JSX
```

---

# 41. Babel Presets

A preset packages a set of plugins/configuration.

For example:

```text
preset-env
```

can select transforms based on target environments.

The key idea:

> A preset is a policy bundle, not a new language feature.

---

# 42. `@babel/preset-env`

Babel's preset-env is designed to transform modern JavaScript based on target environments.

The target should reflect:

```text
actual runtime support
```

not:

```text
“all old browsers just in case.”
```

More transforms mean more generated code.

---

# 43. Babel and Polyfills

Babel syntax transformation does not by itself provide missing APIs.

Polyfill strategy can involve:

```text
core-js
regenerator-runtime
```

or runtime-provided equivalents.

Babel's documentation distinguishes syntax transforms from polyfill support. citeturn236935search2turn236935search3

---

# 44. Polyfill Risk

A polyfill can modify global or built-in behavior.

Potential concerns:

- bundle size,
- global mutation,
- compatibility,
- performance,
- security review,
- interaction with native implementations.

Use only what the target environment requires.

---

# 45. Helper Functions

Transforms can introduce helpers.

Example concept:

```text
class transform
   ↓
helper code
```

If every transformed file includes a copy:

```text
helper
helper
helper
helper
```

output grows.

Tooling can centralize helpers or use runtime helper packages depending on configuration.

---

# 46. Generated Runtime Dependencies

Some transforms require runtime libraries.

This creates another dependency layer:

```text
source
 ↓
transform
 ↓
generated code
 ↓
runtime helper
```

Your application now depends on the helper implementation.

---

# 47. Runtime vs Compile-Time Dependency

A compiler plugin may be:

```text
devDependency
```

while the generated output needs:

```text
runtime dependency
```

Do not confuse:

```text
tool used to generate code
```

with:

```text
library required by generated code
```

---

# 48. TypeScript Emit Pipeline

A simplified `tsc` flow:

```text
.ts / .tsx
   ↓
parse
   ↓
type analysis
   ↓
module determination
   ↓
transform
   ↓
emit .js
   ↓
emit .d.ts
   ↓
emit .map
```

Not every project emits all artifact types.

---

# 49. `noEmit`

A TypeScript project can use:

```json
{
  "compilerOptions": {
    "noEmit": true
  }
}
```

to perform type checking without producing JS.

This is useful when:

```text
bundler handles JS generation
```

or:

```text
another compiler emits output
```

---

# 50. `emitDeclarationOnly`

For libraries:

```json
{
  "compilerOptions": {
    "emitDeclarationOnly": true,
    "declaration": true
  }
}
```

can produce type declarations while another tool emits JavaScript.

This is useful but introduces consistency requirements between:

```text
JS artifact
+
declaration artifact
```

---

# 51. Declaration Files

TypeScript declaration output:

```text
.d.ts
```

describes types available to consumers.

For example:

```ts
export declare function add(a: number, b: number): number;
```

This is not executable JavaScript.

---

# 52. Declaration Files and Module Resolution

A library can have:

```text
working JS
+
broken .d.ts
```

and still fail consumers.

TypeScript's current library guidance highlights this risk, especially when a bundler and declaration emitter use different module-resolution assumptions. citeturn982887search0turn982887search5

---

# 53. Source Maps

A transformed file:

```text
src/main.ts
 ↓
dist/main.js
```

needs debugging information that maps:

```text
dist/main.js:120
```

back to:

```text
src/main.ts:90
```

A source map provides that relationship.

---

# 54. Source Map Pipeline

```text
source
 ↓
transform
 ↓
output.js
 +
output.js.map
```

The map contains mappings between generated and original positions.

---

# 55. Why Source Maps Matter

Without source maps:

```text
stack trace
  ↓
generated helper code
```

With source maps:

```text
stack trace
  ↓
original source
```

This is essential for production debugging.

Chapter 70 goes deeper into source maps.

---

# 56. Node Type Stripping and Source Maps

Node's type-stripping approach intentionally replaces type syntax with whitespace.

Therefore line/column positions remain aligned closely enough that Node documentation says source maps are unnecessary for correct line numbers and Node does not generate them for that feature. citeturn236935search0

This is a clever trade-off:

```text
type removal
+
preserved source positions
=
low-overhead debugging
```

---

# 57. Transforming Source Can Distort Stack Traces

A transform can change:

```text
line
column
function shape
async structure
helper frames
```

Source maps can reduce the mismatch but do not make generated runtime behavior identical to source.

---

# 58. Semantic Preservation

A transformation should ideally preserve observable semantics.

But complete equivalence can be difficult.

Potential differences include:

```text
stack traces
timing
helper allocations
enumerability
`this` behavior
constructors
prototype details
source locations
```

Therefore:

> **“Compiles successfully” is not proof that behavior is preserved.**

---

# 59. Example — Class Transform

Modern:

```js
class User {
  #name;

  constructor(name) {
    this.#name = name;
  }
}
```

A transform targeting old runtimes may use:

```text
WeakMap-like state
helpers
property descriptors
```

The generated output can be substantially larger.

---

# 60. Why Target Modern Runtimes

If production already guarantees:

```text
Node 24+
```

there may be little value in transforming every modern syntax feature to ES5-style code.

Keeping native syntax can improve:

- readability of deployed artifacts,
- startup performance,
- stack traces,
- output size,
- optimization opportunities.

Choose the lowest target you actually support.

---

# 61. Library vs Application Targets

## Application

You know your deployment runtime:

```text
Node 24
```

You can target it directly.

## Library

You do not control consumers.

You may need:

```text
lower target
+
declaration compatibility
+
multiple module outputs
```

TypeScript's current docs explicitly distinguish application and library compilation choices. citeturn982887search0turn982887search5

---

# 62. Library Compilation Strategy

A library may produce:

```text
dist/
  esm/
  cjs/
  types/
```

But each output must be tested.

Do not assume:

```text
ESM works
→
CJS must work
```

because the module systems can expose different APIs and runtime behavior.

TypeScript's current guidance explicitly warns that dual-emit solutions require tests/static analysis across outputs because a single compilation cannot type-check two different emitted module behaviors simultaneously in every case. citeturn982887search0

---

# 63. `nodenext` Library Model

For libraries targeting Node's actual behavior:

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext"
  }
}
```

can align type checking with Node's conditional module system.

---

# 64. `module: preserve`

Modern TypeScript also supports preserving each source module statement rather than converting everything into one common module format in some workflows.

This is useful for:

```text
bundler-driven builds
```

where another tool handles the final module transformation.

Use according to the bundler and runtime contract.

---

# 65. `verbatimModuleSyntax` and Libraries

TypeScript's current library guidance recommends:

```json
{
  "verbatimModuleSyntax": true
}
```

because it avoids ambiguous interoperability assumptions and makes emitted import/export syntax more predictable for consumers. citeturn982887search0turn982887search5

---

# 66. Compilation Pipeline Choices

Common patterns:

### TypeScript only

```text
TS
 ↓
tsc
 ↓
JS
```

### TypeScript check + Babel

```text
TS
 ↓
tsc --noEmit
 ↓
Babel
 ↓
JS
```

### TypeScript check + esbuild/SWC

```text
TS
 ↓
tsc --noEmit
 ↓
fast transformer
 ↓
JS
```

### Node native type stripping

```text
TS
 ↓
Node
 ↓
type strip
 ↓
run
```

### Node stripping + external type checking

```text
TS
 ├── Node → run
 └── tsc → type-check
```

---

# 67. `tsc` Is More Than a Type Checker

This misconception is common.

`tsc` can perform:

```text
type checking
+
transformation
+
JavaScript emit
+
declaration emit
+
source maps
```

Depending on options.

---

# 68. `tsc` Is Also Not a Full Build System

It does not automatically provide:

```text
bundling
asset optimization
code splitting
CSS processing
image optimization
```

Those belong to build/bundler tooling.

---

# 69. Babel Is Not a Type Checker

Babel can parse and transform TypeScript syntax through appropriate tooling, but it generally removes types rather than validating TypeScript's type system.

Therefore:

```text
Babel
≠
TypeScript compiler type checker
```

Use TypeScript or another type checker when static type validation is required.

---

# 70. SWC / esbuild Mental Model

Fast compilers such as:

```text
SWC
esbuild
```

often prioritize high-speed transformation.

They may intentionally leave full type checking to:

```text
tsc
```

This creates a useful split:

```text
type correctness → type checker
code generation → fast transformer
```

---

# 71. Separation of Concerns

A scalable pipeline can be:

```text
TypeScript
     │
     ├── type analysis → errors
     │
     └── syntax transform → compiler
                            │
                            ▼
                           JS
```

This is often faster than asking one tool to perform every task in every build.

---

# 72. But Multiple Tools Increase Complexity

Now you have:

```text
tsconfig
babel config
bundler config
package.json
```

The possibility of disagreement increases.

Example:

```text
TypeScript thinks ESM
Babel outputs CJS
package says ESM
```

Result:

```text
runtime failure
```

---

# 73. Single Source of Truth

A mature build should minimize contradictory configuration.

For each setting ask:

```text
Who owns module format?
Who owns target?
Who owns aliases?
Who owns polyfills?
Who owns source maps?
Who owns minification?
```

Do not let five tools independently decide the same thing.

---

# 74. Babel Configuration Scope

Babel supports project-level and package-local configuration models.

For monorepos:

```text
babel.config.json
```

can provide root-wide policy.

For package-specific behavior:

```text
.babelrc
```

may be appropriate.

Babel's current configuration docs describe these distinctions. citeturn236935search11

---

# 75. Build Determinism

A build should strive toward:

```text
same source
+
same lockfile
+
same compiler version
+
same config
≈
same output
```

Compiler version matters because transforms can change across releases.

---

# 76. Pin Tooling Versions

For production builds, pin or tightly govern:

```text
TypeScript
Babel
SWC
esbuild
plugins
presets
```

A compiler update can alter emitted code without changing application source.

---

# 77. Compiler Plugins Are Build Dependencies

A Babel plugin:

```text
devDependency
```

still changes production code.

Therefore supply-chain security applies to build tools as much as runtime packages.

---

# 78. Generated Artifact as a Product

Treat:

```text
dist/
```

as a release artifact.

Verify:

```text
module format
target syntax
source maps
exports
types
license files
```

before publishing.

---

# 79. Build Output Validation

For a Node library:

```text
dist/index.js
dist/index.cjs
dist/index.d.ts
```

test:

```text
ESM import
CJS require
TypeScript consumer
```

Do not publish based only on:

```text
compiler exited 0
```

---

# 80. Artifact Inspection

Useful checks:

```bash
node --check dist/index.js
```

and direct runtime tests.

For CJS:

```bash
node -e "console.log(require('./dist/index.cjs'))"
```

For ESM:

```bash
node -e "import('./dist/index.js').then(console.log)"
```

The exact command depends on package metadata.

---

# 81. Build Output and Package Exports

From Chapter 66:

```json
{
  "exports": {
    ".": "./dist/index.js"
  }
}
```

must match actual emitted files.

A compiler can succeed while the package is broken if:

```text
exports target
≠
emitted file
```

---

# 82. Compiler Target vs Package Type

Suppose:

```json
{
  "type": "module"
}
```

but your compiler emits CommonJS:

```text
dist/index.js
```

Node may interpret that `.js` as ESM.

The artifact is now incorrectly classified.

The package's:

```text
type
+
file extension
+
compiler output
```

must agree.

---

# 83. `.cts` / `.mts` for TypeScript

Explicit source extensions help.

```text
.mts → ESM
.cts → CJS
```

and emitted:

```text
.mjs
.cjs
```

when using appropriate TypeScript/Node-oriented configuration.

This reduces ambiguity in dual-module packages.

---

# 84. Compile-Time vs Runtime Module Semantics

TypeScript must know:

```text
what module the source file will become
```

to type-check imports correctly.

This is why `module` configuration matters even when:

```json
{
  "noEmit": true
}
```

TypeScript's current module theory documentation explicitly explains this. citeturn982887search2

---

# 85. Transpilation and `this`

Transforms can affect:

```text
this
arguments
super
new.target
```

A good transformer preserves semantics according to its supported targets.

But generated helper logic can make debugging more complex.

---

# 86. Transpilation and Closures

Transforms can rewrite syntax around:

```text
closures
iteration
classes
async functions
```

The compiler must preserve lexical behavior.

This is one reason AST-aware transforms are sophisticated rather than simple text rewriting.

---

# 87. Async Transformation

Modern runtime:

```js
async function main() {
  await work();
}
```

Older-target transformation may create:

```text
generator
+
Promise helpers
```

This can alter:

- stack shape,
- allocations,
- bundle size,
- performance.

When target runtimes support async natively, preserving it is usually preferable.

---

# 88. Generator Transformation

Generators may require runtime helpers or generator runtime support when targeting older environments.

A syntax transform does not necessarily mean:

```text
zero runtime dependencies
```

---

# 89. Class Fields

Modern:

```js
class User {
  name = 'unknown';
}
```

Older targets may need:

```text
constructor assignments
```

or helper logic.

The exact semantics matter around:

```text
initialization order
derived classes
private fields
accessors
```

---

# 90. Optional Chaining

Modern:

```js
user?.profile?.name
```

can be transformed because its behavior can be expressed through conditional checks.

But generated code may be more verbose than native syntax.

This illustrates:

```text
semantic transform
≈
possible but not free
```

---

# 91. Nullish Coalescing

```js
const value = input ?? fallback;
```

cannot always be replaced with:

```js
input || fallback
```

because:

```text
0
false
''
```

must remain valid non-nullish values.

A correct compiler transform preserves this distinction.

This is a good example of why naive transpilation is dangerous.

---

# 92. Destructuring Transform

```js
const { a } = object;
```

can be transformed into older syntax.

But array/object destructuring has semantics around:

- getters,
- iterators,
- defaults,
- evaluation order.

A proper compiler must preserve those semantics.

---

# 93. Spread Transform

```js
const copy = { ...obj };
```

is not necessarily identical to:

```js
Object.assign({}, obj);
```

in every semantic detail or with every target/transform mode.

Compiler transforms must account for language semantics and helper behavior.

---

# 94. Module Transformation

ESM:

```js
import { x } from './x.js';
```

can become CommonJS:

```js
const { x } = require('./x.js');
```

but real interop often requires more machinery.

This is why:

```text
module transform
```

is more than replacing keywords.

---

# 95. ESM ↔ CJS Transform Risks

Module conversion affects:

- live bindings,
- default exports,
- `this`,
- cycle behavior,
- loading timing,
- dynamic import,
- package conditions.

Do not assume:

```text
source semantics
=
transformed module semantics
```

without testing.

---

# 96. Babel `module` Transforms

Babel can transform module syntax depending on configuration.

The chosen output must match:

```text
package "type"
exports
runtime
bundler
```

Otherwise:

```text
compiled successfully
```

but:

```text
runtime fails
```

---

# 97. TypeScript Module Emit

TypeScript can emit different module formats based on:

```text
module
package type
file extension
```

Modern Node-oriented configurations allow per-file interpretation based on Node's package rules.

This is why:

```text
module = nodenext
```

is more than:

```text
output = ESM
```

---

# 98. `preserve` and Bundlers

When a bundler owns final module transformation, preserving module syntax can let the bundler perform:

```text
tree shaking
code splitting
conditional optimization
```

more effectively.

The compiler should not prematurely destroy information the bundler needs.

---

# 99. Bundler Boundary Preview

Chapter 69 goes deeper.

For now:

```text
transpiler
  = changes source representation

bundler
  = builds an application/module graph into release artifacts
```

The distinction matters.

---

# 100. Build Pipeline Example

## Node ESM application

```text
src/*.ts
   ↓
tsc
   ├── type check
   ├── emit JS
   └── source maps
   ↓
dist/*.js
   ↓
Node
```

---

# 101. Build Pipeline Example — Type Checking + Fast Transform

```text
src/*.ts
    │
    ├── tsc --noEmit
    │
    └── esbuild / SWC
             ↓
            dist
             ↓
           Node
```

The two tools must agree about:

```text
module
target
resolution
syntax
```

---

# 102. Build Pipeline Example — Babel

```text
src
 ↓
Babel
 ├── parser
 ├── plugins
 ├── preset-env
 ├── polyfill strategy
 └── generator
 ↓
dist
```

Babel's documentation describes this as a compiler/toolchain for transforming modern JavaScript to compatible output. citeturn236935search2turn236935search3

---

# 103. Build Pipeline Example — Native Node TS

```text
src/*.ts
 ↓
Node
 ↓
type stripping
 ↓
runtime
```

Plus separately:

```text
tsc --noEmit
 ↓
type checking
```

This can be attractive for lightweight scripts and developer tooling.

It is not automatically the best package-publishing pipeline.

---

# 104. When Native Node Type Stripping Fits

Good candidates:

```text
small internal scripts
developer tools
simple CLIs
experiments
projects that already target modern Node
```

provided they stay within erasable syntax constraints.

---

# 105. When Full Compilation Fits

Good candidates:

```text
published libraries
older runtime support
generated declarations
syntax lowering
custom transforms
distribution artifacts
```

---

# 106. Native Node Type Stripping and `node_modules`

Current Node documentation says Node refuses to handle TypeScript files located inside `node_modules`, discouraging packages from publishing raw TypeScript as the runtime distribution. citeturn236935search0

This reinforces the distinction:

```text
authoring in TS
```

versus:

```text
shipping runtime JS
```

---

# 107. `tsx` / `ts-node` vs Native Node

Third-party tools can provide richer TypeScript execution.

Node's current documentation gives `tsx` as an example for full TypeScript support and notes that full support requires third-party tooling. citeturn236935search0

Different tools have different resolution/transform semantics.

Do not assume:

```text
works under tsx
=
works directly under Node
```

---

# 108. Runtime Transpilation

Tools such as Babel register hooks can transform modules on demand.

Conceptually:

```text
require(module)
 ↓
hook
 ↓
transform
 ↓
execute transformed code
```

Babel documents `@babel/register` as a require hook that compiles loaded files on the fly. citeturn236935search10

---

# 109. Runtime Transform vs Precompile

Runtime:

```text
startup
 ↓
transform
 ↓
execute
```

Precompile:

```text
build
 ↓
transform once
 ↓
deploy artifact
 ↓
execute
```

Production systems generally benefit from precompilation when possible because:

- startup is predictable,
- artifacts are inspectable,
- failures happen before deployment,
- production requires less tooling.

---

# 110. Build-Time Failure Is Better Than Runtime Failure

Prefer:

```text
invalid code
   ↓
CI fails
```

over:

```text
production starts
   ↓
runtime transform fails
```

This is one reason production images commonly contain:

```text
compiled output
```

rather than:

```text
full compiler toolchain
```

---

# 111. Build Artifact Minimalism

A production image may need only:

```text
dist/
package.json
production node_modules
```

rather than:

```text
TypeScript compiler
Babel
test framework
source
```

This reduces:

- image size,
- startup complexity,
- attack surface.

---

# 112. Security Considerations

## Compiler supply chain

Your compiler/plugins can execute code during build.

## Build-time secret exposure

A malicious plugin may access:

```text
environment variables
filesystem
CI credentials
```

## Generated code

A transform can introduce runtime dependencies.

## Source maps

Maps can reveal:

```text
internal source paths
source content
repository structure
```

depending on configuration.

---

# 113. Source Map Security

Do not automatically publish source maps publicly if they expose sensitive source material.

Possible strategies:

```text
private source maps
public mappings without sourcesContent
artifact retention in error system
```

Choose based on observability and security needs.

---

# 114. Dependency Supply Chain Connection

From Chapter 67:

```text
compiler
plugin
preset
transformer
```

are dependencies too.

The build system is a supply-chain boundary.

---

# 115. Reproducible Compilation

For higher assurance:

```text
same source
+
same config
+
same compiler
+
same dependency graph
=
predictable output
```

Some organizations additionally verify generated artifacts and compiler provenance.

---

# 116. Performance Considerations

## Build time

AST transforms can be expensive.

## Runtime startup

Generated helper-heavy code can increase parsing/compilation work.

## Output size

Aggressive lowering can increase output size.

## Runtime performance

Native syntax can be faster or easier for modern engines to optimize.

Do not treat “older syntax” as automatically faster.

---

# 117. V8 and Generated Code

Modern V8 knows how to optimize JavaScript patterns well.

A compiler-generated workaround from years ago may be less optimal than native modern syntax.

Therefore:

```text
modern runtime
+
modern syntax
```

is often preferable when your target permits it.

---

# 118. Targeting Strategy

Instead of:

```text
target = oldest environment imaginable
```

use:

```text
minimum supported runtime
```

and transform only what that runtime lacks.

This reduces:

```text
code size
build time
semantic risk
```

---

# 119. Library Targeting Strategy

For a library:

```text
supported consumer matrix
```

may include:

```text
Node
browser
bundler
CJS
ESM
TypeScript
```

The compilation strategy should be based on actual support commitments.

---

# 120. Application Targeting Strategy

For an internal Node application:

```text
production Node = 24+
```

can justify:

```text
modern target
minimal transforms
native ESM
```

Do not preserve compatibility with runtimes you do not actually deploy.

---

# 121. Debugging Compilation Problems

Use:

```text
1. inspect source
2. inspect compiler config
3. inspect emitted JS
4. inspect emitted module format
5. inspect source map
6. run emitted artifact directly
7. compare runtime expectations
```

Do not debug generated behavior only from source.

---

# 122. Debugging Wrong Module Format

Check:

```text
package "type"
file extension
tsconfig module
Babel module transform
exports
```

Potential mismatch:

```text
package says ESM
compiler emits CJS
```

---

# 123. Debugging TypeScript Runtime Alias

Source:

```ts
import x from '@/utils.js';
```

TypeScript:

```text
passes
```

Node:

```text
fails
```

Ask:

```text
Was alias transformed?
Does package imports define it?
Does bundler own resolution?
```

Do not assume `paths` are runtime aliases.

---

# 124. Debugging Missing Types

Runtime works:

```text
JS ✅
```

Consumer TypeScript fails:

```text
types ❌
```

Inspect:

```text
.d.ts
exports
types condition/field
moduleResolution
declaration maps
```

---

# 125. Debugging Generated Code

When source behavior is suspicious:

```text
source.ts
↓
dist/file.js
```

compare them.

A transform may have:

```text
different helper
different closure
different module wrapper
```

The artifact is the runtime truth.

---

# 126. Debugging Source Maps

Check:

```text
sourceMappingURL
map file
sourceRoot
sources
sourcesContent
```

and verify the debugger/error platform understands them.

---

# 127. Common Misconceptions

## Misconception 1

> Transpilation makes unsupported APIs work.

No.

It can transform syntax; runtime APIs require polyfills or newer runtimes.

## Misconception 2

> TypeScript is just JavaScript plus types.

At source level, mostly.

But compiler features such as enums and parameter properties require runtime code generation.

## Misconception 3

> Node can fully execute any `.ts` file now.

False.

Node's built-in support is deliberately limited to erasable syntax and performs no type checking. citeturn236935search0

## Misconception 4

> `tsconfig.json` controls Node native TypeScript execution.

False.

Node's built-in type stripping ignores `tsconfig.json`. citeturn236935search0

## Misconception 5

> `target` means API compatibility.

No.

Syntax target and runtime APIs are separate.

## Misconception 6

> Babel is a TypeScript type checker.

No.

## Misconception 7

> `tsc` is a bundler.

No.

## Misconception 8

> If the compiler succeeds, the package is ready.

No.

The emitted artifacts still need runtime and consumer testing.

---

# 128. Common Mistakes

```text
[ ] Transforming for runtimes you do not support
[ ] Shipping compiler toolchains to production unnecessarily
[ ] Mixing ESM/CJS configurations
[ ] Assuming TypeScript paths are runtime aliases
[ ] Ignoring emitted JavaScript
[ ] Publishing broken declaration files
[ ] Treating source maps as automatic
[ ] Using too many Babel plugins
[ ] Applying polyfills globally without need
[ ] Ignoring compiler/plugin supply-chain risk
[ ] Testing source but not emitted artifact
[ ] Assuming native Node type stripping equals tsc
```

---

# 129. Comparison: `tsc` vs Babel vs Native Node

| Capability | `tsc` | Babel | Node type stripping |
|---|---:|---:|---:|
| Type check TS | ✅ | ❌ generally | ❌ |
| Remove types | ✅ | ✅ | ✅ |
| Transform JS syntax | ✅ | ✅ | ❌ intentional |
| Full TS feature transforms | ✅ | depends on plugins/tooling | ❌ |
| `tsconfig` | ✅ | ❌ unless separately integrated | ❌ |
| JS target lowering | ✅ | ✅ | ❌ |
| Declarations | ✅ | ❌ | ❌ |
| Source maps | ✅ | ✅ | not generated for stripping |
| Runtime TS support | via build/tooling | via tooling | ✅ subset |
| Bundling | ❌ | ❌ primarily | ❌ |
| Primary role | type system + emitter | transformation toolchain | lightweight runtime stripping |

---

# 130. Comparison: Babel vs SWC vs esbuild

| Concern | Babel | SWC | esbuild |
|---|---|---|---|
| AST transform ecosystem | very large | strong | strong but different |
| Speed focus | moderate | high | very high |
| Plugin ecosystem | extensive | smaller | more constrained |
| Type checking | ❌ | ❌ | ❌ |
| Bundling | external/adjacent | tooling support | ✅ |
| Fine-grained transforms | ✅ | ✅ | more opinionated |
| Ecosystem maturity | very high | high | very high |

Select based on requirements, not popularity.

---

# 131. Comparison: Runtime vs Build-Time TypeScript

| Strategy | Advantages | Risks |
|---|---|---|
| Node native stripping | simple, low overhead, minimal tooling | no type check, limited syntax, no tsconfig |
| `tsx` | convenient full TS support | runtime tooling dependency |
| `ts-node` | mature runtime TS workflows | startup/tooling complexity |
| precompile with `tsc` | predictable production artifacts | build step |
| type-check + fast emitter | fast builds | multiple-tool configuration |
| Babel TS transform | rich JS transform ecosystem | type checking separate |

---

# 132. Production Architecture Pattern

For a modern Node backend:

```text
src/**/*.ts
        │
        ├───────────────┐
        ▼               ▼
    tsc --noEmit     fast emitter
     type check          │
        │                ▼
        └───────────► dist/
                         │
                         ▼
                       Node
```

Then CI additionally validates:

```text
package exports
source maps
types
runtime behavior
```

---

# 133. Library Architecture Pattern

```text
src/**/*.ts
      │
      ├── TypeScript analysis
      ├── JavaScript emit
      ├── declaration emit
      └── source maps
              │
              ▼
            dist/
          ├── esm/
          ├── cjs/
          ├── types/
          └── *.map
              │
              ▼
         package exports
```

Only choose multiple outputs when consumers genuinely need them.

---

# 134. Node Native TS Architecture

For an internal modern-Node service:

```text
src/**/*.ts
      │
      ├── tsc --noEmit
      │
      └── Node runtime stripping
              │
              ▼
           execution
```

Conditions:

```text
erasable syntax
modern Node
explicit module rules
```

This can reduce build machinery.

---

# 135. Compiler Configuration Example — Node

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "verbatimModuleSyntax": true,
    "rewriteRelativeImportExtensions": true,
    "erasableSyntaxOnly": true,
    "noEmit": true
  }
}
```

This resembles the modern configuration Node's documentation recommends for lightweight native TypeScript execution. citeturn236935search0

---

# 136. Compiler Configuration Example — Library

```json
{
  "compilerOptions": {
    "module": "node18",
    "target": "ES2020",
    "strict": true,
    "verbatimModuleSyntax": true,
    "declaration": true,
    "sourceMap": true,
    "declarationMap": true,
    "rootDir": "src",
    "outDir": "dist"
  }
}
```

TypeScript's current library guidance provides this style of configuration and explains why these options matter for consumer compatibility. citeturn982887search0turn982887search5

Treat the exact Node/module target as a support-policy decision.

---

# 137. Compiler Configuration Example — Bundler

Bundler-driven TypeScript commonly uses:

```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "noEmit": true,
    "verbatimModuleSyntax": true
  }
}
```

TypeScript's current docs describe this as a bundler-oriented model and warn that bundler resolution can accept imports that would not work under native Node resolution. citeturn982887search0

---

# 138. Avoid Configuration Cargo Cult

Do not copy:

```json
{
  "module": "nodenext",
  "target": "esnext"
}
```

without asking:

```text
What runtime?
Who emits JS?
Who resolves modules?
Who bundles?
Who emits types?
```

Compiler settings are a model of the actual execution pipeline.

---

# 139. Build Contract

Document:

```text
source format:
TypeScript

module runtime:
ESM

target:
Node 24+

type checking:
tsc

JS emitter:
tsc/esbuild/etc.

bundling:
yes/no

source maps:
yes

declarations:
yes/no
```

This makes tool ownership explicit.

---

# 140. Build Failure Contract

CI should fail if:

```text
type check fails
emit fails
declaration emit fails
source-map generation fails
exports target missing
runtime artifact fails smoke test
```

---

# 141. Build Artifact Smoke Test

After compilation:

```bash
node dist/index.js
```

Do not rely only on:

```bash
tsc --noEmit
```

The executable artifact is what production runs.

---

# 142. Clean-Environment Test

Test the release artifact in:

```text
fresh directory
fresh node_modules
actual Node version
actual package manager
actual package metadata
```

This catches:

```text
workspace-only behavior
phantom dependencies
```

and packaging mistakes.

---

# 143. Compilation and Monorepos

Each package may choose:

```text
ESM
CJS
native TS
```

but organization-wide rules should prevent arbitrary mixes.

Use:

```text
package scope
tsconfig
exports
build pipeline
```

as a coherent contract.

---

# 144. Project References

TypeScript project references can structure large repositories:

```text
core
 ↓
domain
 ↓
api
```

This can improve build scalability and package boundaries.

The important design point:

> compilation boundaries should reflect architecture boundaries.

---

# 145. Incremental Compilation

Modern compilers can cache analysis/build information.

Benefits:

```text
faster local rebuilds
faster CI
```

Costs:

```text
cache management
stale-cache bugs
configuration complexity
```

Caches should be invalidated based on actual configuration/source changes.

---

# 146. Compiler Cache Security

Do not treat build caches as blindly trusted.

A compromised cache can theoretically inject unexpected artifacts.

For high-assurance builds:

```text
trusted cache
verified artifacts
isolated runners
```

may matter.

---

# 147. Transform Ordering

Plugin order can affect output.

Conceptually:

```text
plugin A
 ↓
plugin B
```

may differ from:

```text
plugin B
 ↓
plugin A
```

because A may create syntax that B expects, or vice versa.

Do not add transformations without understanding their ordering.

---

# 148. Babel Configuration Drift

A monorepo can accidentally have:

```text
package A → Babel config v1
package B → Babel config v2
```

while sharing:

```text
same compiler assumptions
```

This can produce inconsistent artifacts.

Centralize where possible.

---

# 149. Testing Transformation Semantics

For complex transforms, test:

```text
input behavior
expected runtime behavior
generated artifact
source maps
```

Do not test only snapshot text of generated code because harmless compiler output changes can invalidate snapshots.

Behavior is usually the more important contract.

---

# 150. Snapshotting Generated Code

Generated-code snapshots can be useful for compiler/library development.

But application builds should generally prioritize:

```text
runtime behavior
artifact validity
bundle size/performance
```

rather than exact generated formatting.

---

# 151. Semantic Regression Testing

When upgrading a compiler:

```text
old compiler
vs
new compiler
```

run:

```text
unit
integration
runtime smoke
performance
type tests
```

Compiler upgrades can alter generated semantics/performance.

---

# 152. Performance Benchmark

For a compiler change, measure:

```text
build time
artifact size
startup time
memory
hot-path throughput
```

Do not assume:

```text
faster compiler
=
faster application
```

Those are different dimensions.

---

# 153. Polyfill Strategy

Three broad choices:

### Native target

Require modern runtime.

### Selective polyfills

Load only needed APIs.

### Broad polyfill

Support older environments at larger cost.

Choose intentionally.

---

# 154. Polyfill Ownership

For each missing feature, decide:

```text
runtime provides it?
compiler transforms syntax?
polyfill provides API?
application code provides fallback?
```

Avoid multiple layers solving the same problem.

---

# 155. Compatibility Matrix

Create:

| Feature | Source | Target | Transform? | Polyfill? | Minimum runtime |
|---|---|---|---|---|---|
| optional chaining | modern | Node 18+ | no | no | Node 14+ |
| async/await | modern | modern Node | no | no | current target |
| new API | modern | old runtime | maybe | yes | API-dependent |
| TypeScript types | TS | JS | erase | no | any JS runtime |

The exact supported minimums must come from your actual compatibility contract.

---

# 156. Debugging Exercise — Erasable TypeScript

Given:

```ts
type UserId = string;

const id: UserId = 'u1';
```

Predict what Node native type stripping needs to do.

Then compare with:

```ts
enum Role {
  Admin,
  User,
}
```

Explain why one is erasable and the other requires code generation.

---

# 157. Debugging Exercise — `import type`

```ts
import { User } from './types.ts';

const user = getUser() as User;
```

Assume `User` is type-only.

Why can this fail under native stripping?

Rewrite correctly.

---

# 158. Debugging Exercise — TypeScript Path

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Source:

```ts
import { parse } from '@/parser.js';
```

TypeScript succeeds.

Node fails.

Explain the difference between:

```text
type checker resolution
```

and:

```text
runtime resolution
```

---

# 159. Debugging Exercise — Target Mismatch

Project says:

```text
Node 24+
```

but Babel is configured to target extremely old browsers.

### Task

Identify:

```text
unnecessary transforms
helper overhead
possible startup cost
```

and propose a modern target.

---

# 160. Debugging Exercise — Module Mismatch

Package:

```json
{
  "type": "module"
}
```

Compiler emits:

```text
CommonJS
```

### Task

Predict runtime behavior and identify the configuration mismatch.

---

# 161. Debugging Exercise — Broken Declarations

JS works:

```text
dist/index.js
```

but consumer TypeScript fails:

```text
Cannot find module
```

### Task

Inspect:

```text
.d.ts
exports
moduleResolution
```

and determine whether declarations point to valid runtime/type paths.

---

# 162. Debugging Exercise — Build Works, Publish Fails

Repository build passes.

After `npm pack`:

```text
runtime error
```

### Task

Compare:

```text
source tree
dist
package.json
published files
```

and identify the mismatch.

---

# 163. Code Review Exercise

Review:

```json
{
  "type": "module",
  "scripts": {
    "build": "babel src -d dist"
  },
  "devDependencies": {
    "@babel/core": "^8.0.0",
    "@babel/preset-env": "^8.0.0"
  }
}
```

```js
// src/index.js
export const version = '1.0.0';
```

Ask:

```text
Where is type checking?
What is the target?
What module format is emitted?
What are the package exports?
Are source maps enabled?
Are polyfills required?
Does package type match emitted format?
How is dist tested?
```

---

# 164. Implementation From Scratch

## Stage A — Guided

Build:

```text
src/
  index.ts
  math.ts
```

Compile with `tsc`.

Inspect:

```text
dist/
```

Compare source and output.

---

# 165. Stage B — Partially Guided

Add:

```text
declaration
sourceMap
module resolution
```

Then inspect:

```text
.js
.d.ts
.js.map
```

---

# 166. Stage C — No Reference

Build:

```text
TypeScript
 ↓
type check
 ↓
fast JS transform
 ↓
Node
```

Make module configuration coherent.

---

# 167. Stage D — Edge-Case Hardened

Add:

```text
ESM
CJS
JSON
dynamic import
type-only imports
worker
package exports
```

Test emitted artifacts directly.

---

# 168. Stage E — Production Grade

Add:

```text
CI build
clean install
package artifact
runtime smoke tests
type consumer tests
source-map validation
performance benchmark
compiler upgrade checks
```

---

# 169. Mastery Project — Compiler Comparison

Take the same source and compile it using:

```text
tsc
Babel
esbuild
SWC
```

Compare:

```text
output
size
build time
module format
source maps
runtime behavior
```

Write an architecture decision record.

---

# 170. Mastery Project — Native Node TypeScript

Build a small Node service using:

```text
.ts
Node native stripping
tsc --noEmit
```

Use only erasable syntax.

Then deliberately introduce:

```text
enum
parameter property
```

and observe the limitation.

---

# 171. Mastery Project — Library Build

Build a package with:

```text
ESM
CJS
types
source maps
exports
```

Test consumers from clean installation.

---

# 172. Mastery Project — Semantic Regression

Choose three features:

```text
async/await
optional chaining
private fields
```

Compile to an older target.

Compare:

```text
behavior
performance
artifact size
```

---

# 173. Principal Decision Framework

When choosing a transpilation/compilation strategy, evaluate:

| Dimension | Question |
|---|---|
| Correctness | Does transformed output preserve required semantics? |
| Performance | What do build and runtime costs look like? |
| Memory | Do helpers/metadata increase footprint? |
| Security | Which compiler/plugin code executes in CI? |
| Reliability | Can artifact generation fail before deployment? |
| Maintainability | How many configuration systems must engineers understand? |
| Scalability | Does the pipeline scale across many packages? |
| Observability | Are source maps and artifacts diagnosable? |
| Developer Experience | Are builds and local execution predictable? |
| Operational Complexity | How many stages/tools exist? |
| Future Change | Can targets and compilers evolve safely? |

---

# 174. Production Compilation Checklist

```text
[ ] runtime target is explicit
[ ] source module format is explicit
[ ] output module format is explicit
[ ] package "type" matches output
[ ] moduleResolution matches runtime/toolchain
[ ] type checking is explicit
[ ] emit ownership is explicit
[ ] polyfill strategy is explicit
[ ] source maps are intentional
[ ] declaration output is intentional
[ ] build tools are version-controlled
[ ] compiler plugins are reviewed
[ ] generated artifacts are tested directly
[ ] clean-install tests exist
[ ] package exports match artifacts
[ ] CI runs reproducible builds
[ ] production does not need unnecessary compiler tooling
```

---

# 175. Deep Mental Model

Keep this permanently:

```text
Source
  ↓
Parse
  ↓
AST
  ↓
Analyze
  ├── type analysis
  └── module analysis
  ↓
Transform
  ├── syntax lowering
  ├── module transform
  └── helper insertion
  ↓
Generate
  ├── JS
  ├── declarations
  └── source maps
  ↓
Package
  ↓
Runtime loader
  ↓
JS engine
  ↓
Execution
```

The most important distinction is:

> **A compiler can change the representation of a program, but the runtime still determines the final operational behavior.**

---

# 176. Chapter Connections

## Depends On

- Chapter 41 — Spec Architecture
- Chapter 42 — Abstract Operations
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability
- Chapter 66 — `package.json` and Module Resolution
- Chapter 67 — Dependency Management and Supply Chain

## Builds Toward

- Chapter 69 — Bundlers and Build Systems
- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 80 — Library Authoring
- Chapter 85 — Performance
- Chapter 89 — Code Review and Refactoring
- Chapter 90 — Modern ECMAScript Features
- Chapter 94 — Compatibility Engineering
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-Scale JavaScript Platform

## Related Concepts

- AST
- compiler
- transpiler
- parser
- code generator
- TypeScript
- Babel
- SWC
- esbuild
- target environments
- polyfills
- source maps
- declarations
- module transformation

## Why This Chapter Matters Later

Modern JavaScript production systems are often built from source code that the runtime never sees directly.

A principal engineer therefore needs to know:

```text
What did we write?
What did the compiler understand?
What did it transform?
What artifact was generated?
What module format is it?
What dependencies did the transform add?
What does Node actually execute?
```

Without this model, build failures look mysterious.

With it, the pipeline becomes inspectable.

---

# 177. Spaced Retrieval Plan

## Day 0

Explain:

```text
parse
analyze
transform
generate
```

without notes.

## Day 2

Explain:

```text
type checking
type stripping
syntax lowering
polyfills
```

## Day 7

Build:

```text
TS → JS
```

and inspect every artifact.

## Day 14

Compare:

```text
tsc vs Babel
```

for one feature.

## Day 30

Defend:

> Why should a modern Node application avoid unnecessary transpilation?

---

# 178. Dependency Graph

```text
source language
      │
      ▼
parser / AST
      │
      ├─────────────┐
      ▼             ▼
type analysis    transforms
      │             │
      └──────┬──────┘
             ▼
        code generator
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
     JS    .d.ts   maps
      │
      ▼
package artifacts
      │
      ▼
Node / browser / bundler
      │
      ▼
runtime engine
```

---

# 179. Completion Criteria

```text
[ ] Define transpilation
[ ] Define compilation
[ ] Distinguish build-time from JIT compilation
[ ] Explain parser
[ ] Explain AST
[ ] Explain semantic analysis
[ ] Explain type checking
[ ] Explain type stripping
[ ] Explain transformation
[ ] Explain lowering
[ ] Explain code generation
[ ] Explain polyfills
[ ] Explain source maps
[ ] Explain declaration files
[ ] Explain target environments
[ ] Explain TypeScript target
[ ] Explain TypeScript module
[ ] Explain moduleResolution
[ ] Explain nodenext
[ ] Explain bundler resolution
[ ] Explain verbatimModuleSyntax
[ ] Explain rewriteRelativeImportExtensions
[ ] Explain erasableSyntaxOnly
[ ] Explain Node native TypeScript support
[ ] Explain Node's lack of type checking for type stripping
[ ] Explain Node's tsconfig limitation
[ ] Explain import type
[ ] Explain Babel plugins
[ ] Explain Babel presets
[ ] Explain preset-env
[ ] Explain runtime vs compile-time polyfills
[ ] Explain helper generation
[ ] Explain module transformation
[ ] Explain ESM/CJS transform risks
[ ] Explain compiler/runtime mismatch
[ ] Explain library compilation
[ ] Explain application compilation
[ ] Explain dual-output testing
[ ] Explain build reproducibility
[ ] Explain compiler supply-chain risk
[ ] Debug emitted artifacts
[ ] Debug module mismatch
[ ] Debug declaration failures
[ ] Design a production compilation pipeline
[ ] Pass principal interview questions
```

---

# 180. Mastery Gate

You have mastered this chapter only when you can:

### Understand

Explain source transformation from parser to runtime artifact.

### Explain

Teach the difference between:

```text
type checking
type stripping
syntax transformation
polyfills
bundling
JIT compilation
```

### Predict

Predict what source code will become for a given target/configuration.

### Implement

Build a coherent TypeScript/JavaScript compilation pipeline.

### Debug

Diagnose failures by inspecting emitted artifacts rather than guessing from source.

### Apply

Choose an appropriate compilation strategy for a script, application, library, monorepo, or platform.

### Compare

Defend `tsc`, Babel, SWC, esbuild, and native Node type stripping for a concrete requirement.

### Defend

Explain compiler configuration and target choices to a principal architecture review.

---

# 181. Status

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

# 182. Chapter 68 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain compiler pipeline | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Compare type stripping vs tsc | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain target vs polyfill | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Inspect generated artifact | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design production pipeline | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal defense | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. What is transpilation?
2. What is compilation?
3. Why is type checking different from type stripping?
4. What does Node's native TypeScript support actually do?
5. Why doesn't Node type stripping use tsconfig?
6. What is an AST?
7. Why are syntax transforms not enough for missing APIs?
8. What does target mean?
9. What does moduleResolution mean?
10. Why can bundler resolution disagree with Node?
11. What are declaration files?
12. Why must emitted artifacts be tested?
```

---

# 183. Chapter 68 — Canonical References and Source Discipline

## Primary Node.js references

- Node.js TypeScript support  
  https://nodejs.org/api/typescript.html
- Node.js Modules API  
  https://nodejs.org/api/module.html
- Node.js CLI  
  https://nodejs.org/api/cli.html

Current Node documentation states that built-in TypeScript type stripping is stable, enabled by default in recent Node releases, performs no type checking, ignores `tsconfig.json`, and supports only erasable syntax. citeturn236935search0turn236935search5turn236935search7

## Primary TypeScript references

- TypeScript Modules  
  https://www.typescriptlang.org/docs/handbook/modules
- TypeScript module theory  
  https://www.typescriptlang.org/docs/handbook/modules/theory.html
- TypeScript module options  
  https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html
- TypeScript Node ESM/CommonJS reference  
  https://www.typescriptlang.org/docs/handbook/esm-node.html
- TypeScript compiler options  
  https://www.typescriptlang.org/docs/handbook/compiler-options

The current TypeScript documentation explains that `module`, `moduleResolution`, `verbatimModuleSyntax`, `target`, declarations, source maps, and related options should be selected based on the actual runtime/toolchain. citeturn982887search0turn982887search2turn982887search3turn982887search4

## Primary Babel references

- Babel overview  
  https://babeljs.io/docs/
- Babel usage  
  https://babeljs.io/docs/usage/
- Babel configuration  
  https://babeljs.io/docs/configuration/
- Babel CLI  
  https://babeljs.io/docs/babel-cli/
- Babel register  
  https://babeljs.io/docs/babel-register

Babel describes itself as a JavaScript compiler/toolchain for syntax transformation and compatibility-oriented output, with separate polyfill mechanisms. citeturn236935search2turn236935search3turn236935search10

---

# 184. Source Discipline

1. Distinguish language semantics from compiler transforms.
2. Distinguish compiler output from runtime behavior.
3. Distinguish type checking from syntax transformation.
4. Distinguish syntax transforms from polyfills.
5. Distinguish transpilation from bundling.
6. Verify the exact compiler/tool versions used in production.
7. Inspect emitted artifacts when debugging runtime behavior.
8. Verify package `"type"` and emitted module format together.
9. Test declarations separately for libraries.
10. Treat build tooling as supply-chain code.
11. Do not assume TypeScript configuration controls native Node execution.
12. Do not assume bundler resolution equals Node resolution.
13. Prefer current official documentation for version-sensitive compiler/runtime behavior.
14. Treat generated output as the runtime artifact that must be validated.

---

# 185. Chapter 68 — Completion Snapshot

## Compiler Theory

```text
[ ] Parser
[ ] AST
[ ] Analysis
[ ] Transform
[ ] Lowering
[ ] Code generation
[ ] Source maps
```

## TypeScript

```text
[ ] tsc
[ ] target
[ ] module
[ ] moduleResolution
[ ] nodenext
[ ] verbatimModuleSyntax
[ ] rewriteRelativeImportExtensions
[ ] erasableSyntaxOnly
[ ] declarations
```

## Node

```text
[ ] native TS type stripping
[ ] no type checking
[ ] tsconfig ignored by Node stripping
[ ] erasable syntax
[ ] .ts / .mts / .cts
```

## Babel

```text
[ ] plugins
[ ] presets
[ ] preset-env
[ ] target environments
[ ] syntax transforms
[ ] polyfill strategy
```

## Production

```text
[ ] artifact validation
[ ] source-map validation
[ ] runtime smoke tests
[ ] clean-install testing
[ ] build reproducibility
[ ] compiler supply-chain review
[ ] performance measurement
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

Compilation is the invisible layer between:

```text
what engineers write
```

and:

```text
what production executes
```

That layer can:

```text
erase types
lower syntax
transform modules
insert helpers
generate declarations
generate source maps
introduce runtime dependencies
change performance
change debugging behavior
```

A principal engineer therefore never asks only:

> “Does the source compile?”

The stronger questions are:

```text
What target are we compiling for?
Which transformations happen?
Which features remain native?
Which APIs require polyfills?
Which module format is emitted?
Does package metadata agree with the artifact?
What runtime actually executes the output?
Are declarations consistent with the JS?
Are source maps correct?
Could a compiler upgrade change semantics?
What compiler/plugin code executes in CI?
Can we reproduce the artifact?
```

The deepest lesson is:

> **The source file is not the production program. The compiled artifact plus its runtime environment is the production program.**

Once you understand that distinction, compilation stops being “build tooling.”

It becomes a first-class part of JavaScript runtime engineering.