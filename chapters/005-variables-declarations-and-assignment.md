
# Chapter 05 --- Variables, Declarations, and Assignment

> **Status:** `[+] Completed`\
> **Role in curriculum:** Establishes the precise relationship between
> identifiers, bindings, declarations, initialization, assignment,
> mutability, and global declarations. This chapter is the foundation
> for scope, lexical environments, hoisting, execution contexts,
> closures, modules, and debugging.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain what a variable is in JavaScript without reducing it to a
    physical memory box.
-   Distinguish:
    -   identifier,
    -   binding,
    -   value,
    -   declaration,
    -   initialization,
    -   assignment,
    -   mutation,
    -   reassignment.
-   Explain `var`, `let`, and `const`.
-   Explain how declaration semantics differ between `var` declarations
    and lexical declarations.
-   Explain why `let` and `const` are block-scoped.
-   Explain why `var` is function-scoped.
-   Explain what `const` does and does not guarantee.
-   Explain declaration, initialization, and assignment as separate
    operations.
-   Explain redeclaration vs reassignment.
-   Explain the temporal dead zone concept at a foundational level.
-   Explain global declarations and their relationship with
    `globalThis`.
-   Distinguish global object properties from global lexical bindings.
-   Understand why browser scripts and JavaScript modules have different
    top-level behavior.
-   Predict declaration behavior before execution.
-   Diagnose common bugs involving shadowing, accidental globals,
    hoisting, and mutation.
-   Understand how this chapter connects to lexical environments and
    execution contexts.

------------------------------------------------------------------------

## 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.

Chapter 02 established:

``` text
identifier/binding/value are different concepts
```

This chapter formalizes that distinction.

------------------------------------------------------------------------

# 3. What Is a Variable?

In everyday programming language, developers often say:

> "A variable is a box that stores a value."

That is a useful beginner metaphor, but it is not an accurate complete
model of JavaScript semantics.

A better abstraction is:

``` text
identifier
   ↓
binding
   ↓
current value
```

A binding associates a name with a value within an environment.

For example:

``` js
let age = 25;
```

Conceptually:

``` text
Identifier: age
Binding: mutable lexical binding
Current value: 25
```

Later:

``` js
age = 26;
```

The binding remains:

``` text
age
```

but its current value changes.

------------------------------------------------------------------------

# 4. Declaration

A declaration tells the language that bindings should exist.

Examples:

``` js
var x;
let y;
const z = 10;
```

A declaration is not necessarily the same thing as initialization.

For example:

``` js
let x;
```

declares a binding and initializes it to `undefined` during the
appropriate declaration-instantiation process.

By contrast:

``` js
const x;
```

is a syntax error because a `const` declaration requires an initializer.

This difference becomes important later when studying declaration
instantiation and the temporal dead zone.

------------------------------------------------------------------------

# 5. Initialization

Initialization establishes the initial value associated with a binding.

Example:

``` js
let x = 10;
```

contains:

``` text
declaration
+
initializer expression
```

The initializer expression:

``` js
10
```

produces a value.

The binding is then initialized with that value.

Conceptually:

``` text
create binding
      ↓
evaluate initializer
      ↓
initialize binding with resulting value
```

This is different from later assignment.

------------------------------------------------------------------------

# 6. Assignment

Assignment updates an already-assignable binding.

Example:

``` js
let x = 10;

x = 20;
```

The first line performs declaration and initialization.

The second line performs assignment.

Conceptually:

``` text
x → 10

then:

x → 20
```

The binding remains the same binding.

Only its current value changes.

------------------------------------------------------------------------

# 7. Mutation

Mutation is different from reassignment.

Example:

``` js
const user = {
  name: "A"
};

user.name = "B";
```

The binding is not reassigned.

The object is mutated.

Conceptually:

``` text
user ───→ Object #1

Object #1:
name → "A"

then:

Object #1:
name → "B"
```

Compare:

``` js
let user = {
  name: "A"
};

user = {
  name: "B"
};
```

This reassigns the binding.

Therefore:

``` text
mutation
  ≠
reassignment
```

This distinction should be automatic in your reasoning.

------------------------------------------------------------------------

# 8. `var`, `let`, and `const`

JavaScript provides three major declaration forms:

``` js
var
let
const
```

Their differences include:

``` text
scope
redeclaring
reassigning
declaration instantiation
global behavior
temporal dead zone behavior
```

A high-level comparison:

  -----------------------------------------------------------------------
  Feature           `var`             `let`             `const`
  ----------------- ----------------- ----------------- -----------------
  Function-scoped   Yes               No                No

  Block-scoped      No                Yes               Yes

  Reassignment      Yes               Yes               No
  allowed                                               

  Redeclaration in  Commonly allowed  Not allowed       Not allowed
  same lexical                                          
  scope                                                 

  Must initialize   No                No                Yes
  immediately                                           

  TDZ               No lexical TDZ    Yes               Yes

  `const` binding   ---               ---               Binding cannot be
  immutable                                             reassigned

  Object contents   No                No                No
  automatically                                         
  immutable                                             
  -----------------------------------------------------------------------

