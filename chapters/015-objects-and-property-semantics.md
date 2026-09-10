# Chapter 15 — Objects and Property Semantics

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 02, 06, 07, 10, 13, and 14 — Values/Types, Expressions, Coercion, Scope, Closures, and `this`
>
> **Next:** Chapter 16 — Property Keys, Ordering, and Enumeration

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- define what an ECMAScript object is at the language-semantics level;
- distinguish primitive values from object values;
- explain object identity and reference sharing;
- distinguish an object value from the variables or bindings that refer to it;
- explain properties as keyed associations with values or accessors;
- distinguish data properties from accessor properties;
- explain property descriptors;
- explain writable, enumerable, configurable, getter, and setter semantics;
- use property access with dot notation and bracket notation correctly;
- explain when property keys are strings versus symbols;
- distinguish property lookup from lexical identifier resolution;
- explain own properties versus inherited properties;
- explain how the prototype chain affects property lookup;
- distinguish reading a property from invoking a method stored in a property;
- explain the receiver used by property access and method calls;
- understand `Object.defineProperty`, `Object.getOwnPropertyDescriptor`, and descriptor-driven design;
- explain shallow copying and why object assignment does not clone object state;
- distinguish object mutation, property replacement, and rebinding;
- reason about object extensibility, sealing, and freezing;
- understand accessor side effects and reentrancy;
- identify surprising behavior from getters, setters, proxies, and coercion;
- explain why `hasOwnProperty` patterns can fail on unusual objects;
- compare `in`, `Object.hasOwn`, and direct access;
- reason about property ownership and API invariants;
- implement a simplified object/property engine;
- debug property lookup and mutation systematically;
- assess object design for performance, memory, security, and maintainability;
- explain which object claims are ECMAScript semantics and which are engine implementation details.

---

# 2. Prerequisites

You should already understand:

```text
values and types
bindings
scope and identifier resolution
functions and closures
this and receiver semantics
prototypes at a high level
```

The central progression is:

```text
object value
   ↓
property key
   ↓
property lookup
   ↓
property descriptor / accessor
   ↓
prototype lookup if not own
   ↓
result
```

This chapter is about the semantics of an object itself.

The next chapter will focus more deeply on:

```text
which keys exist
how keys are ordered
how they are enumerated
```

---

# 3. What Is an Object?

At the language level, an object is a collection of properties plus an internal set of behaviors defined by the ECMAScript object model.

Example:

```js
const user = {
  name: "A",
  age: 30
};
```

Conceptually:

```text
user
  │
  ├── "name" → "A"
  └── "age"  → 30
```

But an object is more than a dictionary.

Objects participate in:

- prototype lookup;
- property descriptors;
- getter/setter semantics;
- extensibility rules;
- method invocation;
- coercion hooks;
- internal methods;
- proxy interception;
- inheritance;
- object identity.

Therefore:

> **JavaScript objects are semantic entities, not merely hash maps.**

---

# 4. Object Identity

Consider:

```js
const a = { value: 1 };
const b = a;
```

Then:

```js
a === b
```

is:

```text
true
```

because both bindings refer to the same object.

Think:

```text
a ───────┐
         ↓
      Object #1
         ↑
b ───────┘
```

The bindings are different.

The object identity is shared.

---

# 5. Object Assignment Does Not Clone

Consider:

```js
const a = { value: 1 };
const b = a;

b.value = 2;

console.log(a.value);
```

Result:

```text
2
```

Why?

```text
a → Object #1
b → Object #1
```

Mutating through `b` mutates the same object observed through `a`.

This is one of the most important object-model rules in JavaScript.

---

# 6. Binding vs Object

Consider:

```js
let user = {
  name: "A"
};

user = {
  name: "B"
};
```

The binding:

```text
user
```

was reassigned.

The original object was not mutated.

Compare:

```js
const user = {
  name: "A"
};

user.name = "B";
```

Here:

```text
binding unchanged
object mutated
property changed
```

So distinguish:

```text
binding mutation
object mutation
property mutation
```

These are not the same operation.

---

# 7. Mental Model

Think of:

```js
const user = {
  name: "A",
  age: 30
};
```

as:

```text
Binding:
user
  ↓
Object identity: #42

Object #42:
  property "name"
    descriptor/value → "A"

  property "age"
    descriptor/value → 30
```

The exact physical layout is engine-specific.

The semantic model is property-based.

---

# 8. Properties

A property is a named association on an object.

A property can be:

```text
data property
```

or:

```text
accessor property
```

Data property:

```js
const user = {
  name: "A"
};
```

Conceptually:

```text
name
→ value "A"
```

Accessor:

```js
const user = {
  get name() {
    return "A";
  }
};
```

Conceptually:

```text
name
→ getter function
```

The read behavior is different.

---

# 9. Data Properties

A normal object property is represented semantically using attributes such as:

```text
[[Value]]
[[Writable]]
[[Enumerable]]
[[Configurable]]
```

Example:

```js
const user = {
  name: "A"
};
```

Conceptually:

```text
name:
  value = "A"
  writable = true
  enumerable = true
  configurable = true
```

Object literal properties normally start with ordinary writable/enumerable/configurable behavior unless syntax specifies otherwise.

---

# 10. Accessor Properties

Accessor properties use:

```text
[[Get]]
[[Set]]
[[Enumerable]]
[[Configurable]]
```

Example:

```js
const user = {
  get name() {
    return "A";
  }
};
```

The property does not store a normal `[[Value]]`.

Instead, reading it invokes the getter.

---

# 11. Data vs Accessor

Compare:

```js
const a = {
  value: 10
};
```

with:

```js
const b = {
  get value() {
    return 10;
  }
};
```

Both support:

```js
obj.value
```

but semantics differ.

For `a`:

```text
property read
→ retrieve stored value
```

For `b`:

```text
property read
→ invoke getter
→ produce result
```

This distinction matters for:

- side effects;
- performance;
- exceptions;
- reentrancy;
- debugging;
- security;
- API design.

---

# 12. Property Descriptors

Inspect:

```js
const user = {
  name: "A"
};

console.log(
  Object.getOwnPropertyDescriptor(user, "name")
);
```

Typical conceptual result:

```js
{
  value: "A",
  writable: true,
  enumerable: true,
  configurable: true
}
```

The exact object returned by the API uses JavaScript property names that expose the descriptor state.

---

# 13. Descriptor APIs

Important APIs:

```js
Object.getOwnPropertyDescriptor(obj, key);

Object.getOwnPropertyDescriptors(obj);

Object.defineProperty(obj, key, descriptor);

Object.defineProperties(obj, descriptors);
```

These let you inspect and define property semantics explicitly.

---

# 14. `Object.defineProperty`

Example:

```js
const user = {};

Object.defineProperty(user, "id", {
  value: 42,
  writable: false,
  enumerable: true,
  configurable: false
});
```

Now:

```text
id value = 42
writable = false
enumerable = true
configurable = false
```

This is useful when API invariants matter.

---

# 15. Non-Writable Is Not Immutable Object

Consider:

```js
const user = {};

Object.defineProperty(user, "profile", {
  value: {
    name: "A"
  },
  writable: false
});
```

This does not make the nested object immutable.

You cannot replace:

```js
user.profile = {};
```

through ordinary assignment.

But:

```js
user.profile.name = "B";
```

can still mutate the nested object if it remains mutable.

Thus:

```text
non-writable property
≠
deep immutable object
```

---

# 16. `writable`

For a data property:

```text
[[Writable]] = true
```

means the property value can be changed through applicable assignment operations.

Example:

```js
const obj = {
  value: 1
};

obj.value = 2;
```

With:

```js
Object.defineProperty(obj, "value", {
  writable: false
});
```

ordinary assignment cannot change the value.

In strict mode, inappropriate assignment can throw.

---

# 17. `enumerable`

A property can be visible to enumeration operations.

For example:

```js
Object.keys(obj);
```

returns enumerable own string-keyed properties.

Changing:

```text
enumerable = false
```

does not remove the property.

It changes how certain enumeration APIs observe it.

---

# 18. `configurable`

`configurable` controls whether the property descriptor and existence can be changed in specified ways.

Example:

```js
Object.defineProperty(obj, "id", {
  configurable: false
});
```

Later attempts to delete the property or reconfigure certain attributes can fail.

This supports API invariants.

---

# 19. Getters

Example:

```js
const user = {
  firstName: "A",
  lastName: "B",

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
};

console.log(user.fullName);
```

Reading:

```js
user.fullName
```

invokes the getter.

It is not equivalent to:

```text
stored fullName value
```

---

# 20. Getters Can Have Side Effects

Example:

```js
const obj = {
  get value() {
    console.log("getter ran");
    return 10;
  }
};

console.log(obj.value);
```

Output:

```text
getter ran
10
```

Therefore property reads are not always side-effect-free.

This matters for:

- debugging;
- serialization;
- logging;
- validation;
- reactive systems;
- framework internals;
- security-sensitive code.

---

# 21. Setters

Example:

```js
const user = {
  _name: "",

  set name(value) {
    this._name = value.trim();
  }
};

user.name = " A ";
```

The setter runs during assignment.

Again:

```text
property assignment
```

can execute arbitrary JavaScript.

It is not always:

```text
store value
```

---

# 22. Getter + Setter Pair

Example:

```js
const account = {
  _balance: 0,

  get balance() {
    return this._balance;
  },

  set balance(value) {
    if (value < 0) {
      throw new RangeError("Negative balance");
    }

    this._balance = value;
  }
};
```

The property acts as an API boundary.

This can enforce invariants.

But it also introduces executable behavior into seemingly simple property access.

---

# 23. Property Access vs Identifier Resolution

Compare:

```js
user.name
```

with:

```js
name
```

These are different operations.

```text
name
→ lexical identifier resolution

user.name
→ evaluate user
→ property key "name"
→ object property semantics
```

Do not describe:

```js
user.name
```

as “looking for a variable named name.”

It is property access.

---

# 24. Dot Notation

Dot notation:

```js
obj.name
```

is convenient when the property name is a valid identifier-like name in the grammar.

It accesses a property whose key corresponds to:

```text
"name"
```

---

# 25. Bracket Notation

Bracket notation:

```js
obj["name"]
```

also accesses:

```text
"name"
```

But bracket notation supports dynamic keys:

```js
const key = "name";

obj[key];
```

The key expression is evaluated.

---

# 26. Dynamic Property Access

Example:

```js
const field = "email";

const user = {
  email: "a@example.com"
};

console.log(user[field]);
```

The sequence is:

```text
evaluate user
→ evaluate field
→ "email"
→ property key conversion
→ lookup property
→ return result
```

This will connect deeply to Chapter 16.

---

# 27. Property Key Conversion

Property keys are generally:

```text
string
or
symbol
```

When bracket notation receives another primitive, property-key conversion can occur.

Example:

```js
const obj = {};

obj[1] = "one";

console.log(obj["1"]);
```

Result:

```text
"one"
```

The numeric key is converted to the corresponding string property key.

---

# 28. Objects as Property Keys

Consider:

```js
const obj = {};

const key = {};

obj[key] = 42;
```

For an ordinary object property key operation, the object must be converted to a property key, commonly resulting in a string unless a symbol is produced through coercion behavior.

This is one reason:

```js
Map
```

exists.

A `Map` can preserve object identity as a key.

Ordinary object property keys cannot use arbitrary object identity as a distinct key without conversion.

---

# 29. `Symbol` Keys

Symbols provide property keys that are not ordinary strings.

Example:

```js
const secret = Symbol("secret");

const obj = {
  [secret]: 42
};
```

Access:

```js
obj[secret];
```

Symbols are useful for:

- collision-resistant internal properties;
- protocol hooks;
- metaprogramming;
- well-known language hooks.

They are not automatically private.

A caller with the symbol can access the property.

---

# 30. Own Properties

An own property is stored directly on the object.

Example:

```js
const parent = {
  x: 1
};

const child = Object.create(parent);
child.y = 2;
```

Own properties:

```text
child → y
parent → x
```

`child` does not have its own `x`.

It inherits `x`.

---

# 31. Inherited Properties

Property access follows prototype semantics when an own property is not found.

```js
child.x
```

conceptually:

```text
child has own x?
→ no

prototype of child
→ parent

parent has x?
→ yes

return 1
```

The prototype chain will be studied more deeply in Chapter 17.

---

# 32. `Object.hasOwn`

Modern code can use:

```js
Object.hasOwn(obj, key)
```

to check whether the property is an own property.

Example:

```js
Object.hasOwn(child, "x"); // false
Object.hasOwn(child, "y"); // true
```

This is generally clearer than:

```js
obj.hasOwnProperty(key)
```

because the latter assumes the property exists and is callable from the object's own prototype chain.

---

