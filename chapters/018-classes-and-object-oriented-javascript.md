
# Chapter 18 — Classes and Object-Oriented JavaScript

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 14–17 — `this`, Objects, Property Semantics, Property Keys, Prototypes, and Prototype Chains
>
> **Next:** Chapter 19 — Proxy, Reflect, and Metaprogramming

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- explain what a JavaScript class is semantically;
- distinguish class syntax from the underlying prototype/object model;
- explain class declarations and class expressions;
- distinguish instance methods from static methods;
- explain constructors and instance initialization;
- explain `extends` and the two prototype relationships created by class inheritance;
- explain `super()` and `super.method()` conceptually;
- explain why `this` cannot be used in a derived constructor before `super()`;
- explain base and derived constructor behavior;
- understand public instance fields;
- understand public static fields and initialization;
- understand private fields, private methods, and private access;
- distinguish `#private` fields from symbol properties and naming conventions;
- explain class method placement on prototypes;
- understand class method descriptors and enumerability at a conceptual level;
- explain why class bodies are strict mode;
- understand class declaration TDZ behavior;
- distinguish class methods from arrow-function instance properties;
- explain static methods, static fields, and static initialization blocks;
- reason about subclassing built-ins and species/subclassing concerns at a high level;
- explain inheritance versus composition;
- understand class field initialization ordering;
- reason about derived-class initialization and private-field availability;
- explain how private fields affect nominal identity and branding;
- understand class method `this` behavior when methods are detached;
- identify common class design mistakes;
- implement a simplified class system on top of prototypes;
- implement private state using closures and compare it with private fields;
- debug constructor, inheritance, `super`, field-order, and `this` bugs;
- assess classes for performance, memory, security, API design, and maintainability;
- distinguish specification guarantees from engine-specific implementation details;
- make principal-level decisions about inheritance, composition, abstraction boundaries, and class API contracts.

---

# 2. Prerequisites

You should already understand:

```text
functions
this
objects
property descriptors
prototype chains
constructors
closures
```

The core progression is:

```text
object model
   ↓
prototype model
   ↓
constructor model
   ↓
class syntax
   ↓
inheritance
   ↓
private state
   ↓
object-oriented design
```

The most important conceptual statement in this chapter is:

> **JavaScript classes do not replace the prototype system. Class semantics are built on top of it while adding additional syntax and language rules.**

---

# 3. What Is a Class?

A class declaration creates a class definition with:

- constructor behavior;
- prototype methods;
- optional static members;
- optional fields;
- optional private elements;
- optional inheritance.

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return this.name;
  }
}
```

Conceptually:

```text
User
├── constructor behavior
├── static members
└── User.prototype
      └── greet
```

Instances then relate to:

```text
instance
   ↓ [[Prototype]]
User.prototype
```

---

# 4. Why Do Classes Exist?

JavaScript had prototype-based objects before class syntax.

Classes provide a more structured declaration form for:

```text
construction
methods
inheritance
static behavior
fields
private state
```

They can improve:

```text
readability
organization
discoverability
team consistency
API communication
```

But class syntax is not automatically superior to:

```text
factories
closures
composition
plain objects
functional modules
```

Use the abstraction that best represents the domain.

---

# 5. Class Declaration

Basic form:

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return `Hello ${this.name}`;
  }
}
```

Create an instance:

```js
const user = new User("A");
```

Then:

```js
user.greet();
```

invokes the prototype method.

---

# 6. Class Expressions

A class can be an expression:

```js
const User = class {
  constructor(name) {
    this.name = name;
  }
};
```

Or named:

```js
const User = class UserClass {
  constructor(name) {
    this.name = name;
  }
};
```

The class expression can participate in ordinary expression contexts.

---

# 7. Class Declaration TDZ

Classes behave like lexical declarations.

Therefore:

```js
const user = new User();

class User {}
```

throws:

```text
ReferenceError
```

This connects directly to Chapter 11.

The binding:

```text
User
```

exists in the lexical environment but is uninitialized until class declaration evaluation.

---

# 8. Class Bodies Are Strict

Class method bodies execute under strict-mode semantics.

This affects:

```text
this
assignment behavior
eval
with
some legacy semantics
```

Therefore class code should not be reasoned about using sloppy-function assumptions.

---

# 9. Constructor

The constructor initializes a newly constructed instance.

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }
}
```

Calling:

```js
new User("A")
```

performs constructor-oriented evaluation.

Conceptually:

```text
construct
→ create instance
→ initialize receiver
→ execute constructor
→ finish construction
```

---

# 10. Base Class Constructor

A class without `extends` is a base class.

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }
}
```

Inside the constructor:

```text
this
```

refers to the new instance under normal construction.

---

# 11. Derived Class

Example:

```js
class Admin extends User {
}
```

Now:

```text
Admin.prototype
   ↓ [[Prototype]]
User.prototype
```

and:

```text
Admin
   ↓ [[Prototype]]
User
```

These are two related but distinct inheritance relationships.

---

# 12. Two Prototype Chains in Class Inheritance

For:

```js
class User {
  static role() {
    return "user";
  }

  greet() {
    return "hello";
  }
}

class Admin extends User {}
```

there are conceptually two chains.

### Instance chain

```text
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

### Constructor/static chain

```text
Admin
   ↓
User
   ↓
Function.prototype
   ↓
Object.prototype
   ↓
null
```

Therefore static inheritance and instance inheritance are both present.

---

# 13. `extends`

`extends` establishes inheritance relationships.

Example:

```js
class Admin extends User {
  deleteUser() {
    return true;
  }
}
```

Now:

```js
const admin = new Admin();
```

can find:

```text
deleteUser → Admin.prototype
greet       → User.prototype
```

through the prototype chain.

---

# 14. `super()`

A derived constructor needs to initialize the superclass portion of the instance.

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }
}

class Admin extends User {
  constructor(name, level) {
    super(name);
    this.level = level;
  }
}
```

`super(name)` invokes the superclass constructor behavior.

---

# 15. Why `this` Before `super()` Fails

Consider:

```js
class Child extends Parent {
  constructor() {
    this.value = 1;
    super();
  }
}
```

This is invalid.

A derived constructor does not have an initialized `this` before the required superclass initialization process.

The language enforces this to ensure construction semantics remain valid.

Mental model:

```text
derived constructor starts
       ↓
this not yet initialized
       ↓
super(...)
       ↓
instance receiver becomes available
       ↓
derived initialization continues
```

---

# 16. Base vs Derived Constructor

### Base

```js
class Base {
  constructor() {
    this.ready = true;
  }
}
```

The base constructor establishes the instance receiver directly.

### Derived