The table is a starting point. Exact declaration and global semantics
are more nuanced.

------------------------------------------------------------------------

# 9. `var`

Example:

``` js
var count = 10;
```

`var` creates a variable binding with historical JavaScript semantics.

Most importantly:

``` text
var
→ function-scoped
```

not block-scoped.

Example:

``` js
function example() {
  if (true) {
    var x = 10;
  }

  console.log(x);
}

example();
```

Result:

``` text
10
```

The block does not create a separate `var` scope.

------------------------------------------------------------------------

# 10. `let`

Example:

``` js
let count = 10;
```

`let` creates a lexical binding.

Lexical declarations are block-scoped.

Example:

``` js
{
  let x = 10;
}

console.log(x);
```

This throws a `ReferenceError` because the binding is not available
outside the block.

Conceptually:

``` text
outer scope
   │
   └── block scope
          └── x
```

The binding exists only in the relevant lexical environment.

------------------------------------------------------------------------

# 11. `const`

Example:

``` js
const count = 10;
```

`const` creates a lexical binding that cannot later be reassigned.

This fails:

``` js
const count = 10;
count = 20;
```

The failure concerns the binding.

It does not mean the value itself is universally immutable.

For objects:

``` js
const user = {
  name: "A"
};

user.name = "B";
```

is valid because the object is mutable.

------------------------------------------------------------------------

# 12. `const` Protects a Binding, Not an Object

This deserves emphasis.

``` js
const user = {
  profile: {
    name: "A"
  }
};
```

These are different operations:

``` js
user = {};
```

and:

``` js
user.profile.name = "B";
```

The first is forbidden.

The second mutates reachable object state.

So:

``` text
const
  → no reassignment of the binding

not:

const
  → deep immutability
```

`Object.freeze` is also not deep recursive immutability unless nested
objects are separately frozen.

------------------------------------------------------------------------

# 13. Redeclaration vs Reassignment

These are different errors/concepts.

### Redeclaration

Defining the same declaration name again where the declaration rules do
not permit it.

Example:

``` js
let x = 1;
let x = 2;
```

This is a syntax error.

### Reassignment

Changing the value of an existing mutable binding.

Example:

``` js
let x = 1;
x = 2;
```

This is valid.

So:

``` text
let x = 1;
let x = 2;
```

→ redeclaration

while:

``` text
let x = 1;
x = 2;
```

→ reassignment

------------------------------------------------------------------------

# 14. `var` Redeclaration

Historical `var` semantics allow same-scope redeclaration in cases where
lexical declarations do not.

Example:

``` js
var x = 1;
var x = 2;
```

This is allowed.

The binding is not simply "created twice" as two independent same-name
bindings.

The declaration-instantiation semantics handle the declarations.

This legacy behavior is one reason `var` can produce surprising
interactions with modern lexical declarations.

------------------------------------------------------------------------

# 15. Why `var` Still Exists

JavaScript maintains backward compatibility.

Huge amounts of legacy code depend on `var`.

Replacing it universally or removing it would break existing
applications.

Therefore modern JavaScript provides:

``` text
legacy var semantics
+
modern lexical declarations
```

The language's evolution preserves old behavior while adding better
abstractions.

------------------------------------------------------------------------

# 16. Why `let` and `const` Were Added

Before ECMAScript 2015, JavaScript primarily used:

``` js
var
```

for variable declarations.

This created several design problems:

-   function-scoping often did not match block-oriented code;
-   accidental redeclaration was easier;
-   asynchronous loops could expose confusing behavior;
-   declaration behavior was frequently misunderstood;
-   developers wanted explicit lexical bindings.

`let` and `const` introduced lexical declarations with clearer
block-scoped semantics.

They did not remove `var`.

------------------------------------------------------------------------

# 17. Block Scope

A block is commonly represented by braces:

``` js
{
  // block
}
```

Control structures also create blocks when braces are used:

``` js
if (condition) {
  let x = 10;
}
```

`let` and `const` bindings belong to the relevant lexical scope.

Example:

``` js
{
  const secret = 123;
  console.log(secret);
}

console.log(secret);
```

The second access fails because the binding is outside its scope.

------------------------------------------------------------------------

# 18. Function Scope

A function creates its own function scope.

Example:

``` js
function example() {
  var x = 10;
}

example();

console.log(x);
```

The final access fails because `x` belongs to the function's
environment.

This is what people mean by:

``` text
var is function-scoped
```

A `var` declaration inside a nested block still belongs to the
surrounding function variable environment rather than creating a
block-local `var` binding.