# 33. Why `hasOwnProperty` Can Be Unsafe

Consider:

```js
const obj = Object.create(null);

obj.value = 1;
```

Then:

```js
obj.hasOwnProperty
```

does not exist because:

```text
prototype = null
```

So:

```js
obj.hasOwnProperty("value")
```

fails.

Prefer:

```js
Object.hasOwn(obj, "value");
```

or a carefully controlled equivalent.

---

# 34. `in` Operator

Example:

```js
"x" in child
```

can return:

```text
true
```

even when `x` is inherited.

Thus:

```text
Object.hasOwn(obj, key)
→ own only

key in obj
→ own or inherited
```

This distinction is essential.

---

# 35. Direct Read vs Ownership Test

Consider:

```js
if (obj.value) {
  ...
}
```

This does not answer:

```text
Does obj own `value`?
```

It asks whether the resulting value is truthy.

Possible property states include:

```text
missing
present with undefined
present with null
inherited
accessor returning false
accessor throwing
```

Therefore existence, ownership, and truthiness are distinct questions.

---

# 36. Missing Properties

Example:

```js
const obj = {};

console.log(obj.missing);
```

Result:

```text
undefined
```

This does not mean:

```text
the property exists and stores undefined
```

Compare:

```js
const obj = {
  value: undefined
};

console.log("value" in obj); // true
```

versus:

```js
console.log("missing" in obj); // false
```

This distinction matters for:

- configuration;
- patch semantics;
- serialization;
- defaulting;
- API validation.

---

# 37. Defaulting Operators and Property Presence

Compare:

```js
const value = obj.x || fallback;
```

with:

```js
const value = obj.x ?? fallback;
```

and:

```js
const hasX = Object.hasOwn(obj, "x");
```

These answer different questions:

```text
|| → truthiness
?? → nullishness
hasOwn → ownership
```

Do not substitute one for another when property presence matters.

---

# 38. Property Deletion

Example:

```js
const obj = {
  value: 1
};

delete obj.value;
```

Now the own property is removed when the delete operation is allowed by property configurability.

Afterward:

```js
obj.value
```

produces:

```text
undefined
```

But again:

```text
missing property
```

is different from:

```text
existing property with value undefined
```

---

# 39. Non-Configurable Properties and Delete

Example:

```js
const obj = {};

Object.defineProperty(obj, "id", {
  value: 42,
  configurable: false
});
```

Then:

```js
delete obj.id;
```

cannot remove the property through ordinary semantics.

In strict code, attempting an invalid delete can throw.

This is an important example of descriptor-controlled object invariants.

---

# 40. Object Extensibility

Objects are ordinarily extensible.

You can prevent new own properties with:

```js
Object.preventExtensions(obj);
```

This does not necessarily:

```text
freeze existing properties
```

It primarily affects adding new properties.

---

# 41. Sealing

```js
Object.seal(obj);
```

conceptually:

```text
prevent extensions
+
make own properties non-configurable
```

It does not automatically make data properties non-writable.

---

# 42. Freezing

```js
Object.freeze(obj);
```

conceptually strengthens the object further:

```text
prevent extensions
+
own properties become non-configurable
+
own data properties become non-writable
```

But:

```text
freeze is shallow
```

Nested objects can remain mutable.

---

# 43. Shallow Freeze Example

```js
const config = {
  server: {
    port: 3000
  }
};

Object.freeze(config);

config.server.port = 4000;
```

The nested object can still change.

Deep immutability requires a separate recursive strategy with its own trade-offs.

---

# 44. Object Integrity Levels

Useful hierarchy:

```text
extensible
      ↓
non-extensible
      ↓
sealed
      ↓
frozen
```

These are object-level integrity mechanisms.

They do not change the fact that objects are mutable references in the general language model unless all relevant reachable state is also controlled.

---

# 45. Mutation vs Replacement

Example:

```js
const obj = {
  nested: {
    value: 1
  }
};
```

Mutation:

```js
obj.nested.value = 2;
```

Replacement:

```js
obj.nested = {
  value: 2
};
```

Both produce a different observable state, but their aliasing implications differ.

If another binding points to the old nested object:

```js
const nested = obj.nested;
```

then:

```js
obj.nested.value = 2;
```

is visible through `nested`.

But:

```js
obj.nested = { value: 2 };
```

does not change the object referenced by `nested`.

---

# 46. Copying Objects

Shallow spread:

```js
const copy = { ...original };
```

creates a new outer object.

Nested references are still shared.

Example:

```js
const original = {
  profile: {
    name: "A"
  }
};

const copy = { ...original };

copy.profile.name = "B";

console.log(original.profile.name);
```

Result:

```text
"B"
```

because:

```text
original.profile
copy.profile
```

refer to the same nested object.

---

# 47. `Object.assign`

Similar shallow behavior:

```js
const copy = Object.assign({}, original);
```

It copies enumerable own properties into the target.

It does not clone arbitrary object graphs.

Accessor and descriptor semantics also require care: ordinary copying through these APIs is not equivalent to copying full property descriptors.

---

# 48. Descriptor-Preserving Copy

If descriptor preservation is important:

```js
const clone = Object.create(
  Object.getPrototypeOf(original),
  Object.getOwnPropertyDescriptors(original)
);
```

This is closer to preserving:

```text
descriptors
+
prototype
```

than:

```js
{ ...original }
```

But even this is not a deep clone.

It may preserve accessors that still reference original closure/environment state.

---

# 49. Accessors and Copying

Consider:

```js
const original = {
  get value() {
    return 42;
  }
};
```

Using:

```js
const copy = { ...original };
```

invokes ordinary property access while producing the copied value rather than simply copying the getter descriptor.

By contrast:

```js
Object.defineProperties(
  target,
  Object.getOwnPropertyDescriptors(original)
);
```

can preserve accessor descriptors.

This matters for library authors and metaprogramming.

---

# 50. Property Access Is Executable

Given:

```js
const obj = {
  get value() {
    throw new Error("boom");
  }
};
```

then:

```js
obj.value
```

throws.

Thus an API consumer cannot universally assume:

```text
property read = cheap data retrieval
```

The property may invoke:

- getter;
- proxy trap;
- coercion-related logic;
- other user code.

---

# 51. Reentrancy Through Accessors

Example:

```js
const obj = {
  get value() {
    callback();
    return 10;
  }
};
```

A seemingly internal operation:

```js
const x = obj.value;
```

can run arbitrary external code.

This creates reentrancy possibilities.

Library authors must consider:

```text
object invariants
→ property access
→ callback
→ nested API call
```

---

# 52. Proxies Preview

A Proxy can intercept operations such as:

```js
get
set
has
deleteProperty
defineProperty
getOwnPropertyDescriptor
ownKeys
```

This means property semantics can be customized.

Example:

```js
const proxy = new Proxy(
  { value: 1 },
  {
    get(target, key, receiver) {
      console.log("get:", key);
      return Reflect.get(target, key, receiver);
    }
  }
);
```

Then:

```js
proxy.value;
```

executes the proxy trap.

Chapter 19 will cover this in depth.

---

# 53. Receiver Matters

For:

```js
obj.method()
```

property lookup obtains the function, but the invocation receiver is still:

```text
obj
```

For getters and setters, a receiver also participates in the property access semantics.

This becomes especially important with:

```text
prototype inheritance
Reflect.get
Proxy
accessors
super
```

Therefore:

```text
target
```

and:

```text
receiver
```

are not always interchangeable.

---

# 54. `Reflect.get` Preview

Consider:

```js
Reflect.get(obj, "value, receiver)
```

with the appropriate arguments.

The third argument can affect accessor `this` behavior.

This is important for proxy/prototype forwarding.

The general lesson:

> The receiver of property access can differ from the object where the property descriptor was found.

Chapter 19 will formalize this.

---

# 55. Property Lookup Is Not Assignment

Consider:

```js
const obj = {};

obj.x = 10;
```

Conceptually, assignment must decide:

```text
Does an inherited setter exist?
Is there an own accessor?
Can a new property be created?
Is the object extensible?
Is the existing property writable?
What receiver is being used?
```

Therefore property assignment is a semantic operation, not simply:

```text
hashMap[key] = value
```

---

# 56. Assignment to Inherited Data Properties

Example:

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

child.x = 2;
```

The assignment generally creates an own property on `child` rather than mutating the inherited data property on `parent`, subject to relevant descriptor constraints.

Now:

```text
parent.x → 1
child.x  → 2
```

This is a central prototype/property rule.

---

# 57. Assignment to Inherited Setter

If the prototype provides a setter:

```js
const parent = {
  set x(value) {
    this._x = value;
  }
};

const child = Object.create(parent);

child.x = 10;
```

the inherited setter can run with a receiver corresponding to `child`.

The write does not simply create an own `x` data property.

This demonstrates why:

```text
property location
```

and:

```text
receiver
```

must be modeled separately.

---

# 58. Property Descriptors as Invariants

A library may define:

```js
Object.defineProperty(config, "version", {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false
});
```

This communicates:

```text
version is part of stable API identity
```

Descriptor-based design can enforce invariants at the object boundary.

Use it when the constraint is meaningful enough to justify increased semantic complexity.

---

# 59. Object API Design

A public object should make clear:

```text
which properties are data
which are computed
which are mutable
which are enumerable
which can be removed
which are internal
which are inherited
```

For example:

```js
class User {
  #id;

  constructor(id) {
    this.#id = id;
  }

  get id() {
    return this.#id;
  }
}
```

This is different from:

```js
{
  id: 123
}
```

even though both expose:

```js
user.id
```

---

# 60. Data vs Computed Properties in APIs

Data property:

```js
user.age
```

can be conceptually cheap.

Accessor:

```js
user.age
```

may involve:

```text
calculation
validation
I/O if badly designed
logging
exceptions
```

Do not design getters with surprising heavy side effects.

A useful API principle:

> Property access should generally feel like observation, not hidden workflow execution.

This is a design convention, not a language rule.

---

# 61. Methods Are Properties

Consider:

```js
const obj = {
  greet() {
    return "hello";
  }
};
```

The method is stored as a property whose value is a function.

Then:

```js
obj.greet()
```

combines:

```text
property lookup
+
function call
+
receiver binding
```

That is why Chapter 14 and this chapter connect directly.

---

# 62. Property Read of Function vs Method Call

Compare:

```js
const fn = obj.greet;
```

with:

```js
obj.greet();
```

First:

```text
read property
→ function value
→ no call yet
```

Second:

```text
read property
→ function value
→ invoke with receiver obj
```

This distinction explains detached-method bugs.

---

# 63. Symbols and Hidden Protocols

Well-known symbols such as:

```js
Symbol.iterator
Symbol.toPrimitive
Symbol.toStringTag
```

let objects participate in language protocols.

For example:

```js
const obj = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
  }
};
```

Now the object can participate in:

```js
for (const value of obj) {
  console.log(value);
}
```

Properties therefore do more than hold business data.

They can implement language-level protocols.

---

# 64. Property Semantics and Coercion

Objects can customize primitive conversion through properties such as:

```js
Symbol.toPrimitive
```

Example:

```js
const money = {
  amount: 10,

  [Symbol.toPrimitive](hint) {
    if (hint === "number") return this.amount;
    return `$${this.amount}`;
  }
};
```

Then operators may trigger property lookup and method invocation internally.

This connects object semantics to Chapter 07.

---

# 65. Property Semantics and Serialization

Serialization mechanisms may inspect properties, invoke accessors in some contexts, and omit values according to their own rules.

For example:

```js
const obj = {
  get value() {
    return 10;
  }
};
```

Serializing such an object requires considering:

```text
enumerability
property keys
accessor evaluation
serialization rules
```

This will be explored more deeply in Chapter 28.

---

# 66. Property Semantics and Security

Property access can become a security issue when code mixes untrusted keys with dynamic objects.

Example:

```js
const value = config[userSuppliedKey];
```

Questions:

```text
Can the key reach inherited properties?
Should only own properties be allowed?
Can an accessor execute?
Could a proxy be involved?
Should a null-prototype dictionary be used?
```

Using:

```js
Object.create(null)
```

can eliminate a prototype chain for dictionary-like data, but it also removes inherited convenience methods.

Choose deliberately.

---

# 67. Prototype Pollution Preview

A dangerous family of bugs can occur when untrusted keys are written into ordinary objects and prototype-related behavior is not controlled.

For example, unsafe recursive merge utilities historically mishandled keys associated with prototype mutation.

Principal security lesson:

> Treat dynamic property names from untrusted input as part of the attack surface.

Mitigations depend on the exact operation:

```text
validate keys
use null-prototype dictionaries where appropriate
use Map for arbitrary-key dictionaries
avoid unsafe deep-merge logic
enforce own-property checks
```

Chapter 57 will study JavaScript security engineering more deeply.

---

# 68. Null-Prototype Objects

Example:

```js
const dict = Object.create(null);