```js
class Child extends Base {
  constructor() {
    super();
    this.childReady = true;
  }
}
```

The derived constructor must coordinate with superclass initialization before using its instance receiver.

---

# 17. Default Derived Constructor

If a derived class omits its constructor:

```js
class Child extends Parent {}
```

the language provides derived construction behavior that forwards arguments to the parent construction process.

Conceptually:

```js
constructor(...args) {
  super(...args);
}
```

The specification semantics are more precise, but this is a useful mental model.

---

# 18. Default Base Constructor

If a base class does not define a constructor:

```js
class User {}
```

construction uses default base-class constructor behavior.

Conceptually:

```text
create instance
→ initialize normally
```

---

# 19. Class Methods

Example:

```js
class User {
  greet() {
    return this.name;
  }
}
```

The method is associated with:

```text
User.prototype
```

not copied into each instance by default.

Thus:

```js
const a = new User();
const b = new User();

a.greet === b.greet;
```

is ordinarily:

```text
true
```

because both instances resolve to the same prototype method.

---

# 20. Method Identity

Prototype methods are shared.

Instance arrow properties are not.

Compare:

```js
class User {
  greet() {}
}
```

with:

```js
class User {
  greet = () => {};
}
```

In the first:

```text
one prototype method
```

In the second:

```text
each instance receives its own function
```

This difference affects:

```text
memory
identity
prototype behavior
mocking
inheritance
performance
```

---

# 21. Static Methods

Example:

```js
class User {
  static create(name) {
    return new User(name);
  }
}
```

Call:

```js
User.create("A");
```

The static method belongs to the constructor object rather than:

```text
User.prototype
```

---

# 22. Static Properties

Example:

```js
class Config {
  static version = 1;
}
```

Then:

```js
Config.version;
```

works.

But:

```js
const config = new Config();

config.version;
```

does not mean the same thing.

Static members live on the class/constructor side.

---

# 23. Static Inheritance

Because derived constructors have their own prototype relationship:

```text
Child.[[Prototype]] → Parent
```

static members can be inherited.

Example:

```js
class Parent {
  static hello() {
    return "hello";
  }
}

class Child extends Parent {}

console.log(Child.hello());
```

Lookup can delegate from:

```text
Child
→ Parent
```

This is distinct from:

```text
Child.prototype
→ Parent.prototype
```

---

# 24. Static vs Instance

Always classify a class member:

```text
instance
or
static
```

Example:

```js
class User {
  static find() {}
  save() {}
}
```

Then:

```text
User.find()
```

is static.

```text
user.save()
```

is instance behavior.

Mixing these concepts causes architectural confusion.

---

# 25. Public Instance Fields

Modern classes support field declarations:

```js
class User {
  name = "unknown";
  active = true;
}
```

When an instance is created, those fields are initialized as part of class instance initialization.

They are own properties of the instance.

This differs from prototype methods.

---

# 26. Field Initialization

Example:

```js
class User {
  name = "A";

  constructor() {
    console.log(this.name);
  }
}
```

The precise initialization order matters.

Class fields are initialized as part of instance construction, with timing relative to constructor execution defined by class semantics.

Do not reduce every field to:

```text
assignment after constructor
```

without accounting for base versus derived classes and field initializer evaluation.

---

# 27. Base-Class Field Initialization

In a base class:

```js
class User {
  name = "A";

  constructor() {
    console.log(this.name);
  }
}
```

the field initialization occurs as part of creating the base instance before the constructor body observes the fully initialized instance field state according to class semantics.

This is why:

```js
console.log(this.name);
```

can observe the declared field.

---

# 28. Derived-Class Field Initialization

Consider:

```js
class Parent {
  value = "parent";
}

class Child extends Parent {
  value = "child";

  constructor() {
    super();
  }
}
```

Construction must coordinate:

```text
Parent initialization
+
Child initialization
```

Field initializers execute in the relevant class initialization sequence.

This becomes important when fields:

```text
shadow
override
depend on methods
access private elements
```

---

# 29. Field Initializers Can Execute Code

Example:

```js
class User {
  id = createId();

  constructor() {
    log(this.id);
  }
}
```

The field initializer:

```js
createId()
```

runs during instance initialization.

Therefore class field declarations are not just static data layout.

They are executable expressions.

---

# 30. Field Initializer Ordering

Consider:

```js
class Example {
  first = 1;
  second = this.first + 1;
}
```

The result depends on declaration order.

Conceptually:

```text
first initialized
→ second initializer sees first
```

Reversing fields can change behavior:

```js
class Example {
  second = this.first + 1;
  first = 1;
}
```

At `second` initialization time, `first` has not yet received its field value.

This can produce different results.

---

# 31. Avoid Fragile Field Dependencies

While field ordering is defined, excessive cross-field dependencies can make construction harder to reason about.

Prefer:

```text
simple field defaults
+
constructor-level explicit initialization
```

when initialization logic becomes complex.

Class field syntax should improve clarity, not hide a dependency graph.

---

# 32. Public Static Fields

Example:

```js
class App {
  static version = "1.0.0";
}
```

The static field belongs to the class constructor object.

Its initializer executes during class definition/evaluation.

This is different from per-instance field initialization.

---

# 33. Static Initialization Blocks

Modern JavaScript supports:

```js
class Registry {
  static items = new Map();

  static {
    Registry.items.set("default", true);
  }
}
```

Static initialization blocks are useful when static setup requires multiple statements.

Conceptually:

```text
evaluate class
→ initialize static elements in defined order
```

They are class-definition-time executable logic.

---

# 34. Private Fields

Private fields use `#` syntax:

```js
class User {
  #id;

  constructor(id) {
    this.#id = id;
  }

  getId() {
    return this.#id;
  }
}
```

The private field is not an ordinary string property.

It cannot be accessed using:

```js
user["#id"]
```

or:

```js
user.id
```

---

# 35. Private Means Language-Level Private

Private fields are enforced by the language.

Example:

```js
class User {
  #id = 42;
}

const user = new User();

user.#id;
```

from outside the class body is invalid syntax.

There is no ordinary dynamic property lookup for the private name.

---

# 36. Private Field vs Symbol

A symbol property:

```js
const id = Symbol("id");

const user = {
  [id]: 42
};
```

is not truly private.

Code that obtains:

```text
id
```

can access:

```js
user[id]
```

A private field:

```js
#id
```

has stronger language-enforced access restrictions.

---

# 37. Private Field vs Naming Convention

This:

```js
this._id = 42;
```

is not private.

It is only a convention.

Private fields provide language-level access control.

Use naming conventions when:

```text
ecosystem interoperability
serialization
inspection
simple conventions
```

