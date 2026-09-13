# Chapter 124 — Execution Records, Completion Records & References

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Build a precise specification-level mental model for how ECMAScript tracks execution state, represents control-flow outcomes, and models references to bindings and properties.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime Engineer · Compiler Engineer · Debugging Specialist · Language Specification Reviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Many JavaScript behaviors become simple once you distinguish values, references, execution state, and completion flow.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain what Execution Contexts model
[ ] explain what Environment Records model
[ ] explain what Execution Records mean at the specification level
[ ] distinguish runtime implementation state from specification state
[ ] explain Completion Records
[ ] distinguish normal, throw, return, break, and continue completions
[ ] explain abrupt completions
[ ] explain CompletionValue and completion propagation
[ ] understand the ? and ! specification shorthands
[ ] explain Reference Records
[ ] distinguish environment references from property references
[ ] explain resolvable vs unresolvable references
[ ] explain strictness stored in references
[ ] explain Super Reference Records
[ ] explain Private References
[ ] understand GetValue
[ ] understand PutValue
[ ] understand GetThisValue
[ ] understand InitializeReferencedBinding
[ ] explain why `a = b` is not simply "copy a value"
[ ] explain why `obj.x = y` has a different semantic path
[ ] explain why `delete`, `typeof`, assignment, `super`, and update expressions rely on reference semantics
[ ] trace control flow using completion propagation
[ ] connect specification records to observable JavaScript behavior
[ ] distinguish specification abstractions from engine implementation details
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 01 — JavaScript, ECMAScript, Runtime Landscape
Chapter 05 — Variables, Declarations, Assignment
Chapter 06 — Operators, Expressions
Chapter 10 — Scope / Lexical Environments / Identifier Resolution
Chapter 11 — Hoisting / TDZ
Chapter 12 — Execution Contexts / Execution Model
Chapter 13 — Closures
Chapter 14 — this / Invocation / Binding
Chapter 15 — Objects / Property Semantics
Chapter 17 — Prototypes / Prototype Chains
Chapter 18 — Classes / OOP
Chapter 20 — Symbols / Well-Known Symbols
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 43 — Ordinary Object Internal Methods
Chapter 123 — ECMAScript Grammar, Parsing & Early Errors
```

---

# 3. Why This Chapter Exists

Many JavaScript explanations jump directly from:

```js
source code
```

to:

```text
result
```

without explaining the intermediate semantic machinery.

That becomes difficult when analyzing:

```js
x = y;
obj[key]++;
delete obj[key];
super.x;
typeof missingName;
return value;
break label;
throw error;
```

At specification level, ECMAScript models these operations using abstract records and algorithms.

This chapter focuses on three particularly important concepts:

```text
Execution state
Completion state
Reference state
```

Together they explain a large amount of JavaScript behavior.

---

# 4. Mental Model

Use:

```text
Source Code
   ↓
Evaluation
   ↓
Execution Context / Environment State
   ↓
Intermediate Specification Values
   ↓
Reference / Value
   ↓
Operation
   ↓
Completion
   ↓
Caller / surrounding algorithm
```

A useful distinction is:

```text
Value
    = "What data do I have?"

Reference
    = "Where is this binding/property located?"

Completion
    = "How did this evaluation finish?"

Execution state
    = "Where am I and what environment/runtime state applies?"
```

---

# 5. Specification Records Are Not Ordinary JavaScript Objects

When the ECMAScript specification describes something like:

```text
Reference Record
```

it is not saying JavaScript exposes:

```js
new ReferenceRecord(...)
```

to programs.

These are:

```text
Specification Types
```

used to describe language semantics.

Likewise:

```text
Completion Record
Execution Context
Environment Record
```

are specification abstractions.

An engine may implement them in completely different internal structures.

---

# 6. Execution Context

The ECMAScript specification uses an Execution Context to track evaluation state.

The current specification describes fields including:

```text
LexicalEnvironment
VariableEnvironment
PrivateEnvironment
Function
Realm
ScriptOrModule
code evaluation state
```

for ordinary execution contexts, with additional state for specialized contexts. citeturn449716search1

The important point is:

```text
Execution Context ≠ OS thread
Execution Context ≠ JavaScript object
Execution Context ≠ call stack frame in a literal implementation sense
```

It is a specification model of execution state.

---

# 7. Running Execution Context

At a given point, an ECMAScript agent has a running execution context.

Conceptually:

```text
execution context stack
        ↓
top element
        ↓
running execution context
```

When evaluation transfers into another executable code body, a new execution context can be created and pushed.

When control returns, the caller context can become running again.

---

# 8. Execution Context vs Call Stack

These concepts are closely related but should not be treated as identical implementation structures.

### Execution Context

Specification abstraction.

### Call Stack

Common engine/runtime implementation concept.

For example, an engine may use:

```text
native stack
interpreter frames
optimized frames
heap-allocated continuation state
```

without representing the specification's fields exactly as shown.

The correct reasoning is:

```text
specification says what state must conceptually exist
implementation chooses how to represent it
```

---

# 9. Lexical Environment in Execution Context

An execution context has a:

```text
LexicalEnvironment
```

that identifies the environment used for identifier resolution.

Example:

```js
const x = 10;

