
# Chapter 10 — Scope, Lexical Environments, and Identifier Resolution

> **Status:** `[+] Completed`  
> **Role in curriculum:** Establishes the formal name-resolution model behind JavaScript. Scope explains where bindings are visible; lexical environments and environment records explain how those bindings are represented semantically; identifier resolution explains how a source-level name finds the binding whose value is ultimately read or written. This chapter is the foundation for hoisting, TDZ, closures, execution contexts, modules, and `this` analysis.

---

## 1. Learning Objectives

By the end of this chapter, you should be able to:

- Define scope precisely.
- Explain lexical scope.
- Distinguish:
  - identifier,
  - binding,
  - scope,
  - environment,
  - value,
  - reference.
- Explain global, function, block, module, and nested lexical scope.
- Explain lexical environments.
- Explain environment records.
- Explain the outer-environment relationship.
- Explain identifier resolution.
- Explain lexical shadowing.
- Distinguish shadowing from reassignment.
- Explain how nested scopes find variables from outer scopes.
- Explain why JavaScript uses lexical rather than dynamic scope.
- Explain the difference between lexical visibility and runtime object properties.
- Explain the conceptual relationship between execution contexts and lexical environments.
- Explain how function parameters and local declarations participate in scope.
- Predict scope resolution in nested blocks and functions.
- Explain scope behavior in loops.
- Explain how `var`, `let`, and `const` interact with scope.
- Explain why `var` is function-scoped and `let`/`const` are block-scoped.
- Explain global environment distinctions at a foundational level.
- Prepare a precise mental model for closures.
- Diagnose bugs caused by shadowing, wrong binding resolution, and unexpected scope ownership.

---

# 2. Prerequisites

This chapter depends on:

- Chapter 01 — JavaScript, ECMAScript, and the Runtime Landscape.
- Chapter 02 — Values, Types, and the JavaScript Type System.
- Chapter 05 — Variables, Declarations, and Assignment.
- Chapter 08 — Control Flow and Iteration.
- Chapter 09 — Functions and First-Class Behavior.

The dependency chain is:

```text
bindings
   ↓
scope
   ↓
lexical environments
   ↓
identifier resolution
   ↓
closures
   ↓
execution contexts
```

---

# 3. What Is Scope?

Scope answers:

> **Where can an identifier be resolved?**

Consider:

```js
let name = "outer";

{
  let name = "inner";

  console.log(name);
}
```

Inside the block, the identifier:

```text
name
```

resolves to the inner binding.

Outside:

```js
console.log(name);
```

the identifier resolves to the outer binding.

Scope therefore describes the visibility and resolution region of bindings.

---

# 4. Scope Is Not a Value

This distinction is fundamental.

Consider:

```js
let x = 10;
```

Here:

```text
x
```

is an identifier.

The binding associated with `x` has:

```text
scope
```

and:

```text
current value = 10
```

Scope is not:

```text
10
```

Scope is part of the semantic environment in which the identifier can be resolved.

---

# 5. Lexical Scope

JavaScript uses **lexical scoping**.

This means identifier resolution is determined primarily by the structure of the source code.

Consider:

```js
let x = "outer";

function test() {
  return x;
}
```

The function `test` is written inside the lexical region where `x` is visible.

Therefore its body can resolve `x`.

Now:

```js
function createTest() {
  let x = "inner";

  return function test() {
    return x;
  };
}
```

The returned function resolves `x` according to where it was defined, not according to where it is eventually called.

That observation becomes the foundation of closures.

---

# 6. Lexical Scope vs Dynamic Scope

### Lexical scope

Names are resolved according to source-code structure.

### Dynamic scope

Names would be resolved according to the active runtime call stack.

JavaScript uses lexical scope.

This matters:

```js
let x = "global";

function first() {
  let x = "first";

  return second();
}

function second() {
  return x;
}
```

When:

```js
first();
```

runs, `second()` resolves `x` according to where `second` was defined.

It does not dynamically search the caller's local variables and use:

```text
x = "first"
```

Therefore the result is based on the lexical environment of `second`, not the dynamic caller chain.

---

# 7. Why Lexical Scope Matters

Lexical scope makes code easier to reason about because the source structure determines the potential binding relationships.

You can inspect:

```js
function outer() {
  const x = 10;

  return function inner() {
    return x;
  };
}
```

and determine that `inner` can access:

```text
x
```

without knowing which function will eventually call `inner`.

This supports:

```text
closures
modules
encapsulation
static reasoning
tooling
optimization
```

---

# 8. Scope Chain — Mental Model

A useful simplified model:

```text
Current scope
      ↓
Outer scope
      ↓
Next outer scope
      ↓
Global scope
```

If an identifier is not found in the current scope, resolution proceeds outward.

Example:

```js
const a = 1;

function outer() {
  const b = 2;

  function inner() {
    const c = 3;

    console.log(a, b, c);
  }

  inner();
}
```

Inside `inner`:

```text
c → current scope
b → outer scope
a → outermost/global scope
```

---

# 9. The Scope Chain Is Not Literally One Array

The phrase:

```text
scope chain
```

is useful teaching terminology.

But the ECMAScript specification uses more precise concepts such as:

```text
Lexical Environment
Environment Record
outer environment reference
```

A linked environment structure is a better mental model than imagining one physical array of variable names.

---

# 10. Lexical Environment

A lexical environment is a specification-level mechanism that associates:

```text
an environment record
+
an outer lexical environment
```

Conceptually:

```text
Lexical Environment
├── Environment Record
└── [[OuterEnv]]
```

This gives JavaScript a chain through which identifiers can be resolved.

A simplified diagram:

```text
Environment A
    │
    └── outer → Environment B
                    │
                    └── outer → Environment C
```

---

# 11. Environment Record

An environment record manages bindings.

Conceptually:

```text
Environment Record
------------------
name → value/state
name → value/state
name → value/state
```

The specification distinguishes several kinds of environment records.

Important categories include:

```text
Declarative Environment Record
Function Environment Record
Object Environment Record
Global Environment Record
Module Environment Record
```

You do not need to memorize every internal method yet.

The important insight is:

> Scope is implemented semantically through environments and records that manage bindings.

---

# 12. Declarative Environment Records

Declarative environment records represent bindings that are not simply properties of an object.

Examples include bindings created by:

```js
let
const
class
catch
```

and other lexical constructs.

For example:

```js
{
  let x = 10;
}
```

The binding:

```text
x
```

is represented through lexical/declarative environment machinery.

This is different from saying:

```text
x becomes obj.x
```

---

# 13. Object Environment Records

Some environments use objects as the underlying binding mechanism for lookup behavior.

This is particularly relevant to:

```text
with
```

and aspects of global script semantics.

Modern production code should avoid relying on `with`.

The important conceptual distinction is:

```text
environment binding
```

and:

```text
ordinary object property
```

are not always the same thing.

---

# 14. Function Environment Records

Function calls establish function-specific environment behavior.

Parameters:

```js
function add(a, b) {
  ...
}
```

and certain function-related bindings are represented through function environment semantics.

Function environment records also participate in:

```text
this
super
new.target
```

for relevant function forms.

This creates the bridge from scope to invocation semantics.

---

# 15. Global Environment Record

The global environment is special because it coordinates global declaration behavior.

It can represent distinctions between:

```text
object-backed global declarations
```

and:

```text
declarative global lexical bindings
```

This explains why top-level:

```js
var
```

and:

```js
let
const
```

can differ in relation to the global object.

---

# 16. Module Environment Record

Modules introduce module-specific binding semantics.

Example:

```js
const secret = 123;

export { secret };
```

The binding belongs to the module environment.

It does not simply become:

```js
globalThis.secret
```

Modules therefore provide a strong mechanism for avoiding global namespace pollution.

---

# 17. Outer Environment Reference

Every lexical environment can conceptually point to another outer environment.

Example:

```js
const a = 1;

{
  const b = 2;

  {
    const c = 3;

    console.log(a, b, c);
  }
}
```

Conceptually:

```text
Environment #3
  c → 3
  outer → Environment #2

Environment #2
  b → 2
  outer → Environment #1

Environment #1
  a → 1
```

Identifier resolution walks this chain outward.

---

# 18. Identifier Resolution

Suppose:

```js
const x = 10;

function test() {
  return x;
}
```

When the engine evaluates:

```js
x
```

inside `test`, it must resolve the identifier.

Conceptually:

```text
Current environment
  ↓
Is x here?
  ↓ no
Outer environment
  ↓
Is x here?
  ↓ yes
Use that binding
```

This is identifier resolution.

---

# 19. Resolution Is Name-Based, Not Value-Based

Consider:

```js
const x = 10;
const y = x;
```

When evaluating:

```js
x
```

the runtime does not search for:

```text
the value 10
```

It searches for:

```text
the binding associated with identifier x
```

Then it obtains the value from that binding.

This distinction matters greatly when multiple bindings have the same value.

---

# 20. Reference Records — Preview

At specification level, evaluating an identifier can produce a **Reference**-like semantic record that remembers information needed for later operations.

Conceptually:

```text
identifier
   ↓
Reference
   ↓
binding/environment information
   ↓
GetValue / PutValue
```

This becomes important for:

```text
assignments
property access
this
delete
function calls
```

Chapter 42 will formalize these abstract operations.

---

# 21. Shadowing

Shadowing occurs when an inner scope creates a binding with the same identifier as an outer scope.

Example:

```js
let value = "outer";

{
  let value = "inner";

  console.log(value);
}
```

Inside the block:

```text
value → inner binding
```

The outer binding still exists.

It is simply hidden by normal identifier resolution in that region.

---

# 22. Shadowing Is Not Reassignment

Compare:

```js
let x = "outer";

{
  let x = "inner";
}
```

with:

```js
let x = "outer";

x = "inner";
```

The first creates:

```text
two bindings
```

The second updates:

```text
one binding
```

Therefore:

```text
shadowing
≠
reassignment
```

This distinction should become automatic when debugging.

---

# 23. Shadowing Across Function Boundaries

Example:

```js
const value = "global";

function outer() {
  const value = "outer";

  function inner() {
    const value = "inner";

    return value;
  }

  return inner();
}
```

Inside `inner`:

```text
value
 ↓
inner binding
```

The search stops at the first matching binding.

The outer and global values remain present but are shadowed in that region.

---

# 24. Shadowing Without Mutation

Consider:

```js
let x = 1;

function test() {
  let x = 2;
}

test();

console.log(x);
```

The outer `x` remains:

```text
1
```

because the inner declaration created a different binding.

A common debugging mistake is to say:

> “The function changed x to 2.”

It did not.

It created a new `x`.

---

# 25. Block Scope

Blocks create lexical environments for declarations such as:

```js
let
const
class
```

Example:

```js
{
  const secret = 123;
}

console.log(secret);
```

The outer code cannot resolve:

```text
secret
```

because the binding exists only in the inner lexical environment.

---

# 26. Function Scope

Functions create their own function-level scope.

Example:

```js
function test() {
  var x = 10;
}

console.log(x);
```

The outer code cannot resolve:

```text
x
```

because the `var` binding belongs to the function environment.

---

# 27. `var` Inside Blocks