are more valuable.

Use `#private` when language-enforced encapsulation is important.

---

# 38. Private Methods

Example:

```js
class User {
  #normalize(name) {
    return name.trim();
  }

  constructor(name) {
    this.name = this.#normalize(name);
  }
}
```

Private methods are associated with the class's private brand and are not ordinary public prototype properties.

---

# 39. Private Static Elements

Private elements can also be static:

```js
class Parser {
  static #version = 1;

  static getVersion() {
    return Parser.#version;
  }
}
```

The private element belongs to the class/constructor side.

This differs from private instance fields.

---

# 40. Private Fields Create Branding

A private field access:

```js
this.#id
```

is not based on:

```text
property string lookup
```

The object must have the appropriate private brand.

This means private state has a more nominal flavor than ordinary prototype properties.

Two objects with identical public shape are not interchangeable for a specific private-name access unless they carry the corresponding private brand.

---

# 41. Private Field Brand Check

Consider:

```js
class User {
  #id = 1;

  getId() {
    return this.#id;
  }
}
```

If `getId` is called with an object that does not have the appropriate private brand, private access fails.

This is stronger than:

```js
this.id
```

because ordinary properties are open to prototype/property substitution.

---

# 42. Private Fields and Inheritance

Derived classes can define their own private fields:

```js
class Parent {
  #secret = 1;
}

class Child extends Parent {
  #secret = 2;
}
```

These are distinct private names/brands.

The child does not override the parent's private field through ordinary property shadowing semantics.

Private names belong to their class definitions.

---

# 43. Private State and Reflection

Private fields are not ordinary enumerable properties.

They do not appear through:

```js
Object.keys
Object.getOwnPropertyNames
Reflect.ownKeys
```

This is one reason private state is useful for stronger encapsulation.

But private data can still be observed indirectly through public methods.

---

# 44. Private State and Serialization

Private fields are not automatically serialized into JSON as ordinary properties.

Example:

```js
class User {
  #password = "secret";
}
```

The private field does not appear as a normal JSON property.

This reduces accidental disclosure through generic object serialization.

It does not guarantee the entire object is safe to serialize.

---

# 45. `in` and Private Fields

Private names have special syntax:

```js
#id in obj
```

inside class-capable contexts can test whether the object carries the corresponding private brand.

This is different from:

```js
"id" in obj
```

because the latter is ordinary property lookup through the object/prototype system.

---

# 46. Private Fields vs Closures

Closure-based private state:

```js
function createUser(id) {
  let secret = id;

  return {
    getId() {
      return secret;
    }
  };
}
```

Class private field:

```js
class User {
  #id;

  constructor(id) {
    this.#id = id;
  }

  getId() {
    return this.#id;
  }
}
```

Both can encapsulate state.

Compare:

```text
identity
inheritance
serialization
reflection
method sharing
factory semantics
private brand
testing
debugging
```

There is no universal winner.

---

# 47. Classes and Closures Can Be Combined

Example:

```js
class Service {
  #client;

  constructor(client) {
    this.#client = client;
  }

  createReader() {
    return () => this.#client.read();
  }
}
```

Now:

```text
class private field
+
closure
+
lexical this
```

coexist.

Real systems often combine language features rather than choosing one paradigm exclusively.

---

# 48. Class Methods and `this`

Class methods are ordinary receiver-dependent methods.

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return this.name;
  }
}
```

This works:

```js
user.greet();
```

But:

```js
const greet = user.greet;
greet();
```

detaches the receiver.

The class does not automatically bind the method.

---

# 49. Binding a Class Method

A class may explicitly bind:

```js
class User {
  constructor(name) {
    this.name = name;
    this.greet = this.greet.bind(this);
  }

  greet() {
    return this.name;
  }
}
```

This makes:

```js
const greet = user.greet;
greet();
```

work.

But it changes:

```text
function identity
memory
instance behavior
prototype sharing
```

Use intentionally.

---

# 50. Instance Arrow Method

Another pattern:

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet = () => {
    return this.name;
  };
}
```

This gives lexical `this`.

Trade-off:

```text
per-instance function
```

instead of:

```text
shared prototype method
```

It can be convenient for callbacks.

---

# 51. Class Fields vs Constructor Assignment

These two can express similar state:

```js
class User {
  active = true;
}
```

and:

```js
class User {
  constructor() {
    this.active = true;
  }
}
```

But class field syntax makes initialization part of the class field model.

When field initialization becomes dependent on constructor arguments, constructor assignment may be clearer:

```js
class User {
  active;

  constructor(role) {
    this.active = role !== "disabled";
  }
}
```

---

# 52. `super.method()`

Example:

```js
class User {
  greet() {
    return "hello";
  }
}

class Admin extends User {
  greet() {
    return super.greet() + " admin";
  }
}
```

`super.greet()` finds the superclass method according to `super` semantics and invokes it with the current receiver.

Conceptually:

```text
find super method
+
this remains current instance
```

---

# 53. `super` Does Not Replace `this`

In:

```js
super.greet()
```

the method obtained from the parent prototype can still see:

```text
this === current child instance
```

Thus:

```text
super
→ where to find inherited behavior

this
→ current receiver
```

These are different concepts.

---

# 54. `super` Setter Behavior

`super` can also participate in property assignment:

```js
class Parent {
  set value(v) {
    this._value = v;
  }
}

class Child extends Parent {
  update(v) {
    super.value = v;
  }
}
```

The property is resolved through the super base, while the receiver remains the current instance.

This distinction becomes important in accessor inheritance.

---

# 55. `super` in Static Methods

`super` also works in static methods:

```js
class Parent {
  static greet() {
    return "parent";
  }
}

class Child extends Parent {
  static greet() {
    return super.greet() + " child";
  }
}
```

The relevant super relationship is on the constructor/static side.

This reinforces the two-chain model from earlier.

---

# 56. Constructor Return Behavior

A constructor can explicitly return an object:

```js
class User {
  constructor() {
    return {
      replacement: true
    };
  }
}
```

An object return can affect the result of construction.

Returning a primitive does not replace the constructed object in the same way.

Avoid surprising constructor returns in normal application design unless the behavior is intentional and documented.

---

# 57. Subclassing Built-ins

Modern JavaScript supports subclassing built-in constructors:

```js
class MyArray extends Array {}
```

Instances can inherit from:

```text
MyArray.prototype
→ Array.prototype
```

and participate in specialized built-in semantics.

However, built-ins can have internal slots and construction behaviors that are more complex than ordinary user-defined classes.

Do not assume every built-in is equivalent to:

```text
plain object + prototype methods
```

---