------------------------------------------------------------------------

# 19. The Classic `var` Loop Problem

Consider:

``` js
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

The callbacks can all observe the final value associated with the same
`var` binding.

The exact output and event-loop timing will be explored deeply in later
asynchronous chapters.

The foundational issue here is:

``` text
one function-scoped `var` binding
```

rather than:

``` text
one block-scoped binding per iteration
```

Modern `let` semantics solve this class of problem differently.

------------------------------------------------------------------------

# 20. `let` in Loops

Compare:

``` js
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

The loop semantics create per-iteration lexical bindings in the relevant
cases.

Conceptually:

``` text
iteration 0 → i = 0
iteration 1 → i = 1
iteration 2 → i = 2
```

This is a language semantic mechanism, not merely a compiler trick.

Later chapters will connect this behavior to lexical environments and
closures.

------------------------------------------------------------------------

# 21. The Temporal Dead Zone

`let` and `const` declarations are associated with a period between
entering their scope and initialization where accessing the binding is
prohibited.

This is commonly called the:

``` text
Temporal Dead Zone (TDZ)
```

Example:

``` js
console.log(x);

let x = 10;
```

This throws:

``` text
ReferenceError
```

The important point is:

``` text
the binding exists in the relevant lexical environment
but has not yet been initialized for use
```

This is different from saying:

> "The variable does not exist at all."

The specification model is more precise.

------------------------------------------------------------------------

# 22. TDZ Is Not Simply "Hoisting Does Not Happen"

A common but misleading explanation is:

> "`let` and `const` are not hoisted."

A better explanation is:

> Their lexical bindings are created during declaration instantiation,
> but access before initialization is restricted.

Therefore:

``` text
declaration creation
```

and:

``` text
initialization
```

are separate concepts.

Chapter 11 will explore TDZ and hoisting formally.

------------------------------------------------------------------------

# 23. `var` and Early Access

Compare:

``` js
console.log(x);

var x = 10;
```

Typical result:

``` text
undefined
```

Why?

The `var` binding is created and initialized to `undefined` as part of
the relevant declaration instantiation.

Then the assignment occurs when execution reaches:

``` js
x = 10;
```

Conceptually:

``` text
binding exists
value = undefined

then execution:
value = 10
```

This differs fundamentally from lexical TDZ behavior.

------------------------------------------------------------------------

# 24. Declaration Instantiation

Before executable statements in a scope run, the language creates the
appropriate bindings for declarations.

The exact process differs between:

``` text
var
lexical declarations
function declarations
class declarations
```

This is one of the first places where specification terminology becomes
useful.

The runtime must establish the environment in which code executes.

This process is not simply:

``` text
run each line from top to bottom and create names when encountered
```

The language's declaration-instantiation semantics are more structured.

------------------------------------------------------------------------

# 25. Assignment Evaluation

Consider:

``` js
let x = 1;

x = 2;
```

The assignment expression:

``` js
x = 2
```

does several conceptual things:

1.  Evaluate the left-hand side as a reference to a binding.
2.  Evaluate the right-hand side.
3.  Perform the appropriate assignment operation.
4.  Update the binding.

Later chapters will formalize this using specification concepts such as:

``` text
Reference
GetValue
PutValue
```

This is why assignments are more than simple "box writes."

------------------------------------------------------------------------

# 26. The Assignment Operator Produces a Value

Assignment is an expression.

Example:

``` js
let x;

const result = (x = 42);

console.log(result);
```

Result:

``` text
42
```

The assignment updates `x` and also produces the assigned value as the
result of the assignment expression.

This enables constructs such as:

``` js
let a;
let b;

a = b = 10;
```

Conceptually:

``` text
b = 10
→ 10

a = 10
→ 10
```

Although valid, chained assignment should be used carefully for
readability.

------------------------------------------------------------------------

# 27. Compound Assignment

JavaScript supports:

``` js
+=
-=
*=
/=
%=
**=
<<=
>>=
>>>=
&=
|=
^=
&&=
||=
??=
```

Example:

``` js
let count = 10;

count += 5;
```

This has semantics related to reading the old value, applying the
operator, and writing the result.

Do not assume every compound assignment is equivalent to a textual
substitution without considering:

-   getters/setters,
-   coercion,
-   evaluation order,
-   side effects.

Those topics become important when assignments target properties rather
than simple bindings.

------------------------------------------------------------------------

# 28. Destructuring Declarations

Declarations can destructure values.

Example:

``` js
const user = {
  name: "A",
  age: 25
};

const { name, age } = user;
```

This creates bindings:

``` text
name → "A"
age  → 25
```

Array destructuring works similarly:

``` js
const [first, second] = [10, 20];
```

Destructuring is not a separate type mechanism.

It is syntax that performs binding initialization/assignment according
to defined semantics.