Consider:

```js
function test() {
  if (true) {
    var x = 10;
  }

  console.log(x);
}
```

This works because `var` is function-scoped.

Conceptually:

```text
function environment
    x → 10
```

The `if` block does not create a separate `var` binding.

---

# 28. `let` Inside Blocks

Compare:

```js
function test() {
  if (true) {
    let x = 10;
  }

  console.log(x);
}
```

The final access cannot resolve the block-scoped `x`.

Conceptually:

```text
function environment
   │
   └── block environment
          x → 10
```

Once outside the block, `x` is not visible.

---

# 29. Nested Block Scope

Example:

```js
let a = 1;

{
  let b = 2;

  {
    let c = 3;

    console.log(a, b, c);
  }
}
```

Inside the innermost block:

```text
c → current block
b → outer block
a → outer/global scope
```

Resolution moves outward until a matching binding is found.

---

# 30. Closures Are Built on Lexical Environments

Consider:

```js
function outer() {
  const x = 10;

  return function inner() {
    return x;
  };
}

const fn = outer();
```

When:

```js
fn();
```

runs, `x` is still accessible.

Why?

The returned function was created in the lexical environment where `x` existed.

This is the foundational relationship:

```text
function
+
lexical environment
=
closure capability
```

Chapter 13 will explain the lifetime and memory implications in depth.

---

# 31. Lexical Scope Explains Closures

If JavaScript used dynamic scope, this would behave very differently.

With lexical scope:

```js
function make() {
  let value = 42;

  return function () {
    return value;
  };
}
```

the returned function remembers its lexical environment relationship.

This is why closures support:

```text
private state
factories
callbacks
memoization
event handlers
module patterns
```

---

# 32. Function Parameters Are Scoped Bindings

Example:

```js
function add(a, b) {
  return a + b;
}
```

Inside the invocation:

```text
a
b
```

are bindings.

Outside the function, these parameter identifiers are not automatically visible.

Therefore:

```js
console.log(a);
```

outside the function fails unless another binding named `a` exists.

---

# 33. Local Variables and Parameters Share Lexical Structure

Example:

```js
function test(a) {
  const b = 10;

  return a + b;
}
```

The function invocation has access to:

```text
parameter a
local binding b
```

Identifier resolution can find both.

Their exact environment-record relationship becomes more nuanced when default parameters are involved, but the key principle remains:

```text
function invocation
→ parameter/local bindings
```

---

# 34. Default Parameters and Scope

Consider:

```js
function f(a = 10) {
  const b = 20;
  return a + b;
}
```

The default parameter is initialized before the function body executes.

Parameter initialization can therefore introduce bindings and environment behavior that matter independently of body execution.

This becomes important for cases such as:

```js
function f(a = b, b = 10) {}
```

where parameter initialization order affects whether `b` is available.

---

# 35. Scope in `for` Loops

Example:

```js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

The loop introduces lexical scope.

In relevant loop semantics, successive iterations can receive distinct per-iteration bindings.

This is what allows:

```js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}
```

to preserve distinct `i` values.

This behavior is directly connected to closures.

---

# 36. Scope in `for...of`

Example:

```js
for (const value of values) {
  console.log(value);
}
```

The loop variable is lexical.

Each iteration has appropriate iteration-binding semantics.

This makes:

```js
const
```

perfectly valid in a loop even though the binding cannot be reassigned within a given iteration.

---

# 37. Scope and `catch`

Modern JavaScript supports a block-like binding for catch parameters:

```js
try {
  ...
} catch (error) {
  console.log(error);
}
```

The binding:

```text
error
```

is scoped to the catch clause.

This is another example of a language construct creating a lexical environment.

---

# 38. Scope and Classes

Class declarations are block-scoped lexical declarations.

Example:

```js
{
  class User {}
}

console.log(User);
```

The outer access fails because the class binding belongs to the inner lexical scope.

Classes also have internal name-binding behavior during class evaluation.

Those details will be covered in the object/class chapters.

---

# 39. Scope and Modules

Modules create their own top-level lexical environment.

Example:

```js
const databaseUrl = "...";

export function connect() {
  return databaseUrl;
}
```

The function can resolve:

```text
databaseUrl
```

because it is lexically defined within the module environment.

Other modules do not automatically receive direct access to that binding.

They use imports.

This creates strong boundaries for application architecture.

---

# 40. Imports Are Live Bindings

Consider conceptually:

```js
// module-a.js
export let count = 0;
```

and:

```js
// module-b.js
import { count } from "./module-a.js";
```

The imported name is not simply a copied snapshot in the ordinary language model.

ES modules use **live bindings**.

This means the imported binding reflects the exported binding's current value subject to module semantics.

This becomes important in Chapter 64.

---

# 41. Lexical Scope and Tooling

Understanding lexical environments helps explain why developer tools can show:

```text
local variables
closure variables
global variables
```

in stack frames.

When paused inside a function, the debugger can expose values from:

```text
current scope
outer lexical scopes
global/module scope
```

This is a reflection of the language/runtime model.

---

# 42. Scope and Static Analysis

Because JavaScript is lexically scoped, tools can often detect:

```text
undefined identifiers
shadowed variables
unused bindings
unreachable references
module dependencies
```

using static analysis.

Linters and compilers can reason about much of this without executing the program.

This is one of the practical advantages of lexical scoping.

---

# 43. Lexical Scope Does Not Mean Everything Is Static

JavaScript is still dynamic.

At runtime:

```js
obj[key]
```

can determine a property dynamically.

Objects can mutate.

Functions can be replaced.

Proxies can intercept operations.

`eval` can complicate static reasoning.

So:

```text
lexical scope
≠
statically typed or fully statically determined language
```

Lexical scope specifically describes how identifier bindings are resolved.

---

# 44. `eval` as a Scope Complication

Direct `eval` can interact with the surrounding lexical environment under defined rules.

Example:

```js
function test() {
  let x = 10;
  eval("console.log(x)");
}