{
  const y = 20;

  console.log(x + y);
}
```

The inner block can have an Environment Record whose:

```text
[[OuterEnv]]
```

links outward.

The execution context points to the relevant current environment.

---

# 10. Variable Environment

The specification also distinguishes:

```text
LexicalEnvironment
VariableEnvironment
```

These can point to different Environment Records.

This matters especially when reasoning about:

```text
var
let
const
function declarations
```

The distinction is a specification mechanism rather than a recommendation for ordinary coding.

---

# 11. Private Environment

Class private names require another environment concept.

Example:

```js
class User {
  #name = "A";
}
```

The specification tracks private names separately from ordinary identifier bindings.

This helps explain why:

```js
this.#name
```

is not equivalent to:

```js
this.name
```

The semantic machinery is different.

---

# 12. Execution Contexts Can Be Suspended

Some execution contexts are not simply:

```text
created
→ run
→ destroyed
```

Generators and some async mechanisms can suspend/resume evaluation.

For example:

```js
function* sequence() {
  yield 1;
  yield 2;
}
```

A generator can preserve execution state so that later evaluation resumes.

The specification explicitly allows suspended execution-context state to remain available when required by generators. citeturn449716search6turn449716search9

The important mental model:

```text
execution can pause without losing its semantic state
```

---

# 13. What Is a Completion Record?

A Completion Record represents how an ECMAScript algorithm or evaluation step finishes.

The current specification uses completion records for control flow such as:

```text
normal
break
continue
return
throw
```

and includes fields conceptually like:

```text
[[Type]]
[[Value]]
[[Target]]
```

The specification uses Completion Records to model non-local transfers of control. citeturn449716search0

---

# 14. Completion Type

A completion's:

```text
[[Type]]
```

identifies how evaluation finished.

Common forms:

```text
normal
throw
return
break
continue
```

The exact specification machinery is richer than simply saying:

```text
"the function returned a value."
```

A statement can complete normally while producing a value used by an enclosing evaluation algorithm.

---

# 15. Normal Completion

Conceptually:

```text
NormalCompletion(value)
```

means:

```text
evaluation finished normally
```

with a value that may be:

```text
undefined
```

or another ECMAScript value.

Example:

```js
1 + 2
```

evaluates to:

```text
3
```

as a normal completion.

---

# 16. Throw Completion

A throw completion represents abrupt control flow caused by an exception.

Example:

```js
throw new Error("boom");
```

Conceptually:

```text
Completion {
  [[Type]]: throw,
  [[Value]]: Error(...)
}
```

The actual internal representation is specification-level.

---

# 17. Return Completion

Example:

```js
function f() {
  return 10;
}
```

The `return` statement does not merely calculate:

```text
10
```

and continue.

It initiates a non-local transfer of control.

Conceptually:

```text
ReturnCompletion(10)
```

propagates until the function-call evaluation machinery consumes the return completion.

---

# 18. Break Completion

Example:

```js
outer: {
  for (const x of values) {
    if (x === 3) {
      break outer;
    }
  }
}
```

The break produces a completion carrying:

```text
type = break
target = "outer"
```

The target allows surrounding algorithms to determine which labeled construct should consume the completion.

---

# 19. Continue Completion

Example:

```js
for (const value of values) {
  if (value < 0) {
    continue;
  }

  process(value);
}
```

A `continue` represents another non-local control transfer.

The surrounding iteration machinery determines where execution resumes.

Conceptually:

```text
ContinueCompletion(empty-or-target)
```

---

# 20. Abrupt Completion

A completion that is not:

```text
normal
```

is generally treated as an:

```text
abrupt completion
```

This includes:

```text
throw
return
break
continue
```

Why is this useful?

Because specification algorithms often need to say:

```text
if this step did not complete normally,
propagate that outcome instead of continuing.
```

---

# 21. Completion Is About Control Flow, Not Only Errors

A common misconception is:

```text
Completion Record = exception object
```

Incorrect.

Completion includes:

```text
normal
return
break
continue
throw
```

Therefore:

```text
completion
```

is better understood as:

```text
evaluation outcome + control-flow mode
```

---

# 22. Completion Value

A completion can carry a:

```text
[[Value]]
```

For example:

```text
normal + 42
return + 42
throw + Error
```

The meaning of that value depends on the completion type.

For:

```text
return
```

the value is what the function is returning.

For:

```text
throw
```

the value is the thrown value.

For:

```text
break
```

the value is generally not the useful part; the target matters.

---

# 23. Completion Target

A completion can carry a:

```text
[[Target]]
```

used for labeled control flow.

Example:

```js
search: {
  for (const item of items) {
    if (match(item)) {
      break search;
    }
  }
}
```

The completion carries the label needed to decide which construct consumes it.

This is why the specification needs more than:

```text
success/failure
```

---

# 24. Completion Propagation

Suppose:

```js
function f() {
  throw new Error("boom");
}

function g() {
  return f();
}
```

Conceptually:

```text
f evaluation
→ throw completion

g's evaluation of f()
→ receives abrupt completion

g propagates it
```

unless some surrounding mechanism handles it.

This is the specification-level foundation of:

```text
exception propagation
```

---

# 25. `try/catch` as Completion Handling

Example:

```js
try {
  throw new Error("boom");
} catch (error) {
  console.log("caught");
}
```

At a conceptual level:

```text
throw completion
        ↓
try statement receives abrupt result
        ↓
catch handler selected
        ↓
handler executes
        ↓
new completion emerges
```

The catch block does not magically erase the exception.

It participates in the algorithm that handles the prior completion.

---

# 26. `finally` and Completion Replacement

Consider:

```js
function f() {
  try {
    return 1;
  } finally {
    return 2;
  }
}
```

The first part produces:

```text
return 1
```

but `finally` executes before the function completes.

The later:

```text
return 2
```

can replace the earlier completion.

This is one reason `return` inside `finally` is dangerous.

The key model is:

```text
completion generated
→ cleanup/finalization executes
→ cleanup can replace or propagate the completion
```

---

# 27. `finally` Can Mask Errors

Example:

```js
function f() {
  try {
    throw new Error("important");
  } finally {
    return 10;
  }
}
```

The throw completion can be replaced by the return completion.

Observable result:

```text
10
```

not:

```text
Error("important")
```

This is not an engine accident.

It follows from completion propagation/control-flow semantics.

---

# 28. Specification Shorthand `?`

ECMAScript algorithms frequently use:

```text
?
```

before an operation.

Conceptually:

```text
? operation()
```

means:

```text
perform operation
if it produces an abrupt completion,
propagate it immediately
otherwise extract its normal value
```

The specification defines this shorthand explicitly. citeturn449716search7

---

# 29. Specification Shorthand `!`

The:

```text
!
```

shorthand means, conceptually:

```text
the operation is expected to produce a normal completion;
if not, the algorithm's assumptions are violated
```

The specification's rewrite rules express this as an assertion of normal completion followed by extraction of the value. citeturn449716search7

These operators are specification notation.

They are not JavaScript syntax.

---

# 30. Why `?` Matters

Consider an abstract operation:

```text
Get(O, P)
```

which can fail.

Another algorithm may say:

```text
Let value be ? Get(O, P).
```

This compactly expresses:

```text
perform Get
if Get throws:
    propagate the abrupt completion
