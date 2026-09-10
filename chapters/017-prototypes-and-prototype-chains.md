
# Chapter 17 — Prototypes and Prototype Chains

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 14–16 — `this`, Objects/Property Semantics, Property Keys/Enumeration
>
> **Next:** Chapter 18 — Classes and Object-Oriented JavaScript

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- define the JavaScript prototype model precisely;
- distinguish an object's own properties from inherited properties;
- explain `[[Prototype]]` as an internal object relationship;
- explain the prototype chain;
- trace property lookup across multiple prototype levels;
- distinguish prototype inheritance from copying;
- explain `Object.create()`;
- distinguish `Object.getPrototypeOf()` from the legacy `__proto__` accessor;
- explain how methods placed on a prototype are shared across instances;
- explain why `this` still refers to the call receiver when a method is found on a prototype;
- explain how assignment behaves when a property is inherited;
- reason about inherited data properties versus inherited accessors;
- explain `hasOwn` versus `in` in prototype-aware code;
- explain how constructors connect instances with `.prototype`;
- distinguish a function's `.prototype` property from an object's internal `[[Prototype]]`;
- explain `instanceof` conceptually through prototype-chain relationships;
- understand how classes use prototype-based semantics beneath class syntax;
- distinguish class inheritance from JavaScript's underlying prototype mechanism;
- explain shadowing of inherited properties;
- understand prototype mutation and its architectural risks;
- identify prototype pollution risks;
- explain null-prototype objects;
- reason about prototype identity and cross-realm behavior;
- distinguish ECMAScript prototype semantics from engine optimizations such as hidden classes/shapes;
- implement a simplified prototype-aware object model;
- debug property lookup and inheritance problems;
- assess prototype design for correctness, memory, performance, security, and maintainability;
- make principal-level decisions about inheritance versus composition.

---

# 2. Prerequisites

You should understand:

```text
Chapter 15
objects and property semantics

Chapter 16
property keys and enumeration

Chapter 14
this and receiver semantics
```

The conceptual chain is:

```text
object
   ↓
own property lookup
   ↓
property missing
   ↓
[[Prototype]]
   ↓
prototype lookup
   ↓
continue until null
```

This chapter explains the inheritance mechanism underneath:

```text
object-oriented JavaScript
classes
constructors
instance methods
instanceof
```

---

# 3. What Is a Prototype?

Every ordinary JavaScript object can have an internal prototype relationship.

At the specification level this is represented conceptually by:

```text
[[Prototype]]
```

Example:

```js
const parent = {
  greet() {
    return "hello";
  }
};

const child = Object.create(parent);
```

Now:

```text
child
  own properties: none

child.[[Prototype]]
  ↓
parent
```

When:

```js
child.greet()
```

is evaluated:

```text
child has own "greet"?
→ no

prototype has "greet"?
→ yes

use inherited function
```

---

# 4. Why Does the Prototype System Exist?

JavaScript needs a way to share behavior and establish object relationships without requiring every object to carry a separate copy of every method.

Prototype inheritance supports:

```text
shared methods
delegation
inheritance
constructor-instance relationships
language protocols
built-in behavior
```

For example:

```js
const a = [];
const b = [];
```

Both arrays can use behavior provided through:

```text
Array.prototype
```

without each array needing an independent implementation of every array method.

---

# 5. Mental Model

Imagine:

```text
child object
   ↓
prototype object
   ↓
prototype's prototype
   ↓
...
   ↓
null
```

Property lookup walks this chain until:

```text
property found
```

or:

```text
prototype === null
```

Example:

```js
const grandparent = {
  role: "grandparent"
};

const parent = Object.create(grandparent);
const child = Object.create(parent);

console.log(child.role);
```

Lookup:

```text
child.role
→ child has role? no
→ parent has role? no
→ grandparent has role? yes
→ return "grandparent"
```

---

# 6. Prototype Chain Is Not Property Copying

Consider:

```js
const parent = {
  value: 10
};

const child = Object.create(parent);
```

The child does not contain an independent copied property:

```text
child.value
```

Instead:

```text
child
  ↓ [[Prototype]]
parent
  └── value = 10
```

If:

```js
parent.value = 20;
```

then:

```js
child.value
```

can observe:

```text
20
```

because the child still delegates lookup to the parent.

---

# 7. Identity of Prototypes

The prototype relationship is identity-based.

```js
const parentA = {};
const parentB = {};

const child = Object.create(parentA);

Object.getPrototypeOf(child) === parentA;
```

is:

```text
true
```

Changing:

```text
parentA
```

to a structurally identical object does not preserve the same identity relationship.

---

# 8. `Object.create()`

`Object.create(proto)` creates an object whose internal prototype is:

```text
proto
```

Example:

```js
const base = {
  greet() {
    return "hello";
  }
};

const obj = Object.create(base);

console.log(obj.greet());
```

The method is inherited.

---

# 9. Null Prototype

You can create:

```js
const dict = Object.create(null);
```

Now:

```text
dict.[[Prototype]] = null
```

There is no inherited `Object.prototype`.

Therefore:

```js
dict.toString
```

is:

```text
undefined
```

unless explicitly defined.

This is useful for dictionary-like structures.

---

# 10. Object Literal Prototype

An object literal such as:

```js
const obj = {};
```

normally gets:

```text
Object.prototype
```

as its prototype.

Thus:

```js
obj.toString
```

can resolve to an inherited method.

The object literal itself only contains its own explicitly defined properties.

---

# 11. Function Objects and `.prototype`

A crucial distinction:

```js
function User() {}
```

creates a function object.

That function object typically has a property named:

```text
prototype
```

This property is an ordinary data property of the function.

Separately, objects created via:

```js
new User()
```

typically receive a `[[Prototype]]` that points to:

```js
User.prototype
```

So:

```text
User.prototype
```

and:

```text
Object.getPrototypeOf(new User())
```

are conceptually connected.

---

# 12. `.prototype` Is Not `[[Prototype]]`