# 58. Built-in Subclassing Caveats

When subclassing built-ins, consider:

```text
internal slots
species behavior
constructibility
returning derived instances
cross-realm behavior
performance
compatibility
```

Some built-ins have specialized semantics.

This prepares for Chapter 21 on species/subclassing.

---

# 59. `instanceof` with Classes

Example:

```js
class User {}

const user = new User();

user instanceof User;
```

typically returns:

```text
true
```

because the class has a prototype relationship compatible with `instanceof`.

The mechanism remains prototype/protocol-oriented.

Classes do not turn `instanceof` into nominal runtime typing in the way some statically typed languages do.

---

# 60. Class Identity vs Structural Shape

These objects can have identical public properties:

```js
const a = { name: "A" };
const b = { name: "A" };
```

but they are not instances of a particular class merely because the shape matches.

Likewise:

```js
class User {
  constructor(name) {
    this.name = name;
  }
}
```

does not mean every `{ name: "A" }` is a `User`.

JavaScript object shape and class/prototype identity are different concepts.

---

# 61. Inheritance vs Composition

Class inheritance:

```js
class Admin extends User {}
```

Composition:

```js
function createAdmin(user, permissions) {
  return {
    user,
    permissions
  };
}
```

Inheritance can provide:

```text
polymorphism
shared protocol
hierarchical substitution
```

Composition can provide:

```text
explicit dependencies
flexible behavior assembly
less hierarchy coupling
```

Choose based on actual domain relationships.

---

# 62. The Liskov Substitution Question

If:

```text
Admin extends User
```

then ask:

```text
Can an Admin be safely used anywhere a User is expected?
```

If not, inheritance may encode a misleading relationship.

JavaScript's flexibility makes it easy to create inheritance hierarchies that are syntactically valid but architecturally weak.

---

# 63. Prefer Stable Abstractions

A base class should expose a contract that subclasses can honor.

Avoid base classes that depend on:

```text
fragile initialization order
protected-by-convention mutable state
implicit callback timing
subclass implementation quirks
```

The more implicit the contract, the harder the hierarchy becomes to evolve.

---

# 64. Composition for Optional Behavior

Instead of:

```js
class LoggedAdmin extends AuditedUser {}
class CachedLoggedAdmin extends LoggedAdmin {}
class SecureCachedLoggedAdmin extends CachedLoggedAdmin {}
```

consider composing capabilities:

```text
service
+
logger
+
cache
+
authorization
```

This can avoid combinatorial inheritance trees.

---

# 65. Class Design and Dependency Injection

A class can receive dependencies explicitly:

```js
class UserService {
  constructor(repository, logger) {
    this.repository = repository;
    this.logger = logger;
  }
}
```

This makes dependencies visible.

But a class is not required for dependency injection.

A factory can achieve the same:

```js
function createUserService(repository, logger) {
  return {
    ...
  };
}
```

Choose based on abstraction and lifecycle.

---

# 66. Abstract Classes Are a Convention

JavaScript has no built-in abstract-class keyword enforcing all abstract-method rules as some statically typed languages do.

You can enforce contracts through:

```text
runtime checks
throwing base methods
symbols
private methods
TypeScript
documentation
tests
```

For example:

```js
class Repository {
  save() {
    throw new Error("save() must be implemented");
  }
}
```

This is a runtime convention.

---

# 67. Mixins

Mixins compose behavior without deep class inheritance.

Example:

```js
const Timestamped = Base => class extends Base {
  timestamp() {
    return Date.now();
  }
};
```

Then:

```js
class User {}

class TimestampedUser extends Timestamped(User) {}
```

Mixin patterns can be powerful but may create:

```text
complex prototype graphs
name collisions
method-order issues
debugging difficulty
```

Use them deliberately.

---

# 68. Multiple Inheritance Is Not Built In

JavaScript class `extends` supports one superclass.

To combine multiple behaviors, use:

```text
composition
mixins
delegation
interfaces via conventions/typing
```

Do not fake multiple inheritance unless the complexity is justified.

---

# 69. Class Methods Are Not Enumerable

Class methods defined in the class body have different descriptor defaults from methods assigned using:

```js
Class.prototype.method = ...
```

Class methods are generally non-enumerable.

This can affect:

```text
Object.keys
for...in
reflection
serialization assumptions
```

This distinction matters when migrating from constructor/prototype assignment style to class syntax.

---

# 70. Class Fields Are Own Properties

Public instance fields create own properties on instances.

Compare:

```js
class User {
  greet() {}
  active = true;
}
```

Then:

```text
greet
→ inherited through User.prototype

active
→ own property on instance
```

This difference affects:

```text
enumeration
ownership
memory
shadowing
property descriptors
```

---

# 71. Field Enumerability

Public fields are ordinary public properties with their own descriptor behavior.

This means they can participate in property enumeration unlike class methods, subject to their descriptor attributes and later mutation.

Therefore:

```text
class method
vs
public field
```

is not merely a syntax difference.

---

# 72. Private Fields Are Not Enumerable

Private state does not participate in ordinary property reflection.

This provides a stronger encapsulation boundary:

```text
public own properties
vs
private class elements
```

A debugger may display private state separately, but this is tooling behavior, not ordinary property enumeration.

---

# 73. Class Initialization Ordering

When evaluating a class, several categories can participate:

```text
heritage evaluation
constructor definition
instance fields
private names
static fields
static methods
static blocks
```

The language specifies ordering.

Do not make assumptions based solely on visual source order without considering whether an element is:

```text
instance
static
private
computed
```

---

# 74. Computed Class Names

Class elements can use computed keys:

```js
const methodName = "greet";

class User {
  [methodName]() {
    return "hello";
  }
}
```

The computed expression is evaluated during class definition according to class element evaluation rules.

This connects class semantics to object property-key evaluation.

---

# 75. Computed Fields

Example:

```js
const field = "name";

class User {
  [field] = "A";
}
```

The resulting public field is a property whose key is computed.

This makes class bodies executable and dynamic.

---

# 76. Class Static Initialization Errors

Static field initializers and static blocks execute during class evaluation.

Therefore:

```js
class Config {
  static value = riskyOperation();
}
```

can throw while the class definition is being evaluated.

A failed class evaluation can prevent the surrounding module/script from progressing normally.

Do not treat class definitions as passive declarations.

---

# 77. Private Static Initialization

Private static fields and methods are established as part of class definition processing.

Therefore a static block can access:

```js
class Registry {
  static #map = new Map();

  static {
    Registry.#map.set("x", 1);
  }
}
```

The private static element is available according to class-element initialization order.

---

# 78. Class and Module Architecture

Classes work especially well inside ES modules:

```text
module
├── exported class
├── private implementation
├── dependencies
└── helper functions
```

The module provides a file-level boundary.

The class provides an instance/object-level boundary.

Closures can provide smaller internal boundaries.

A robust architecture often combines these layers.

---

# 79. Class and Closure Encapsulation

A practical hierarchy:

```text
module boundary
    ↓
class boundary
    ↓
private field/method
    ↓
closure-local state
```

Do not force every concern into a class.

Use the narrowest abstraction that expresses ownership clearly.

---

# 80. Memory Considerations

Prototype methods are typically shared.

Public instance fields are per-instance.

Private instance fields are per-instance state.

Instance arrow functions are per-instance function values.

Therefore for large populations, compare:

```text
prototype method
vs
bound method
vs
instance arrow
```

with:

```text
instance field count
private fields
closure captures
```

Do not assume syntax alone predicts exact memory use.

---

# 81. Performance Considerations

Class syntax is not inherently faster or slower than factories.

Relevant performance factors include:

```text
object shape stability
field initialization order
method sharing
allocation rate
inheritance depth
private-field access
closure creation
call-site optimization
instance count
```

Modern engines optimize many class patterns effectively.

Measure the actual workload.

---

# 82. Hidden Classes / Shapes

A class with stable field initialization such as:

```js
class User {
  constructor(name) {
    this.name = name;
    this.active = true;
  }
}
```

can provide consistent object layouts across instances.

If initialization is highly conditional:

```js
class User {
  constructor(name, admin) {
    if (admin) {
      this.role = "admin";
    }

    this.name = name;
  }
}
```

different instances can have different property layouts.

This is engine-specific performance behavior, not an ECMAScript requirement.

---

# 83. Security Considerations

Classes improve encapsulation when private fields are appropriate.

But:

```text
private ≠ authorized
```

A private field can prevent accidental external access.

It does not automatically enforce:

```text
who may invoke public methods
what input is valid
whether an operation is authorized
```

Security still requires explicit trust-boundary design.

---

# 84. Prototype Pollution and Classes

Class instances still participate in object/prototype semantics.

Class usage does not automatically prevent:

```text
prototype pollution
unsafe dynamic property writes
inherited property confusion
```

Be careful when classes expose generic object mutation APIs.

---

# 85. Private Fields and Security

Private fields can reduce accidental disclosure and property collisions.

Example:

```js
class AuthContext {
  #token;

  constructor(token) {
    this.#token = token;
  }

  isValid() {
    return Boolean(this.#token);
  }
}
```

The private field is not a normal enumerable property.

Still:

```text
token secret handling
logging
memory lifetime
authorization
```

must be designed separately.

---

# 86. Memory Retention Through Class Instances

A class instance can retain:

```text
large fields
closures
listeners
services
request state
caches
```

For example:

```js
class Controller {
  data = hugeObject;

  handler = () => this.process();
}
```

If:

```text
handler
```

remains registered somewhere long-lived, the instance may remain reachable through the closure.

Class syntax does not remove closure-lifetime concerns.

---

# 87. Reentrancy in Class Methods

Example:

```js
class Store {
  update() {
    this.state = "updating";
    this.notify();
    this.state = "ready";
  }
}
```

If `notify()` invokes external code synchronously, that code can call:

```js
store.update();
```

again.

Classes do not protect against reentrancy.

The same execution-model rules from Chapters 12–14 apply.

---

# 88. Debugging Constructor Order

When constructor state is wrong, inspect:

```text
base or derived?
super() called?
field initializer order?
constructor assignment order?
static vs instance?
private field initialized?
```

Do not debug only by reading the final object state.

Trace construction.

---

# 89. Debugging `super`

When:

```js
super.method()
```

behaves unexpectedly:

```text
1. Identify current class.
2. Identify superclass.
3. Identify method's home-object context.
4. Determine super property lookup target.
5. Determine current receiver (`this`).
6. Determine whether method was overridden further down.
```

This prevents the common error:

```text
super = this.parent
```

which is not an accurate model.

---

# 90. Debugging Private Fields

When you see a private-field error:

```text
1. Identify the class that declared the private name.
2. Verify the object carries the relevant private brand.
3. Check whether the method was called with the expected receiver.
4. Check cross-object / detached-method behavior.
5. Check inheritance assumptions.
```

Private access is not ordinary property lookup.

---

# 91. Debugging Field Initializers

Given:

```js
class User {
  id = createId();
  label = `user-${this.id}`;
}
```

trace:

```text
create instance
→ initialize id
→ initialize label
→ constructor
```

Then test the reversed declaration order.

Initialization order is observable.

---

# 92. Code Review Exercise — Deep Inheritance

Review:

```js
class A {}
class B extends A {}
class C extends B {}
class D extends C {}
class E extends D {}
class F extends E {}
```

Questions:

```text
Is each inheritance edge semantically justified?
What contract does each layer provide?
Can composition flatten the hierarchy?
How difficult is debugging method lookup?
What happens when A changes?
```

Deep inheritance should have a strong reason.

---

# 93. Code Review Exercise — Stateful Base Class

Review:

```js
class Base {
  state = {};

  reset() {
    this.state = {};
  }
}

class Child extends Base {
  save() {
    return this.state;
  }
}
```

Questions:

```text
Who owns state?
Can subclasses safely mutate it?
Is replacement expected?
Would private fields improve invariants?
Should state be composed instead?
```

Base-class mutable state is an API contract.

---

# 94. Code Review Exercise — Constructor Logic

Review:

```js
class User {
  constructor(data) {
    this.data = data;

    if (data.admin) {
      this.role = "admin";
    }

    if (data.active) {
      this.active = true;
    }
  }
}
```

Potential concerns:

```text
inconsistent instance shapes
implicit defaults
input validation mixed with initialization
conditional field existence
```

A clearer design may define stable state:

```js
class User {
  constructor(data) {
    this.data = data;
    this.role = data.admin ? "admin" : "user";
    this.active = Boolean(data.active);
  }
}
```

The correct design depends on domain requirements.

---

# 95. Code Review Exercise — Private Field

Review:

```js
class Account {
  #balance = 0;

  get balance() {
    return this.#balance;
  }

  deposit(amount) {
    this.#balance += amount;
  }
}
```

Questions:

```text
Is negative amount allowed?
Can overflow occur?
Who is authorized to call deposit?
Should returned balance be a copy or primitive?
Does the API maintain invariants?
```

Private fields hide representation but do not automatically enforce business rules.

---

# 96. Code Review Exercise — Class vs Factory

Compare:

```js
class UserService {
  constructor(repo) {
    this.repo = repo;
  }

  find(id) {
    return this.repo.find(id);
  }
}
```

and:

```js
function createUserService(repo) {
  return {
    find(id) {
      return repo.find(id);
    }
  };
}
```

Evaluate:

```text
identity
inheritance
method sharing
testing
composition
serialization
lifecycle
team conventions
```

Either can be correct.

---

# 97. Implementation From Scratch

Build a simplified class system on top of the prototype model.

Required features:

```text
class definition
constructor
prototype methods
static methods
new
extends
super
```

Do not implement private fields initially.

---

# 98. Constructor-Based Class Simulation

Start with:

```js
function defineClass(constructor, methods = {}) {
  Object.assign(constructor.prototype, methods);
  return constructor;
}
```

Example:

```js
const User = defineClass(
  function User(name) {
    this.name = name;
  },
  {
    greet() {
      return this.name;
    }
  }
);

const user = new User("A");
```

This approximates only the basic prototype-sharing model.

---

# 99. `extends` Simulation

Build:

```js
function extend(Child, Parent) {
  Object.setPrototypeOf(Child.prototype, Parent.prototype);
  Object.setPrototypeOf(Child, Parent);

  return Child;
}
```

Then:

```js
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return this.name;
};

function Admin(name) {
  User.call(this, name);
}

extend(Admin, User);
```

This approximates the two-chain inheritance idea.

It is not a full `class extends` implementation.

---

# 100. Static Method Simulation

Add:

```js
function defineStatic(Ctor, methods) {
  Object.assign(Ctor, methods);
  return Ctor;
}
```

Then:

```js
defineStatic(User, {
  create(name) {
    return new User(name);
  }
});
```

This demonstrates:

```text
static side
vs
prototype side
```

---

# 101. Private State Simulator

Use closures:

```js
function createPrivateUser(name) {
  let privateName = name;

  return {
    getName() {
      return privateName;
    }
  };
}
```

Then compare with:

```js
class User {
  #name;

  constructor(name) {
    this.#name = name;
  }

  getName() {
    return this.#name;
  }
}
```

Document:

```text
identity
inheritance
reflection
memory
serialization
API ergonomics
```

---

# 102. Guided Implementation

Implement:

```text
defineClass
defineMethod
defineStatic
extend
construct
superCall
privateClosureState
```

Then create:

```text
User
Admin
```

and test:

```text
instance methods
static methods
inheritance
overrides
constructor arguments
```

---

# 103. Partially Guided Implementation

Add:

```text
field initialization
method overriding
super method calls
private-state simulation
bound callback methods
```

Then log:

```text
constructor
field-init
method-call
super-call
return
```

---

# 104. No-Reference Implementation

Build a mini class runtime from scratch.

Support:

```text
base class
derived class
constructor
prototype methods
static methods
new
extends
super
```

Then compare with native class behavior.

---

# 105. Edge-Case Hardening

Add:

- default constructors;
- derived constructors;
- missing `super`;
- field initialization order;
- private-state checks;
- static initialization;
- method extraction;
- bound methods;
- inherited accessors;
- overridden methods;
- built-in subclassing considerations.

---

# 106. Production-Grade Exercise

Design a small domain model:

```text
Entity
User
Admin
```

Then design the same model using:

```text
composition
factories
closures
```

Compare:

```text
behavior sharing
state ownership
testability
serialization
inheritance
memory
extensibility
API stability
```

Write a decision memo choosing one approach.

---

# 107. Interview Questions

## Beginner

1. What is a JavaScript class?
2. Is JavaScript class-based or prototype-based?
3. Where do class methods live?
4. What does `new` do?
5. What is the constructor?
6. What does `extends` do?
7. What does `super()` do?

## Intermediate

8. Why can `this` not be used before `super()` in a derived constructor?
9. What is the difference between static and instance methods?
10. What are public class fields?
11. What are private fields?
12. Why are class methods not automatically bound?
13. Why does extracting a class method lose `this`?
14. What is the difference between `#private` and `_private`?
15. Why are private fields not visible through normal reflection?

## Advanced

16. Explain the two prototype chains created by class inheritance.
17. How do class fields initialize?
18. How do static initialization blocks work?
19. How do private brands differ from ordinary property lookup?
20. How do classes interact with closures?
21. What are mixins?
22. What are the trade-offs between prototype methods and instance arrows?
23. What happens when a constructor returns an object?
24. What issues arise when subclassing built-ins?

## Principal

25. When should inheritance be replaced by composition?
26. How would you design a stable base-class contract?
27. How would you evaluate class vs factory for a library API?
28. How would you design millions of instances with memory efficiency?
29. When are private fields useful versus closure encapsulation?
30. How would you audit a deep inheritance hierarchy for architectural risk?
31. Which class/performance claims are ECMAScript guarantees versus engine implementation details?
32. How would you design subclassing hooks without creating fragile inheritance coupling?

---

# 108. Predict-the-Output Exercises

Predict before running.

## Exercise A

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return this.name;
  }
}

const user = new User("A");

console.log(user.greet());
```

## Exercise B

```js
class User {
  greet() {
    return this.name;
  }
}

const user = new User();
const greet = user.greet;

console.log(greet());
```

## Exercise C

```js
class Parent {
  static value = 1;
}

class Child extends Parent {}

console.log(Child.value);
```

## Exercise D

```js
class Parent {
  greet() {
    return "parent";
  }
}

class Child extends Parent {
  greet() {
    return super.greet() + " child";
  }
}

console.log(new Child().greet());
```

## Exercise E

```js
class Parent {
  value = 1;
}

class Child extends Parent {
  value = 2;
}

console.log(new Child().value);
```

## Exercise F

```js
class User {
  #id = 42;

  getId() {
    return this.#id;
  }
}

const user = new User();

console.log(user.getId());
console.log(Object.keys(user));
```

## Exercise G

```js
class User {
  first = 1;
  second = this.first + 1;
}

console.log(new User().second);
```

## Exercise H

```js
class User {
  second = this.first + 1;
  first = 1;
}

console.log(new User().second);
```

## Exercise I

```js
class Parent {
  constructor() {
    this.value = 1;
  }
}

class Child extends Parent {
  constructor() {
    super();
    this.value = 2;
  }
}

console.log(new Child().value);
```

## Exercise J

```js
class User {
  static {
    this.ready = true;
  }
}