otherwise:
    store the normal value
```

Without the shorthand, specification algorithms would become extremely repetitive.

---

# 31. What Is a Reference Record?

A Reference Record models a resolved or potentially unresolved binding/property location.

The current specification describes fields including:

```text
[[Base]]
[[ReferencedName]]
[[Strict]]
[[ThisValue]]
```

and uses Reference Records to explain behavior of:

```text
delete
typeof
assignment
super
other language features
```

The left side of an assignment is a classic example. citeturn449716search0

---

# 32. Value vs Reference

This distinction is foundational.

Consider:

```js
const x = 10;
```

The expression:

```js
x
```

can be evaluated through a reference to the binding.

But when an operation needs the actual value, the reference is resolved.

Conceptually:

```text
identifier
→ Reference Record
→ GetValue
→ 10
```

This explains why JavaScript semantics cannot be modeled entirely as:

```text
"every expression immediately produces a value."
```

Some expressions first produce a reference-like semantic result.

---

# 33. Identifier Reference

For:

```js
x
```

the language needs to determine:

```text
Which environment contains x?
```

A Reference Record can represent:

```text
base = Environment Record
name = "x"
strict = ...
```

Then:

```text
GetValue(reference)
```

obtains the binding's value.

---

# 34. Property Reference

For:

```js
obj.x
```

the semantic path is different.

The reference conceptually identifies:

```text
base = obj
name = "x"
```

Then:

```text
GetValue(propertyReference)
```

performs property access through the appropriate object internal method.

---

# 35. Environment Reference vs Property Reference

This distinction is important:

```text
environment reference
```

represents:

```text
binding in an Environment Record
```

whereas:

```text
property reference
```

represents:

```text
property access on an object-like base
```

This affects:

```text
GetValue
PutValue
delete
strict behavior
this/super behavior
```

---

# 36. Unresolvable Reference

Suppose:

```js
missingName;
```

and no binding exists.

Conceptually, identifier evaluation can produce an:

```text
unresolvable Reference
```

Then:

```text
GetValue(unresolvableReference)
```

throws:

```text
ReferenceError
```

The specification explicitly defines this behavior. citeturn449716search0

---

# 37. Why `typeof missingName` Is Special

Consider:

```js
typeof missingName;
```

This does not throw the same way an ordinary value read would.

Its semantics contain special treatment for unresolved references.

This is one reason the language requires reference semantics instead of immediately reducing every identifier to:

```text
"give me the value or throw"
```

The operation:

```text
typeof
```

can inspect the reference before ordinary value retrieval.

---

# 38. Strictness in Reference Records

A Reference Record contains:

```text
[[Strict]]
```

This allows later operations to know whether the reference came from strict-mode code.

That information can affect behavior such as:

```text
assignment to unresolvable references
property deletion rules
```

The reference therefore carries not only:

```text
where
```

but also:

```text
contextual semantic information
```

---

# 39. `PutValue`

`PutValue` takes:

```text
Reference Record
+
new value
```

and performs the appropriate update.

Conceptually:

```text
identifier reference
→ environment binding update
```

or:

```text
property reference
→ object property update
```

The current specification explicitly defines `PutValue` for these cases. citeturn449716search0

---

# 40. Assignment Is Not Just "Set Variable"

Consider:

```js
x = 10;
```

Conceptual path:

```text
evaluate x
→ Reference Record
→ evaluate 10
→ PutValue(reference, 10)
```

Now consider:

```js
obj.x = 10;
```

Conceptual path:

```text
evaluate obj
→ property reference for "x"
→ evaluate 10
→ PutValue(propertyReference, 10)
```

The same surface operator:

```text
=
```

uses different reference bases.

---

# 41. Compound Assignment

Consider:

```js
x += 2;
```

A simplified semantic model is:

```text
evaluate x
→ Reference
→ GetValue(reference)
→ evaluate 2
→ apply addition semantics
→ PutValue(reference, result)
```

This explains an important rule:

```text
the left-hand side is evaluated as a reference before its current value is retrieved.
```

---

# 42. Update Expressions

Consider:

```js
obj.count++;
```

Conceptually:

```text
evaluate obj.count
→ property Reference
→ GetValue
→ numeric conversion
→ increment
→ PutValue
→ produce old/new result according to ++ semantics
```

This involves the reference remaining meaningful across multiple steps.

---

# 43. Why `a.b` Can Be a Reference

The expression:

```js
obj.name
```

is not necessarily just:

```text
"produce current value of name"
```

in the semantic pipeline.

When used as an assignment target:

```js
obj.name = "new";
```

the language needs a representation of:

```text
which property is being targeted
```

That is why the evaluation can produce a Reference Record that `PutValue` later consumes.

---

# 43. `GetValue`

The specification defines:

```text
GetValue(V)
```

for either:

```text
Reference Record
```

or:

```text
ordinary ECMAScript language value
```

If `V` is already a value:

```text
return V
```

If it is a Reference:

```text
resolve binding/property
```

The current specification explicitly defines the property-reference and environment-reference paths. citeturn449716search0

---

# 45. Property `GetValue` Path

Conceptually:

```text
Reference
  ↓
IsPropertyReference?
  ↓ yes
ToObject(base)
  ↓
private or ordinary property handling
  ↓