This is one of the most important interview distinctions.

For:

```js
function User() {}
```

there are two different things:

```text
User.[[Prototype]]
```

the prototype of the function object itself,

and:

```text
User.prototype
```

an ordinary property whose value is intended to be the prototype of instances created by `new User()`.

Do not say:

> `User.prototype` is the prototype of User.

More precisely:

> `User.prototype` is an object referenced by the function's `prototype` property and commonly becomes the `[[Prototype]]` of instances constructed by that function.

---

# 13. Constructor Example

```js
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return this.name;
};

const a = new User("A");
const b = new User("B");
```

Conceptually:

```text
a ──[[Prototype]]──→ User.prototype
b ──[[Prototype]]──→ User.prototype
```

and:

```text
User.prototype
└── greet
```

Thus `a` and `b` share the same `greet` function.

---

# 14. Why Prototype Methods Save Per-Instance Duplication

Compare:

```js
class User {
  greet() {
    return this.name;
  }
}
```

with:

```js
function User(name) {
  this.name = name;

  this.greet = function () {
    return this.name;
  };
}
```

In the second pattern, every instance gets a new `greet` function.

With a prototype method:

```text
one shared method
+
many instances
```

This can reduce memory overhead for large numbers of instances.

Actual performance depends on engine behavior and object shape.

---

# 15. Prototype Method Does Not Change `this`

Suppose:

```js
const proto = {
  greet() {
    return this.name;
  }
};

const child = Object.create(proto);
child.name = "A";
```

Then:

```js
child.greet();
```

finds `greet` on the prototype.

But:

```text
this === child
```

because the receiver is `child`.

The method's storage location does not determine `this`.

The invocation form does.

This connects directly to Chapter 14.

---

# 16. Method Lookup and Receiver

For:

```js
child.greet()
```

think in two stages:

```text
1. Property lookup:
   child → prototype → find greet

2. Call:
   invoke found function
   with receiver = child
```

This separation is essential.

---

# 17. Detached Prototype Method

Consider:

```js
const proto = {
  greet() {
    return this.name;
  }
};

const child = Object.create(proto);
child.name = "A";

const fn = child.greet;

fn();
```

The function was found through the prototype chain.

But once extracted:

```js
fn()
```

is a plain call.

The original receiver is lost.

This proves:

```text
method lookup location
≠
receiver
```

---

# 18. Inherited Data Property Read

Example:

```js
const parent = {
  x: 10
};

const child = Object.create(parent);

console.log(child.x);
```

Lookup:

```text
child own x?
→ no

parent x?
→ yes

return 10
```

No own property is created merely by reading.

---

# 19. Inherited Data Property Assignment

Now:

```js
child.x = 20;
```

For an inherited writable data property, ordinary assignment usually creates an own property on the child rather than mutating the parent property.

Result:

```text
parent.x → 10
child.x  → 20
```

This is one of the most important inheritance rules.

---

# 20. Shadowing an Inherited Property

After:

```js
child.x = 20;
```

the child now has:

```text
own x
```

Lookup becomes:

```text
child.x
→ own x found
→ 20
```

The inherited property is shadowed.

---

# 21. Shadowing Is Not Mutation of Parent

Example:

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

child.x = 2;
```

This does not necessarily mean:

```text
parent.x = 2
```

Instead:

```text
child.x creates/shadows own x
```

This distinction is fundamental to prototype inheritance.

---

# 22. Inherited Accessor Property

Consider:

```js
const parent = {
  get x() {
    return this._x;
  },

  set x(value) {
    this._x = value;
  }
};

const child = Object.create(parent);

child.x = 10;
```

The inherited setter can execute.

The receiver is:

```text
child
```

so:

```js
child._x
```

can be created.

The setter belongs to the prototype.

The receiver is the child.

---

# 23. Inherited Non-Writable Property

Consider:

```js
const parent = {};

Object.defineProperty(parent, "x", {
  value: 10,
  writable: false,
  configurable: true
});

const child = Object.create(parent);

child.x = 20;
```

Because the inherited data property is non-writable, ordinary assignment cannot simply create a shadowing own property in the same way as the writable case.

In strict code, the assignment can throw.

This is an important descriptor/prototype interaction.

---

# 24. Property Lookup Algorithm

For ordinary objects, a useful simplified model is:

```text
Get(object, key)

1. Look for own property.
2. If found:
   - data property → return value
   - accessor → invoke getter with receiver
3. If not found:
   - get [[Prototype]]
   - if null → undefined
   - otherwise repeat
```

The real specification includes receiver and internal-method details.

Use the simplified model to reason, not as a complete implementation specification.

---

# 25. Prototype Chain Termination

Every prototype chain eventually reaches:

```text
null
```

for ordinary objects.

Example:

```js
const obj = {};

Object.getPrototypeOf(obj) === Object.prototype;
Object.getPrototypeOf(Object.prototype) === null;
```

Thus lookup eventually terminates.

There is no infinite prototype chain in a valid ordinary object graph.

---

# 26. Prototype Cycles

Objects cannot normally be arranged into an arbitrary prototype cycle through ordinary `[[SetPrototypeOf]]` operations.

The language maintains invariants that prevent invalid prototype cycles.

This matters because property lookup must have a terminating chain.

---

# 27. `Object.getPrototypeOf`

Use:

```js
Object.getPrototypeOf(obj)
```

to inspect the internal prototype relation.

Example:

```js
const parent = {};
const child = Object.create(parent);

console.log(Object.getPrototypeOf(child) === parent);
```

Result:

```text
true
```

This is the preferred explicit API for observing the prototype.

---

# 28. `Object.setPrototypeOf`

You can change an object's prototype using:

```js
Object.setPrototypeOf(obj, proto);
```

Example:

```js
const parent = {
  greet() {
    return "hello";
  }
};

const child = {};

Object.setPrototypeOf(child, parent);

