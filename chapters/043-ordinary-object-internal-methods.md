# Chapter 43 — Ordinary Object Internal Methods

## Chapter Metadata

```text
Chapter: 43
Title: Ordinary Object Internal Methods
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what an ECMAScript internal method is.
- Explain what makes an object “ordinary”.
- Distinguish ordinary objects from exotic objects.
- Explain why ordinary objects are defined through a standard set of internal methods.
- Understand the core ordinary object internal methods:
  - `[[GetPrototypeOf]]`
  - `[[SetPrototypeOf]]`
  - `[[IsExtensible]]`
  - `[[PreventExtensions]]`
  - `[[GetOwnProperty]]`
  - `[[DefineOwnProperty]]`
  - `[[HasProperty]]`
  - `[[Get]]`
  - `[[Set]]`
  - `[[Delete]]`
  - `[[OwnPropertyKeys]]`
  - `[[Call]]`
  - `[[Construct]]` where applicable to function objects.
- Understand the conceptual object-internal-method interface.
- Explain the role of internal slots such as `[[Prototype]]` and `[[Extensible]]`.
- Explain how ordinary property access is implemented semantically.
- Explain how property descriptors participate in object operations.
- Explain data properties versus accessor properties.
- Explain how `[[Get]]` traverses prototypes.
- Explain how `[[Set]]` differs from `[[Get]]`.
- Explain the importance of the receiver argument.
- Explain how setters behave through inherited properties.
- Explain how own-property definition differs from assignment.
- Explain how non-writable and non-configurable properties constrain mutation.
- Explain how object extensibility constrains property creation.
- Explain the difference between:
  - own property;
  - inherited property;
  - enumerable property;
  - configurable property;
  - writable property.
- Explain `[[HasProperty]]` versus own-property checks.
- Explain `[[OwnPropertyKeys]]` and property-key ordering.
- Explain deletion semantics.
- Explain prototype mutation semantics.
- Explain invariants imposed by non-extensibility and property descriptors.
- Understand how Proxy objects relate to ordinary internal methods.
- Understand why Proxy behavior is constrained by invariants.
- Understand which internal methods are relevant to ordinary data objects versus function objects.
- Trace JavaScript syntax through abstract operations into ordinary internal methods.
- Implement a simplified ordinary object model.
- Implement simplified property descriptors.
- Implement prototype-aware `[[Get]]` and `[[Set]]`.
- Implement `[[DefineOwnProperty]]`.
- Implement `[[HasProperty]]`, `[[Delete]]`, and `[[OwnPropertyKeys]]`.
- Debug descriptor/prototype/accessor edge cases.
- Distinguish specification semantics from engine object representations.
- Use internal-method reasoning during code review and interview questions.
- Defend ordinary-object semantics at principal-engineer depth.

### Mastery Gate

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

Required:

- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations

Strongly related:

- Chapter 14 — `this` / Invocation / Binding
- Chapter 21 — Species / Subclassing / Derived Constructors
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 43 itself leads directly into Chapter 44 — Realms / Agents / Execution Isolation

---

## 3. What Is It?

An **internal method** is a specification-defined semantic operation associated with an ECMAScript object.

Examples:

```text
[[Get]]
[[Set]]
[[Delete]]
[[HasProperty]]
[[DefineOwnProperty]]
[[GetPrototypeOf]]
[[SetPrototypeOf]]
[[IsExtensible]]
[[PreventExtensions]]
[[OwnPropertyKeys]]
```

These methods are not normal JavaScript methods.

You cannot generally write:

```js
obj.[[Get]]("x");
```

Instead, source-level syntax and built-in APIs trigger the appropriate specification operations.

An **ordinary object** is an object whose internal-method behavior follows the ordinary object algorithms defined by ECMAScript, rather than using specialized exotic semantics for those operations.

A useful model is:

```text
JavaScript source
      ↓
abstract operation
      ↓
ordinary internal method
      ↓
object state
      ↓
result / completion
```

---

## 4. Why Does It Exist?

JavaScript objects appear simple:

```js
obj.x
obj.x = 10
delete obj.x
"x" in obj
```

But these operations must account for:

- own properties;
- inherited properties;
- descriptors;
- getters;
- setters;
- prototypes;
- extensibility;
- deletion;
- Proxies;
- Symbols;
- receiver semantics.

The specification therefore defines a small set of object-level internal methods.

Instead of every language feature inventing its own property rules, they reuse the same semantic operations.

The result is:

```text
consistent object semantics
+
reusable algorithms
+
explicit invariants
```

---

## 5. Mental Model

Think of an ordinary object as:

```text
Ordinary Object
├── own properties
├── [[Prototype]]
└── [[Extensible]]
```

and a semantic interface:

```text
                Ordinary Object
                       │
       ┌───────────────┼────────────────┐
       │               │                │
     [[Get]]         [[Set]]        [[Delete]]
       │               │                │
       ├── prototype   ├── descriptor   └── descriptor
       │   traversal   │   rules
       │               ├── receiver
       │               └── prototype
       │
       └── accessor/data rules
```

The most important insight:

> Object syntax is a surface language over internal object semantics.

For:

```js
obj.x
```

think:

```text
Evaluate MemberExpression
→ ToPropertyKey
→ Get
→ [[Get]]
→ property descriptor / prototype rules
→ value
```

For:

```js
obj.x = 10
```

think:

```text
Evaluate assignment
→ ToPropertyKey
→ PutValue
→ Set
→ [[Set]]
→ descriptor / prototype / receiver rules
→ completion
```

---

## 6. Core Rules

### Rule 1 — Internal methods are semantic operations

They are not ordinary user-visible methods.

### Rule 2 — Ordinary objects use ordinary internal-method algorithms

Specialized objects can use different internal behavior.

### Rule 3 — Property descriptors are central

Object-property semantics cannot be understood precisely without descriptors.

### Rule 4 — `[[Get]]` and `[[Set]]` are different

Reading and writing have different algorithms.

### Rule 5 — Prototype traversal is part of ordinary `[[Get]]`

A missing own property can be searched through the prototype chain.

### Rule 6 — Prototype traversal is also relevant to `[[Set]]`

Inherited setters and writable inherited data properties affect assignment.

### Rule 7 — Receiver matters

The object where lookup occurs and the object that ultimately receives an accessor-defined write are not always conceptually identical.

### Rule 8 — Non-extensibility constrains creation

A non-extensible ordinary object cannot simply acquire arbitrary new own properties.

### Rule 9 — Descriptor invariants constrain redefinition

Non-configurable properties impose strict limits on later changes.

### Rule 10 — Deletion is not assignment to `undefined`

```js
delete obj.x
```

has its own semantics.

### Rule 11 — `in` is not an own-property test

It corresponds conceptually to prototype-aware property existence.

### Rule 12 — Own-key enumeration is a semantic operation

Property ordering is defined at the specification level.

### Rule 13 — Accessor properties can execute code

A getter or setter is part of the property descriptor.

### Rule 14 — Internal methods can fail abruptly

A semantic operation can produce an abrupt completion.

### Rule 15 — Proxy behavior is constrained by object invariants

A Proxy cannot freely violate all underlying semantic requirements.

---

## 7. Syntax

The internal-method names are written as:

```text
[[Get]]
[[Set]]
[[Delete]]
[[HasProperty]]
[[DefineOwnProperty]]
[[GetOwnProperty]]
[[GetPrototypeOf]]
[[SetPrototypeOf]]
[[IsExtensible]]
[[PreventExtensions]]
[[OwnPropertyKeys]]
```

The language-level operations that connect to these include:

```js
obj.x
obj[x]
obj.x = value
delete obj.x
"x" in obj
Object.getOwnPropertyDescriptor(obj, "x")
Reflect.get(obj, "x")
Reflect.set(obj, "x", value)
Reflect.defineProperty(obj, "x", descriptor)
Object.getPrototypeOf(obj)
Object.setPrototypeOf(obj, proto)
```

These APIs are not simply wrappers in every detail, but they provide useful source-level entry points for understanding the internal methods.

---

## 8. Basic Examples

### Example 1 — Data property

```js
const obj = {
  x: 10
};