test();
```

This can access the surrounding lexical state.

`eval` complicates optimization, security, and static analysis.

Modern production code should avoid dynamic `eval` unless there is a specific, well-understood reason.

---

# 45. `with` as a Scope Complication

Historically, JavaScript provides:

```js
with (obj) {
  ...
}
```

It changes name-resolution behavior using object properties.

This makes code harder to:

```text
analyze
optimize
secure
understand
```

Strict mode disallows `with`.

Production code should not use it.

Its main relevance today is understanding legacy language semantics.

---

# 46. Scope vs Property Lookup

Compare:

```js
const x = 10;
```

with:

```js
const obj = { x: 10 };
```

These are different mechanisms.

Identifier:

```js
x
```

uses lexical environment resolution.

Property access:

```js
obj.x
```

uses object property lookup.

A property named `x` does not become a lexical variable named `x`.

---

# 47. Global Identifier vs Global Property

Consider browser classic-script contexts where:

```js
var x = 10;
```

can participate in global-object behavior.

The identifier:

```js
x
```

and property:

```js
globalThis.x
```

may be connected under applicable global declaration rules.

But:

```js
let y = 20;
```

creates a global lexical binding that does not behave as an ordinary global object property.

Therefore:

```text
global scope
≠
global object
```

as a universal identity.

---

# 48. Environment vs Object Property

This distinction is fundamental:

```text
environment:
  x → 10

object:
  property x → 10
```

They can sometimes interact in global semantics, but they are not conceptually identical structures.

This distinction explains why:

```js
delete x
```

and:

```js
delete obj.x
```

have very different meanings.

---

# 49. Identifier Resolution and Shadowing

Consider:

```js
const x = "A";

function outer() {
  const x = "B";

  return function inner() {
    const x = "C";
    return x;
  };
}
```

Resolution inside `inner`:

```text
x?
 ↓
inner environment
 ↓
found C
```

The search stops immediately.

It does not continue toward:

```text
B
A
```

because the nearest binding shadows the outer ones.

---

# 50. Assignment Uses Resolved Binding

Consider:

```js
let x = 10;

function test() {
  x = 20;
}

test();
```

Inside `test`, resolution finds the outer `x`.

The assignment updates that binding.

There is no local `x`.

Now:

```js
function test() {
  let x = 20;
}
```

creates a local binding.

The outer binding is untouched.

This is one of the most important practical scope distinctions.

---

# 51. Assignment Does Not Automatically Create Local Scope

A common misconception:

```js
function test() {
  x = 10;
}
```

does not mean:

```text
create local x
```

If `x` is unresolved, behavior depends on strictness and source context.

In strict/module code, it results in an error.

Never rely on accidental global creation.

---

# 52. Scope and Strict Mode

Strict mode eliminates or restricts several legacy behaviors.

Modern modules are strict by default.

This helps make scope behavior more predictable by rejecting certain accidental or dangerous patterns.

Production JavaScript should normally prefer:

```text
modules
+
strict semantics
+
explicit declarations
```

over legacy sloppy-mode behavior.

---

# 53. Scope and Reentrancy

Consider:

```js
let shared = 0;

function increment() {
  shared++;
}
```

The binding:

```text
shared
```

is shared across calls.

By contrast:

```js
function increment() {
  let local = 0;
  local++;
  return local;
}
```

creates fresh local binding state per invocation.

This distinction matters for:

```text
concurrency
reentrancy
testing
closures
request isolation
```

---

# 54. Scope and Request Isolation

Dangerous server pattern:

```js
let currentUser;