child.greet();
```

This works, but prototype mutation has significant architectural and performance implications.

---

# 29. Why Prototype Mutation Is Often Discouraged

Dynamic prototype changes can complicate:

```text
object shape assumptions
optimization
debugging
reasoning
inheritance design
security
```

Instead of:

```js
Object.setPrototypeOf(existingObject, anotherProto);
```

prefer establishing the correct prototype when creating the object:

```js
Object.create(proto);
```

or:

```js
new Constructor();
```

when appropriate.

---

# 30. `__proto__`

`__proto__` is a legacy accessor associated with `Object.prototype`.

Example:

```js
const child = {};
child.__proto__ = parent;
```

This can change the object's prototype.

Modern code should generally prefer:

```js
Object.getPrototypeOf(obj);
Object.setPrototypeOf(obj, proto);
Object.create(proto);
```

The legacy accessor remains relevant for compatibility and security discussions.

---

# 31. Prototype Pollution Preview

Prototype pollution can occur when attacker-controlled property manipulation changes shared prototype state or causes inherited attacker-controlled properties to appear in security-sensitive code.

Risky assumptions include:

```text
for...in sees only my input fields
obj[key] = value is always a local write
```

These are not universally safe assumptions.

Security-sensitive code should explicitly control:

```text
keys
prototype
ownership
merge semantics
```

Chapter 57 will cover this in depth.

---

# 32. `Object.hasOwn` vs `in`

Given:

```js
const parent = {
  role: "admin"
};

const child = Object.create(parent);
```

Then:

```js
"role" in child
```

is:

```text
true
```

But:

```js
Object.hasOwn(child, "role")
```

is:

```text
false
```

The difference:

```text
in
→ own or inherited

hasOwn
→ own only
```

This distinction is essential in security-sensitive record processing.

---

# 33. `for...in` and Prototypes

`for...in` can enumerate inherited enumerable string properties.

Example:

```js
const parent = {
  inherited: 1
};

const child = Object.create(parent);
child.own = 2;

for (const key in child) {
  console.log(key);
}
```

Possible output:

```text
own
inherited
```

The exact ordering is governed by the language's enumeration rules.

---

# 34. Prototype Method Sharing

Prototype-based design is useful when many objects share behavior.

Example:

```js
const animal = {
  speak() {
    return "sound";
  }
};

const dog = Object.create(animal);
const cat = Object.create(animal);
```

Both can use:

```js
dog.speak();
cat.speak();
```

without duplicating the method definition.

---

# 35. Prototype Delegation

JavaScript prototype inheritance can be described more accurately as:

```text
delegation
```

rather than classical copied inheritance.

An object delegates missing property lookup to its prototype.

This makes the model:

```text
object → prototype
```

rather than:

```text
object copied from class blueprint
```

The class syntax later provides a class-oriented surface over these semantics.

---

# 36. Changing the Prototype Changes Behavior

Consider:

```js
const parentA = {
  value: 1
};

const parentB = {
  value: 2
};

const obj = Object.create(parentA);

console.log(obj.value);

Object.setPrototypeOf(obj, parentB);

console.log(obj.value);
```

Conceptually:

```text
1
2
```

The object's own properties did not change.

Its delegation relationship changed.

This illustrates the behavioral role of prototype identity.

---

# 37. Prototype Mutation and Hidden Coupling

If one part of a system modifies:

```js
Object.prototype.foo = ...
```

unrelated objects can observe:

```text
foo
```

through inherited lookup.

This creates global shared behavior.

Avoid modifying built-in prototypes in application code unless there is an exceptionally strong compatibility reason and the implications are fully understood.

---

# 38. Built-in Prototypes

Examples include:

```text
Object.prototype
Array.prototype
Function.prototype
String.prototype
Number.prototype
RegExp.prototype
Map.prototype
Set.prototype
```

These provide shared methods and language behavior.

For:

```js
const arr = [];
```

methods such as:

```js
arr.map
arr.push
```

are normally resolved through the relevant prototype chain.

---

# 39. Built-in Prototype Shadowing

An object can shadow inherited built-in properties.

Example:

```js
const obj = {
  toString: () => "custom"
};
```

Now:

```js
obj.toString()
```

uses the own property rather than:

```text
Object.prototype.toString
```

This is another example of:

```text
own property wins over inherited property
```

---

# 40. `hasOwnProperty` and Shadowing

An object can define:

```js
const obj = {
  hasOwnProperty: "not a function"
};
```

Now:

```js
obj.hasOwnProperty("x")
```

fails.

This is another reason:

```js
Object.hasOwn(obj, "x")
```

is safer and clearer.

---

# 41. Constructors and Prototype Identity

For:

```js
function User(name) {
  this.name = name;
}
```

the function object has:

```js
User.prototype
```

When:

```js
const user = new User("A");
```

the constructed instance typically has:

```text
[[Prototype]] → User.prototype
```

Therefore:

```js
Object.getPrototypeOf(user) === User.prototype
```

is ordinarily true.

---

# 42. Adding Methods After Instance Creation

Consider:

```js
function User() {}

const a = new User();

User.prototype.greet = function () {
  return "hello";
};

console.log(a.greet());
```

This works because `a` delegates to the same prototype object.

Adding the method later changes what existing instances can observe.

This is dynamic prototype behavior.

---

# 43. Replacing the Constructor `.prototype`

Consider:

```js
function User() {}

const a = new User();

User.prototype = {
  greet() {
    return "hello";
  }
};

const b = new User();
```

Now:

```text
a → old prototype object
b → new prototype object
```

Changing the constructor's `prototype` property later does not retroactively change the existing instance's internal `[[Prototype]]`.

This is a high-value distinction.

---

# 44. `constructor` Property

The default prototype object created for a function constructor commonly has:

```js
constructor
```

pointing back to the function.

Example:

```js
function User() {}