console.log(obj.x);
```

Conceptual path:

```text
Get(obj, "x")
→ obj.[[Get]]("x", obj)
→ own data property found
→ value = 10
```

### Example 2 — Inherited property

```js
const parent = {
  x: 10
};

const child = Object.create(parent);

console.log(child.x);
```

Conceptually:

```text
child.[[Get]]("x", child)
→ no own property
→ prototype = parent
→ parent.[[Get]]("x", child)
→ data property found
→ 10
```

### Example 3 — Getter

```js
const obj = {
  get x() {
    return 10;
  }
};

console.log(obj.x);
```

Conceptually:

```text
[[Get]]
→ accessor descriptor
→ getter callable
→ Call(getter, receiver)
→ 10
```

### Example 4 — Setter

```js
const obj = {
  set x(value) {
    console.log(value);
  }
};

obj.x = 10;
```

Conceptually:

```text
[[Set]]
→ accessor descriptor
→ setter callable
→ Call(setter, receiver, [10])
```

### Example 5 — Non-writable data property

```js
const obj = {};

Object.defineProperty(obj, "x", {
  value: 10,
  writable: false,
  configurable: false
});
```

Assignment cannot freely replace it.

### Example 6 — Non-extensible object

```js
const obj = {};

Object.preventExtensions(obj);

obj.x = 10;
```

Whether the source assignment throws or silently fails depends on strictness/context, but the object cannot simply become extensible.

---

## 9. Execution Walkthrough

Consider:

```js
const result = child.x;
```

Suppose:

```js
const parent = {
  get x() {
    return 20;
  }
};

const child = Object.create(parent);
```

### Step 1 — Evaluate `child`

The identifier resolves to the child object.

### Step 2 — Determine property key

```text
"x"
```

is the property key.

### Step 3 — Abstract `Get`

The semantic property-read operation is performed.

### Step 4 — `child.[[Get]]`

The child has no own property `"x"`.

### Step 5 — Obtain child prototype

The child prototype is:

```text
parent
```

### Step 6 — Continue lookup

The ordinary `[[Get]]` algorithm continues using the prototype.

### Step 7 — Find descriptor

The parent has an accessor descriptor:

```text
get: function
set: absent
```

### Step 8 — Call getter

The getter executes with the correct receiver.

Conceptually:

```text
Call(getter, child, [])
```

### Step 9 — Return result

The getter returns:

```text
20
```

### Critical lesson

The getter belongs to:

```text
parent
```

but its receiver is:

```text
child
```

That distinction explains many prototype/accessor behaviors.

---

## 10. Internal Mechanics

### 10.1 Internal-method table

An ordinary object can conceptually support:

| Internal Method | Purpose |
|---|---|
| `[[GetPrototypeOf]]` | Obtain prototype |
| `[[SetPrototypeOf]]` | Change prototype subject to constraints |
| `[[IsExtensible]]` | Check extensibility |
| `[[PreventExtensions]]` | Prevent future extensions |
| `[[GetOwnProperty]]` | Retrieve own-property descriptor |
| `[[DefineOwnProperty]]` | Create/update own property |
| `[[HasProperty]]` | Test own/inherited property existence |
| `[[Get]]` | Read property |
| `[[Set]]` | Write property |
| `[[Delete]]` | Delete own property |
| `[[OwnPropertyKeys]]` | Produce ordered own property keys |

Function objects additionally have callable/constructable behavior where applicable.

### 10.2 `[[GetPrototypeOf]]`

Returns the object's prototype.

Conceptually:

```text
object
→ [[Prototype]]
→ prototype object or null
```

### 10.3 `[[SetPrototypeOf]]`

Attempts to change the prototype.

The operation must respect constraints such as:

```text
non-extensibility
prototype cycles
existing prototype relationships
```

### 10.4 `[[IsExtensible]]`

Answers:

```text
Can new own properties still be added?
```

### 10.5 `[[PreventExtensions]]`

Makes the object non-extensible.

This does not automatically make existing properties non-writable or non-configurable.

Important distinction:

```text
non-extensible
≠
frozen
```

### 10.6 `[[GetOwnProperty]]`

Returns the descriptor for an own property if present.

Conceptually:

```text
own property exists?
→ descriptor
else
→ undefined
```

The result is specification-level descriptor information.

### 10.7 Property descriptors

A descriptor can conceptually contain:

```text
value
writable
get
set
enumerable
configurable
```

There are two broad categories:

```text
data descriptor
accessor descriptor
```

A data descriptor has:

```text
value / writable
```

An accessor descriptor has:

```text
get / set
```

Both include relevant:

```text
enumerable
configurable
```

attributes.

### 10.8 `[[DefineOwnProperty]]`

Creates or changes an own property according to descriptor compatibility rules.

This is more general than ordinary assignment.

### 10.9 `[[HasProperty]]`

Conceptually:

```text
own descriptor exists?
→ true

otherwise:
prototype exists?
→ recurse

otherwise:
→ false
```

### 10.10 `[[Get]]`

Conceptually:

```text
descriptor = [[GetOwnProperty]](P)

if descriptor is undefined:
    prototype = [[GetPrototypeOf]]()
    if prototype is null:
        return undefined
    return prototype.[[Get]](P, Receiver)

if data descriptor:
    return descriptor.[[Value]]

if accessor descriptor:
    getter = descriptor.[[Get]]
    if getter is undefined:
        return undefined
    return Call(getter, Receiver, [])
```

This is a simplified teaching model. Exact specification details and completion handling matter.

### 10.11 `[[Set]]`

`[[Set]]` is more complicated than:

```text
write into object
```

It must account for:

- own descriptor;
- inherited descriptor;
- writable data properties;
- setters;
- receiver;
- extensibility;
- descriptor compatibility.

A simplified mental flow:

```text
find own descriptor
→ if absent, inspect prototype
→ if accessor, call setter
→ if non-writable data, fail
→ otherwise create/update property on Receiver subject to constraints
```

### 10.12 Receiver

Consider:

```js
const parent = {
  set x(value) {
    this._x = value;
  }
};

const child = Object.create(parent);