function handleRequest(user) {
  currentUser = user;

  // asynchronous work
}
```

Multiple requests can interact with the same shared binding.

Better designs use:

```text
request-local variables
parameters
AsyncLocalStorage where appropriate
explicit context objects
```

Scope should match data lifetime.

This principle becomes important in Node.js architecture.

---

# 55. Scope and State Ownership

A production design should ask:

```text
Who owns this binding?
Who can access it?
Who can mutate it?
How long should it live?
```

Examples:

```text
function-local state
request-scoped state
module state
process-global state
external persistent state
```

A variable's scope is therefore part of architecture, not merely syntax.

---

# 56. Memory Implications of Scope

Bindings can keep values reachable.

If a long-lived function captures:

```js
const largeObject = ...
```

then the associated lexical environment may remain reachable through that function.

This can influence garbage collection.

Therefore:

```text
scope lifetime
+
function lifetime
+
captured state
```

can affect memory.

Chapter 13 and Chapter 45 will explore this fully.

---

# 57. Security Implications of Scope

Reducing the visibility of sensitive data reduces accidental exposure.

Prefer:

```js
function authenticate(credentials) {
  const token = createToken(credentials);
  ...
}
```

over:

```js
globalThis.token = ...
```

A secret placed in a broad shared scope may be accessible to unrelated code.

Scope is therefore one tool for reducing authority and exposure.

---

# 58. Common Misconceptions

### “Scope means where the variable is stored.”

Incomplete.

Scope describes where a binding is visible/resolvable.

### “JavaScript uses dynamic scope.”

False.

JavaScript uses lexical scoping.

### “The scope chain is the call stack.”

False.

Lexical scope follows source structure; the call stack describes active execution.

### “Shadowing changes the outer variable.”

False.

It creates or selects a different inner binding.

### “Object properties are the same as lexical variables.”

False.

Property lookup and identifier resolution are distinct mechanisms.

### “`let` doesn't exist until its line runs.”

Incomplete.

The binding is created during declaration instantiation; access before initialization is restricted by TDZ semantics.

### “Functions only see variables from the caller.”

False.

Functions use lexical scope based on where they were defined.

### “A block creates scope for `var`.”

No.

`var` is function-scoped.

### “Every global binding is a `globalThis` property.”

False.

Global lexical declarations are distinct from object-backed global properties.

---

# 59. Common Mistakes

### Mistake: confusing scope with lifetime

Visibility and lifetime are related but not identical concepts.

### Mistake: confusing shadowing with reassignment

Two bindings can share the same name.

### Mistake: using global variables for request-specific data

This creates shared mutable state.

### Mistake: treating object properties as lexical variables

They use different lookup systems.

### Mistake: relying on accidental globals

Use explicit declarations.

### Mistake: ignoring loop binding semantics

`let` and `var` can produce radically different closure behavior.

### Mistake: using `eval` to manipulate local scope

This harms clarity, tooling, and security.

### Mistake: using `with`

Avoid entirely in modern code.

---

# 60. Comparison Table — Scope Categories

| Scope | Typical source | Main property |
|---|---|---|
| Global | script/module top level | outermost application environment |
| Module | ES module | module-private top-level bindings |
| Function | function body/parameters | function-local bindings |
| Block | `{}` with lexical declarations | `let`/`const`/`class` visibility |
| Catch | `catch (error)` | catch binding scope |
| Nested lexical | blocks/functions inside scopes | resolves outward |

These categories can overlap structurally. A function body itself also contains lexical environments relevant to its declarations.

---

# 61. Comparison Table — `var`, `let`, `const`

| Declaration | Scope behavior | Reassignment | TDZ behavior |
|---|---|---:|---|
| `var` | function-scoped | Yes | No lexical TDZ |
| `let` | block/lexical | Yes | Yes |
| `const` | block/lexical | No | Yes |

This table is a foundation; exact declaration-instantiation behavior is covered in Chapter 11.

---

# 62. Execution Walkthrough — Nested Resolution

Consider:

```js
const a = 1;

function outer() {
  const b = 2;

  function inner() {
    const c = 3;

    return a + b + c;
  }

  return inner();
}
```

Inside `inner`, resolve:

```text
c
```

Current environment:

```text
c → 3
```

Resolve:

```text
b
```

Current environment does not contain `b`.

Move outward:

```text
outer environment
b → 2
```

Resolve:

```text
a
```

Not in `inner`.

Not in `outer`.

Move outward:

```text
global/module environment
a → 1
```

Result:

```text
6
```

This is lexical resolution in action.

---

# 63. Execution Walkthrough — Shadowing

```js
const x = "A";

function test() {
  const x = "B";

  return x;
}
```

Inside `test`:

```text
x
 ↓
test environment
 ↓
x exists
 ↓
use "B"
```

The outer `x` is never consulted.

The outer binding still exists.

This is shadowing.

---

# 64. Execution Walkthrough — Reassignment

```js
let x = "A";

function test() {
  x = "B";
}

test();
```

Inside `test`:

```text
x
 ↓
current environment
 ↓
not found
 ↓
outer environment
 ↓
x found
 ↓
assignment updates outer binding
```

Final:

```text
x = "B"
```

No inner binding was created.

---

# 65. Execution Walkthrough — Separate Local Binding

```js
let x = "A";

function test() {
  let x = "B";
  return x;
}

test();

console.log(x);
```

Inside:

```text
x → local "B"
```

Outside:

```text
x → outer "A"
```

Two bindings exist.

The outer value remains unchanged.

---

# 66. Execution Walkthrough — Closure Foundation

```js
function outer() {
  const value = 42;

  return function inner() {
    return value;
  };
}

const fn = outer();
```

When `outer()` returns:

```text
inner function value
+
lexical relationship to outer environment
```

remain relevant.

Then:

```js
fn();
```

resolves:

```text
value
```

through its lexical environment chain.

This is the foundation of closure behavior.

---

# 67. Execution Walkthrough — `var` vs `let`

```js
function test() {
  if (true) {
    var a = 1;
    let b = 2;
  }

  return [a, typeof b];
}
```

Conceptually:

```text
a
→ function-scoped
→ visible after block

b
→ block-scoped
→ not visible after block
```

Therefore:

```text
a → 1
b → not resolvable
```

The `typeof b` behavior itself must be interpreted carefully because unresolved identifiers have special `typeof` semantics; this is a useful edge case for Chapter 06/07 and should not be confused with ordinary scope visibility.

---

# 68. Implementation From Scratch — Scope Chain

Build a tiny scope simulator.

Represent:

```js
{
  bindings: new Map(),
  outer: parent
}
```

Implement:

```text
declare(name, value)
resolve(name)
assign(name, value)
```

Example:

```text
global
  x → 1

function
  y → 2

block
  z → 3