Later operator and pattern chapters will explore it more deeply.

------------------------------------------------------------------------

# 29. Declaration Destructuring vs Assignment Destructuring

Compare:

``` js
const { name } = user;
```

with:

``` js
({ name } = user);
```

The first is a declaration.

The second is an assignment pattern and requires syntax that avoids
being parsed as a block in statement position.

This demonstrates again:

``` text
declaration
≠
assignment
```

even when the same pattern syntax is involved.

------------------------------------------------------------------------

# 30. Default Values in Declarations

Example:

``` js
const { name = "Unknown" } = user;
```

The default applies when the relevant extracted value is `undefined`.

It does not mean:

``` text
any falsy value
```

For example:

``` js
const { name = "Unknown" } = { name: "" };
```

The result is:

``` text
""
```

because the property exists with an empty String value.

This connects declaration patterns to the distinction between:

``` text
undefined
```

and other falsy values.

------------------------------------------------------------------------

# 31. Shadowing

A nested scope can declare a binding with the same name as an outer
scope.

Example:

``` js
let x = "outer";

{
  let x = "inner";
  console.log(x);
}

console.log(x);
```

Output:

``` text
inner
outer
```

The inner binding shadows the outer binding within its scope.

Conceptually:

``` text
outer environment
  x → "outer"

inner environment
  x → "inner"
```

Identifier resolution finds the nearest applicable binding.

Chapter 10 will formalize this.

------------------------------------------------------------------------

# 32. Shadowing Is Not Reassignment

Compare:

``` js
let x = "outer";

{
  let x = "inner";
}
```

with:

``` js
let x = "outer";

x = "inner";
```

The first creates a new binding.

The second changes the existing binding.

Therefore:

``` text
shadowing
≠
reassignment
```

This distinction matters enormously when debugging closures and nested
functions.

------------------------------------------------------------------------

# 33. Accidental Globals

In non-strict legacy script scenarios, assignment to an unresolvable
identifier can have special global behavior.

Example often associated with sloppy-mode code:

``` js
x = 10;
```

without a prior declaration.

This is dangerous.

Modern production code should use explicit declarations:

``` js
const x = 10;
```

or:

``` js
let x = 10;
```

Strict mode turns many accidental-global patterns into errors rather
than silently creating global properties.

The exact behavior depends on source type and strictness.

------------------------------------------------------------------------

# 34. Why Accidental Globals Are Dangerous

Unexpected global state creates:

``` text
hidden coupling
name collisions
hard-to-debug mutations
security risks
test contamination
concurrency-like state problems
```

A global value can be modified by unrelated code.

Production engineering should minimize mutable global state.

------------------------------------------------------------------------

# 35. Global Scope Is Special

JavaScript has a global environment.

But:

``` text
global binding
```

and:

``` text
global object property
```

are not universally identical concepts.

This distinction is especially important in browsers.

Consider classic scripts:

``` js
var x = 10;
```

The `var` declaration has historical interaction with the global object.

But:

``` js
let y = 20;
```

creates a global lexical binding that does not behave as an ordinary
global object property.

This is why:

``` js
globalThis
```

should not be treated as the exact synonym for "the global scope."

------------------------------------------------------------------------

# 36. `globalThis`

`globalThis` provides a standardized way to obtain the global object
associated with the current execution environment.

Example:

``` js
console.log(globalThis);
```

The exact object and exposed properties depend on the host.

Examples include:

-   browser global object behavior,
-   Node.js global object behavior,
-   worker contexts.

`globalThis` improves portability compared with environment-specific
names such as:

``` js
window
global
```

But access to `globalThis` itself is not equivalent to access to every
global lexical binding.

------------------------------------------------------------------------

# 37. Browser Global Behavior

In a browser classic script:

``` js
var x = 10;
```

can create a corresponding global object property under the applicable
global declaration rules.

But:

``` js
let y = 20;
```

creates a global lexical binding.

Therefore:

``` js
globalThis.x
```

and:

``` js
globalThis.y
```

do not necessarily behave equivalently.

This is a specification-level distinction that prevents many confusing
global-scope assumptions.

------------------------------------------------------------------------

# 38. Modules Change Top-Level Semantics

JavaScript modules are not just scripts with `import` statements added.

A module has its own top-level lexical environment and module-specific
execution model.

Top-level declarations in modules do not become ordinary global object
properties simply because they are at the top level of a file.

Example:

``` js
const secret = 123;

export { secret };
```

`secret` is module-scoped.

This isolation is one of the core reasons modern JavaScript favors
modules over shared global script state.

------------------------------------------------------------------------

# 39. `var` in Modules

Even though `var` has function-scoped behavior, top-level `var` in a
module is not equivalent to a browser classic script's global `var`.

Example:

``` js
// module
var x = 10;
```

This does not create a normal global object property merely because the
declaration is at top level.