[[Get]]
  ↓
value
```

Notice that the object semantics eventually reach:

```text
[[Get]]
```

which connects this chapter to:

```text
Chapter 43 — Ordinary Object Internal Methods
```

---

# 46. Environment `GetValue` Path

Conceptually:

```text
Reference
  ↓
not a property reference
  ↓
Base = Environment Record
  ↓
GetBindingValue(name, strict)
  ↓
value
```

This connects Reference Records to:

```text
lexical environments
bindings
TDZ
strict mode
```

---

# 47. `GetThisValue`

A property reference can have special behavior for:

```text
super
```

The specification defines:

```text
GetThisValue
```

to recover the appropriate receiver.

For an ordinary property reference:

```text
base
```

is used.

For a Super Reference Record:

```text
[[ThisValue]]
```

is retained explicitly. citeturn449716search0

---

# 48. Why `super` Needs Extra Reference State

Consider:

```js
class Child extends Parent {
  method() {
    return super.value;
  }
}
```

The property lookup starts from the parent prototype chain, but method invocation should still use the current instance as the receiver.

Conceptually:

```text
lookup base
    ≠
receiver / this value
```

The Reference Record's additional:

```text
[[ThisValue]]
```

field lets the specification model that distinction.

---

# 49. Super Reference

A Super Reference can be thought of as:

```text
property reference
+
special receiver
```

The base is where the property lookup begins.

The `this` value is the receiver used for access.

This explains behaviors such as:

```js
super.method()
```

where:

```text
method lookup
```

and:

```text
this binding
```

are not identical operations.

---

# 50. Private References

Private fields create another specialized reference category.

Example:

```js
class A {
  #value = 1;

  read() {
    return this.#value;
  }
}
```

A private reference carries a:

```text
Private Name
```

rather than ordinary property-key semantics.

`GetValue` and other reference operations therefore have special branches for private references. citeturn449716search0

---

# 51. Property Key vs Private Name

These are different semantic concepts:

```text
"secret"
```

as an ordinary property key is not the same as:

```text
#secret
```

as a private name.

The language intentionally prevents arbitrary dynamic lookup of private fields with:

```js
obj["#secret"]
```

That is a different property access altogether.

---

# 52. `InitializeReferencedBinding`

There is also a distinct operation:

```text
InitializeReferencedBinding
```

used when a binding that is not yet initialized receives its initial value.

This is especially useful for understanding:

```text
let
const
class
TDZ
declaration instantiation
```

The specification performs initialization through the appropriate Environment Record methods. citeturn449716search0

---

# 53. Initialization vs Assignment

These are not the same.

Consider:

```js
let x;
```

and later:

```js
x = 10;
```

The binding lifecycle includes:

```text
create binding
→ initialize binding
→ later assignments
```

For `const`:

```text
create binding
→ initialize once
→ later PutValue attempts are restricted
```

The semantic machinery distinguishes:

```text
InitializeReferencedBinding
```

from:

```text
PutValue
```

---

# 54. TDZ Through This Model

Consider:

```js
{
  console.log(x);
  let x = 10;
}
```

The binding exists structurally before the initializer runs, but its state is uninitialized.

Conceptually:

```text
identifier evaluation
→ Reference to x
→ GetValue
→ Environment Record sees uninitialized binding
→ ReferenceError
```

This gives a specification-level explanation of TDZ.

---

# 55. `delete` and References

Consider:

```js
delete obj.x;
```

The operation cares about the property location.

Likewise:

```js
delete missingName;
```

has different semantics from:

```js
delete obj.x;
```

This is another reason reference semantics exist:

```text
delete
```

sometimes needs to operate on:

```text
where the expression refers
```

rather than simply:

```text
what value did the expression produce?
```

---

# 56. Why Values Alone Are Insufficient

Imagine the expression:

```js
obj.x
```

produces:

```text
10
```

If all semantic information were reduced immediately to:

```text
10
```

then later:

```js
obj.x = 20;
```

would have no way to know:

```text
which object
which property
```

should be updated.

Reference semantics preserve the target information until the operation consuming it needs a value or performs an update.

---

# 57. Reference Is Not a Pointer

Do not over-map Reference Records to:

```text
C pointer
memory address
JavaScript object pointer
```

A Reference Record is a specification type.

It can represent:

```text
environment binding
object property
super access
private-name access
```

The engine may use addresses, handles, registers, stack slots, inline caches, or other mechanisms.

The specification does not require a literal pointer.

---

# 58. Reference Lifecycle

A useful conceptual model:

```text
source expression
      ↓
produce Reference
      ↓
consumer asks:
  GetValue?
  PutValue?
  delete?
  typeof?
      ↓
reference-specific semantics
```

Reference Records are often transient specification artifacts.

A program cannot directly store:

```text
"the Reference Record for x"
```

as a JavaScript value.

---

# 59. Assignment Evaluation Walkthrough

Example:

```js
let x = 1;

x = 2;
```

Conceptual sequence:

```text
1. resolve x
2. produce environment Reference
3. evaluate 2
4. PutValue(reference, 2)
5. assignment completes normally
```

The observable result is simple.

The semantic machinery is not.

---

# 60. Property Assignment Walkthrough

Example:

```js
const obj = { x: 1 };

obj.x = 2;
```

Conceptual sequence:

```text
1. evaluate obj
2. create property Reference for x
3. evaluate 2
4. PutValue(propertyReference, 2)
5. object internal [[Set]] semantics execute
6. assignment completes
```

This connects references to:

```text
property descriptors
prototype behavior
accessors
Proxy
strict mode
```

---

# 61. Accessor Property Connection

Consider:

```js
const obj = {
  set x(value) {
    console.log("setter", value);
  }
};

obj.x = 10;
```

`PutValue` eventually leads into property-setting semantics.

Those semantics may invoke:

```text
setter
```

rather than simply storing data.

This demonstrates:

```text
Reference
→ PutValue
→ internal object method
→ user-defined behavior
```

---

# 62. Proxy Connection

Consider:

```js
const target = {};
const proxy = new Proxy(target, {
  set(target, key, value) {
    console.log("set", key, value);
    return Reflect.set(target, key, value);
  }
});