```

Then resolve:

```text
x
y
z
```

from the innermost environment.

The goal is not to reproduce ECMAScript.

The goal is to make the lexical lookup algorithm tangible.

---

# 69. Implementation — Shadowing

Extend the simulator so that:

```text
inner.declare("x", 2)
```

does not mutate:

```text
outer.x = 1
```

Then implement:

```text
resolve("x")
```

so that the nearest binding wins.

This gives a concrete model of shadowing.

---

# 70. Implementation — Assignment Resolution

Add:

```text
assign(name, value)
```

with behavior:

```text
find nearest existing binding
→ update it
```

Then compare:

```text
declare("x", ...)
```

with:

```text
assign("x", ...)
```

This mirrors the conceptual distinction between:

```text
new binding
```

and:

```text
existing binding update
```

---

# 71. Implementation Progression

### Guided

Implement:

```text
Environment
declare
resolve
assign
```

### Partially Guided

Add:

```text
nested environments
shadowing
errors for unresolved assignment
```

### No Reference

Build a small interpreter for variable declarations and identifier expressions.

Support:

```text
let
const
assignment
identifier lookup
blocks
```

### Edge-Case Hardened

Test:

```text
shadowing
nested blocks
function scope
unresolved names
reassignment
const assignment
```

### Production-Grade

Add:

```text
clear diagnostics
scope visualization
tests
documentation
```

---

# 72. Debugging Exercises

### Exercise 1

Explain:

```js
const x = 1;

function test() {
  const x = 2;
  return x;
}
```

Which `x` is returned?

### Exercise 2

Explain:

```js
let x = 1;

function test() {
  x = 2;
}

test();
console.log(x);
```

Which binding changes?

### Exercise 3

Explain:

```js
function test() {
  if (true) {
    var x = 1;
  }

  return x;
}
```

Why is `x` visible?

### Exercise 4

Explain:

```js
function test() {
  if (true) {
    let x = 1;
  }

  return x;
}
```

Why is `x` not visible?

### Exercise 5

Explain the lexical-scope result:

```js
let x = "global";

function first() {
  let x = "first";
  return second();
}

function second() {
  return x;
}
```

### Exercise 6

Find the hidden shared-state bug:

```js
let currentUser;

function handleRequest(user) {
  currentUser = user;
  doAsyncWork();
}
```

### Exercise 7

Explain why:

```js
const obj = { x: 10 };
```

does not make `x` a lexical binding.

---

# 73. Code Review Exercise — Scope Ownership

Review:

```js
let cache = {};

function getUser(id) {
  cache[id] = loadUser(id);
  return cache[id];
}
```

Ask:

```text
Who owns cache?
Should it be shared across requests?
Can it grow without bound?
Is it safe for concurrent access?
Should it be module-scoped?
Should it have eviction?
Does it expose mutable global/module state?
```

Scope is an architectural decision because it defines who can access shared state.

---

# 74. Code Review Exercise — Shadowing

Review:

```js
const config = loadConfig();

function start(config) {
  if (config.enabled) {
    const config = normalizeConfig(config);
    run(config);
  }
}
```

The repeated `config` identifier creates multiple bindings.

This may be intentional, but it increases cognitive load.

A clearer version may use:

```js
function start(config) {
  if (config.enabled) {
    const normalizedConfig = normalizeConfig(config);
    run(normalizedConfig);
  }
}
```

The lesson is not:

> “Never shadow.”

The lesson is:

> Use shadowing deliberately and make the binding transition obvious.

---

# 75. Code Review Exercise — Global Request State

Review:

```js
let userId;

async function requestHandler(request) {
  userId = request.userId;
  await serviceCall();

  return getDataForUser(userId);
}
```

This is dangerous in a server handling overlapping requests because:

```text
userId
```

is shared across invocations.

The correct design is generally to keep request-specific data:

```text
inside the request invocation
```

or use an explicit request context mechanism designed for the runtime.

---

# 76. Security Considerations

Lexical scope can reduce accidental access to sensitive values.

For example:

```js
function authenticate(credentials) {
  const secret = loadSecret();

  return verify(credentials, secret);
}
```

The secret is not made process-global merely because the function needs it.

However, scope is not a complete security boundary.

Closures, module exports, object references, debugging tools, and process-level authority still matter.

Use scope to reduce exposure, not as a substitute for a security architecture.

---

# 77. Security — Global Mutable State

Global mutable values create broad authority and shared state.

Avoid:

```js
globalThis.currentUser = user;
```

for request-specific state.

Potential consequences include:

```text
cross-request data leakage
race conditions
authorization confusion
test contamination
```

Scope and lifetime should match the security sensitivity of the data.

---

# 78. Performance Considerations

Lexical scope itself is not normally a performance problem.

Modern engines optimize common lexical access patterns aggressively.

Performance concerns can arise from:

```text
dynamic scope features
eval
with
large closure environments
retained references
deoptimization
```

These are implementation/workload considerations.

Do not rewrite clean lexical code for imagined scope-access micro-costs without measurement.

---

# 79. Memory Considerations

A long-lived closure can keep an outer lexical environment reachable.

Example:

```js
function createHandler(largeObject) {
  return () => largeObject.id;
}
```

If the returned function stays alive, the environment may remain reachable.

Therefore:

```text
scope
+
closure
+
lifetime
```

can affect memory retention.

Chapter 13 will formalize this relationship.

---

# 80. Why Scope Is Foundational

The following concepts depend directly on scope:

```text
hoisting
TDZ
closures
function invocation
modules
imports/exports
async callbacks
event handlers
private state
```

Even `this` analysis benefits from separating:

```text
lexical variable resolution
```

from:

```text
invocation-time this binding
```

A strong JavaScript engineer should never use “scope” as a vague synonym for “where the variable lives.”

---

# 81. Common Misconceptions

### “Scope means memory location.”

No. It describes binding visibility/resolution.

### “The caller determines which variables a function sees.”

False for ordinary lexical variables.

### “The scope chain is the call stack.”

False.

### “Shadowing overwrites the outer variable.”

False.

### “A property and a variable are the same thing.”

False.

### “`var` is globally scoped.”

False. `var` is function-scoped; global `var` has special global-environment behavior.

### “`let` is only scoped to the line where it appears.”

False. It is scoped to the relevant lexical block/environment.

### “Closures happen because nested functions are special.”

Incomplete. The critical fact is preserved lexical access to an outer environment.

---

# 82. Common Mistakes

- Using `var` because block scope is misunderstood.
- Creating request-specific state at module/global scope.
- Confusing shadowing with mutation.
- Treating property lookup as identifier lookup.
- Using `eval` to modify lexical state.
- Assuming lexical scope means values cannot change.
- Ignoring how loops create lexical iteration bindings.
- Capturing large objects in long-lived functions without considering lifetime.
- Assuming module bindings are ordinary global variables.
- Assuming all global bindings are `globalThis` properties.

---

# 83. Interview Questions

## Junior

1. What is scope?
2. What is lexical scoping?
3. What is block scope?
4. What is function scope?
5. What is variable shadowing?
6. What is the difference between shadowing and reassignment?

## Mid-Level

7. What is a lexical environment?
8. What is an environment record?
9. How does identifier resolution work?
10. Why does a nested function see variables from where it was defined?
11. Why is `var` function-scoped?
12. Why are `let` and `const` block-scoped?
13. What is the difference between lexical scope and the call stack?
14. Why are object properties different from lexical variables?

## Senior

15. Explain the outer-environment relationship.
16. Explain function parameter scope.
17. Explain scope in `for` loops and its relationship to closures.
18. Explain global environment differences.
19. Explain module scope and live bindings.
20. Why do `eval` and `with` complicate scope analysis?
21. How can scope design affect memory?
22. How can global state create request-isolation problems?

## Staff / Principal

23. Design a scope/state-ownership strategy for a Node.js service.
24. How would you eliminate accidental process-global request state?
25. How would you review a large codebase for harmful shadowing?
26. How would you explain lexical scoping to engineers who think scope is the same as stack lifetime?
27. How does scope influence security authority?
28. How would you design module boundaries to minimize mutable shared state?
29. How would you diagnose a memory leak caused by a closure retaining a large environment?
30. How does lexical scoping enable static tooling while JavaScript remains dynamically typed?

---

# 84. Predict-the-Output Exercises

Predict before execution.

### 1

```js
let x = "outer";