child.x = 10;
```

The setter is found through:

```text
parent
```

but the receiver is:

```text
child
```

Therefore:

```js
this
```

inside the setter refers to the receiver in the relevant invocation semantics.

### 10.13 `[[Delete]]`

Deletion considers:

```text
does property exist?
is it configurable?
```

A non-configurable property cannot simply be deleted.

### 10.14 `[[OwnPropertyKeys]]`

Returns the object's own property keys according to specification ordering rules.

Conceptually:

```text
integer-index-like string keys
→ other string keys
→ symbol keys
```

with the exact ordering rules defined by the specification.

### 10.15 Ordinary `[[OwnPropertyKeys]]`

An ordinary object has specification-defined key ordering behavior rather than arbitrary hash-table iteration semantics.

### 10.16 Non-extensibility

Once:

```text
[[Extensible]] = false
```

new own properties cannot be added through ordinary successful property creation.

### 10.17 Property compatibility

Descriptor changes are constrained.

Examples:

```text
configurable: false
```

makes many later descriptor changes invalid.

### 10.18 Data-to-accessor changes

Changing:

```text
data descriptor
```

to:

```text
accessor descriptor
```

is constrained by configurability.

### 10.19 Accessor-to-data changes

The reverse transition is similarly constrained.

### 10.20 Writable transitions

A non-writable, non-configurable data property imposes especially strong restrictions.

### 10.21 Prototype cycles

Prototype relationships cannot be manipulated arbitrarily.

Creating an invalid cycle can fail.

### 10.22 Ordinary versus Proxy

Proxy objects use Proxy-specific internal methods.

But Proxy behavior must obey semantic invariants imposed by the specification.

### 10.23 Reflect API

The `Reflect` API exposes source-level operations that closely correspond to object semantic operations:

```js
Reflect.get(obj, key);
Reflect.set(obj, key, value);
Reflect.has(obj, key);
Reflect.deleteProperty(obj, key);
Reflect.defineProperty(obj, key, descriptor);
Reflect.getOwnPropertyDescriptor(obj, key);
Reflect.ownKeys(obj);
```

This is extremely useful for learning internal-method semantics.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Ordinary object invariant set

Ordinary object behavior is defined through a consistent family of internal methods.

These operations coordinate:

```text
properties
prototype
extensibility
descriptors
accessors
```

### 11.2 `[[GetPrototypeOf]]`

The ordinary object returns its `[[Prototype]]` value.

### 11.3 `[[SetPrototypeOf]]`

The ordinary algorithm applies constraints before changing `[[Prototype]]`.

A central concern is preserving a valid prototype structure.

### 11.4 `[[IsExtensible]]`

Reflects the object's extensibility state.

### 11.5 `[[PreventExtensions]]`

Transitions the object into a non-extensible state.

### 11.6 `[[GetOwnProperty]]`

Provides the semantic property-descriptor view of an own property.

### 11.7 `[[DefineOwnProperty]]`

Validates whether a requested descriptor transition is permitted.

This operation is foundational for:

```js
Object.defineProperty
Object.defineProperties
Reflect.defineProperty
```

and many built-in behaviors.

### 11.8 `[[HasProperty]]`

Prototype-aware existence:

```js
"x" in obj
```

conceptually reaches property-existence semantics rather than only own-property checking.

### 11.9 `[[Get]]`

The ordinary algorithm:

1. Looks for an own descriptor.
2. If absent, consults the prototype.
3. If a data descriptor exists, uses its value.
4. If an accessor descriptor exists, uses the getter with the receiver.
5. If no applicable property exists, returns `undefined`.

### 11.10 `[[Set]]`

The ordinary algorithm is receiver-sensitive.

It must distinguish:

```text
own writable data property
own non-writable data property
own accessor with setter
inherited writable data property
inherited accessor with setter
no property
```

### 11.11 Receiver semantics

The receiver allows an inherited setter or inherited writable path to operate on the appropriate target object.

### 11.12 `[[Delete]]`

Deletion is constrained by configurability.

### 11.13 `[[OwnPropertyKeys]]`

Ordinary objects provide deterministic key ordering according to standardized ordering rules.

### 11.14 `Object.freeze`

Conceptually combines:

```text
prevent extensions
+
make existing properties non-writable/non-configurable where applicable
```

The exact semantics operate through property-descriptor/internal-method machinery.

### 11.15 `Object.seal`

Conceptually:

```text
prevent extensions
+
make existing properties non-configurable
```

but does not generally make writable data properties non-writable.

### 11.16 `Object.preventExtensions`

Only extensibility changes.

It does not automatically rewrite existing descriptors.

### 11.17 Descriptor queries

```js
Object.getOwnPropertyDescriptor(obj, "x");
```

exposes a source-level representation of the relevant descriptor information.

### 11.18 Own versus inherited

These are different semantic questions:

```text
GetOwnProperty
→ own property only

HasProperty
→ own + prototype chain
```

### 11.19 Accessor receiver semantics

A getter/setter is associated with a property descriptor found during lookup, but the receiver can be another object.

This is one of the most important reasons `[[Get]]`/`[[Set]]` deserve separate study.

### 11.20 `super`

`super.x` depends on special reference/receiver semantics and ultimately interacts with property access machinery.

Therefore:

```text
super lookup target
≠
this receiver
```

This becomes easier once internal methods are understood.

---

## 12. Advanced Behavior

### 12.1 Property lookup is recursive through prototypes

Conceptually:

```text
obj
 ↓
prototype
 ↓
prototype
 ↓
null
```

### 12.2 Getter receiver differs from home object

```js
const proto = {
  get x() {
    return this._x;
  }
};

const obj = Object.create(proto);
obj._x = 100;

console.log(obj.x);
```

The getter is found on `proto` but executes with `obj` as receiver.

### 12.3 Setter receiver matters even more

```js
const proto = {
  set x(value) {
    this._x = value;
  }
};

const obj = Object.create(proto);