console.log(User.ready);
```

---

# 109. Mastery Exercises

## Level 1 — Understand

Define:

```text
class
constructor
instance
static member
prototype method
public field
private field
derived class
super
private brand
```

---

## Level 2 — Explain

Explain why:

```text
class syntax
```

does not eliminate:

```text
prototype chains
```

---

## Level 3 — Predict

Predict:

```text
constructor order
field initialization
super
this
private fields
static initialization
method lookup
```

---

## Level 4 — Implement

Build the simplified class runtime.

---

## Level 5 — Debug

Diagnose:

```text
missing super
detached class method
wrong field initialization order
private brand error
unexpected static inheritance
shared state on prototype
```

---

## Level 6 — Defend

Defend:

> A class is a structured abstraction over JavaScript's object/prototype model, not a replacement for that model.

---

# 110. Principal-Level Reasoning Problems

## Problem 1 — Inheritance pressure

A team has:

```text
BaseService
AuthenticatedService
CachedAuthenticatedService
AuditedCachedAuthenticatedService
RegionAwareAuditedCachedAuthenticatedService
```

Evaluate the hierarchy.

Ask:

```text
Which behaviors are orthogonal?
Which are true subtype relationships?
Could composition replace inheritance?
What happens when another independent capability is introduced?
```

---

## Problem 2 — Private fields at scale

A service creates 10 million short-lived objects.

Compare:

```js
class Item {
  #id;
  process() {}
}
```

with closure factories and public/private-by-convention alternatives.

Evaluate:

```text
memory
allocation
lookup
debugging
encapsulation
engine behavior
```

Do not guess exact performance without measurement.

---

## Problem 3 — Base-class contract

Design a base class:

```js
class Repository {}
```

that supports:

```text
save
find
delete
transaction
```

Define:

```text
invariants
error semantics
async contract
subclass expectations
extension points
```

Then decide whether a base class is actually better than an interface-oriented composition design.

---

# 111. Production Class Design Checklist

Before introducing a class, ask:

```text
1. Is there meaningful instance identity?
2. Is there a lifecycle?
3. Is state owned by instances?
4. Is method sharing valuable?
5. Is inheritance genuinely needed?
6. Could a factory be simpler?
7. Could composition be clearer?
8. Is private state required?
9. Are callbacks extracted?
10. Are methods receiver-dependent?
11. Are constructor invariants explicit?
12. Are field initialization dependencies simple?
13. Are static members justified?
14. Are subclassing contracts documented?
15. Does the class need to survive across module boundaries?
```

---

# 112. Class vs Factory Decision Table

| Requirement | Often favors |
|---|---|
| Strong instance identity | Class |
| Prototype-shared methods | Class |
| Deep inheritance/polymorphism | Class, cautiously |
| Simple dependency closure | Factory |
| Narrow private state | Closure or private fields |
| Composable capabilities | Factory/composition |
| Static constructor API | Class/factory |
| Serialization as plain data | Plain object / DTO |
| Public inheritance contract | Class |
| Minimal abstraction | Plain function/object |

This is guidance, not a universal rule.

---

# 113. Security Review Checklist

For class-based systems:

```text
1. Are private fields being mistaken for authorization?
2. Are public methods validating untrusted input?
3. Can prototype mutation alter behavior?
4. Are dynamic property writes exposed?
5. Are instances retaining secrets longer than needed?
6. Are stack traces/logs leaking internals?
7. Are subclasses trusted?
8. Does deserialization reconstruct trusted class instances safely?
```

Private syntax protects representation.

Security architecture must still protect authority.

---

# 114. Performance Review Checklist

Measure:

```text
instance creation rate
instance count
field initialization
function allocation
prototype sharing
private-field access
inheritance depth
callback binding
GC pressure
hot call sites
```

Avoid:

```text
class syntax = slow
private fields = slow
inheritance = slow
factories = fast
```

These are not valid universal conclusions.

---

# 115. Memory Review Checklist

Identify:

```text
per-instance fields
private fields
bound methods
instance arrows
captured closures
listeners
cached services
prototype references
static caches
```

Then ask:

```text
What is shared?
What is per-instance?
What is process-long-lived?
What is request-scoped?
```

This exposes hidden retention.

---

# 116. Specification-Oriented Vocabulary

Important concepts:

```text
Class Definition Evaluation
ClassDeclaration
ClassExpression
ConstructorMethod
MethodDefinition
ClassFieldDefinition
PrivateName
PrivateElement
[[Prototype]]
[[Construct]]
[[Call]]
Home Object
SuperProperty
InitializeInstanceElements
InitializeStaticElements
```

These are specification concepts and algorithm vocabulary.

Do not reduce them to:

```text
"classes are syntactic sugar"
```

because modern class features include semantics that are not captured by a simplistic constructor/prototype rewrite.

---

# 117. Is `class` Just Syntactic Sugar?

The answer is:

```text
partly, but not completely.
```

At a high level, class instance methods map naturally onto prototype behavior.

But class syntax also carries specialized semantics for:

```text
strict mode
constructors
derived construction
private fields
private methods
class fields
static fields
static blocks
super
class element initialization
```

Therefore:

> **“Class is just syntactic sugar for constructor functions” is an incomplete statement.**

---

# 118. Engine Considerations

Engines can optimize class-heavy code using the same underlying runtime strategies used for objects/functions:

```text
hidden classes
shapes
inline caches
optimized code
deoptimization
specialized field layouts
```

Private fields may have specialized implementation strategies.

These details are engine-specific.

Never turn:

```text
V8 implementation detail
```

into:

```text
ECMAScript guarantee
```

---

# 119. Built-in Subclassing and Exotic Objects

Some built-ins have special internal behavior.

Examples:

```text
Array
Map
Set
TypedArray
Promise
Error
```

Subclassing can involve:

```text
internal slots
species
constructor propagation
brand checks
specialized methods
```

This is why Chapter 21 exists as a separate topic.

---

# 120. Debugging Workflow

When a class behaves unexpectedly:

```text
1. Is the class base or derived?
2. Where was the object created?
3. What is its prototype?
4. Which method was found?
5. What is the current `this`?
6. Was the method extracted?
7. Was the method bound?
8. What fields initialize before the constructor body?
9. Was `super()` called?
10. Which class declared the private element?
11. Is the member static or instance?
12. Could inherited state be shared?
```

---

# 121. Completion Criteria

### Understand

You can define:

- class;
- constructor;
- instance;
- prototype method;
- static member;
- class field;
- private field;
- derived class;
- `super`;
- private brand.

### Explain

You can explain:

- class declarations/expressions;
- constructors;
- `extends`;
- two prototype chains;
- `super`;
- `this` initialization;
- public fields;
- private fields/methods;
- static members;
- static blocks;
- class-method descriptors;
- class/prototype relationship;
- inheritance vs composition.

### Predict

You can correctly predict:

- construction order;
- `super`;
- field initialization;
- private-field access;
- static initialization;
- method lookup;
- detached methods;
- inherited static behavior.

### Implement

You can build:

```text
constructor
prototype methods
static methods
extends
super-like behavior
field initialization
closure-based private state
```

### Debug

You can diagnose:

```text
missing super
wrong receiver
method extraction
field-order bug
private brand error
shared prototype state
static/instance confusion
```

### Principal Judgment

You can choose between:

```text
class inheritance
composition
factory
closure
plain object
```

using:

```text
identity
lifecycle
encapsulation
memory
performance
testability
extensibility
security
API stability
```

**Evidence of mastery:**

- 90%+ prediction accuracy;
- working class/prototype simulator;
- correct two-chain inheritance explanation;
- correct field/constructor order reasoning;
- correct private-field model;
- defensible inheritance-vs-composition decision.

---

# 122. Key Takeaways

1. **JavaScript classes are structured object-model constructs built on prototype semantics.**
2. **Class syntax does not remove prototype chains.**
3. **Instance methods are normally shared through the class prototype.**
4. **Static members live on the class/constructor side.**
5. **Class inheritance creates both instance and constructor/static prototype relationships.**
6. **Derived constructors require superclass initialization before `this` can be used normally.**
7. **`super()` and `super.method()` are not aliases for a generic parent object.**
8. **`this` remains the current receiver when inherited methods are invoked through an instance.**
9. **Detaching a class method can lose its receiver.**
10. **Classes do not automatically bind instance methods.**
11. **Instance arrow functions can preserve lexical `this`, but they are per-instance functions.**
12. **Public instance fields become own properties.**
13. **Class methods are generally prototype methods rather than own instance properties.**
14. **Field initializers are executable and their ordering is observable.**
15. **Static fields and static blocks execute on the class-definition/static side.**
16. **Private fields and methods are language-enforced private elements, not ordinary properties.**
17. **Private fields use class-specific branding rather than ordinary string-property lookup.**
18. **Private fields differ fundamentally from `_private` naming conventions and symbol properties.**
19. **Classes can combine with closures; they are not competing abstractions in every design.**
20. **Inheritance should encode a meaningful subtype relationship, not merely code reuse.**
21. **Composition often scales better when behaviors are orthogonal.**
22. **Prototype-shared methods can be memory-efficient for large instance populations.**
23. **Per-instance arrows and bound methods trade memory/identity for callback convenience.**
24. **Classes do not automatically solve security; private representation is not authorization.**
25. **Class instances can still participate in prototype pollution, aliasing, reentrancy, and memory-retention problems.**
26. **Built-in subclassing can involve specialized internal behavior beyond ordinary user-defined classes.**
27. **“Classes are just syntactic sugar” is incomplete because modern class features have additional semantics.**
28. **Engine optimizations around classes are implementation details, not ECMAScript guarantees.**
29. **Principal-level class design is about lifecycle, ownership, abstraction, invariants, and change cost.**

---

# 123. Final Mastery Drill

For each example, answer:

```text
1. Base or derived?
2. Constructor involved?
3. Instance or static?
4. What prototype is used?
5. Where is the method stored?
6. What is this?
7. When are fields initialized?
8. Which fields are public/private?
9. What private brand is involved?
10. What happens if the method is detached?
11. What state is shared?
12. What state is per-instance?
13. What happens through inheritance?
14. What production trade-off follows?
```

### Drill 1

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return this.name;
  }
}
```