{
  let x = "inner";
  console.log(x);
}

console.log(x);
```

### 2

```js
var x = "outer";

{
  var x = "inner";
}

console.log(x);
```

### 3

```js
function test() {
  if (true) {
    var x = 10;
  }

  return x;
}

console.log(test());
```

### 4

```js
function test() {
  if (true) {
    let x = 10;
  }

  return x;
}

console.log(test());
```

Determine whether it returns or throws.

### 5

```js
const x = "global";

function test() {
  const x = "local";
  return x;
}

console.log(test());
console.log(x);
```

### 6

```js
let x = "global";

function first() {
  let x = "first";
  return second();
}

function second() {
  return x;
}

console.log(first());
```

### 7

```js
function makeCounter() {
  let count = 0;

  return () => ++count;
}

const a = makeCounter();
const b = makeCounter();

console.log(a());
console.log(a());
console.log(b());
```

### 8

```js
const callbacks = [];

for (var i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

### 9

```js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

### 10

```js
let x = 1;

function test() {
  x = 2;
}

test();

console.log(x);
```

### 11

```js
let x = 1;

function test() {
  let x = 2;
}

test();

console.log(x);
```

### 12

```js
const obj = { x: 10 };

function test() {
  const x = 20;
  return obj.x + x;
}

console.log(test());
```

### 13

```js
let value = 1;

{
  let value = 2;

  {
    let value = 3;
    console.log(value);
  }

  console.log(value);
}

console.log(value);
```

### 14

```js
function outer() {
  const x = 10;

  return function inner() {
    return x;
  };
}

const fn = outer();

console.log(fn());
```

### 15

```js
const x = 10;

function outer() {
  function inner() {
    return x;
  }

  return inner;
}

const fn = outer();

console.log(fn());
```

---

# 85. Mastery Exercises

## Exercise A — Scope Chain Diagram

For:

```js
const a = 1;

function outer() {
  const b = 2;

  {
    const c = 3;

    function inner() {
      const d = 4;
      return a + b + c + d;
    }

    return inner;
  }
}
```

Draw every relevant lexical environment and its outer relationship.

Then explain the lookup path for:

```text
a
b
c
d
```

## Exercise B — Shadowing Audit

Create:

```text
global x
function x
block x
nested function x
```

Explain which `x` each expression resolves to.

## Exercise C — Scope Simulator

Implement:

```text
Environment
declare
resolve
assign
```

and support nested environments.

## Exercise D — State Ownership Audit

Take a backend service and classify every important variable as:

```text
function-local
request-scoped
module-scoped
process-global
external state
```

For each, explain why the scope matches the required lifetime.

## Exercise E — Closure Retention

Create a closure capturing a large object.

Then design three versions:

```text
captures everything
captures only required data
uses explicit state passed at invocation
```

Compare their memory/lifetime implications conceptually.

## Exercise F — Lexical vs Dynamic Scope

Simulate what the result would be under dynamic scope for a selected example.

Then compare with actual JavaScript lexical behavior.

---

# 86. Production Scope Decision Framework

For every important piece of state, ask:

```text
1. Who owns it?
2. Who needs access?
3. How long should it live?
4. Should multiple invocations share it?
5. Should multiple requests share it?
6. Can it be mutated?
7. Should it be immutable?
8. Is it security-sensitive?
9. Can a closure retain it?
10. Can it leak through a global/module boundary?
```

Then choose:

```text
local
request-scoped
module-scoped
process-global
externalized
```

Scope is an ownership decision.

---

# 87. Scope and Architecture

A large application can be understood as nested authority boundaries:

```text
process
  ↓
module/service
  ↓
request
  ↓
function
  ↓
block
```

The smaller the scope:

```text
the fewer components can directly access the state
```

This often improves:

```text
maintainability
testability
security
reasoning
concurrency isolation
```

But overly fragmented scope can make dependency flow difficult to follow.

Good architecture balances locality with explicit sharing.

---

# 88. Scope vs Lifetime

Do not equate:

```text
scope
```

with:

```text
lifetime
```

A binding can be lexically inaccessible while an object remains alive through another reference.

A function can outlive the scope in which it was created because of a closure.

An object can be globally reachable while only one module logically owns its mutation.

Therefore:

```text
visibility
≠
lifetime
≠
ownership
```

These are separate engineering concepts.

---

# 89. Concept Connections

## Depends On

- Chapters 01–09, especially bindings, declarations, functions, and control flow.

## Builds Toward

- Chapter 11 — Hoisting and the Temporal Dead Zone.
- Chapter 12 — Execution Contexts and the Execution Model.
- Chapter 13 — Closures.
- Chapter 14 — `this`, Invocation, and Binding.
- Chapter 18 — Classes.
- Chapter 25 — Iterables and Iterators.
- Chapter 35 — Promises.
- Chapter 36 — Async/Await.
- Chapter 45 — Memory and Garbage Collection.
- Chapter 46 — Weak References and Finalization.
- Chapter 64 — ES Modules.
- Chapter 65 — CommonJS and Interoperability.
- Chapter 78 — Production JavaScript Architecture.
- Chapter 83 — Observability.
- Chapter 84 — Reliability.
- Chapter 88 — Debugging.

## Related Concepts

```text
identifier
binding
scope
lexical environment
environment record
outer environment
identifier resolution
shadowing
closure
global environment
module environment
function environment
reference records
```

## Concepts Revisited Later

- declaration instantiation,
- TDZ,
- execution contexts,
- `GetValue` / `PutValue`,
- `this`,
- closures,
- garbage collection,
- module live bindings,
- async context propagation,
- memory retention.

## Why This Chapter Matters Later

Scope is one of the central organizing structures of JavaScript.

Without a precise model of:

```text
which binding exists
where it lives semantically
how resolution searches outward
which binding is shadowing another
```

it is nearly impossible to reason rigorously about:

```text
hoisting
TDZ
closures
functions
modules
async callbacks
memory
```

---

# 90. Key Takeaways

1. Scope describes where an identifier can be resolved.
2. JavaScript uses lexical scope.
3. Lexical scope is determined by source-code structure rather than the dynamic caller chain.
4. A lexical environment conceptually contains an environment record and an outer-environment relationship.
5. Environment records manage bindings.
6. JavaScript has multiple environment-record categories with different purposes.
7. Identifier resolution searches the current environment and then moves outward.
8. The first matching binding found by resolution wins.
9. Shadowing creates a new binding with the same name in an inner scope.
10. Shadowing does not mutate or overwrite the outer binding.
11. Reassignment updates an existing mutable binding.
12. Object property lookup is distinct from lexical identifier resolution.
13. `var` is function-scoped.
14. `let` and `const` are block/lexically scoped.
15. Functions create function-related scope and parameter bindings.
16. Loops with lexical declarations can create per-iteration binding semantics.
17. Closures rely on the relationship between function values and lexical environments.
18. Global environments contain distinct semantics for object-backed and lexical declarations.
19. Module scope isolates top-level bindings from the global object.
20. ES module imports use live bindings rather than ordinary copied snapshots.
21. Lexical scope supports static analysis and tooling even though JavaScript remains dynamically typed.
22. `eval` and `with` complicate scope analysis and should generally be avoided.
23. Scope determines visibility, but it is not identical to lifetime or ownership.
24. Long-lived closures can keep lexical state reachable.
25. Global mutable state can create request-isolation, security, and concurrency problems.
26. Scope should be treated as part of state ownership and architecture.
27. A strong debugging question is: “Which binding does this identifier resolve to?”
28. A strong architecture question is: “Who should be allowed to access this state, and for how long?”

---

# 91. Completion Criteria

### Understand

- Define scope, lexical environments, and environment records.
- Explain lexical scoping.
- Explain function/block/module/global scope.
- Explain identifier resolution.

### Explain

- Explain the outer-environment relationship.
- Explain shadowing vs reassignment.
- Explain property lookup vs lexical lookup.
- Explain closure foundations.
- Explain global and module environment distinctions.

### Predict

- Predict nested identifier resolution.
- Predict shadowing.
- Predict `var` vs `let` scope.
- Predict loop binding behavior.
- Predict closure lookup.

### Implement

- Build a scope-chain simulator.
- Implement `declare`, `resolve`, and `assign`.
- Build a simple variable/binding interpreter.

### Debug

- Diagnose wrong-binding bugs.
- Diagnose shadowing.
- Diagnose accidental shared state.
- Diagnose global request-state bugs.
- Diagnose closure-retention risks.

### Apply

- Design scope/state ownership for production services.
- Keep request-specific data local to requests/invocations.
- Use module scope intentionally.
- Avoid accidental globals.

### Compare

- lexical vs dynamic scope
- shadowing vs reassignment
- lexical variable vs object property
- scope vs lifetime
- scope vs ownership
- `var` vs `let`/`const`
- script/global vs module scope

### Defend

- Defend a scope/state-ownership strategy.
- Explain exactly which binding an identifier resolves to.
- Explain how scope affects security and memory.
- Explain why lexical scope is foundational to closures and modules.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 20 — Symbols and Well-Known Symbols