obj.x = 10;
```

The setter can create:

```js
obj._x
```

rather than modifying `proto`.

### 12.4 Inherited writable data property

Conceptually:

```text
prototype has writable x
child does not have x
child.x = 10
```

The write can create an own property on the receiver rather than changing the prototype property.

### 12.5 Inherited non-writable data property

A non-writable inherited data property can block assignment.

### 12.6 Inherited accessor without setter

Assignment can fail because there is no setter.

### 12.7 Accessor and data descriptors are mutually exclusive

A property cannot simultaneously be both a data and accessor descriptor in the same descriptor state.

### 12.8 Non-configurable property

Once a property becomes non-configurable, many structural changes become impossible.

### 12.9 Non-writable plus same value

Some descriptor/assignment operations distinguish:

```text
attempted write of same value
```

from:

```text
attempted write of different value
```

under the relevant compatibility rules.

### 12.10 `Object.defineProperty`

Unlike ordinary assignment, it directly requests descriptor semantics.

This is why:

```js
obj.x = 1;
```

and:

```js
Object.defineProperty(obj, "x", { value: 1 });
```

are not interchangeable.

### 12.11 Silent failure versus throw

Whether failed assignment becomes:

```text
silent failure
```

or:

```text
TypeError
```

depends on execution context such as strictness.

### 12.12 `delete`

A deletion attempt on a non-configurable property cannot simply remove the property.

The resulting source-level behavior depends on context.

### 12.13 `in`

```js
"x" in obj
```

can return true for inherited properties.

### 12.14 `hasOwn`

```js
Object.hasOwn(obj, "x");
```

answers a different question:

```text
Does obj itself own x?
```

### 12.15 Prototype mutation

`Object.setPrototypeOf` can affect later lookup behavior.

Prototype mutation may also have engine-performance consequences, but the standardized semantics are the primary concern here.

### 12.16 Null prototype objects

```js
const obj = Object.create(null);
```

has:

```text
[[Prototype]] = null
```

and therefore does not inherit ordinary Object.prototype properties.

### 12.17 Ordinary object without Object.prototype

Ordinary behavior does not require:

```text
Object.prototype
```

specifically.

An object can remain ordinary with:

```text
[[Prototype]] = null
```

### 12.18 Arrays are specialized

Arrays have specialized/exotic behavior beyond the ordinary object model, particularly around indexed properties and `length`.

This chapter establishes the ordinary baseline required to understand those specialized behaviors.

### 12.19 Functions are specialized objects

Function objects combine object property behavior with callable semantics.

### 12.20 Proxy objects are exotic

Proxy internal methods route through Proxy state and traps.

### 12.21 Proxy invariants

Proxy traps cannot produce arbitrary answers in cases where the specification requires consistency with target invariants.

### 12.22 Receiver and Proxy

`Reflect.get` and `Reflect.set` make receiver-sensitive behavior visible at the source level:

```js
Reflect.get(target, key, receiver);
Reflect.set(target, key, value, receiver);
```

### 12.23 Prototype poisoning

Changing prototypes can alter property resolution and inherited behavior.

### 12.24 Descriptor aliasing misconceptions

Descriptors returned from APIs are JavaScript objects representing descriptor information.

They are not direct references to internal specification records.

### 12.25 Enumeration and ordering

`Object.keys`, `Object.getOwnPropertyNames`, `Object.getOwnPropertySymbols`, and `Reflect.ownKeys` expose different projections of own-key semantics.

### 12.26 Property definition versus assignment

Definitions can bypass setter invocation in cases where assignment would invoke accessors, because the operations have different semantics.

### 12.27 Getter side effects

A seemingly harmless property read can invoke arbitrary JavaScript.

### 12.28 Setter side effects

A property write can execute arbitrary JavaScript.

### 12.29 Proxy side effects

A Proxy can intercept many object operations and invoke user code.

### 12.30 Prototype traversal cost

Long prototype chains can increase lookup work, although engines may optimize common paths.

### 12.31 Internal method recursion

Specification algorithms can recursively delegate:

```text
[[Get]]
→ prototype.[[Get]]
```

until they find a result or reach `null`.

### 12.32 Completion propagation

A getter may throw:

```js
const obj = {
  get x() {
    throw new Error("boom");
  }
};
```

Then:

```text
[[Get]]
→ Call getter
→ abrupt completion
→ propagate
```

### 12.33 Setter failure

The same applies to setters.

### 12.34 Define-property failure

Descriptor incompatibility can produce an abrupt completion or false result depending on the specific operation/API.

### 12.35 Internal method versus Reflect return conventions

Some source-level `Reflect` methods return booleans where corresponding throwing object operations may instead throw.

This distinction is important for precise API reasoning.

---

## 13. Edge Cases

- `Object.preventExtensions` does not freeze existing properties.
- `Object.seal` does not generally make writable properties read-only.
- `Object.freeze` affects existing descriptors as appropriate but does not recursively freeze nested objects.
- `Object.create(null)` creates an ordinary object with a null prototype.
- `obj.x` can execute a getter.
- `obj.x = value` can execute a setter.
- An inherited setter can write to the receiver rather than the prototype.
- An inherited writable data property can result in creation of an own property on the receiver.
- An inherited non-writable property can block assignment.
- An accessor with no setter can block writes.
- Non-configurable properties cannot be freely redefined.
- Non-extensible objects cannot gain arbitrary new own properties.
- `delete` does not mean “set to undefined”.
- `"x" in obj` can be true for inherited properties.
- `Object.hasOwn(obj, "x")` considers only own properties.
- A prototype can be `null`.
- Property ordering is not simply “whatever the hash table returns”.
- Symbols participate in own-key ordering as a separate category.
- Arrays require specialized semantics beyond ordinary objects.
- Functions require callable/constructable internal behavior beyond ordinary property operations.
- Proxies require exotic semantics and invariant checks.
- Descriptor objects exposed by APIs are ordinary JavaScript representations of descriptor information, not internal specification records.
- Failed assignments may throw or fail silently depending on language context.
- A getter/setter may throw.
- `Object.defineProperty` and assignment do not have identical semantics.
- A long prototype chain does not imply a particular engine data structure.
- Prototype mutation may change engine performance without changing the specification-level operation itself.

---

## 14. Common Misconceptions

### Misconception 1 — “JavaScript objects are just hash maps.”

No. The semantic model includes:

```text
properties
descriptors
prototypes
accessors
extensibility
internal methods
```

### Misconception 2 — “Property access only checks the object's own properties.”

No. Ordinary `[[Get]]` can traverse the prototype chain.

### Misconception 3 — “Assignment modifies whichever object supplied the property.”

Not necessarily.

Receiver semantics matter.

### Misconception 4 — “A getter belongs to the receiver.”

The getter is part of the descriptor found during lookup; it can execute with another object as receiver.

### Misconception 5 — “Non-extensible means immutable.”

No.

Existing properties can still be writable/configurable depending on their descriptors.

### Misconception 6 — “Frozen means deeply immutable.”

No.

`Object.freeze` is shallow with respect to referenced nested objects.

### Misconception 7 — “`in` checks own properties.”

No. It is prototype-aware.

### Misconception 8 — “`delete obj.x` sets `obj.x` to undefined.”

No.

It removes a property when deletion is permitted.

### Misconception 9 — “Object.defineProperty is just assignment with options.”

No. It invokes descriptor-definition semantics.

### Misconception 10 — “Symbols are strings internally.”

No. Symbols are distinct property-key values.

### Misconception 11 — “Internal methods are methods I can call.”

No.

### Misconception 12 — “Every object uses ordinary semantics.”

No. Arrays, functions, Proxies, and other specialized objects can have exotic/internal specialization.

### Misconception 13 — “A Proxy can return whatever it wants.”

No. Trap results are constrained by specification invariants.

### Misconception 14 — “Null-prototype objects are exotic.”

A null prototype alone does not make an object exotic.

### Misconception 15 — “Non-configurable means non-writable.”

Not necessarily.

### Misconception 16 — “Property order is implementation-defined.”

Standardized own-key ordering exists.

---

## 15. Common Mistakes

1. Ignoring descriptors.
2. Treating `[[Get]]` as a simple dictionary lookup.
3. Ignoring prototypes.
4. Ignoring receiver semantics.
5. Assuming setters always mutate the object where the setter was found.
6. Confusing own-property checks with prototype-aware property checks.
7. Treating non-extensible as immutable.
8. Treating sealed as frozen.
9. Treating frozen as deep immutability.
10. Assuming assignment and `defineProperty` are equivalent.
11. Ignoring strict-mode assignment failures.
12. Ignoring getter/setter exceptions.
13. Ignoring Proxy invariants.
14. Assuming null-prototype objects use non-ordinary semantics.
15. Assuming internal slots are ordinary properties.
16. Assuming specification key ordering is hash-table iteration order.
17. Inferring engine layout from the specification.
18. Explaining receiver-sensitive behavior only through “this is the object”.
19. Forgetting that internal methods can produce abrupt completions.
20. Treating Reflect APIs as unrelated utilities instead of useful semantic windows.

---

## 16. Comparison With Related Concepts

| Concept | Main question |
|---|---|
| `[[GetOwnProperty]]` | What is this object's own descriptor for property P? |
| `[[HasProperty]]` | Does P exist on this object or its prototype chain? |
| `[[Get]]` | What value results from reading P? |
| `[[Set]]` | Can/How should P receive value V? |
| `[[DefineOwnProperty]]` | Can this descriptor be defined on the object? |
| `[[Delete]]` | Can own property P be removed? |
| `[[OwnPropertyKeys]]` | What are the object's own keys in specification order? |
| `[[GetPrototypeOf]]` | What is the object's prototype? |
| `[[SetPrototypeOf]]` | Can the prototype be changed? |
| `[[IsExtensible]]` | Can new properties be added? |
| `[[PreventExtensions]]` | Can extension be disabled? |

### `[[GetOwnProperty]]` vs `[[HasProperty]]`

```text
GetOwnProperty
→ own only