User.prototype.constructor === User;
```

But this is just a property relationship.

It is not magical proof that an object was produced by that constructor.

---

# 45. `constructor` Is Not a Security Guarantee

An object can have:

```js
obj.constructor
```

from its prototype chain.

A malicious or unusual object can also define or override it.

Do not use:

```js
obj.constructor === ExpectedConstructor
```

as a universal security or type-validation mechanism.

It is easy to alter prototype relationships.

---

# 46. `instanceof`

Example:

```js
function User() {}

const user = new User();

console.log(user instanceof User);
```

typically returns:

```text
true
```

Conceptually, `instanceof` checks whether the constructor's relevant prototype object appears in the left-hand value's prototype chain, subject to the language's `instanceof` protocol and overrides.

---

# 47. `instanceof` Is Prototype-Based

The simplified model:

```text
user prototype
   ↓
User.prototype?
```

If the same prototype identity is encountered:

```text
true
```

Otherwise:

```text
false
```

This means `instanceof` is not a simple textual class-name comparison.

---

# 48. `instanceof` Can Be Customized

JavaScript allows objects to define:

```js
Symbol.hasInstance
```

which participates in `instanceof`.

Example:

```js
class EvenNumber {
  static [Symbol.hasInstance](value) {
    return typeof value === "number" && value % 2 === 0;
  }
}
```

Then:

```js
2 instanceof EvenNumber
```

can evaluate according to the custom protocol.

Therefore:

> `instanceof` is a language protocol, not merely a hard-coded prototype-chain test.

---

# 49. Cross-Realm Prototype Identity

Objects created in different realms can have different intrinsic prototypes.

For example, an array from another browser realm can have a different:

```text
Array.prototype
```

identity from the local realm's:

```text
Array.prototype
```

This can make:

```js
value instanceof Array
```

fail when the value originated in another realm.

This is an important browser and iframe boundary concept.

Later Chapter 44 will examine realms formally.

---

# 50. Safer Array Detection

For cross-realm array detection:

```js
Array.isArray(value)
```

is usually more robust than:

```js
value instanceof Array
```

because it tests array semantics rather than relying on same-realm constructor prototype identity.

The exact algorithm is specified by ECMAScript.

---

# 51. `Object.prototype` and the End of the Chain

For a normal object:

```js
Object.getPrototypeOf(obj) === Object.prototype
```

Then:

```js
Object.getPrototypeOf(Object.prototype) === null
```

So:

```text
obj
 ↓
Object.prototype
 ↓
null
```

is the typical simplest prototype chain.

---

# 52. Custom Multi-Level Chains

Example:

```js
const a = {
  level: "a"
};

const b = Object.create(a);
const c = Object.create(b);
const d = Object.create(c);
```

Lookup:

```text
d
 ↓
c
 ↓
b
 ↓
a
 ↓
null
```

If the property exists at:

```text
b
```

lookup stops there.

Properties farther up the chain are not consulted for that key.

---

# 53. Prototype Shadowing

Example:

```js
const a = { value: "a" };
const b = Object.create(a);
const c = Object.create(b);

b.value = "b";

console.log(c.value);
```

Result:

```text
"b"
```

Lookup:

```text
c has value? no
b has value? yes
stop
```

---

# 54. Deleting a Shadowing Property

```js
delete b.value;
```

Now:

```js
c.value
```

again resolves to:

```text
a.value
```

This demonstrates dynamic delegation:

```text
own property inserted
→ shadows inherited
→ deleted
→ inherited property visible again
```

---

# 55. Prototype Chain and Mutation

Inherited objects are shared.

Consider:

```js
const parent = {
  config: {
    enabled: true
  }
};

const child = Object.create(parent);
```

Then:

```js
child.config.enabled = false;
```

mutates the shared nested object.

The child did not automatically receive an independent copy.

Prototype inheritance therefore interacts with aliasing and mutable state.

---

# 56. Prototype State and Instance Isolation

If prototype objects contain mutable state:

```js
const proto = {
  state: []
};
```

then:

```js
const a = Object.create(proto);
const b = Object.create(proto);
```

both can share:

```text
proto.state
```

This may be surprising.

For instance-specific mutable state, prefer:

```js
a.state = [];
b.state = [];
```

or constructor/class initialization.

---

# 57. Prototype Methods vs Prototype Data

Sharing behavior is usually safer than sharing mutable state.

Good candidate:

```js
User.prototype.greet = function () {};
```

Potentially dangerous shared state:

```js
User.prototype.permissions = [];
```

Every instance can observe the same array unless shadowed.

This distinction matters in object-oriented design.

---

# 58. Inheritance vs Composition

Prototype inheritance is not always the best architecture.

Inheritance is useful when:

```text
shared behavioral relationship
specialization
polymorphism
stable hierarchy
```

Composition can be better when:

```text
behavior is assembled from independent capabilities
hierarchy is unstable
reuse is orthogonal
dependencies should be explicit
```

Principal-level JavaScript design should not assume:

```text
inheritance = reuse
```

is always the best answer.

---

# 59. Prototype Inheritance vs Class Inheritance

Class syntax:

```js
class Admin extends User {}
```

does not replace the prototype model.

It builds on it.

Conceptually:

```text
Admin.prototype
   ↓ [[Prototype]]
User.prototype
```

Instances then delegate through:

```text
instance
   ↓
Admin.prototype
   ↓
User.prototype
   ↓
...
```

Thus classes are a higher-level syntax over prototype-oriented semantics.

---

# 60. `extends` and Prototype Chains

For:

```js
class User {}
class Admin extends User {}
```

there are important prototype relationships:

```text
Admin.prototype.[[Prototype]] === User.prototype
```

and:

```text
Admin.[[Prototype]] === User
```

The first relates instance method inheritance.

The second relates static inheritance.

Both matter.

---

# 61. Instance Side vs Constructor Side

Think of a class hierarchy as two related chains:

```text
INSTANCE SIDE

admin instance
   ↓
Admin.prototype
   ↓
User.prototype
   ↓
Object.prototype
   ↓
null
```

and:

```text
CONSTRUCTOR / STATIC SIDE