The module environment changes the surrounding declaration semantics.

This is a key reason source type matters when analyzing code.

------------------------------------------------------------------------

# 40. Global Environment as Two Related Parts

At specification level, the global environment includes mechanisms for
handling:

``` text
object environment behavior
+
declarative lexical bindings
```

This helps explain why:

``` js
var
```

and:

``` js
let
const
```

behave differently at the global level.

The distinction is deeper than:

> "var is old and let is new."

It reflects different binding models.

------------------------------------------------------------------------

# 41. Environment Model

A useful conceptual model:

``` text
Execution context
      ↓
Lexical Environment
      ↓
Environment Record
      ↓
Bindings
      ↓
Values
```

This chapter does not yet fully define execution contexts or environment
records.

Those come in Chapter 10 and Chapter 12.

For now, understand:

``` text
bindings live in environments
```

and:

``` text
scope determines where bindings are available
```

------------------------------------------------------------------------

# 42. Binding Mutability

A binding can be:

``` text
mutable
```

or:

``` text
non-reassignable
```

`let` creates a mutable lexical binding:

``` js
let x = 1;
x = 2;
```

`const` creates a binding that cannot be reassigned after
initialization:

``` js
const x = 1;
// x = 2; // TypeError
```

This is a property of the binding.

It says nothing by itself about whether the value is an object that can
be mutated.

------------------------------------------------------------------------

# 43. Binding Initialization Is Not Object Mutation

Compare:

``` js
const user = {
  name: "A"
};
```

and:

``` js
user.name = "B";
```

The first initializes the binding.

The second mutates the object.

The object can have mutable state even though the binding cannot be
reassigned.

This distinction becomes crucial in state management and functional
programming.

------------------------------------------------------------------------

# 44. Readonly Is Not a Native General Object Type

JavaScript's `const` is not equivalent to a language-wide deep immutable
type.

There is no ECMAScript declaration syntax like:

``` text
immutable object
```

that recursively freezes arbitrary object graphs.

Application-level immutability can instead be achieved through:

-   `Object.freeze`,
-   immutable data structures,
-   disciplined programming conventions,
-   libraries,
-   TypeScript's compile-time types,
-   architectural boundaries.

Those mechanisms solve different problems.

------------------------------------------------------------------------

# 45. Declaration Initializers Are Expressions

Consider:

``` js
const total = price * quantity;
```

The initializer is an expression.

That expression can involve:

-   function calls,
-   property access,
-   arithmetic,
-   objects,
-   arrays,
-   `await` in permitted contexts,
-   exceptions.

For example:

``` js
const result = riskyOperation();
```

If `riskyOperation()` throws, the declaration does not complete
normally.

This connects declarations to error propagation and execution contexts.

------------------------------------------------------------------------

# 46. Declaration Order Matters

Consider:

``` js
const a = b;
const b = 10;
```

This fails due to lexical access before `b` is initialized.

Compare:

``` js
var a = b;
var b = 10;
```

The `var` declarations are instantiated differently, so the first read
encounters the current `undefined` state associated with `b`.

The contrast is an excellent demonstration that declaration order,
initialization timing, and binding creation are distinct concepts.

------------------------------------------------------------------------

# 47. Multiple Declarations

JavaScript permits:

``` js
let a = 1, b = 2, c = 3;
```

This is one lexical declaration containing multiple declarators.

The initializer expressions are evaluated according to the language's
evaluation rules.

For production code, separate declarations may sometimes be clearer:

``` js
const a = 1;
const b = 2;
const c = 3;
```

Readability is an engineering concern even when semantics are
equivalent.

------------------------------------------------------------------------

# 48. Variable Declaration vs Function Declaration

JavaScript has multiple declaration forms:

``` js
var x;
let y;
const z = 1;
function f() {}
class C {}
```

They do not all participate in initialization identically.

This is why saying:

> "JavaScript hoists everything the same way"

is misleading.

Different declaration forms have different declaration-instantiation
behavior.

Chapter 11 will compare them directly.

------------------------------------------------------------------------

# 49. Assignment to `const`

Consider:

``` js
const x = 10;
x += 1;
```

This fails because compound assignment still attempts to assign a new
value to the `const` binding.

Likewise:

``` js
++x;
```

fails because increment requires an assignment-like update.

The fact that an operator is syntactically different does not change the
underlying binding constraint.

------------------------------------------------------------------------

# 50. Assignment Through Object Properties

A `const` object can have a property assignment:

``` js
const user = {};
user.name = "A";
```

This does not assign to the `user` binding.

It assigns to a property of the object value.

This is why:

``` js
const user = {};
user = {};
```

fails while:

``` js
user.name = "A";
```

can succeed.

The left-hand side expressions represent different kinds of references.

------------------------------------------------------------------------

# 51. Strict Mode and Assignment