HasProperty
→ own + prototype chain
```

### `[[Get]]` vs `[[GetOwnProperty]]`

```text
GetOwnProperty
→ returns descriptor

Get
→ returns property value
```

### `[[Set]]` vs `[[DefineOwnProperty]]`

Assignment:

```text
receiver-sensitive write semantics
```

Definition:

```text
explicit descriptor semantics
```

### Ordinary vs exotic objects

```text
ordinary:
standard internal methods

exotic:
specialized internal-method behavior
```

### Data vs accessor properties

```text
data:
value / writable

accessor:
get / set
```

### Extensible vs writable

```text
extensible:
can new properties be added?

writable:
can a particular data property value change?
```

These are independent dimensions.

### Configurable vs writable

```text
configurable:
can property structure/descriptor be changed?

writable:
can data value be changed?
```

A property can be:

```text
writable but non-configurable
```

---

## 17. Performance Considerations

### 17.1 Specification does not define one object layout

Do not infer:

```text
hash table
tree
shape
hidden class
dictionary
```

from ordinary-object semantics alone.

### 17.2 Property access

Engines often optimize common property-access patterns.

The semantic baseline remains:

```text
[[Get]]
```

and related operations.

### 17.3 Prototype chains

Long or frequently mutated prototype structures can complicate optimization.

### 17.4 Accessors

Getters/setters add callable execution and can limit simple property-access optimization.

### 17.5 Proxies

Proxy operations can substantially reduce opportunities for certain assumptions/optimizations.

### 17.6 Prototype mutation

Frequent changes to prototypes can affect engine optimization strategies.

### 17.7 Descriptor transitions

Changing object property characteristics can affect implementation representations.

### 17.8 Enumeration

Key enumeration can have different costs depending on object shape and property distribution.

### 17.9 `Object.defineProperty`

Repeated descriptor manipulation can be more expensive than simple property assignment in many implementations, but exact costs are engine-specific.

### 17.10 Performance rule

Separate:

```text
specified semantics
```

from:

```text
engine optimization strategy
```

before making a performance claim.

---

## 18. Memory Considerations

### 18.1 Specification state versus memory layout

Internal slots such as:

```text
[[Prototype]]
[[Extensible]]
```

do not define exact physical memory representation.

### 18.2 Property descriptors

Descriptor information may be represented efficiently or compactly by an engine.

### 18.3 Accessors

Getters/setters reference functions and therefore keep those functions/closures relevant to object lifetime.

### 18.4 Prototype retention

An object retains access to its prototype chain for lookup semantics.

### 18.5 Long prototype chains

A long chain can maintain a larger reachable object graph.

### 18.6 Proxies

Proxy objects have specialized state connecting them with:

```text
target
handler
```

and can therefore retain both.

### 18.7 Symbols

Symbol-keyed properties remain part of object state and can affect retention just like other properties.

### 18.8 Memory model

Use:

```text
specification
→ engine representation
→ heap/profile measurement
```

for accurate memory analysis.

---

## 19. Security Considerations

### 19.1 Prototype pollution

Understanding:

```text
[[Set]]
[[Get]]
[[SetPrototypeOf]]
```

helps reason about prototype-manipulation attacks.

### 19.2 Getters

Property reads can execute attacker-controlled code when objects are not trusted.

### 19.3 Setters

Property writes can trigger arbitrary side effects.

### 19.4 Proxies

Proxy traps can make apparently simple operations invoke arbitrary code.

### 19.5 Key conversion

Computed property keys can execute conversion hooks.

### 19.6 `in`

Prototype-aware checks may accept inherited properties unexpectedly.

### 19.7 Null-prototype dictionaries

Objects with:

```js
Object.create(null)
```

can reduce some prototype-related hazards in dictionary-style data structures, but do not automatically make data safe.

### 19.8 Descriptor hardening

Non-configurable/non-writable properties can be useful for protecting critical invariants.

### 19.9 Freezing

Freezing can protect an object's own structure but does not recursively freeze referenced objects.

### 19.10 Access-control assumptions

Do not assume:

```text
property exists
→ property is harmless
```

The property may be accessor-backed or Proxy-intercepted.

---

## 20. Production Usage

### 20.1 Defensive object boundaries

For dictionary-like data with untrusted keys, understand prototype behavior before choosing:

```text
Object
Object.create(null)
Map
```

### 20.2 API design

Document whether your API accepts:

```text
ordinary objects
Proxies
accessors
null-prototype objects
```

### 20.3 Configuration objects

Be cautious when reading configuration properties from unknown objects because getters can execute code.

### 20.4 Serialization

Property reads during serialization may trigger getters depending on the serializer/operation.

### 20.5 Validation

Schema validation that traverses properties may invoke user code.

### 20.6 Immutable configuration

Where required, use appropriate descriptor/freezing strategies and understand their shallow nature.

### 20.7 Metaprogramming

Use `Reflect` when explicit internal-method-like behavior is useful:

```js
Reflect.get
Reflect.set
Reflect.has
Reflect.deleteProperty
Reflect.defineProperty
Reflect.ownKeys
```

### 20.8 Proxy design

When creating Proxies, reason about:

```text
target invariants
non-configurable properties
non-extensibility
receiver
```

### 20.9 Code review

Ask:

```text
Is this own or inherited?
Can this access invoke code?
Can this setter redirect the write?
Could a Proxy alter behavior?
Is the object extensible?
What descriptor constraints exist?
```

### 20.10 Production debugging

When property behavior surprises you:

```text
1. inspect prototype
2. inspect own descriptor
3. inspect extensibility
4. inspect accessor/data type
5. inspect Proxy possibility
6. determine receiver
7. trace Get/Set/Delete semantics
```

---

## 21. Implementation From Scratch

The goal is a teaching implementation of ordinary object semantics.

### Stage 1 — Internal State

Create:

```js
class OrdinaryObject {
  constructor(proto = null) {
    this.prototype = proto;
    this.extensible = true;
    this.properties = new Map();
  }
}
```

Keep the teaching representation explicit.

### Stage 2 — Property Descriptors

Represent:

```js
{
  kind: "data",
  value,
  writable,
  enumerable,
  configurable
}
```

or:

```js
{
  kind: "accessor",
  get,
  set,
  enumerable,
  configurable
}
```

### Stage 3 — `GetOwnProperty`

Implement:

```text
getOwnDescriptor(object, key)
```

### Stage 4 — `GetPrototypeOf`

Implement:

```text
getPrototype(object)
```

### Stage 5 — `HasProperty`

Implement recursive prototype traversal.

### Stage 6 — `Get`

Implement:

```text
own descriptor
→ prototype traversal
→ data value
→ getter call with receiver
```

### Stage 7 — `Set`

Implement:

```text
own descriptor
→ inherited descriptor
→ setter
→ writable data
→ receiver property creation/update
```

### Stage 8 — `DefineOwnProperty`

Implement descriptor compatibility rules.

### Stage 9 — `Delete`

Implement configurability checks.

### Stage 10 — `OwnPropertyKeys`

Implement deterministic key ordering for:

```text
integer-index-like strings
other strings
symbols
```

### Stage 11 — Prototype Mutation

Implement:

```text
getPrototypeOf
setPrototypeOf
isExtensible
preventExtensions
```

with relevant invariants.

### Stage 12 — Reflect Adapter

Build source-level helpers:

```js
reflectGet(object, key, receiver);
reflectSet(object, key, value, receiver);
reflectHas(object, key);
reflectDelete(object, key);
reflectDefineProperty(object, key, descriptor);
```

### Stage 13 — Production-Grade Teaching Model

Add:

- completion representation;
- strict/sloppy assignment policy;
- non-configurable invariants;
- descriptor validation;
- accessor exceptions;
- Proxy-like interception simulation;
- tracing;
- test suite.

---

## 22. Debugging Exercises

### Exercise 1 — Prototype Read

```js
const proto = { x: 10 };
const obj = Object.create(proto);