Admin
   ↓ [[Prototype]]
User
   ↓ [[Prototype]]
Function.prototype
   ↓
Object.prototype
   ↓
null
```

This is a high-value class/prototype mental model.

---

# 62. Static Methods

Example:

```js
class User {
  static create() {
    return new User();
  }
}
```

Then:

```js
User.create();
```

looks up:

```text
create
```

on the constructor object itself and its prototype chain.

It does not come from:

```text
User.prototype
```

This distinction becomes important in Chapter 18.

---

# 63. Prototype Lookup and Accessors

Consider:

```js
const base = {
  get value() {
    return this._value;
  }
};

const child = Object.create(base);
child._value = 42;
```

Reading:

```js
child.value
```

finds the getter on:

```text
base
```

but executes it with receiver:

```text
child
```

Thus the getter returns:

```text
child._value
```

This is why receiver semantics are essential to prototype reasoning.

---

# 64. `Reflect.get` and Prototype Receiver

A more advanced form:

```js
Reflect.get(target, key, receiver);
```

allows explicit receiver control.

Suppose an accessor exists on:

```text
target's prototype
```

but you want its `this` to be:

```text
receiver
```

This is fundamental to proxy forwarding and advanced object mechanics.

---

# 65. `super` Preview

Inside a method:

```js
super.method()
```

does not simply mean:

```text
find method on this.__proto__
```

It uses the method's home-object/super semantics.

Conceptually:

```text
current method
   ↓
home object
   ↓
super base
   ↓
lookup inherited method
   ↓
call with current this
```

This becomes important in classes.

---

# 66. Prototype Mutation and Performance

Modern engines often use internal object-shape structures to optimize property access.

Frequent prototype mutation can interfere with optimization strategies.

This is engine-specific behavior.

Do not claim:

```text
Object.setPrototypeOf is always slow
```

as a universal law.

Better:

> Dynamic prototype mutation can create optimization challenges, and should be benchmarked in actual hot paths.

---

# 67. Hidden Classes and Prototypes

An engine may represent objects using:

```text
shape / hidden class
```

information that describes property layout.

Prototype relationships can participate in optimized property lookup.

Again:

```text
ECMAScript guarantee:
prototype semantics

Engine implementation:
shape structures and caches
```

Keep these layers separate.

---

# 68. Prototype Chain Lookup Performance

Conceptually, a property read:

```js
obj.value
```

might require multiple prototype levels in a naive model.

Modern engines often avoid repeated linear work through:

```text
inline caches
shape checks
optimized property access
```

Therefore:

```text
prototype chain depth
```

and:

```text
actual property-read performance
```

are related but not a simple one-to-one cost formula.

Measure real workloads.

---

# 69. Security: Prototype Pollution

A common dangerous pattern is recursively assigning attacker-controlled keys:

```js
target[key] = value;
```

without a strong policy.

Problems can arise when:

```text
key affects prototype-related state
```

or:

```text
merged object inherits attacker-controlled properties
```

Security review should examine:

```text
source of keys
prototype of targets
own-property checks
merge algorithm
descriptor handling
```

---

# 70. Security: Safe Dictionary Designs

When the requirement is:

```text
arbitrary user-controlled string keys
```

consider:

```js
const dict = Object.create(null);
```

or:

```js
const dict = new Map();
```

depending on the API.

This can reduce assumptions about inherited properties.

Still validate values and keys according to the threat model.

---

# 71. Security: Do Not Treat `__proto__` as Ordinary Data Everywhere

Because legacy environments expose special `__proto__` behavior, generic merge/assignment code should treat it carefully.

Safer designs can:

```text
validate keys
use Map
use null-prototype targets
define exact merge semantics
```

Do not rely on casual property assignment for untrusted structural data.

---

# 72. Security: Built-in Prototype Modification

Code such as:

```js
Object.prototype.isAdmin = true;
```

can affect unrelated objects.

This can lead to:

```text
authorization bugs
unexpected enumeration
framework behavior changes
global state corruption
```

Application code should generally avoid modifying shared built-in prototypes.

---

# 73. Memory Considerations

Prototype methods are shared.

This can be memory-efficient for large object populations.

Prototype-held mutable data is shared, which can be a correctness and retention concern.

A long-lived prototype can also indirectly retain objects if properties on it reference large object graphs.

Therefore inspect:

```text
prototype-owned state
closures stored on prototypes
static caches
global references
```

when analyzing memory.

---

# 74. Object Lifetime and Prototype Lifetime

If many objects point to:

```text
prototype object
```

the prototype remains reachable as long as those objects or other roots maintain the relationship.

Typical built-in prototypes are process/realm-long-lived.

Custom prototype graphs should be designed with lifecycle in mind.

---

# 75. Production Usage

Use prototypes deliberately for:

```text
shared behavior
domain models
class methods
built-in object behavior
delegation
polymorphic method lookup
```

Avoid prototypes as an accidental global shared-state container.

For simple record data, plain objects may be enough.

For large inheritance hierarchies, consider whether composition is clearer.

---

# 76. Implementation From Scratch

Build a simplified object model:

```text
Object
├── own properties
├── [[Prototype]]
└── extensibility
```

Implement:

```text
get
set
has
getPrototypeOf
setPrototypeOf
own
inherited
```

---

# 77. Prototype-Aware Object

```js
class ProtoObject {
  constructor(proto = null) {
    this.proto = proto;
    this.properties = new Map();
  }

  hasOwn(key) {
    return this.properties.has(key);
  }