proxy.x = 10;
```

The reference target eventually reaches property-setting behavior that can involve Proxy internal methods.

This connects:

```text
Reference Records
```

to:

```text
Proxy
Reflect
internal methods
metaprogramming
```

---

# 63. Completion + Reference Together

Consider:

```js
function f(obj) {
  obj.value = 10;
  return obj.value;
}
```

Conceptually:

```text
obj.value
→ Reference
→ PutValue
→ normal completion

obj.value
→ Reference
→ GetValue
→ normal completion with 10

return 10
→ return completion

function call machinery consumes return completion
```

This combines:

```text
reference semantics
+
completion semantics
```

---

# 64. Evaluation Produces More Than Values

A useful conceptual table:

| Expression / operation | Useful semantic result |
|---|---|
| `1 + 2` | value |
| `x` | reference before value retrieval |
| `obj.x` | property reference before value retrieval |
| `x = 2` | completion after assignment |
| `return 2` | return completion |
| `break label` | break completion |
| `throw error` | throw completion |

This is a much richer model than:

```text
"every expression just returns a value."
```

---

# 65. Completion and Value Propagation

Specification algorithms often compose like:

```text
A
↓
result
↓
if abrupt → propagate
↓
otherwise continue with value
```

This is why algorithm notation frequently uses:

```text
?
```

The shorthand expresses a control-flow discipline.

---

# 66. Completion and `await`

Async functions introduce additional layers because ordinary synchronous completion becomes part of Promise/async machinery.

Conceptually:

```text
evaluation
→ completion
→ async function machinery
→ Promise settlement
```

A throw completion inside an async function contributes to:

```text
Promise rejection
```

rather than behaving exactly like a top-level synchronous throw.

The exact async algorithms belong to later chapters.

---

# 67. Completion and Generators

Generators add:

```text
yield
suspend
resume
```

which interact with completion-like control states.

Example:

```js
function* g() {
  yield 1;
  return 2;
}
```

The generator protocol exposes results such as:

```js
{ value: 1, done: false }
{ value: 2, done: true }
```

The specification internally tracks richer execution/completion state than this public object shape.

---

# 68. `return`, `throw`, and Generator Control

Generator methods such as:

```js
iterator.return(value)
iterator.throw(error)
```

inject control into the suspended generator computation.

Conceptually:

```text
external request
→ resume suspended execution with control input
→ completion handling
→ generator result
```

This is another example of why completion semantics matter beyond ordinary functions.

---

# 69. `break` and `continue` Are Not Exceptions

A common misconception is:

```text
"break and continue work like exceptions."
```

They do not have the same public semantics or intended abstraction.

The specification uses Completion Records to model all of these non-local control transfers using one general mechanism.

That is a modeling technique.

It does not imply:

```text
break = exception
```

at the engine or performance level.

---

# 70. Completion Records and Engine Internals

An optimizing engine might implement:

```text
return
break
continue
throw
```

using:

```text
branch instructions
register state
jump targets
runtime calls
exception tables
```

The language specification does not require an engine to literally allocate:

```text
CompletionRecord objects
```

for every operation.

This distinction is essential for performance discussions.

---

# 71. Specification → Implementation Translation

A useful translation exercise:

### Specification

```text
Evaluate identifier
→ Reference
→ GetValue
```

### Engine reality might be

```text
scope lookup
→ stack slot / context slot
→ load value
```

Likewise:

### Specification

```text
PutValue(reference, value)
```

### Engine reality might be

```text
resolve environment slot
→ store
```

or:

```text
property inline cache
→ optimized store
```

The external semantics must match.

The internal representation does not have to.

---

# 72. Execution Context → Engine Mapping

Conceptually:

```text
Execution Context
├─ LexicalEnvironment
├─ VariableEnvironment
├─ Realm
├─ Function
├─ ScriptOrModule
└─ execution state
```

Possible engine representation:

```text
stack frame
+
function metadata
+
context object
+
register state
+
bytecode/interpreter state
```

or:

```text
optimized frame
+
deoptimization metadata
+
heap environment object
```

The mapping is implementation-dependent.

---

# 73. Completion → Engine Mapping

Conceptually:

```text
Completion {
  Type,
  Value,
  Target
}
```

Possible implementation:

```text
return → jump
break → jump target
continue → loop back-edge
throw → exception path
normal → ordinary control flow
```

Again:

```text
specification representation
```

is not necessarily:

```text
runtime allocation
```

---

# 74. Reference → Engine Mapping

Conceptually:

```text
Reference {
  Base,
  ReferencedName,
  Strict,
  ThisValue?
}
```

Possible engine representations include:

```text
scope slot
property key + receiver
bytecode operand
register
hidden metadata
temporary compiler state
```

An engine can optimize away the explicit abstraction.

---

# 75. Edge Case — Assignment Through a Getter/Setter

```js
const object = {
  get x() {
    return 1;
  },
  set x(value) {
    console.log(value);
  }
};

object.x = 10;
```

Reason using:

```text
property Reference
→ PutValue
→ [[Set]]
→ accessor semantics
```

Do not conclude:

```text
"x is just overwritten with 10."
```

---

# 76. Edge Case — Prototype Setter

```js
const proto = {
  set x(value) {
    console.log("setter", value);
  }
};

const obj = Object.create(proto);

obj.x = 10;
```

Reason through:

```text
property reference
→ PutValue
→ prototype-aware [[Set]]
→ inherited accessor
```

This connects directly to:

```text
prototype chain
property descriptors
internal methods
```

---

# 77. Edge Case — Primitive Base

```js
const text = "hello";