dict["toString"] = "custom";
```

There is no inherited `toString` method because:

```text
prototype = null
```

Advantages:

```text
clean dictionary semantics
no prototype-name collisions
```

Trade-offs:

```text
missing Object methods
different debugging experience
must use Object APIs explicitly
```

A `Map` may still be more appropriate for many dictionary workloads.

---

# 69. Object vs Map

Use object when:

```text
JSON-like record
named fields
structural data
protocol object
configuration
```

Use `Map` when:

```text
dynamic key-value collection
arbitrary key types
frequent insertion/deletion
key identity matters
```

This is not a strict performance rule.

Choose based on semantics first.

---

# 70. Object Identity and Equality

Consider:

```js
{} === {}
```

Result:

```text
false
```

Each literal creates a distinct object identity.

Compare:

```js
const a = {};
const b = a;

a === b;
```

Result:

```text
true
```

because both refer to the same object.

---

# 71. Property Value Equality vs Object Equality

Consider:

```js
const a = { x: 1 };
const b = { x: 1 };
```

Then:

```js
a.x === b.x
```

is:

```text
true
```

but:

```js
a === b
```

is:

```text
false
```

Equal contents do not imply equal identity.

This distinction matters in:

- caches;
- memoization;
- React-like rendering systems;
- dependency tracking;
- weak collections;
- database identity mapping.

---

# 72. Mutation Observability Through Aliases

Example:

```js
const state = {
  count: 0
};

const observer = state;

state.count++;

console.log(observer.count);
```

Result:

```text
1
```

Because:

```text
state
observer
```

both refer to the same identity.

This is why hidden aliasing is a major source of application bugs.

---

# 73. Defensive Copying

When exposing object state:

```js
class Store {
  getState() {
    return this.state;
  }
}
```

a caller can mutate internal state.

A defensive copy:

```js
getState() {
  return { ...this.state };
}
```

can protect the outer structure.

But shallow copies do not protect nested references.

Use:

```text
immutability
deep cloning
structural sharing
immutable data structures
encapsulation
```

according to actual requirements.

---

# 74. Object Freezing as an API Boundary

Sometimes:

```js
Object.freeze(config)
```

can enforce a shallow configuration invariant.

But freezing everything recursively can:

- increase startup work;
- complicate performance;
- break expected mutability;
- fail on certain host objects;
- still not create complete isolation.

Do not use freezing as a generic substitute for architecture.

---

# 75. Object Extensibility and Library Design

A library may choose to expose objects that are:

```text
extensible
sealed
frozen
class instances
proxy-backed
```

Consider compatibility implications.

Freezing or sealing a public object can surprise consumers who expect extension points.

Design the contract explicitly.

---

# 76. Specification-Oriented Vocabulary

Important object-model vocabulary includes:

```text
Object
property
property key
data property
accessor property
property descriptor
[[Value]]
[[Writable]]
[[Enumerable]]
[[Configurable]]
[[Get]]
[[Set]]
prototype
receiver
own property
extensible
internal method
[[Get]]
[[Set]]
[[HasProperty]]
[[Delete]]
[[DefineOwnProperty]]
```

The internal methods are specification concepts, not ordinary JavaScript methods named with double brackets.

---

# 77. Internal Methods

A useful conceptual model for ordinary property access is:

```text
obj.x
   ↓
Get operation
   ↓
object internal `[[Get]]`
   ↓
find own property
   ↓
if absent, consult prototype
   ↓
if data property, return value
   ↓
if accessor, call getter
   ↓
result
```

For assignment:

```text
obj.x = value
   ↓
Set operation
   ↓
object internal `[[Set]]`
   ↓
descriptor / prototype / receiver rules
   ↓
store, invoke setter, or reject
```

This is much closer to the specification-level mental model than:

```text
object[key] = value
```

as a simple dictionary mutation.

---

# 78. Ordinary Objects

An ordinary object follows the default internal-method algorithms defined by ECMAScript.

Special objects can differ.

Examples include:

```text
Array
Function
Proxy
Module namespace objects
Typed arrays
Date
RegExp
```

Some have specialized internal behavior.

This chapter focuses primarily on ordinary object property semantics.

---

# 79. Proxy Preview

A Proxy can replace default behavior:

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});
```

Now:

```js
proxy.value
```

can trigger custom code before the result is obtained.

This demonstrates why:

```text
ordinary object semantics
```

and:

```text
proxy semantics
```

must be kept separate.

---

# 80. Performance Considerations

Do not assume:

```text
object property access = hash table lookup
```

Modern engines can use optimized structures such as:

```text
hidden classes
shapes
inline caches
property offsets
specialized machine code
```

The exact mechanism varies by engine.

Common performance considerations:

- stable object shapes can help optimization;
- wildly changing property layouts can make access less predictable;
- excessive dynamic meta-programming can inhibit optimization;
- creating many unnecessary accessor layers can affect hot paths;
- repeated object cloning can increase allocation and GC pressure.

Do not optimize shape behavior without measurement.

---

# 81. Hidden Classes / Shapes

A common engine strategy is to assign internal structural metadata to objects with similar property layouts.

For conceptual purposes:

```text
Object A
shape S1
properties: a, b

Object B
shape S1
properties: a, b
```

may share structural metadata.

Adding properties in inconsistent orders can create different shapes.

This is engine-specific implementation knowledge, not an ECMAScript guarantee.

---

# 82. Inline Caches

Engines may optimize repeated property access such as:

```js
function readX(obj) {
  return obj.x;
}
```

based on observed object shapes.

A monomorphic access site might repeatedly see:

```text
same structural shape
```

while a highly polymorphic site may see many.

Again:

> This is a useful engine-performance mental model, not a language-level requirement.

---

# 83. Memory Considerations

Every object has semantic properties and identity.

Physical memory representation may include:

```text
object header
shape metadata
property storage
elements storage
embedded values
external resources
```

Exact layout varies by engine.

Important application-level memory questions:

```text
How many objects exist?
How large are nested graphs?
How long are references retained?
Are objects copied repeatedly?
Are closures retaining them?
Are caches bounded?
```

---

# 84. Security Considerations

Object semantics become security-sensitive when:

```text
keys are attacker-controlled
prototypes are mutable
accessors execute code
proxies are involved
objects cross trust boundaries
```

Safe engineering practices include:

```text
validate dynamic keys
avoid unsafe merges
use own-property checks
use Map where key identity matters
treat getters as executable code
avoid exposing unnecessary object authority
```

---

# 85. Production Usage

Object semantics are used constantly in:

```text
configuration
API payloads
database records
state objects
domain models
caches
request contexts
service containers
framework internals
protocol objects
```

For production systems, ask:

```text
Who owns this object?
Who may mutate it?
What properties are public?
Which properties are computed?
What is the expected prototype?
Can external code extend it?
How long does it remain reachable?
```

Those questions expose hidden coupling.

---

# 86. Implementation From Scratch

Build a simplified object model.

Requirements:

```text
object identity
own property table
property descriptors
get
set
hasOwn
delete
prototype link
extensibility
```

Do not attempt to reproduce every ECMAScript rule initially.

Start with ordinary data properties.

---

# 87. Basic Property Store

```js
class SimpleObject {
  constructor(prototype = null) {
    this.prototype = prototype;
    this.properties = new Map();
    this.extensible = true;
  }

  define(key, descriptor) {
    if (!this.extensible && !this.properties.has(key)) {
      throw new TypeError("Object is not extensible");
    }

    this.properties.set(key, {
      configurable: true,
      enumerable: true,
      writable: true,
      ...descriptor
    });
  }

  hasOwn(key) {
    return this.properties.has(key);
  }
}
```

This is a teaching model, not a specification implementation.

---

# 88. Property Read

Add:

```js
get(key) {
  if (this.properties.has(key)) {
    return this.properties.get(key).value;
  }

  if (this.prototype) {
    return this.prototype.get(key);
  }

  return undefined;
}
```

This models the simplest:

```text
own property
→ prototype
→ undefined
```

lookup path.

---

# 89. Property Write

Add:

```js
set(key, value) {
  const own = this.properties.get(key);

  if (own) {
    if (own.writable === false) {
      throw new TypeError("Property is not writable");
    }

    own.value = value;
    return;
  }

  if (!this.extensible) {
    throw new TypeError("Object is not extensible");
  }

  this.define(key, { value });
}
```

Then add prototype/setter semantics later.

---

# 90. Own vs Inherited Read

Extend:

```js
hasProperty(key) {
  if (this.properties.has(key)) {
    return true;
  }

  return this.prototype
    ? this.prototype.hasProperty(key)
    : false;
}
```

Now simulate:

```text
hasOwn
vs
hasProperty
```

which corresponds conceptually to:

```text
Object.hasOwn
vs
`in`
```

---

# 91. Accessor Support

Represent:

```js
{
  get,
  set,
  enumerable,
  configurable
}
```

Then:

```text
get property
→ if getter exists, invoke it
→ otherwise use data value
```

and:

```text
set property
→ if setter exists, invoke it
→ otherwise use writable data semantics
```

---

# 92. Receiver-Aware Simulator

Extend:

```js
get(key, receiver = this)
```

and:

```js
set(key, value, receiver = this)
```

This prepares the simulator for:

```text
prototype setters
accessor `this`
Proxy
Reflect
```

The receiver is one of the most important advanced object-model concepts.

---

# 93. Guided Implementation

Implement:

```text
defineProperty
getOwnProperty
get
set
hasOwn
hasProperty
delete
preventExtensions
```

Then test:

```text
own property
inherited property
readonly property
missing property
getter
setter
non-extensible object
```

---

# 94. Partially Guided Implementation

Build support for:

```text
data descriptor
accessor descriptor
prototype chain
receiver
extensibility
```

Add validation:

```text
invalid descriptor transitions
non-configurable properties
non-writable properties
setter/getter conflicts
```

---

# 95. No-Reference Implementation

Implement a mini ordinary-object engine from memory.

Support:

```text
{
  property definitions
  descriptors
  prototype
  get
  set
  has
  delete
}
```

Do not look at the implementation from earlier sections.

Then compare your result with expected ECMAScript behavior.

---

# 96. Edge-Case Hardening

Add:

- symbol keys;
- numeric-looking keys;
- inherited accessors;
- non-extensible objects;
- non-configurable properties;
- strict assignment failures;
- getter exceptions;
- setter exceptions;
- receiver differences;
- null-prototype objects;
- prototype chains with multiple levels.

---

# 97. Production-Grade Exercise

Build a configuration object system with:

```text
immutable version field
validated setters
private/internal symbols
null-prototype dictionary for arbitrary user keys
own-property validation
prototype-safe merging
audit logging
```

Then write tests proving:

```text
forbidden writes fail
unexpected inherited keys are ignored
getter/setter invariants hold
dynamic keys do not alter object prototypes
```

---

# 98. Debugging Workflow

When a property behaves unexpectedly:

```text
1. What is the exact object?
2. What is its prototype?
3. Is the property own?
4. What is the property key?
5. Is it a data or accessor property?
6. What descriptor applies?
7. Is it inherited?
8. Is a proxy involved?
9. What is the receiver?
10. Is the object extensible?
11. Is the value missing, undefined, null, or falsy?
```

This checklist prevents many common mistakes.

---

# 99. Debugging Exercise 1

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

console.log(child.x);
console.log(Object.hasOwn(child, "x"));
console.log("x" in child);
```

Explain:

```text
read
→ inherited

hasOwn
→ false

in
→ true
```

---

# 100. Debugging Exercise 2

```js
const obj = {
  value: undefined
};

console.log(obj.value);
console.log("value" in obj);
console.log(Object.hasOwn(obj, "value"));
```

The result demonstrates:

```text
existing property
+
undefined value
```

is different from:

```text
missing property
```

---

# 101. Debugging Exercise 3

```js
const obj = {
  get value() {
    throw new Error("boom");
  }
};

console.log(obj.value);
```

Question:

```text
Why can a property read throw?
```

Answer using:

```text
accessor property
→ getter execution
→ abrupt completion
```

---

# 102. Debugging Exercise 4

```js
const parent = {
  set value(next) {
    this._value = next;
  }
};

const child = Object.create(parent);

child.value = 10;

console.log(child._value);
console.log(Object.hasOwn(child, "value"));
```

Reason about:

```text
inherited setter
receiver = child
setter writes child._value
```

---

# 103. Debugging Exercise 5

```js
const obj = Object.create(null);

obj.value = 1;