Strict mode makes several legacy behaviors stricter.

For example, assigning to an undeclared identifier:

``` js
x = 10;
```

causes a `ReferenceError` in strict mode.

JavaScript modules are implicitly strict.

This means modern module code naturally avoids certain sloppy-mode
hazards.

Strict mode will be discussed more deeply in the `this`, functions, and
legacy JavaScript chapters.

------------------------------------------------------------------------

# 52. Automatic Semicolon Insertion and Declarations

Declarations can interact with statement parsing.

For example:

``` js
let x
= 10;
```

is valid because line breaks do not necessarily terminate the
declaration in this case.

But JavaScript's automatic semicolon insertion rules can create
surprising results in other contexts.

The important principle:

> Source formatting and grammar are related but not identical.

Do not reason about declaration boundaries solely by visual line breaks.

------------------------------------------------------------------------

# 53. Production Engineering --- Prefer Explicit Bindings

Modern production JavaScript generally favors:

``` js
const
```

by default, and:

``` js
let
```

when reassignment is genuinely required.

Use:

``` js
var
```

primarily when deliberately maintaining or interfacing with legacy
semantics.

A practical rule:

``` text
Need a binding?
   ↓
Can it remain assigned to the same value?
   ↓ yes → const
   ↓ no  → let
```

This improves readability because the binding's reassignment semantics
are visible to the reader.

It does not guarantee object immutability.

------------------------------------------------------------------------

# 54. Production Engineering --- Avoid Accidental Globals

Bad:

``` js
userCount = 10;
```

Better:

``` js
const userCount = 10;
```

or:

``` js
let userCount = 10;
```

Better still, keep state scoped to the smallest appropriate
module/function/component/service boundary.

Global mutable state increases coupling and complicates testing.

------------------------------------------------------------------------

# 55. Production Engineering --- Minimize Shared Mutable State

A production architecture becomes easier to reason about when:

``` text
state ownership
```

is explicit.

Prefer:

``` text
module-owned state
service-owned state
request-scoped state
function-local state
```

over:

``` text
process-wide mutable globals
```

This is not a blanket rule that globals are forbidden.

Some globals are intentionally used for:

-   configuration,
-   process-wide immutable registries,
-   observability setup,
-   shared caches.

The engineering question is:

> Who owns the state, who can mutate it, and what guarantees exist
> around access?

------------------------------------------------------------------------

# 56. Performance Considerations

Declaration syntax is not normally where application performance
problems originate.

The major costs usually come from:

-   algorithmic complexity,
-   allocation,
-   object shape changes,
-   memory pressure,
-   I/O,
-   serialization,
-   garbage collection,
-   excessive work.

Do not choose `var` over `let` because of vague claims such as:

> "var is faster."

Modern engine behavior is complex, implementation-specific, and
workload-dependent.

Prefer the declaration semantics that make the program correct and
maintainable.

Measure performance before optimizing declaration choices.

------------------------------------------------------------------------

# 57. Memory Considerations

Bindings can remain reachable through:

-   active execution contexts,
-   closures,
-   global environments,
-   module state,
-   event handlers.

For example:

``` js
function createCounter() {
  let count = 0;

  return () => ++count;
}
```

The `count` binding can remain reachable after `createCounter` returns
because the returned function closes over it.

This connects declarations directly to Chapter 13 on closures and later
garbage-collection analysis.

------------------------------------------------------------------------

# 58. Security Considerations

Global mutable bindings can create security-sensitive shared state.

Potential problems include:

``` text
authorization flags
request-specific secrets
tenant identifiers
user data
security configuration
```

being accidentally stored in process-global mutable state.

In server applications, that can cause request isolation failures.

For example, a mutable global:

``` js
let currentUser;
```

is generally a dangerous architecture for a concurrent server because
requests can overlap.

State ownership should match the lifetime of the data.

------------------------------------------------------------------------

# 59. Debugging Methodology

When a variable behaves unexpectedly, ask these questions in order:

``` text
1. What is the identifier?
2. Which binding does it resolve to?
3. Which scope/environment contains that binding?
4. Is this a declaration, initialization, assignment, or mutation?
5. Can the binding be reassigned?
6. Has the binding been initialized yet?
7. Is another inner binding shadowing it?
8. Is the value an object with mutable state?
9. Is this a global/module/function/block binding?
10. Is the unexpected behavior caused by the language, runtime, or application architecture?
```

This checklist will become more powerful after the scope and
execution-context chapters.

------------------------------------------------------------------------

# 60. Debugging Exercise --- Reassignment vs Mutation

Given:

``` js
const user = {
  name: "A"
};

user.name = "B";

console.log(user);
```

Explain why this is valid.

Now:

``` js
user = {
  name: "C"
};
```

Explain why this fails.

The answer must explicitly distinguish:

``` text
binding update
```

from:

``` text
object property update
```

------------------------------------------------------------------------

# 61. Debugging Exercise --- Shadowing

Analyze:

``` js
let value = "outer";

function test() {
  let value = "inner";
  console.log(value);
}

test();

console.log(value);
```

Identify:

``` text
outer binding
inner binding
identifier resolution inside test
identifier resolution outside test
```

Do not describe the second declaration as "changing" the first value.

It creates a different binding.

------------------------------------------------------------------------

# 62. Debugging Exercise --- Global Binding

In an appropriate browser classic-script context, compare:

``` js
var x = 1;
let y = 2;

console.log(globalThis.x);
console.log(globalThis.y);
```

Determine why the results differ.

Then repeat in a module context and explain why top-level behavior
changes.

The important skill is understanding global declaration categories
rather than memorizing browser quirks.

------------------------------------------------------------------------

# 63. Implementation Exercise --- Declaration Classifier

Build a static-analysis-style utility or checklist that takes a
declaration and classifies:

``` text
declaration kind
scope kind
reassignable?
requires initializer?
lexical TDZ?
same-scope redeclaration behavior
global-object interaction
```

For:

``` js
var x;
let y;
const z = 1;
```

Do not rely only on `typeof` or runtime evaluation.

The exercise is about declaration semantics.

------------------------------------------------------------------------

# 64. Implementation Progression

### Guided

Create a table covering:

``` text
var
let
const
```

### Partially Guided

Write examples demonstrating:

``` text
function scope
block scope
reassignment
redeclaration
shadowing
TDZ
global behavior
```

### No Reference

Build a small test suite containing at least 30 declaration edge cases.

### Edge-Case Hardened

Include:

``` text
nested blocks
loops
functions
modules
strict mode
global script
object mutation
destructuring
default values
compound assignment
shadowing
```

### Production-Grade

Turn the suite into documentation-backed tests for an actual project
coding standard.

------------------------------------------------------------------------

# 65. Predict-the-Output Exercises

Predict before executing.

### 1

``` js
var x = 10;
console.log(x);
```

### 2

``` js
let x = 10;
console.log(x);
```

### 3

``` js
const x = 10;
console.log(x);
```

### 4

``` js
var x = 10;
var x = 20;
console.log(x);
```

### 5

``` js
let x = 10;
let x = 20;
```

### 6

``` js
const x = 10;
x = 20;
```

### 7

``` js
let x;

console.log(x);
```

### 8

``` js
console.log(x);
var x = 10;
```

### 9

``` js
console.log(x);
let x = 10;
```

### 10

``` js
const user = {};
user.name = "A";

console.log(user.name);
```

### 11

``` js
const user = {};
user = {};
```

### 12

``` js
let x = "outer";

{
  let x = "inner";
  console.log(x);
}

console.log(x);
```

### 13

``` js
let x = 1;

function test() {
  let x = 2;
  return x;
}

console.log(test(), x);
```

### 14

``` js
let x = 1;

{
  var x = 2;
}
```

Determine whether the code is valid before predicting output.

### 15

``` js
let x = 1;

{
  {
    let x = 2;
    console.log(x);
  }

  console.log(x);
}
```

------------------------------------------------------------------------

# 66. Interview Questions

## Junior

1.  What is the difference between `var`, `let`, and `const`?
2.  What does `const` actually make immutable?
3.  What is function scope?
4.  What is block scope?
5.  What is reassignment?
6.  What is mutation?

## Mid-Level

7.  Why can you mutate an object declared with `const`?
8.  What is redeclaration?
9.  What is shadowing?
10. Why is `var` function-scoped?
11. Why was `let` introduced?
12. Why does `console.log(x); let x = 1;` throw?

## Senior

13. Explain declaration vs initialization vs assignment.
14. Explain the temporal dead zone.
15. Explain why "let is not hoisted" is an incomplete statement.
16. Explain global `var` vs global `let`.
17. Explain top-level declarations in modules.
18. Explain why accidental globals are dangerous.

## Staff / Principal

19. Design a state-ownership policy for a Node.js service.
20. Explain why global mutable request state is unsafe.
21. How would you review a codebase for accidental global state?
22. How do declaration semantics affect closure behavior?
23. How would you establish a `const-by-default` coding standard without
    falsely claiming it creates deep immutability?
24. How would you diagnose a bug caused by shadowing in a large module?
25. How would you explain declaration-instantiation semantics to
    engineers without oversimplifying them into "hoisting"?

------------------------------------------------------------------------

# 67. Mastery Exercises

## Exercise A --- Seven-Term Distinction

Explain, with one code example each:

``` text
identifier
binding
value
declaration
initialization
assignment
mutation
```

## Exercise B --- Scope Matrix

Create examples showing:

``` text
var in function
var in block
let in block
const in block
top-level var in script
top-level let in script
top-level declarations in module
```