console.log(text.length);
```

The specification can model property access involving a primitive base.

`GetValue` uses the property-reference path and appropriate object conversion/internal behavior.

Do not infer that the source necessarily creates a long-lived wrapper object visible to the programmer.

The specification itself notes that an intermediate object may be an abstract/runtime optimization detail rather than a persistent object exposed externally. citeturn449716search0

---

# 78. Edge Case — `super.x = value`

```js
class Parent {
  set x(value) {
    console.log("parent setter", value);
  }
}

class Child extends Parent {
  method() {
    super.x = 10;
  }
}
```

Reason using:

```text
Super Reference
→ [[Base]]
→ [[ThisValue]]
→ PutValue
→ property-set semantics
```

The property lookup base and actual receiver must not be confused.

---

# 79. Edge Case — Private Field Access

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}
```

Reason using:

```text
Private Reference
→ private name
→ PrivateGet
→ value
```

This is different from:

```js
this["#name"]
```

which is ordinary property access.

---

# 80. Edge Case — Assignment to Unresolvable Name

Non-strict vs strict behavior differs for unresolved assignment.

Conceptually:

```js
missing = 10;
```

requires the specification to know:

```text
Is the reference unresolvable?
Is the reference strict?
```

Reference semantics therefore preserve information that affects the eventual result.

Do not reduce unresolved assignment to:

```text
"JavaScript creates a variable."
```

without specifying the code mode and exact semantics.

---

# 81. Edge Case — `delete` and Strict Mode

Consider:

```js
"use strict";

delete obj.property;
```

versus deletion of an unqualified identifier.

The legality and runtime result are governed by different semantic paths.

Again:

```text
Reference category
+
strictness
```

matter.

---

# 82. Edge Case — Reference vs Value Aliasing

Consider:

```js
let a = 1;
let b = a;

b = 2;
```

There is no persistent reference alias between:

```text
a
```

and:

```text
b
```

The semantic reference used to read `a` during initializer evaluation is resolved to a value.

The later assignment targets:

```text
b's binding
```

This helps distinguish:

```text
reference used during evaluation
```

from:

```text
stored JavaScript value
```

---

# 83. Common Misconceptions

### Misconception 1

> “Reference Record is a JavaScript pointer.”

Correction:

```text
Reference Record is a specification type.
```

### Misconception 2

> “Completion Record means exception.”

Correction:

```text
Completion models normal and abrupt control flow.
```

### Misconception 3

> “Every expression immediately becomes a value.”

Correction:

```text
Some evaluations first produce references.
```

### Misconception 4

> “`return` is just a value.”

Correction:

```text
return initiates non-local control flow represented by a return completion.
```

### Misconception 5

> “`?` is JavaScript syntax.”

Correction:

```text
It is ECMAScript specification notation.
```

### Misconception 6

> “Completion Records are allocated objects at runtime.”

Correction:

```text
They are specification abstractions; engines may implement them differently.
```

---

# 84. Common Mistakes

```text
[ ] confusing Environment Records with Reference Records
[ ] confusing a reference with its resolved value
[ ] ignoring strictness in reference semantics
[ ] treating property references as environment bindings
[ ] forgetting Super Reference receiver semantics
[ ] treating private names as normal property keys
[ ] thinking return/break/continue are ordinary values
[ ] ignoring completion propagation
[ ] interpreting ? as runtime syntax
[ ] assuming specification records map one-to-one to heap objects
```

---

# 85. Comparison With Related Concepts

| Concept | Core Question |
|---|---|
| Environment Record | Where are bindings stored/resolved? |
| Reference Record | What binding/property location does this expression denote? |
| Value | What data has been resolved? |
| Completion Record | How did evaluation finish? |
| Execution Context | What execution state applies right now? |
| Execution stack/frame | How might an implementation represent execution state? |
| Internal Method | What semantic operation does an object perform? |
| Promise | How is asynchronous result state represented publicly? |

---

# 86. Performance Considerations

Never assume:

```text
Reference Record
```

means:

```text
extra allocation
```

or:

```text
Completion Record
```

means:

```text
heap object per statement
```

Optimizing engines can represent these concepts efficiently.

Performance discussions should instead investigate:

```text
scope access
property access
inline caches
deoptimization
exception paths
control-flow structure
allocation
```

where relevant.

---

# 87. Memory Considerations

The specification abstractions do not directly determine:

```text
heap usage
stack usage
object lifetime
allocation count
```

Those depend on implementation.

However, the semantic model helps identify where state must conceptually exist.

For example:

```text
closure
generator suspension
async continuation
```

can require an implementation to preserve semantic state across time.

---

# 88. Security Considerations

Reference semantics matter indirectly for security-sensitive reasoning.

Examples:

```text
prototype lookup
property assignment
private fields
environment bindings
strict-mode assignment
```

Understanding:

```text
where a name/property resolves
```

helps prevent authorization and object-integrity mistakes.

The specification model does not itself provide application authorization.

---

# 89. Production Usage

This chapter is especially useful when:

```text
debugging tricky JavaScript
reviewing language semantics
reading ECMAScript algorithms
implementing interpreters
building compilers
building static analyzers
building transpilers
analyzing Proxy behavior
understanding class/private semantics
debugging engine behavior
```

---

# 90. Implementation From Scratch — Mini Evaluator

Implement a tiny evaluator for a restricted language containing:

```text
identifiers
integer literals
assignment
addition
return
```

Represent:

```text
Reference
Completion
```

as ordinary implementation data structures in your toy interpreter.

Example internal model:

```js
{
  kind: "reference",
  base: environment,
  name: "x",
  strict: false
}
```

and:

```js
{
  type: "return",
  value: 10
}
```

Do not claim these objects are the ECMAScript specification's internal objects.

They are your pedagogical implementation.

---

# 91. Mini Evaluator Milestones

### Milestone 1

Environment lookup.

### Milestone 2

Reference creation.

### Milestone 3

GetValue.

### Milestone 4

PutValue.

### Milestone 5

Normal completion.

### Milestone 6

Return completion.