console.log(obj.x);
```

Trace:

```text
Get
→ [[Get]]
→ GetOwnProperty
→ prototype
→ [[Get]]
```

### Exercise 2 — Getter Receiver

```js
const proto = {
  get x() {
    return this.value;
  }
};

const obj = Object.create(proto);
obj.value = 100;

console.log(obj.x);
```

Why is the result based on `obj`?

### Exercise 3 — Inherited Setter

```js
const proto = {
  set x(value) {
    this._x = value;
  }
};

const obj = Object.create(proto);

obj.x = 10;

console.log(obj._x);
```

Trace the receiver.

### Exercise 4 — Inherited Non-Writable Property

```js
const proto = {};

Object.defineProperty(proto, "x", {
  value: 10,
  writable: false
});

const obj = Object.create(proto);

obj.x = 20;
```

Predict behavior in strict and non-strict contexts.

### Exercise 5 — Non-Configurable Property

Create:

```text
configurable = false
```

then attempt incompatible redefinition.

### Exercise 6 — Non-Extensible Object

```js
const obj = {};
Object.preventExtensions(obj);
```

Attempt to add a property and inspect the result.

### Exercise 7 — Delete

Create:

```text
configurable = false
```

and attempt:

```js
delete obj.x;
```

### Exercise 8 — `in` Versus Own

Compare:

```js
"x" in obj
Object.hasOwn(obj, "x")
```

with inherited `x`.

### Exercise 9 — Null Prototype

```js
const dict = Object.create(null);
```

Test:

```js
dict.toString
```

and compare with a normal object.

### Exercise 10 — Reflect Receiver

Use:

```js
Reflect.get(proto, "x", child);
Reflect.set(proto, "x", 10, child);
```

and explain the receiver effect.

---

## 23. Code Review Exercise

Review:

```js
function getConfig(config, key) {
  return config[key] ?? defaultValue;
}
```

Analyze the assumptions:

```text
What if config is a Proxy?
What if key maps to a getter?
What if the property is inherited?
What if the getter throws?
What if key conversion executes code?
What if defaultValue itself is expensive?
```

Then compare with:

```js
Object.hasOwn(config, key)
```

and:

```js
Object.prototype.hasOwnProperty.call(config, key)
```

Identify which question each approach answers.

---

## 24. Interview Questions

### Foundational

1. What is an internal method?
2. What is an ordinary object?
3. What are the main ordinary object internal methods?
4. What is `[[Get]]`?
5. What is `[[Set]]`?
6. What is `[[Delete]]`?
7. What is `[[HasProperty]]`?
8. What is `[[GetOwnProperty]]`?
9. What is `[[DefineOwnProperty]]`?
10. What is `[[OwnPropertyKeys]]`?

### Intermediate

11. Explain ordinary `[[Get]]`.
12. Explain ordinary `[[Set]]`.
13. What is a property descriptor?
14. Compare data and accessor descriptors.
15. What is the receiver?
16. Why does receiver matter?
17. What is the difference between `HasProperty` and `GetOwnProperty`?
18. What does non-extensible mean?
19. What does configurable mean?
20. What does writable mean?

### Advanced

21. Explain inherited setters.
22. Explain inherited writable data properties.
23. Explain inherited non-writable properties.
24. Explain accessor receiver semantics.
25. Explain `Object.defineProperty` versus assignment.
26. Explain `Object.preventExtensions`, `seal`, and `freeze`.
27. Explain Proxy invariants.
28. Explain ordinary versus exotic objects.
29. Explain property-key ordering.
30. Explain `Reflect.get` / `Reflect.set` receiver semantics.

### Principal-Level

31. Trace `obj.x` from syntax to `[[Get]]`.
32. Trace `obj.x = value` through `[[Set]]`.
33. Explain why `super` needs receiver-aware property semantics.
34. Design a standards-faithful teaching implementation of `[[Get]]`.
35. Design a standards-faithful teaching implementation of `[[Set]]`.
36. Prove why inherited setters usually operate on the receiver.
37. Explain how Proxy invariants preserve object-model consistency.
38. Diagnose a bug involving prototype mutation and property lookup.
39. Diagnose a bug involving `Object.freeze` and nested state.
40. Defend the statement:

> Ordinary objects are best understood as semantic objects defined by internal methods, descriptors, prototype state, and extensibility—not as generic hash maps.

---

## 25. Predict-the-Output Exercises

For each:

```text
Predict → Run → Compare → Trace internal methods → Explain
```

### Exercise A

```js
const parent = { x: 10 };
const child = Object.create(parent);

console.log(child.x);
```

### Exercise B

```js
const proto = {
  get x() {
    return this._x;
  }
};

const obj = Object.create(proto);
obj._x = 20;

console.log(obj.x);
```

### Exercise C

```js
const proto = {
  set x(value) {
    this._x = value;
  }
};

const obj = Object.create(proto);

obj.x = 30;

console.log(obj._x);
console.log(proto._x);
```

### Exercise D

```js
const proto = {};

Object.defineProperty(proto, "x", {
  value: 10,
  writable: false
});

const obj = Object.create(proto);

obj.x = 20;