  hasProperty(key) {
    if (this.hasOwn(key)) {
      return true;
    }

    return this.proto
      ? this.proto.hasProperty(key)
      : false;
  }
}
```

This models the simplest prototype chain.

---

# 78. Prototype-Aware Get

```js
get(key) {
  if (this.properties.has(key)) {
    return this.properties.get(key);
  }

  return this.proto
    ? this.proto.get(key)
    : undefined;
}
```

This demonstrates:

```text
own
→ prototype
→ prototype
→ ...
→ undefined
```

---

# 79. Prototype-Aware Set

Start with:

```js
set(key, value) {
  if (this.properties.has(key)) {
    this.properties.set(key, value);
    return;
  }

  this.properties.set(key, value);
}
```

Then improve it to account for:

```text
inherited descriptors
writable
setters
receiver
```

The simplified version intentionally demonstrates shadowing first.

---

# 80. Guided Implementation

Implement:

```text
createObject(proto)
getPrototypeOf
setPrototypeOf
get
set
hasOwn
has
delete
```

Test:

```text
own property
inherited property
shadowing
prototype replacement
null prototype
```

---

# 81. Partially Guided Implementation

Add descriptor support.

For inherited property lookup:

```text
find first descriptor in chain
```

Then:

```text
data writable?
accessor setter?
missing?
```

Apply the appropriate receiver behavior.

---

# 82. No-Reference Implementation

Build a prototype-aware object runtime from memory.

Support:

```text
prototype chain
property lookup
shadowing
inherited setters
non-writable inherited properties
own/in checks
```

Then compare behavior with native JavaScript.

---

# 83. Edge-Case Hardening

Add:

- null-prototype objects;
- multi-level chains;
- accessors;
- non-writable inherited properties;
- non-extensible objects;
- proxy-like interception;
- symbols;
- prototype reassignment;
- constructor metadata;
- `instanceof`.

---

# 84. Production-Grade Exercise

Build a domain model with:

```text
BaseEntity
User
Admin
Guest
```

Use shared prototype methods.

Then intentionally implement the same model with composition.

Measure and compare:

```text
extensibility
testability
memory
behavioral coupling
type checks
serialization
inheritance depth
```

Document which model you would choose and why.

---

# 85. Debugging Workflow

When inherited behavior surprises you:

```text
1. Identify the object.
2. Inspect Object.getPrototypeOf(obj).
3. Check own property first.
4. Walk prototype levels.
5. Identify the first descriptor found.
6. Determine data vs accessor.
7. Determine receiver.
8. Check shadowing.
9. Check whether prototype was mutated.
10. Check whether a proxy is involved.
```

This is the definitive prototype-debugging workflow.

---

# 86. Debugging Exercise 1

```js
const parent = {
  role: "admin"
};

const child = Object.create(parent);

console.log(child.role);
console.log(Object.hasOwn(child, "role"));
console.log("role" in child);
```

Explain every result.

---

# 87. Debugging Exercise 2

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

child.x = 2;

console.log(parent.x);
console.log(child.x);
console.log(Object.hasOwn(child, "x"));
```

Trace shadowing.

---

# 88. Debugging Exercise 3

```js
const parent = {
  set x(value) {
    this._x = value;
  }
};

const child = Object.create(parent);

child.x = 10;

console.log(child._x);
console.log(Object.hasOwn(child, "x"));
```

Explain inherited setter + receiver behavior.

---

# 89. Debugging Exercise 4

```js
function User() {}

const user = new User();

console.log(Object.getPrototypeOf(user) === User.prototype);
console.log(user instanceof User);
```

Explain the two results from different perspectives:

```text
prototype identity
instanceof protocol
```

---

# 90. Debugging Exercise 5

```js
function User() {}

const a = new User();

User.prototype = {
  greet() {
    return "hello";
  }
};

const b = new User();

console.log(typeof a.greet);
console.log(typeof b.greet);
```

Explain why existing and future instances can differ.

---

# 91. Debugging Exercise 6

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

Object.setPrototypeOf(child, null);

console.log(child.x);
```

Trace the behavior before and after prototype replacement.

---

# 92. Code Review Exercise

Review:

```js
function merge(target, source) {
  for (const key in source) {
    target[key] = source[key];
  }

  return target;
}
```

Prototype-related concerns include:

```text
inherited source keys
inherited target behavior
setter invocation
prototype pollution
unexpected special keys
accessor execution
```

The contract must specify which keys are allowed.

---

# 93. Code Review Exercise — Mutable Prototype State

Review:

```js
function User() {}

User.prototype.permissions = [];

const a = new User();
const b = new User();

a.permissions.push("read");
```

Question:

```text
Can b.permissions observe "read"?
```

Yes.

Reason:

```text
a.permissions
→ no own property
→ User.prototype.permissions
→ shared array
```

This is an important real-world bug pattern.

---

# 94. Code Review Exercise — Prototype Mutation

Review:

```js
const user = {};

Object.setPrototypeOf(user, adminPrototype);
```

Questions:

```text
Why was the prototype changed after creation?
Could Object.create(adminPrototype) be clearer?
Does runtime performance matter?
Is this part of an intentional object-state transition?
```

Prototype mutation should be deliberate.

---

# 95. Code Review Exercise — Constructor Check

Review:

```js
if (value.constructor === User) {
  grantAccess();
}
```

Why is this weak?

Because:

```text
constructor is a normal property lookup
prototype can be changed
constructor can be shadowed
cross-realm identity may differ
```

Security-sensitive authorization should not rely on this pattern.

---

# 96. Interview Questions

## Beginner

1. What is a prototype?
2. What is the prototype chain?
3. What is `Object.create()`?
4. What is the difference between own and inherited properties?
5. Why does `obj.method()` work when `method` is on the prototype?
6. What is `Object.getPrototypeOf()`?

## Intermediate

7. What is the difference between `.prototype` and `[[Prototype]]`?
8. How does assignment interact with inherited properties?
9. What is shadowing?
10. What does `instanceof` check?
11. Why can `instanceof Array` fail across realms?
12. Why is `Object.hasOwn()` different from `in`?
13. Why are prototype methods often shared?

## Advanced

14. Explain inherited accessors and receiver semantics.
15. What happens when you replace a constructor's `.prototype`?
16. Why can mutable prototype properties create shared state bugs?
17. Why is modifying `Object.prototype` dangerous?
18. What is prototype pollution?
19. Why are null-prototype objects useful?
20. What is the relationship between classes and prototypes?
21. How does `super` relate to prototype lookup?
22. Why can prototype mutation hurt optimization?

## Principal

23. When should inheritance be replaced with composition?
24. How would you design a prototype-safe merge utility?
25. How would you audit a system for prototype pollution?
26. Which prototype claims are ECMAScript guarantees and which are engine-specific?
27. How would you model prototype relationships in a debugger?
28. How would you decide between prototype methods and per-instance methods for millions of objects?
29. How would you handle cross-realm type checks?
30. What prototype-related invariants belong in a public library API?

---

# 97. Predict-the-Output Exercises

Predict first.

## Exercise A

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

console.log(child.x);
```