### Milestone 7

Throw completion.

### Milestone 8

Break/continue completion.

### Milestone 9

Property reference.

### Milestone 10

Proxy-like interception in the toy runtime.

---

# 92. Implementation From Scratch — Completion Propagation

Implement:

```text
evaluate(statement)
```

so it can return:

```text
Normal(value)
Return(value)
Throw(error)
Break(target)
Continue(target)
```

Then write a block evaluator that:

```text
continues on Normal
propagates Return
propagates Throw
consumes matching Break
consumes matching Continue
```

This makes the completion model tangible.

---

# 93. Debugging Exercises

## Exercise A — Wrong Binding

```js
let x = 1;

function f() {
  let x = 2;
  return x;
}
```

Trace:

```text
Reference for x
→ environment lookup
→ GetValue
```

Identify which binding is selected.

---

## Exercise B — Property Target

```js
const obj = { x: 1 };

obj.x = 2;
```

Trace:

```text
object evaluation
→ property reference
→ PutValue
→ property set
```

---

## Exercise C — `super`

```js
class Parent {
  method() {
    return this.value;
  }
}

class Child extends Parent {
  test() {
    return super.method();
  }
}
```

Trace the distinction between:

```text
method lookup
receiver
```

---

# 94. Code Review Exercise

Review:

```js
function assign(target, key, value) {
  const current = target[key];

  if (current) {
    console.log("existing");
  }

  target[key] = value;
}
```

Questions:

```text
What reference is created for target[key]?

When does GetValue occur?

When does PutValue occur?

Can getter behavior occur?

Can Proxy behavior occur?

Can prototype behavior matter?

Can reading and writing invoke different behavior?
```

Then rewrite the explanation in specification-oriented terminology.

---

# 95. Interview Questions

### Fundamentals

```text
1. What is a Completion Record?
2. What is a Reference Record?
3. What is an Execution Context?
4. Why does JavaScript need Reference Records?
5. What is an abrupt completion?
```

### Advanced

```text
6. What is the difference between a Reference and a value?
7. What does GetValue do?
8. What does PutValue do?
9. Why does typeof missingName work?
10. Why does strictness exist in a Reference Record?
11. What is a Super Reference?
12. Why does super need a separate this value?
13. How are private references different?
14. How does return propagate?
15. How does finally affect completion?
```

### Principal

```text
16. How would you explain Completion Records without claiming engines allocate them?
17. How does Reference semantics explain Proxy behavior?
18. How would you build a toy interpreter using Completion and Reference abstractions?
19. How does specification state map onto optimized engine internals?
20. Why is the distinction between specification abstraction and implementation detail critical?
```

---

# 96. Predict-the-Behavior Exercises

Before checking, classify:

```text
reference
value
normal completion
abrupt completion
```

### Exercise 1

```js
let x = 1;
x = 2;
```

Identify the reference during assignment.

### Exercise 2

```js
const obj = { x: 1 };
obj.x++;
```

Identify:

```text
reference
GetValue
PutValue
```

### Exercise 3

```js
function f() {
  return 10;
}
```

Identify the completion generated by:

```text
return 10
```

### Exercise 4

```js
try {
  throw new Error("x");
} finally {
  return 1;
}
```

Identify which completion ultimately escapes.

### Exercise 5

```js
typeof notDefined;
```

Explain why the ordinary unresolved-reference path is not observed as a ReferenceError here.

---

# 97. Mastery Exercises

### Exercise 1 — Reference Tracer

Create a toy evaluator that logs:

```text
Reference created
GetValue
PutValue
```

for:

```js
x = y + 1
```

### Exercise 2 — Completion Tracer

Trace:

```js
function f() {
  try {
    return 1;
  } finally {
    return 2;
  }
}
```

### Exercise 3 — Control-Flow Interpreter

Implement:

```text
break
continue
return
throw
```

using completion objects.

### Exercise 4 — Property Semantics

Implement a toy object model supporting:

```text
data property
getter
setter
prototype lookup
```

Then connect it to your Reference abstraction.

### Exercise 5 — Specification Reading

Take one ECMAScript algorithm that uses:

```text
?
!
GetValue
PutValue
Completion
```

and rewrite it in plain English without losing the propagation logic.

---

# 98. Track A — Core Theory

Master:

```text
Execution Context
Environment Record
Completion Record
normal completion
abrupt completion
return/break/continue/throw
Reference Record
environment reference
property reference
unresolvable reference
strict reference
super reference
private reference
GetValue
PutValue
GetThisValue
InitializeReferencedBinding
```

Deliverable:

```text
explain the semantic pipeline for an expression/control-flow construct
```

---

# 99. Track B — Implementation

Build:

```text
toy interpreter
reference model
completion model
property semantics
control-flow propagation
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

Deliverable:

```text
demonstrate how language semantics can be implemented
```

---

# 100. Track C — Interview / Reasoning

Practice:

```text
"Why is x not just a value?"

"How does x = 10 work semantically?"

"Why does obj.x behave differently from x?"

"How does super preserve the receiver?"

"How does return leave nested statement evaluation?"

"Why doesn't typeof missingName throw?"
```

Deliverable:

```text
precise specification-level explanations
```

---

# 101. Specification / Runtime Source Discipline

Prefer:

```text
ECMAScript specification
→ abstract operations
→ internal methods
→ engine implementation
→ host behavior
```

For this chapter, the primary normative material is the ECMAScript specification's sections on:

```text
Execution Contexts
Environment Records
Completion Records
Reference Records
```

The current specification defines Reference Record fields and `GetValue`/`PutValue` semantics, and defines Execution Context state as a specification model. citeturn449716search0turn449716search1

When discussing implementation:

```text
label the statement as engine-specific
```

rather than treating a particular stack-frame or register representation as universal.

---

# 102. Common Failure Modes

```text
Failure 1:
Confusing a Reference with a pointer.

Failure 2:
Confusing a Reference with a value.