### Drill 2

```js
class Admin extends User {
  constructor(name, level) {
    super(name);
    this.level = level;
  }

  greet() {
    return super.greet() + " admin";
  }
}
```

### Drill 3

```js
class Registry {
  static items = new Map();

  static {
    Registry.items.set("default", true);
  }
}
```

### Drill 4

```js
class Account {
  #balance = 0;

  deposit(amount) {
    this.#balance += amount;
  }

  getBalance() {
    return this.#balance;
  }
}
```

### Drill 5

```js
class Controller {
  handle = () => this.run();
}
```

### Drill 6

```js
class Example {
  first = 1;
  second = this.first + 1;
}
```

### Drill 7

```js
class Example {
  second = this.first + 1;
  first = 1;
}
```

### Drill 8

```js
class A {
  static value = 1;
}

class B extends A {}
```

### Drill 9

```js
class Base {
  set value(v) {
    this._value = v;
  }
}

class Child extends Base {
  update(v) {
    super.value = v;
  }
}
```

### Drill 10

```js
class User {
  #id = 42;

  getId() {
    return this.#id;
  }
}

const getId = User.prototype.getId;
```

Explain what happens when `getId` is invoked with:

```text
correct instance
wrong object
undefined receiver
```

---

# 124. Transition to Chapter 19

Chapter 18 established:

```text
classes
constructors
instance methods
static methods
inheritance
super
private fields
class fields
object-oriented design
```

Chapter 19 will move from normal object semantics to programmable object behavior:

```text
What is Proxy?
What is Reflect?
How can property access be intercepted?
What does `get` really intercept?
What does `set` intercept?
How do ownKeys traps work?
What invariants must proxies preserve?
How do proxies affect `this`, receivers, prototypes, and descriptors?
Why can metaprogramming break ordinary optimization assumptions?
```

The next progression is:

```text
ordinary object
   ↓
internal methods
   ↓
interception
   ↓
Proxy
   ↓
Reflect
   ↓
metaprogramming
```

This will connect the object model to frameworks, reactivity systems, validation layers, membranes, virtualization, and advanced runtime abstractions.

---

# 125. Chapter Completion Record

**Chapter:** 18 — Classes and Object-Oriented JavaScript

**Status:** `[+] Completed`

**Strong Areas Expected:**

- class declarations/expressions;
- constructors;
- prototype methods;
- static members;
- `extends`;
- `super`;
- derived `this` initialization;
- public fields;
- private fields/methods;
- static blocks;
- class inheritance;
- closure-vs-private-state;
- composition-vs-inheritance.

**Revision Triggers:**

- saying classes replace prototypes;
- saying classes are only syntax sugar;
- confusing `.prototype` with `[[Prototype]]`;
- forgetting the two inheritance chains;
- using `this` before `super()`;
- assuming class methods are bound;
- confusing static and instance members;
- treating public fields like prototype methods;
- treating private fields as ordinary properties;
- assuming private state provides authorization;
- storing mutable shared state on prototypes.

**Evidence of Mastery:**

- 90%+ prediction accuracy;
- working simplified class runtime;
- correct `super` and constructor reasoning;
- correct field-initialization tracing;
- correct private-brand explanation;
- correct static/instance distinction;
- defensible choice between class, factory, closure, and composition.
---