console.log(Object.hasOwn(obj, "value"));
console.log(obj.hasOwnProperty);
```

Explain why the second result is:

```text
undefined
```

and why `Object.hasOwn` is appropriate.

---

# 104. Code Review Exercise

Review:

```js
function merge(target, source) {
  for (const key in source) {
    target[key] = source[key];
  }

  return target;
}
```

Questions:

```text
Does this copy inherited properties?
Can a key trigger a setter?
Can source be a proxy?
Can target be a proxy?
Can dynamic keys create security issues?
Does it preserve descriptors?
```

This is not a generic safe deep-merge utility.

A robust merge design needs an explicit contract.

---

# 105. Code Review Exercise — Truthiness Bug

Review:

```js
function getPort(config) {
  return config.port || 3000;
}
```

Potential issue:

```text
port = 0
```

is treated as missing.

Better, depending on requirements:

```js
return config.port ?? 3000;
```

But even that asks only about nullishness, not ownership.

If the API requires own-property presence:

```js
if (Object.hasOwn(config, "port")) {
  return config.port;
}

return 3000;
```

The correct choice depends on the contract.

---

# 106. Code Review Exercise — Returning Internal State

Review:

```js
class Store {
  constructor() {
    this.state = {
      users: []
    };
  }

  getState() {
    return this.state;
  }
}
```

Question:

```text
Can callers mutate internal state?
```

Yes.

Potential designs:

```text
defensive copy
readonly API
immutable state
controlled mutation methods
```

Choose according to performance and API requirements.

---

# 107. Interview Questions

## Beginner

1. What is an object in JavaScript?
2. What is object identity?
3. Why does `const a = {}; const b = a;` make `a === b` true?
4. What is a property?
5. What is the difference between dot and bracket notation?
6. What is an own property?

## Intermediate

7. What is a property descriptor?
8. What are writable, enumerable, and configurable?
9. What is the difference between data and accessor properties?
10. What is the difference between `in` and `Object.hasOwn`?
11. Why can a getter throw?
12. Why does object assignment not clone an object?
13. What does `Object.freeze` actually do?

## Advanced

14. Explain inherited property assignment.
15. Explain the receiver used by a setter.
16. Why are descriptors important to library authors?
17. Why is `{ ...obj }` only a shallow copy?
18. What is the difference between an object and a dictionary?
19. Why can null-prototype objects be useful?
20. Why can dynamic property keys create security risk?
21. How do properties interact with `Proxy` and `Reflect`?

## Principal

22. How would you design a safe user-defined key/value store?
23. When should an API expose data properties versus getters?
24. How would you diagnose an unexpected inherited property?
25. How would you decide between object, null-prototype object, and `Map`?
26. How would you audit a merge function for prototype pollution risk?
27. Which object-layout/performance claims are language guarantees and which are engine-specific?
28. How would you design object invariants using descriptors without making an API unnecessarily rigid?

---

# 108. Predict-the-Output Exercises

Predict before execution.

## Exercise A

```js
const a = {};
const b = a;

b.x = 10;

console.log(a.x);
```

## Exercise B

```js
const a = {};
const b = {};

console.log(a === b);
```

## Exercise C

```js
const parent = { x: 1 };
const child = Object.create(parent);

console.log(child.x);
console.log(Object.hasOwn(child, "x"));
console.log("x" in child);
```

## Exercise D

```js
const obj = {
  value: undefined
};

console.log("value" in obj);
console.log(Object.hasOwn(obj, "value"));
```

## Exercise E

```js
const obj = {
  get value() {
    return 42;
  }
};

console.log(obj.value);
```

## Exercise F

```js
const obj = {};

Object.defineProperty(obj, "x", {
  value: 1,
  writable: false
});

obj.x = 2;

console.log(obj.x);
```

## Exercise G

```js
const parent = {
  set x(value) {
    this.y = value;
  }
};

const child = Object.create(parent);

child.x = 10;

console.log(child.y);
console.log(Object.hasOwn(child, "x"));
```

## Exercise H

```js
const original = {
  nested: {
    value: 1
  }
};

const copy = { ...original };

copy.nested.value = 2;

console.log(original.nested.value);
```

## Exercise I

```js
const obj = Object.create(null);

obj.x = 1;

console.log(obj.hasOwnProperty);
console.log(Object.hasOwn(obj, "x"));
```

## Exercise J

```js
const obj = {};

Object.preventExtensions(obj);

obj.x = 1;

console.log(Object.hasOwn(obj, "x"));
```

Predict the difference between strict and non-strict code for the failed write.

---

# 109. Mastery Exercises

## Level 1 — Understand

Explain:

```text
object identity
property
descriptor
data property
accessor property
own property
inherited property
receiver
prototype
```

---

## Level 2 — Explain

Explain why:

```js
const b = a;
```

does not clone `a`.

---

## Level 3 — Predict

Predict:

```text
property reads
property writes
accessors
inheritance
missing vs undefined
freeze/seal
shallow copies
```

---

## Level 4 — Implement

Build the object-property simulator.

---

## Level 5 — Debug

Diagnose:

```text
unexpected inherited field
getter side effect
setter receiver bug
prototype pollution vector
accidental aliasing
```

---

## Level 6 — Defend

Defend:

> JavaScript objects are not generic hash maps; they have descriptors, prototype semantics, receiver-aware access, and internal methods.

---

# 110. Principal-Level Reasoning Problems

## Problem 1 — Object vs Map

A service receives 100,000 dynamic keys from users and needs arbitrary object identities as keys.

Compare:

```text
Object
Object.create(null)
Map
```

Evaluate:

```text
key semantics
prototype safety
iteration
deletion
API clarity
memory
performance assumptions
serialization
```

Choose a design and defend it.

---

## Problem 2 — Getter API

A team proposes:

```js
user.permissions
```

where the getter performs a database query.

Challenge the design.

Discuss:

```text
surprise
latency
side effects
reentrancy
caching
testing
observability
```

A property should not silently behave like an unbounded workflow unless the contract is extremely clear.

---

## Problem 3 — Merge Security

A utility accepts:

```js
merge({}, untrustedInput)
```

Design a threat model.

Consider:

```text
prototype-related keys
accessors
proxies
inherited properties
recursive structures
descriptor behavior
```

Then specify safe invariants for the implementation.

---

# 111. Production Design Framework

For a public object, ask:

```text
Ownership
→ who owns the object?

Mutation
→ who may change it?

Visibility
→ which properties are enumerable/public?

Inheritance
→ should inherited properties matter?

Identity
→ is identity significant?

Lifecycle
→ how long should it live?

Extensibility
→ should consumers add properties?

Security
→ can untrusted keys reach it?