## Exercise C --- Binding Ownership

For a production service, classify every important state variable as:

``` text
local
request-scoped
module-scoped
process-global
externalized
```

Then explain why each choice is safe.

## Exercise D --- Shadowing Audit

Take a real module and identify every shadowed identifier.

For each one, decide:

``` text
intentional
accidental
dangerous
acceptable
```

## Exercise E --- Declaration Reasoning

For each of these, state:

``` text
binding creation
initialization state
assignment state
scope
reassignment allowed?
```

``` js
var a;
let b;
const c = 1;
```

------------------------------------------------------------------------

# 68. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.

## Builds Toward

-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.
-   Chapter 09 --- Functions.
-   Chapter 10 --- Scope, Lexical Environments, and Identifier
    Resolution.
-   Chapter 11 --- Hoisting and the Temporal Dead Zone.
-   Chapter 12 --- Execution Contexts and the Execution Model.
-   Chapter 13 --- Closures.
-   Chapter 14 --- `this`, Invocation, and Binding.
-   Chapter 64 --- ES Modules.
-   Chapter 78 --- Production JavaScript Architecture.

## Related Concepts

``` text
scope
binding
environment
hoisting
TDZ
shadowing
global object
globalThis
modules
mutability
immutability
closures
```

## Concepts Revisited Later

-   Lexical environments.
-   Variable environments.
-   Environment records.
-   Reference records.
-   `GetValue`.
-   `PutValue`.
-   declaration instantiation.
-   execution contexts.
-   global environments.
-   module environments.
-   closure capture.

## Why This Chapter Matters Later

Almost every advanced JavaScript behavior can be reduced partly to
questions about bindings:

``` text
Which binding?
Created where?
Initialized when?
Resolved from which scope?
Mutable or not?
Shadowed by what?
Captured by which closure?
```

Once those questions become habitual, later topics such as hoisting,
closures, modules, asynchronous callbacks, and execution contexts become
much easier to reason about.

------------------------------------------------------------------------

# 69. Key Takeaways

1.  A JavaScript "variable" is best understood through bindings and
    values rather than a simplistic physical memory-box model.
2.  An identifier is a name used in source code; a binding connects that
    name to a value in an environment.
3.  Declaration, initialization, assignment, and mutation are separate
    concepts.
4.  `var` is function-scoped.
5.  `let` and `const` are lexically scoped and block-scoped.
6.  `let` can be reassigned after initialization.
7.  `const` cannot be reassigned after initialization.
8.  `const` does not make referenced objects deeply immutable.
9.  Redeclaration and reassignment are different operations.
10. `var` has historical redeclaration behavior that differs from
    lexical declarations.
11. `let` and `const` have TDZ behavior before initialization.
12. Saying "`let` is not hoisted" is an incomplete explanation.
13. A binding can exist before it becomes initialized for successful
    access.
14. Shadowing creates a different binding; it does not mutate the outer
    binding.
15. `globalThis` refers to the global object, but global object
    properties and global lexical bindings are not universally
    identical.
16. Modules have their own top-level scope and do not treat top-level
    declarations like browser global scripts.
17. Accidental globals create hidden shared state and should be avoided
    in production code.
18. State lifetime and binding scope should match the lifetime and
    ownership of the data.
19. Declaration semantics are foundational to scope, hoisting, closures,
    modules, and execution contexts.

------------------------------------------------------------------------

# 70. Completion Criteria

### Understand

-   Define declaration, binding, initialization, assignment,
    reassignment, and mutation.
-   Explain `var`, `let`, and `const`.

### Explain

-   Explain function scope vs block scope.
-   Explain TDZ at a foundational level.
-   Explain global binding vs global object property.
-   Explain `const` vs immutability.

### Predict

-   Predict declaration and assignment outcomes.
-   Predict shadowing behavior.
-   Identify TDZ failures.

### Implement

-   Build a declaration-semantics test suite.
-   Build a binding/scope classification utility or checklist.

### Debug

-   Diagnose redeclaration errors.
-   Diagnose shadowing.
-   Diagnose accidental globals.
-   Diagnose mutation vs reassignment mistakes.
-   Diagnose early lexical access.

### Apply

-   Use `const` by default where the binding does not need reassignment.
-   Use `let` when reassignment is intentional.
-   Use `var` deliberately rather than accidentally.
-   Keep state ownership explicit.

### Compare

-   `var` vs `let`.
-   `let` vs `const`.
-   redeclaration vs reassignment.
-   mutation vs reassignment.
-   global property vs global lexical binding.
-   script vs module top-level declarations.

### Defend

-   Defend a scope/state-ownership design.
-   Explain why a declaration choice improves correctness or
    maintainability.
-   Explain exactly which guarantee comes from the language and which is
    merely a coding convention.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 06 --- Operators and Expressions