console.log(obj.x);
```

Explain strict versus non-strict behavior.

### Exercise E

```js
const obj = {};

Object.defineProperty(obj, "x", {
  value: 10,
  configurable: false
});

console.log(delete obj.x);
```

### Exercise F

```js
const proto = { x: 10 };
const obj = Object.create(proto);

console.log("x" in obj);
console.log(Object.hasOwn(obj, "x"));
```

### Exercise G

```js
const obj = {};
Object.preventExtensions(obj);

obj.x = 10;

console.log(Object.hasOwn(obj, "x"));
```

### Exercise H

```js
const obj = Object.create(null);

console.log(obj.toString);
console.log(Object.getPrototypeOf(obj));
```

### Exercise I

```js
const proto = {
  get x() {
    return this.value;
  }
};

const child = {
  value: 42
};

Object.setPrototypeOf(child, proto);

console.log(Reflect.get(proto, "x", child));
```

### Exercise J

```js
const obj = {};

Object.defineProperty(obj, "x", {
  value: 1,
  writable: false,
  configurable: false
});

Object.defineProperty(obj, "x", {
  value: 1
});
```

Determine why the second definition can differ from a definition using another value or descriptor.

---

## 26. Mastery Exercises

### Exercise 1 — Descriptor Model

Implement:

```text
data descriptor
accessor descriptor
ValidateAndApplyPropertyDescriptor
```

at a teaching level.

### Exercise 2 — Ordinary `[[Get]]`

Implement:

```js
ordinaryGet(object, key, receiver)
```

with:

- own descriptor;
- prototype traversal;
- accessor getter;
- receiver;
- abrupt completion.

### Exercise 3 — Ordinary `[[Set]]`

Implement:

```js
ordinarySet(object, key, value, receiver)
```

with:

- inherited properties;
- setters;
- writable data properties;
- receiver creation;
- non-extensibility;
- non-writable constraints.

### Exercise 4 — Define Property

Implement:

```js
ordinaryDefineOwnProperty(object, key, descriptor)
```

with descriptor compatibility validation.

### Exercise 5 — Delete

Implement:

```text
ordinaryDelete
```

with configurable-property semantics.

### Exercise 6 — HasProperty

Implement recursive:

```text
ordinaryHasProperty
```

and measure prototype depth.

### Exercise 7 — Own Keys

Implement deterministic ordering for:

```text
integer-index-like string keys
other strings
symbols
```

### Exercise 8 — Object State Machine

Build:

```text
prototype
extensible
properties
```

and allow controlled state transitions.

### Exercise 9 — Reflect Semantic Adapter

Build source-level APIs mapping to your internal model:

```text
get
set
has
delete
defineProperty
ownKeys
getPrototypeOf
setPrototypeOf
```

### Exercise 10 — Principal Challenge

Implement a miniature ordinary-object engine and prove it with a conformance suite covering:

```text
data properties
accessors
inheritance
receiver behavior
descriptor compatibility
non-extensibility
deletion
key ordering
null prototypes
abrupt completion
```

---

## 27. Key Takeaways

1. Internal methods are specification-level semantic operations.
2. Ordinary objects implement a standard family of internal methods.
3. `[[Get]]` defines property-read semantics.
4. `[[Set]]` defines property-write semantics.
5. `[[HasProperty]]` is prototype-aware.
6. `[[GetOwnProperty]]` describes only own-property state.
7. `[[DefineOwnProperty]]` controls descriptor-based property creation/redefinition.
8. `[[Delete]]` controls property removal.
9. `[[OwnPropertyKeys]]` defines own-key retrieval/order semantics.
10. `[[GetPrototypeOf]]` and `[[SetPrototypeOf]]` control prototype semantics subject to constraints.
11. `[[IsExtensible]]` and `[[PreventExtensions]]` govern extensibility.
12. Property descriptors are central to object semantics.
13. Data and accessor descriptors are different semantic categories.
14. Prototype lookup is recursive through `[[Get]]`/related operations.
15. The receiver is critical for inherited accessors and assignment semantics.
16. Non-extensible does not mean immutable.
17. Sealed does not mean frozen.
18. Frozen is not deep immutability.
19. Assignment and `defineProperty` are different semantic operations.
20. `in` is not the same as an own-property test.
21. Null-prototype objects can still be ordinary objects.
22. Arrays, functions, and Proxies can have specialized/exotic behavior.
23. Reflect APIs provide useful source-level windows into object semantics.
24. Proxy traps are constrained by specification invariants.
25. The specification defines semantics, not a mandatory hash-table/hidden-class/object-layout implementation.
26. The central principle is:

> An ordinary JavaScript object is not merely a bag of key/value pairs; it is a semantic object governed by descriptors, prototype state, extensibility, receivers, and a standardized set of internal methods.

---

## 28. Concept Connections

### Depends On

- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations

### Builds Toward

- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 45 — Memory / GC
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 64 — ES Modules
- Chapter 90 — Modern ECMAScript Features
- Chapter 91 — TC39 Proposal Tracking
- Chapter 94 — Compatibility Engineering
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios

### Related Concepts

- Property Descriptor
- Data Descriptor
- Accessor Descriptor
- Internal Method
- Internal Slot
- `[[Get]]`
- `[[Set]]`
- `[[Delete]]`
- `[[HasProperty]]`
- `[[GetOwnProperty]]`
- `[[DefineOwnProperty]]`
- `[[OwnPropertyKeys]]`
- `[[GetPrototypeOf]]`
- `[[SetPrototypeOf]]`
- `[[IsExtensible]]`
- `[[PreventExtensions]]`
- Prototype chain
- Receiver
- Getter
- Setter
- Proxy
- Reflect
- Own property
- Inherited property
- Extensibility
- Configurability
- Writability
- Enumeration

### Concepts Revisited

This chapter revisits:

- objects;
- property keys;
- property ordering;
- prototypes;
- Proxy;
- Reflect;
- descriptors;
- accessors;
- `this`;
- specification abstract operations.

### Why This Chapter Matters Later

Chapter 43 turns the abstract operation model into a concrete semantic object model.

The learner should now be able to reason about:

```text
obj.x
obj.x = value
delete obj.x
"x" in obj
Reflect.get(...)
Reflect.set(...)
Object.defineProperty(...)
```

without reducing every operation to:

> “JavaScript looks in the object.”

Instead, the reasoning becomes:

```text
source syntax
→ abstract operation
→ internal method
→ descriptor/prototype/receiver state
→ completion/result
```

That model becomes essential for:

- Proxies;
- classes;
- accessors;
- engine optimizations;
- prototype-related security;
- metaprogramming;
- specification-level debugging.

---

## 29. Completion Criteria

### Conceptual Understanding

- [ ] Define internal method.
- [ ] Define ordinary object.
- [ ] Distinguish ordinary and exotic objects.
- [ ] Explain `[[GetPrototypeOf]]`.
- [ ] Explain `[[SetPrototypeOf]]`.
- [ ] Explain `[[IsExtensible]]`.
- [ ] Explain `[[PreventExtensions]]`.
- [ ] Explain `[[GetOwnProperty]]`.
- [ ] Explain `[[DefineOwnProperty]]`.
- [ ] Explain `[[HasProperty]]`.
- [ ] Explain `[[Get]]`.
- [ ] Explain `[[Set]]`.
- [ ] Explain `[[Delete]]`.
- [ ] Explain `[[OwnPropertyKeys]]`.
- [ ] Explain property descriptors.
- [ ] Explain data vs accessor descriptors.
- [ ] Explain receiver semantics.
- [ ] Explain prototype traversal.
- [ ] Explain extensibility/configurability/writability.

### Predictive Mastery

- [ ] Predict own property access.
- [ ] Predict inherited property access.
- [ ] Predict getter invocation.
- [ ] Predict setter receiver.
- [ ] Predict inherited writable-property assignment.
- [ ] Predict inherited non-writable-property assignment.
- [ ] Predict deletion of configurable/non-configurable properties.
- [ ] Predict non-extensible assignment.
- [ ] Predict `in` vs `Object.hasOwn`.
- [ ] Predict null-prototype behavior.
- [ ] Predict Reflect receiver behavior.
- [ ] Predict descriptor compatibility.

### Implementation

- [ ] Implement descriptors.
- [ ] Implement `[[GetOwnProperty]]`.
- [ ] Implement `[[GetPrototypeOf]]`.
- [ ] Implement `[[HasProperty]]`.
- [ ] Implement `[[Get]]`.
- [ ] Implement `[[Set]]`.
- [ ] Implement `[[DefineOwnProperty]]`.
- [ ] Implement `[[Delete]]`.
- [ ] Implement `[[OwnPropertyKeys]]`.
- [ ] Implement extensibility/prototype operations.
- [ ] Build a Reflect-style adapter.
- [ ] Build a conformance suite.

### Debugging

- [ ] Diagnose prototype lookup.
- [ ] Diagnose getter execution.
- [ ] Diagnose setter receiver behavior.
- [ ] Diagnose inherited property writes.
- [ ] Diagnose descriptor incompatibility.
- [ ] Diagnose non-extensibility.
- [ ] Diagnose deletion failures.
- [ ] Diagnose `in` vs own-property bugs.
- [ ] Diagnose Proxy-related object behavior.
- [ ] Trace internal-method completion failures.

### Production Engineering

- [ ] Review untrusted object/property access.
- [ ] Review prototype-pollution risk.
- [ ] Review configuration-object getters.
- [ ] Review Proxy assumptions.
- [ ] Choose Object vs null-prototype Object vs Map appropriately.
- [ ] Use descriptors/freezing deliberately.
- [ ] Use Reflect when explicit receiver semantics are needed.
- [ ] Separate standards semantics from engine performance assumptions.

### Interview Readiness

- [ ] Explain internal methods.
- [ ] Explain ordinary `[[Get]]`.
- [ ] Explain ordinary `[[Set]]`.
- [ ] Explain descriptors.
- [ ] Explain receiver.
- [ ] Explain inheritance.
- [ ] Explain extensibility.
- [ ] Explain ordinary vs exotic objects.
- [ ] Explain Proxy invariants.
- [ ] Implement simplified `[[Get]]` and `[[Set]]`.

### Track A — Core Theory

- [ ] Internal-method model understood.
- [ ] Descriptor model understood.
- [ ] Prototype traversal understood.
- [ ] Receiver semantics understood.
- [ ] Extensibility understood.
- [ ] Ordinary/exotic distinction understood.

### Track B — Implementation

- [ ] Guided object model completed.
- [ ] Partially guided object model completed.
- [ ] No-reference object model completed.
- [ ] Edge-case hardened object model completed.
- [ ] Conformance suite reviewed.

### Track C — Interview / Reasoning

- [ ] Output prediction completed.
- [ ] Internal-method tracing completed.
- [ ] Descriptor debugging completed.
- [ ] Receiver reasoning completed.
- [ ] Proxy invariant reasoning completed.
- [ ] Principal-level object-model defense completed.

### Mastery Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Do not mark `[*] Mastered` until the learner can independently:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# Chapter 43 — Revision / Retrieval Record

## Retrieval Prompts

1. What is an internal method?
2. What is an ordinary object?
3. What is an exotic object?
4. What does `[[GetOwnProperty]]` do?
5. What does `[[DefineOwnProperty]]` do?
6. What does `[[HasProperty]]` do?
7. What does `[[Get]]` do?
8. What does `[[Set]]` do?
9. What does `[[Delete]]` do?
10. What does `[[OwnPropertyKeys]]` do?
11. What does `[[GetPrototypeOf]]` do?
12. What does `[[SetPrototypeOf]]` do?
13. What does `[[IsExtensible]]` do?
14. What does `[[PreventExtensions]]` do?
15. What is a data descriptor?
16. What is an accessor descriptor?
17. What is the receiver?
18. Why does receiver matter?
19. How does ordinary `[[Get]]` traverse the prototype chain?
20. How does ordinary `[[Set]]` handle inherited setters?
21. How does ordinary `[[Set]]` handle inherited writable data properties?
22. What happens with inherited non-writable data properties?
23. What is the difference between `HasProperty` and own-property lookup?
24. What is the difference between assignment and `defineProperty`?
25. What does non-extensible mean?
26. What does configurable mean?
27. What does writable mean?
28. How do `preventExtensions`, `seal`, and `freeze` differ?
29. Why is frozen not deep immutability?
30. Why can null-prototype objects still be ordinary?
31. Why are arrays specialized?
32. Why are functions specialized?
33. Why are Proxies exotic?
34. What are Proxy invariants?
35. How does `Reflect.get` expose receiver semantics?
36. How would you trace `obj.x` from source to internal method?
37. How would you trace `obj.x = value`?
38. How would you diagnose a surprising setter behavior?

## Weak Areas

```text
-
-
-
```

## Revision Queue

```text
- [ ] Revisit internal-method taxonomy
- [ ] Revisit property descriptors
- [ ] Revisit GetOwnProperty
- [ ] Revisit DefineOwnProperty
- [ ] Revisit HasProperty
- [ ] Revisit Get
- [ ] Revisit Set
- [ ] Revisit Delete
- [ ] Revisit OwnPropertyKeys
- [ ] Revisit prototype mutation
- [ ] Revisit extensibility
- [ ] Revisit receiver semantics
- [ ] Revisit ordinary vs exotic objects
- [ ] Revisit Proxy invariants
- [ ] Revisit Reflect
```

## Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

## Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 43 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — primary source for ordinary object internal methods, property descriptors, prototypes, and object invariants.
2. TC39 materials — language evolution/context.
3. Browser/WHATWG standards — host behavior that interacts with ECMAScript objects.
4. Node.js documentation — runtime behavior.
5. Engine documentation/source — implementation strategies.
6. Developer documentation such as MDN — practical API explanations.

Always distinguish:

```text
ordinary internal method
vs
abstract operation
vs
source-level API
vs
engine implementation
```

For difficult object behavior, trace:

```text
source syntax
→ abstract operation
→ internal method
→ property descriptor/prototype/receiver state
→ completion
→ observable result
```

Do not infer a specific hidden-class, dictionary, hash-table, or object-layout implementation from ordinary-object semantics alone.

---

# Chapter 43 — Completion Snapshot

```text
Chapter: 43
Title: Ordinary Object Internal Methods
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```