Performance
→ is it on a hot path?

Observability
→ can mutation be traced?
```

This turns object semantics into architecture.

---

# 112. Property Semantics Review Checklist

Before shipping object-heavy code:

```text
1. Is the object used as a record or dictionary?
2. Are keys trusted?
3. Could inherited properties matter?
4. Are own-property checks explicit?
5. Could getters/setters execute?
6. Could a proxy be involved?
7. Are descriptors important?
8. Is shallow copying sufficient?
9. Are aliases intentional?
10. Is mutation ownership documented?
11. Could long-lived references retain large graphs?
12. Are object integrity levels appropriate?
```

---

# 113. Completion Criteria

### Understand

You can define:

- object identity;
- property;
- descriptor;
- data property;
- accessor property;
- own property;
- inherited property;
- receiver;
- prototype.

### Explain

You can explain:

- assignment vs mutation;
- property lookup;
- data/accessor semantics;
- descriptors;
- extensibility;
- sealing/freezing;
- shallow copies;
- inherited assignment;
- null-prototype objects;
- `Object.hasOwn` vs `in`;
- object vs `Map`.

### Predict

You can correctly predict:

- aliasing;
- missing vs undefined;
- getters/setters;
- descriptor restrictions;
- prototype lookup;
- inherited setters;
- shallow-copy aliasing;
- extension failures.

### Implement

You can build:

```text
object
property table
descriptor
prototype chain
get/set
own/inherited checks
extensibility
```

### Debug

You can diagnose:

```text
wrong receiver
unexpected inheritance
getter exception
setter behavior
aliasing bug
unsafe dynamic key
prototype-related vulnerability
```

### Principal Judgment

You can design object APIs using:

```text
ownership
mutation
identity
lifecycle
security
performance
extensibility
observability
```

**Evidence of mastery:**

- 90%+ prediction accuracy;
- functional mini object engine;
- safe handling of dynamic keys;
- correct descriptor reasoning;
- correct own-vs-inherited reasoning;
- defensible object-vs-Map decisions.

---

# 114. Key Takeaways

1. **JavaScript objects are semantic entities, not merely hash maps.**
2. **Object identity is distinct from object contents.**
3. **Bindings can point to objects; rebinding and object mutation are different operations.**
4. **Object properties can be data properties or accessor properties.**
5. **Descriptors control important property behavior such as writability, enumerability, and configurability.**
6. **A non-writable property does not make a referenced object deeply immutable.**
7. **Getters and setters execute code during property access.**
8. **Property access is different from lexical identifier resolution.**
9. **Bracket notation evaluates a key expression and can trigger property-key conversion.**
10. **Ordinary object keys are strings or symbols.**
11. **Own properties and inherited properties must be distinguished explicitly.**
12. **`Object.hasOwn()` answers a different question from `in`.**
13. **Missing properties and properties whose value is `undefined` are different states.**
14. **`delete`, extensibility, sealing, and freezing are descriptor/integrity operations.**
15. **Object spread and `Object.assign` perform shallow copying.**
16. **Prototype lookup is part of ordinary property semantics.**
17. **Assignment can interact with inherited setters and receiver semantics.**
18. **Null-prototype objects are useful for certain dictionary problems but have trade-offs.**
19. **`Map` is often preferable when arbitrary key identity is required.**
20. **Properties can implement language protocols through symbols.**
21. **Dynamic property keys and prototype-related behavior can create security vulnerabilities.**
22. **Engine optimizations such as shapes and inline caches are implementation details, not ECMAScript guarantees.**
23. **Long-lived object references and aliases are important memory-design considerations.**
24. **Principal-level object design is about ownership, invariants, key semantics, and lifecycle.**

---

# 115. Final Mastery Drill

For each example, answer:

```text
1. What object identity is involved?
2. What is the property key?
3. Is the property own or inherited?
4. What descriptor applies?
5. Is it data or accessor?
6. What is the prototype?
7. What is the receiver?
8. Can code execute during the operation?
9. Can the operation fail?
10. What production implication follows?
```

### Drill 1

```js
const parent = {
  x: 1
};

const child = Object.create(parent);

child.x = 2;
```

### Drill 2

```js
const parent = {
  set x(value) {
    this._x = value;
  }
};

const child = Object.create(parent);

child.x = 10;
```

### Drill 3

```js
const obj = {
  get value() {
    return this._value;
  },

  _value: 10
};
```

### Drill 4

```js
const original = {
  nested: {
    value: 1
  }
};

const copy = { ...original };
```

### Drill 5

```js
const obj = Object.create(null);
```

### Drill 6

```js
Object.defineProperty(obj, "id", {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false
});
```

### Drill 7

```js
const key = userInput;
target[key] = value;
```

For each drill, explain:

```text
language-level semantics
+
security implications
+
memory implications
+
performance assumptions
+
API-design consequences
```

---

# 116. Transition to Chapter 16

Chapter 15 established:

```text
What an object is.
What a property is.
How property reads/writes work.
How descriptors and prototypes participate.
```

Chapter 16 will narrow the focus to:

```text
What exactly is a property key?
Why are integer-like keys special?
What ordering does Object.keys use?
How do for...in, Object.keys, Reflect.ownKeys, and JSON differ?
Where do symbol keys appear?
Why does enumeration order matter for deterministic systems?
```

The conceptual progression becomes:

```text
object
   ↓
property
   ↓
property key
   ↓
key ordering
   ↓
enumeration
   ↓
deterministic object traversal
```

---

# 117. Chapter Completion Record

**Chapter:** 15 — Objects and Property Semantics

**Status:** `[+] Completed`

**Strong Areas Expected:**

- object identity;
- property descriptors;
- data/accessor properties;
- own vs inherited properties;
- receiver semantics;
- mutation vs replacement;
- object integrity levels;
- shallow copying;
- object vs Map;
- dynamic-key security;
- ordinary object internal-method mental model.

**Revision Triggers:**

- treating objects as simple hash maps;
- confusing bindings with objects;
- confusing missing with `undefined`;
- confusing `in` with own-property checks;
- assuming getters are stored values;
- assuming `Object.freeze` is deep;
- assuming spread clones nested objects;
- ignoring inherited setters;
- treating engine shapes as language guarantees.

**Evidence of Mastery:**

- 90%+ prediction accuracy;
- working object-property simulator;
- correct descriptor reasoning;
- correct prototype/receiver reasoning;
- successful review of dynamic-key security;
- defensible Object vs null-prototype Object vs Map choices.

---