Failure 3:
Treating Completion as exception-only machinery.

Failure 4:
Ignoring abrupt completion propagation.

Failure 5:
Missing the role of [[Strict]].

Failure 6:
Treating super property lookup and this receiver as the same thing.

Failure 7:
Treating private names as normal property keys.

Failure 8:
Assuming the specification's records are literal runtime objects.

Failure 9:
Forgetting GetValue before ordinary value computation.

Failure 10:
Forgetting that assignment needs a target, not only a value.
```

---

# 103. Principal Decision Framework

When reading or implementing specification-level semantics, evaluate:

```text
Correctness
Specification fidelity
Runtime mapping
Performance implications
Memory implications
Debuggability
Tooling value
Maintainability
Future language evolution
Implementation complexity
```

The goal is not to reproduce the specification's prose literally in code.

The goal is:

```text
preserve observable semantics
```

while choosing an efficient implementation strategy.

---

# 104. Retrieval Record

```md
# Chapter 124 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Execution Context
-

## Completion Model
-

## Reference Model
-

## GetValue / PutValue
-

## Super / Private References
-

## Strongest Areas
-

## Weakest Areas
-

## Specification Reading Difficulty
-

## Implementation Progress
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 105. Spaced Retrieval Schedule

### Day 0

Study:

```text
Execution Context
Completion
Reference
```

and complete the prediction exercises.

### Day 1

Trace:

```text
x = 10
obj.x = 10
obj.x++
```

using reference/completion notation.

### Day 3

Explain:

```text
normal vs abrupt completion
```

without notes.

### Day 7

Implement the toy completion evaluator.

### Day 14

Explain:

```text
super
private reference
typeof unresolved name
```

from memory.

### Day 21

Read one specification algorithm and rewrite it in plain language.

### Day 30

Build the mini evaluator again without the original implementation.

---

# 106. Dependency Graph

```text
Chapter 10
scope / identifier resolution
        ↓
Chapter 11
TDZ / declarations
        ↓
Chapter 12
execution contexts
        ↓
Chapter 15
objects / property semantics
        ↓
Chapter 17
prototype chains
        ↓
Chapter 18
classes
        ↓
Chapter 41
specification architecture
        ↓
Chapter 42
abstract operations
        ↓
Chapter 43
ordinary object internal methods
        ↓
Chapter 123
grammar / parsing / early errors
        ↓
Chapter 124
execution records + completion records + references
        ↓
Chapter 125
Promise internals + Promise capability machinery
```

---

# 107. Concept Connections

## Depends On

```text
scope
environment records
execution contexts
objects
property access
classes
private fields
strict mode
specification algorithms
```

## Builds Toward

```text
Promise internals
module linking/evaluation
async function semantics
engine implementation
language tooling
interpreter/compiler design
```

## Related Concepts

```text
AST
bytecode
stack frames
closures
internal methods
Proxy
Reflect
generator suspension
async continuations
```

## Concepts Revisited

```text
TDZ
this
super
private fields
assignment
delete
typeof
return
throw
break
continue
```

## Why This Chapter Matters

This chapter is the bridge between:

```text
JavaScript syntax
```

and:

```text
JavaScript semantic machinery
```

Once you can distinguish:

```text
Reference
Value
Completion
Execution State
```

many advanced JavaScript behaviors stop looking like isolated exceptions.

---

# 108. Completion Snapshot

```md
# Chapter 124 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Primary Gaps:
-

Execution Context:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Completion Records:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Reference Records:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

GetValue / PutValue:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Super / Private:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Specification Reading:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Toy Interpreter:
[ ] not started
[ ] partial
[ ] complete
[ ] hardened
```

---

# 109. Completion Criteria

```text
[ ] Execution Context explained
[ ] Environment Record relationship explained
[ ] Completion Record explained
[ ] normal completion understood
[ ] abrupt completion understood
[ ] return/break/continue/throw distinguished
[ ] completion propagation understood
[ ] ? and ! specification notation understood
[ ] Reference Record explained
[ ] environment reference understood
[ ] property reference understood
[ ] unresolvable reference understood
[ ] strict reference semantics understood
[ ] Super Reference understood
[ ] Private Reference understood
[ ] GetValue understood
[ ] PutValue understood
[ ] GetThisValue understood
[ ] InitializeReferencedBinding understood
[ ] value/reference distinction understood
[ ] assignment traced semantically
[ ] property assignment traced semantically
[ ] control-flow completion traced
[ ] toy evaluator implemented
```

---

# 110. Final Principal Mental Model

When reading difficult ECMAScript semantics, ask:

```text
What code is being evaluated?

What execution context is active?

Which environment is consulted?

Does evaluation produce a value or a reference?

If a reference exists, what is its base?

What is the referenced name?

Is it a property, binding, super reference, or private reference?

When does GetValue occur?

When does PutValue occur?

How does evaluation complete?

Is the completion normal or abrupt?

If abrupt, who consumes or propagates it?

What part is language semantics?

What part is engine implementation?
```

This is the semantic tracing discipline required for advanced ECMAScript work.

---

# 111. Final Principal Principle

> **A value tells you what something is. A reference tells you what location or binding an expression denotes. A completion tells you how evaluation ended. An execution context tells you what semantic state surrounds that evaluation.**

The mature model is:

```text
Execution State
      ↓
Evaluation
      ↓
Reference or Value
      ↓
GetValue / PutValue / other operation
      ↓
Completion
      ↓
Propagation or handling
      ↓
Next semantic step
```

Once this model becomes intuitive, you can read difficult ECMAScript algorithms without translating every step into vague “JavaScript magic.”

You can instead reason:

```text
state
→ reference
→ operation
→ completion
→ propagation
```

That is the foundation for understanding:

```text
assignment
scope
TDZ
super
private fields
Proxy
exceptions
return
break
continue
generators
async functions
Promises
```

and eventually:

```text
how a JavaScript engine turns language semantics into executable machinery.
```