## Exercise B

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

child.x = 2;

console.log(parent.x);
console.log(child.x);
```

## Exercise C

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

console.log(Object.hasOwn(child, "x"));
console.log("x" in child);
```

## Exercise D

```js
const parent = {
  get x() {
    return this._x;
  }
};

const child = Object.create(parent);
child._x = 5;

console.log(child.x);
```

## Exercise E

```js
function User() {}

User.prototype.greet = function () {
  return "hello";
};

const user = new User();

console.log(user.greet());
console.log(Object.getPrototypeOf(user) === User.prototype);
```

## Exercise F

```js
function User() {}

const a = new User();

User.prototype.greet = function () {
  return "hello";
};

console.log(a.greet());
```

## Exercise G

```js
function User() {}

const a = new User();

User.prototype = {
  greet() {
    return "hello";
  }
};

const b = new User();

console.log(typeof a.greet);
console.log(typeof b.greet);
```

## Exercise H

```js
function User() {}

const user = new User();

console.log(user.constructor === User);
```

Explain why the answer is based on property lookup rather than magical constructor metadata.

## Exercise I

```js
const base = {
  value: 1
};

const obj = Object.create(base);

Object.setPrototypeOf(obj, {
  value: 2
});

console.log(obj.value);
```

## Exercise J

```js
const proto = {
  state: []
};

const a = Object.create(proto);
const b = Object.create(proto);

a.state.push(1);

console.log(b.state);
```

---

# 98. Mastery Exercises

## Level 1 — Understand

Define:

```text
prototype
[[Prototype]]
prototype chain
own property
inherited property
shadowing
delegation
constructor.prototype
instanceof
```

---

## Level 2 — Explain

Explain:

```text
User.prototype
```

versus:

```text
Object.getPrototypeOf(new User())
```

---

## Level 3 — Predict

Predict:

```text
property lookup
shadowing
inherited setters
prototype mutation
instanceof
constructor replacement
```

---

## Level 4 — Implement

Build the prototype-aware object simulator.

---

## Level 5 — Debug

Diagnose:

```text
unexpected inherited property
shared prototype state
wrong instanceof result
detached prototype method
prototype mutation
prototype pollution vector
```

---

## Level 6 — Defend

Defend:

> JavaScript inheritance is fundamentally prototype delegation; classes provide a higher-level syntax over that model.

---

# 99. Principal-Level Reasoning Problems

## Problem 1 — Shared mutable state

A team proposes:

```js
Base.prototype.cache = new Map();
```

for application-wide optimization.

Analyze:

```text
ownership
sharing
thread/worker boundaries
lifetime
memory
security
test isolation
```

Would you keep it on the prototype?

---

## Problem 2 — Dynamic prototype switching

A state machine changes an object's prototype at runtime:

```js
Object.setPrototypeOf(state, nextPrototype);
```

Evaluate:

```text
correctness
debuggability
optimization
invariants
serialization
team understanding
```

Could composition or explicit state dispatch be clearer?

---

## Problem 3 — Type validation

A security-sensitive service uses:

```js
value.constructor === Admin
```

Replace it with a robust design.

Consider:

```text
authorization
branding
schema validation
capabilities
cross-realm values
prototype spoofing
```

---

# 100. Production Design Framework

For inheritance, ask:

```text
1. Is the relationship truly hierarchical?
2. Are methods the primary shared artifact?
3. Is mutable shared state avoided?
4. Is the hierarchy stable?
5. Is polymorphism valuable?
6. Will consumers subclass this API?
7. Does composition produce clearer ownership?
8. Can cross-realm objects appear?
9. Are prototype mutations required?
10. Does the design create security assumptions?
```

---

# 101. Prototype Review Checklist

Before shipping prototype-based code:

```text
Prototype
→ Is it intentional?

Own state
→ Is instance-specific state actually own?

Shared state
→ Is any mutable data accidentally shared?

Lookup
→ Could inherited values be mistaken for input data?

Mutation
→ Is the prototype changed after creation?

Security
→ Can untrusted keys reach the prototype?

Identity
→ Is instanceof relied upon?

Cross-realm
→ Could objects come from another realm?

Debugging
→ Can developers easily inspect the chain?
```

---

# 102. Completion Criteria

### Understand

You can define:

- prototype;
- `[[Prototype]]`;
- prototype chain;
- own/inherited;
- shadowing;
- delegation;
- constructor `.prototype`.

### Explain

You can explain:

- `Object.create`;
- `Object.getPrototypeOf`;
- `Object.setPrototypeOf`;
- `__proto__`;
- inherited property lookup;
- inherited setters;
- shadowing;
- prototype methods;
- `instanceof`;
- classes over prototypes;
- static vs instance prototype chains.

### Predict

You can correctly predict:

- inherited reads;
- inherited writes;
- own shadowing;
- accessors;
- prototype replacement;
- mutable prototype state;
- `instanceof`;
- constructor replacement.

### Implement

You can build:

```text
prototype-aware object
lookup
shadowing
inheritance
descriptors
instanceof-like behavior
```

### Debug

You can diagnose:

```text
wrong prototype
unexpected inheritance
shared mutable state
wrong instanceof
prototype mutation
prototype pollution
```

### Principal Judgment

You can choose:

```text
prototype inheritance
class hierarchy
composition
explicit delegation
```

based on:

```text
ownership
stability
memory
security
debuggability
extensibility
team understanding
```

**Evidence of mastery:**

- 90%+ prediction accuracy;
- working prototype simulator;
- correct explanation of `.prototype` vs `[[Prototype]]`;
- successful identification of shared prototype state;
- correct prototype-pollution threat analysis;
- defensible inheritance-vs-composition decision.

---

# 103. Key Takeaways

1. **A prototype is an internal object relationship represented conceptually by `[[Prototype]]`.**
2. **Property lookup can delegate from an object to its prototype chain.**
3. **The prototype chain terminates at `null`.**
4. **Prototype inheritance is delegation, not automatic property copying.**
5. **Own properties shadow inherited properties.**
6. **Reading an inherited data property does not create an own property.**
7. **Assigning an inherited writable data property typically creates an own shadowing property on the receiver.**
8. **Inherited accessors can execute with the child object as receiver.**
9. **Non-writable inherited properties can constrain writes to the receiver.**
10. **`Object.create(proto)` establishes a chosen prototype at creation time.**
11. **`Object.getPrototypeOf()` explicitly exposes the prototype relationship.**
12. **`Object.setPrototypeOf()` can mutate that relationship but should be used deliberately.**
13. **`__proto__` is legacy/prototype-accessor behavior; explicit Object APIs are clearer.**
14. **A function's `.prototype` property is different from the function object's own `[[Prototype]]`.**
15. **Instances created with `new` commonly get `Constructor.prototype` as their `[[Prototype]]`.**
16. **Prototype methods are shared across instances.**
17. **A prototype method still receives the actual call receiver as `this`.**
18. **Detaching a prototype method changes its invocation semantics and can lose the receiver.**
19. **Replacing `Constructor.prototype` does not retroactively change existing instances' `[[Prototype]]`.**
20. **Mutable data stored on a prototype is shared across instances and can cause accidental coupling.**
21. **`instanceof` is prototype/protocol-based, not merely a class-name check.**
22. **`Symbol.hasInstance` can customize `instanceof`.**
23. **Cross-realm values can make `instanceof` surprising; `Array.isArray` is often safer for arrays.**
24. **Classes are built on top of prototype-based semantics.**
25. **Class inheritance creates related prototype chains on instance and constructor sides.**
26. **Modifying built-in prototypes can create global shared behavior and security issues.**
27. **Prototype pollution is a security risk when untrusted keys can influence prototype-related state.**
28. **Null-prototype objects can be useful for dictionary-like data.**
29. **Engine optimizations such as hidden classes and inline caches are implementation details, not language guarantees.**
30. **Principal-level inheritance design requires explicit decisions about hierarchy, ownership, sharing, security, and composition.**

---

# 104. Final Mastery Drill

For every example, answer:

```text
1. What object is being accessed?
2. What is its [[Prototype]]?
3. Is the property own?
4. If missing, which prototype is searched next?
5. Which descriptor is found first?
6. Is the property data or accessor?
7. What is the receiver?
8. Does the operation shadow or mutate?
9. What identity relationships matter?
10. What security/performance/lifecycle implication follows?
```

### Drill 1

```js
const base = {
  value: 1
};

const child = Object.create(base);
```

### Drill 2

```js
child.value = 2;
```

### Drill 3

```js
delete child.value;
```

### Drill 4

```js
Object.setPrototypeOf(child, null);
```

### Drill 5

```js
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return this.name;
};
```

### Drill 6

```js
const a = new User("A");
const b = new User("B");
```

### Drill 7

```js
User.prototype = {
  greet() {
    return "new";
  }
};
```

Explain what changes for:

```text
a
b
future instances
```

### Drill 8

```js
User.prototype.permissions = [];
```

Explain:

```text
ownership
sharing
mutation
memory
security
```

### Drill 9

```js
const value = new (class {})();
```

Explain the prototype relationship without relying on class folklore.

### Drill 10

```js
value instanceof Object
```

Explain it as a prototype/protocol question.

---

# 105. Transition to Chapter 18

Chapter 17 established:

```text
prototype
   ↓
prototype chain
   ↓
delegation
   ↓
shared behavior
   ↓
constructor-instance relationships
```

Chapter 18 will place a higher-level abstraction over this mechanism:

```text
class
constructor
instance methods
static methods
extends
super
private fields
derived constructors
class initialization
```

The key progression becomes:

```text
prototype mechanics
   ↓
class syntax
   ↓
instance construction
   ↓
inheritance
   ↓
super
   ↓
object-oriented design
```

Chapter 18 will also examine where class syntax maps cleanly to prototypes and where class semantics introduce additional rules that cannot be reduced to a simplistic “class = constructor + prototype” slogan.

---

# 106. Chapter Completion Record


**Chapter:** 17 — Prototypes and Prototype Chains

**Status:** `[+] Completed`

**Strong Areas Expected:**

- `[[Prototype]]`;
- prototype chain lookup;
- own vs inherited;
- shadowing;
- inherited setters;
- constructor `.prototype`;
- prototype methods;
- `instanceof`;
- class/prototype relationship;
- prototype pollution;
- shared prototype state.

**Revision Triggers:**

- confusing `.prototype` with `[[Prototype]]`;
- saying prototypes copy properties into objects;
- assuming inherited writes always mutate the parent;
- forgetting inherited accessor receiver behavior;
- treating `constructor` as trusted type metadata;
- treating `instanceof` as a class-name check;
- storing mutable instance state on prototypes;
- mutating built-in prototypes without a compelling reason;
- forgetting cross-realm identity issues.

**Evidence of Mastery:**

- 90%+ prediction accuracy;
- working prototype-aware object simulator;
- correct `.prototype`/`[[Prototype]]` explanation;
- correct assignment/descriptor reasoning;
- correct `instanceof` reasoning;
- prototype-pollution analysis;
- inheritance-vs-composition design defense.

---