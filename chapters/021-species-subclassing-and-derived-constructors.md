
# Chapter 21 — Species / Subclassing / Derived Constructors

## Chapter Status

`[~] In Progress`

**Part:** III — Objects  
**Primary theme:** Constructor inheritance, derived classes, `super()`, derived initialization, built-in subclassing, and `Symbol.species`.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what subclassing means in JavaScript.
2. Explain the relationship between:
   - a base class,
   - a derived class,
   - constructor functions,
   - prototypes,
   - `extends`,
   - `super`.
3. Explain why a derived constructor behaves differently from a base constructor.
4. Explain why `this` is not available in a derived constructor before `super()`.
5. Explain the internal meaning of a derived constructor's pending `this` state.
6. Explain how `super()` initializes the derived instance.
7. Explain the role of the base constructor in derived-instance creation.
8. Explain why omitting a derived constructor causes JavaScript to synthesize constructor behavior.
9. Explain `super.method()` and why it is not equivalent to `this.method()`.
10. Explain the dual prototype relationships created by class inheritance.
11. Explain static inheritance separately from instance inheritance.
12. Explain why `extends` affects both constructor inheritance and prototype relationships.
13. Distinguish:
    - subclassing,
    - composition,
    - delegation,
    - mixins.
14. Explain subclassing of built-ins such as:
    - `Array`,
    - `Map`,
    - `Set`,
    - `Error`,
    - typed-array families.
15. Explain `Symbol.species`.
16. Explain why some built-in methods construct result objects rather than always returning the same concrete class.
17. Explain the conceptual difference between:
    - "what class is this object?"
    - "what constructor should create the result?"
18. Explain `this.constructor` and why it is not always a reliable substitute for species semantics.
19. Understand how subclassing interacts with:
    - `new`,
    - `super`,
    - private fields,
    - public fields,
    - static members,
    - custom constructors,
    - built-in methods.
20. Implement custom subclasses of built-ins.
21. Implement and reason about `Symbol.species`.
22. Identify surprising or dangerous subclass behavior in production code.
23. Explain why inheritance hierarchies can create semantic coupling.
24. Reason about when subclassing is appropriate and when composition is superior.
25. Debug initialization-order failures in derived constructors.
26. Predict constructor/prototype behavior without executing the code.
27. Evaluate subclassing choices at principal-engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 09 — Functions / First-Class Behavior
- Chapter 10 — Scope / Lexical Environments / Identifier Resolution
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 14 — `this` / Invocation / Binding
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 18 — Classes / OOP
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols

The learner should already understand:

```text
constructor functions
prototype chains
class syntax
this
new
private fields
static members
Symbol-based protocols
```

---

## 3. What Is It?

A **subclass** is a class whose construction and/or behavior is derived from another constructor through:

```js
class Child extends Parent {
}
```

The base class is:

```text
Parent
```

The derived class is:

```text
Child
```

Example:

```js
class Animal {
  speak() {
    return "sound";
  }
}

class Dog extends Animal {
  bark() {
    return "woof";
  }
}
```

A `Dog` instance can access both:

```js
const dog = new Dog();

dog.bark();
dog.speak();
```

because the instance prototype chain includes `Dog.prototype` and then `Animal.prototype`.

### But subclassing is more than method lookup

JavaScript class inheritance also affects construction.

This is why:

```js
class Child extends Parent {
  constructor() {
    super();
  }
}
```

has semantics that cannot be reduced to:

```js
Object.setPrototypeOf(Child.prototype, Parent.prototype);
```

and nothing more.

Subclassing affects:

- constructor behavior,
- `new`,
- `super`,
- instance initialization,
- static inheritance,
- built-in construction,
- result creation in selected built-in methods.

---

## 4. Why Does It Exist?

Inheritance exists partly to express:

> "This abstraction is a specialized form of another abstraction."

For example:

```text
Animal
   ↓
Dog
```

The derived type can reuse or specialize behavior.

JavaScript also needs a mechanism for built-in subclassing.

For example:

```js
class MyArray extends Array {}
```

This allows domain-specific collection abstractions while preserving much of the built-in array behavior.

However, inheritance comes with coupling:

```text
derived behavior
      ↓
base constructor
      ↓
base prototype semantics
      ↓
built-in assumptions
```

That coupling is especially visible in constructor and species semantics.

### Species exists for a different problem

Suppose:

```js
class MyArray extends Array {}
```

and a method creates a new array-like result.

Should that result be:

```text
Array
```

or:

```text
MyArray
```

JavaScript defines species-related semantics so certain built-ins can determine the constructor used for derived result objects.

This separates:

```text
the receiver's class
```

from:

```text
the constructor intended for a derived result
```

---

## 5. Mental Model

Use this picture.

```text
                 Parent constructor
                        ↑
                        |
                     super()
                        |
                Child constructor
                        |
                     instance
                        |
                Child.prototype
                        |
                Parent.prototype
                        |
                Object.prototype
                        |
                       null
```

There are actually two important inheritance relationships.

### Instance side

```js
Object.getPrototypeOf(Child.prototype) === Parent.prototype
```

### Constructor/static side

Conceptually:

```js
Object.getPrototypeOf(Child) === Parent
```

This allows:

```js
Child.someStaticMethod()
```

to inherit static behavior from `Parent`.

### Derived constructor mental model

A derived constructor begins in a state where it does not yet have an initialized `this` binding.

Then:

```js
super(...)
```

delegates construction to the base constructor and establishes the actual instance.

Think:

```text
derived constructor begins
        ↓
this not initialized
        ↓
evaluate super(...)
        ↓
base construction
        ↓
actual receiver/instance established
        ↓
derived initialization continues
```

---

## 6. Core Rules

### Rule 1 — A derived class is declared with `extends`

```js
class Child extends Parent {}
```

---

### Rule 2 — A derived constructor cannot use `this` before `super()`

Invalid:

```js
class Child extends Parent {
  constructor() {
    this.value = 1;
    super();
  }
}
```

This throws because derived construction has not initialized `this`.

---

### Rule 3 — `super()` must happen before returning a normal derived instance

A derived constructor normally must either:

- initialize `this` through `super()`, or
- explicitly return an appropriate object.

The usual pattern is:

```js
class Child extends Parent {
  constructor() {
    super();
  }
}
```

---

### Rule 4 — If a derived constructor returns an object explicitly, `super()` is not necessarily required

Example:

```js
class Base {}

class Child extends Base {
  constructor() {
    return {};
  }
}
```

This demonstrates an important specification detail:

> Constructor completion and instance initialization are related but not identical.

Do not oversimplify the rule to "derived constructors must always call super."

In ordinary object construction, returning a valid object can determine the result.

---

### Rule 5 — `super()` invokes the base constructor with derived construction context

Conceptually:

```js
super(value);
```

does not mean:

```js
Parent.call(this, value);
```

because `this` may not yet exist as an initialized receiver in a derived constructor.

---

### Rule 6 — `super.method()` is different from `this.method()`

```js
super.speak();
```

starts lookup from the base class's prototype.

```js
this.speak();
```

starts ordinary property lookup from the actual receiver.

---

### Rule 7 — Instance inheritance and static inheritance are both established by `extends`

```js
class Parent {
  static hello() {
    return "hello";
  }
}

class Child extends Parent {}

Child.hello(); // "hello"
```

---

### Rule 8 — Derived constructors have different initialization rules

A base constructor can use `this` immediately.

A derived constructor cannot.

---

### Rule 9 — Public field initialization has an order

Field initialization is part of class construction semantics and interacts with base/derived constructors.

Do not reason about:

```js
field = expression;
```

as though it were always equivalent to the first line of the constructor body.

---

### Rule 10 — Private fields require the correct class initialization

Derived classes do not automatically get access to private fields declared by a base class.

```js
class Base {
  #x = 1;
}

class Child extends Base {}
```

`Child` cannot directly access:

```js
this.#x
```

because the private name is branded to the declaring class.

---

### Rule 11 — `Symbol.species` affects selected derived built-in result construction

It does not globally redefine:

```text
"What class is this object?"
```

It is a constructor-selection protocol for operations that explicitly use species semantics.

---

### Rule 12 — Not every method uses species

Do not assume every built-in method does.

The correct question is:

> Does the relevant specification algorithm perform species-based construction?

---

## 7. Syntax

### Basic inheritance

```js
class Child extends Parent {}
```

### Derived constructor

```js
class Child extends Parent {
  constructor(...args) {
    super(...args);
  }
}
```

### Overriding instance method

```js
class Child extends Parent {
  method() {
    return "child";
  }
}
```

### Calling base implementation

```js
super.method();
```

### Static inheritance

```js
class Child extends Parent {
  static create() {
    return new this();
  }
}
```

### Species override

```js
class SubArray extends Array {
  static get [Symbol.species]() {
    return Array;
  }
}
```

---

## 8. Basic Examples

### Example 1 — Simple subclass

```js
class Animal {
  speak() {
    return "sound";
  }
}

class Dog extends Animal {
  bark() {
    return "woof";
  }
}

const dog = new Dog();

console.log(dog.speak());
console.log(dog.bark());
```

Output:

```text
sound
woof
```

---

### Example 2 — Derived constructor

```js
class Person {
  constructor(name) {
    this.name = name;
  }
}

class Employee extends Person {
  constructor(name, role) {
    super(name);
    this.role = role;
  }
}

const employee = new Employee("Asha", "Engineer");

console.log(employee.name);
console.log(employee.role);
```

---

### Example 3 — `super.method()`

```js
class Parent {
  greet() {
    return "parent";
  }
}

class Child extends Parent {
  greet() {
    return `${super.greet()} + child`;
  }
}

console.log(new Child().greet());
```

Output:

```text
parent + child
```

---

### Example 4 — Static inheritance

```js
class Parent {
  static kind() {
    return "parent";
  }
}

class Child extends Parent {}

console.log(Child.kind());
```

Output:

```text
parent
```

---

### Example 5 — Built-in subclass

```js
class Scores extends Array {
  average() {
    return this.reduce((sum, value) => sum + value, 0) / this.length;
  }
}

const scores = new Scores(10, 20, 30);

console.log(scores.average());
console.log(scores instanceof Scores);
console.log(scores instanceof Array);
```

Output:

```text
20
true
true
```

---

## 9. Execution Walkthrough

Consider:

```js
class Person {
  constructor(name) {
    this.name = name;
  }
}

class Employee extends Person {
  constructor(name, role) {
    super(name);
    this.role = role;
  }
}

const employee = new Employee("Milan", "Engineer");
```

### Step 1 — Evaluate `new Employee(...)`

The construction operation recognizes that `Employee` is a constructor.

### Step 2 — Determine constructor mode

`Employee` is a derived constructor.

Its constructor cannot simply start with an already-initialized ordinary `this`.

### Step 3 — Enter derived constructor

Parameters receive:

```text
name → "Milan"
role → "Engineer"
```

The constructor reaches:

```js
super(name);
```

### Step 4 — Evaluate `super(...)`

The base constructor is:

```js
Person
```

Construction is delegated to the base constructor using the derived construction context.

### Step 5 — Base constructor initializes instance state

Inside:

```js
this.name = name;
```

the object now has its `name` field.

### Step 6 — Return from base construction

The base construction result becomes the derived constructor's receiver.

### Step 7 — Continue derived constructor

Now:

```js
this.role = role;
```

is allowed.

### Step 8 — Complete construction

The resulting object has access to:

```text
Employee.prototype
        ↓
Person.prototype
```

So:

```js
employee instanceof Employee
employee instanceof Person
```

both produce true.

---

## 10. Internal Mechanics

### 10.1 `extends` establishes prototype relationships

For:

```js
class Child extends Parent {}
```

the instance prototype chain is structured so that:

```js
Child.prototype
      ↓
Parent.prototype
```

The constructor functions also have an inheritance relationship relevant to static lookup:

```text
Child
 ↓
Parent
```

This is why both instance and static inheritance work.

---

### 10.2 `new` and constructor kind

Constructors have an internal distinction between base and derived constructor behavior.

A base constructor has ordinary instance initialization behavior.

A derived constructor relies on construction through the base constructor unless it explicitly returns an object.

This distinction is fundamental to understanding:

```js
super()
```

---

### 10.3 `super()` is not ordinary function-call syntax

Inside a constructor:

```js
super(args)
```

is a dedicated form of syntax.

It invokes the [[Construct]] behavior associated with the base constructor.

This matters because JavaScript constructors are not necessarily ordinary callable functions in the same sense.

---

### 10.4 Derived `this` state

At the beginning of a derived constructor, `this` is conceptually uninitialized.

Operations such as:

```js
this.x
this.x = 1
```

before `super()` are errors.

This is not merely a parser restriction.

The runtime semantics enforce an initialization rule.

---

### 10.5 The base constructor can produce a different object

Derived construction does not guarantee that the resulting object is literally an object allocated by a generic "new Child" algorithm before the base constructor runs.

The base constructor participates in determining the actual object used as the receiver/result.

This becomes important for exotic built-ins and constructors that can return objects.

---

### 10.6 Constructor return behavior

For a base constructor:

```js
class Base {
  constructor() {
    return { custom: true };
  }
}
```

the explicit object can become the result.

For a derived constructor, an explicit object return can also determine the construction result, subject to derived constructor rules.

Returning a primitive is not treated as a replacement object.

This is part of why constructor completion must be learned separately from field initialization.

---

### 10.7 `super.method()` lookup

Suppose:

```js
class Parent {
  method() {
    return "parent";
  }
}

class Child extends Parent {
  method() {
    return super.method();
  }
}
```

The `super` reference does not mean:

```text
use Parent as this
```

The method is still called with the actual receiver.

Conceptually:

```text
lookup starts from Parent.prototype
receiver remains Child instance
```

So if the parent implementation uses:

```js
this.value
```

it observes the child instance.

---

### 10.8 Static `super`

Inside a static method:

```js
class Parent {
  static value() {
    return "parent";
  }
}

class Child extends Parent {
  static value() {
    return `${super.value()} + child`;
  }
}
```

`super.value()` starts lookup from the parent constructor's property inheritance context.

Again:

```text
lookup base
receiver remains Child
```

for the method invocation.

---

### 10.9 Derived field initialization

Class fields are initialized according to class construction semantics.

For a base class, instance fields are initialized as part of base construction.

For a derived class, initialization occurs after the base construction establishes the instance, with the precise placement tied to the class field initialization algorithms.

This is why:

```js
class Base {
  value = "base";
}

class Child extends Base {
  value = "child";
}
```

does not mean the child field exists before `super()`.

---

### 10.10 Private fields and brands

Consider:

```js
class Base {
  #secret = 1;

  getSecret() {
    return this.#secret;
  }
}

class Child extends Base {}
```

A `Child` instance carries the Base private-field brand because base initialization occurs during construction.

But code written inside `Child` cannot refer to:

```js
this.#secret
```

because private names are lexical and class-scoped.

This distinction is central:

```text
instance has Base private brand
```

does not imply:

```text
Child can name Base's private field
```

---

## 11. ECMAScript / Specification Semantics

### 11.1 Constructor categories

ECMAScript distinguishes base and derived constructors.

The class definition determines whether the constructor is derived.

Conceptually:

```text
base constructor
    ↓
can initialize this directly

derived constructor
    ↓
does not initialize this until super-construction
```

This distinction drives many later rules.

---

### 11.2 `[[ConstructorKind]]`

Specification-level reasoning includes constructor kind information.

A derived constructor has a `"derived"` constructor kind.

This affects construction semantics and the handling of `this`.

---

### 11.3 `[[Construct]]`

When:

```js
new Child(...)
```

is evaluated, construction ultimately proceeds through the constructor's internal `[[Construct]]` behavior.

The internal algorithm differs depending on constructor kind.

For derived constructors, base construction via `super()` becomes a central step.

---

### 11.4 `OrdinaryCreateFromConstructor`

Class and constructor initialization is connected to the abstract operation:

```text
OrdinaryCreateFromConstructor
```

which is used in many ordinary construction paths to create an object using the constructor's prototype information.

For built-in and derived construction, the exact path can differ because some constructors and built-ins have special internal behavior.

The important learning goal is:

> "new" is not merely shorthand for allocation plus calling a function.

---

### 11.5 `GetPrototypeFromConstructor`

Prototype selection is conceptually related to:

```text
GetPrototypeFromConstructor
```

This means the constructor's `prototype` property and realm-sensitive default prototype logic can affect object creation.

This becomes especially important when constructors are inherited, replaced, or cross-realm.

---

### 11.6 `super()` constructor semantics

A `super()` call in a derived constructor invokes the base constructor through its construction semantics.

The specification keeps track of the current function's home object and active function context so that:

```js
super()
```

can resolve the correct base constructor.

---

### 11.7 `SuperProperty`

For:

```js
super.method()
```

the specification conceptually performs a special super-property reference rather than ordinary property access from `this`.

The home object's prototype determines where lookup begins.

The receiver is still the actual `this`.

This distinction is crucial.

---

### 11.8 `this` binding after super construction

Derived constructor semantics use the base construction result to initialize the `this` binding for the derived constructor.

That is why this works:

```js
class Child extends Base {
  constructor() {
    super();
    this.value = 1;
  }
}
```

while this fails:

```js
class Child extends Base {
  constructor() {
    this.value = 1;
    super();
  }
}
```

---

### 11.9 Constructor return completion

A constructor's completion behavior determines the final result of construction.

Object returns can replace the ordinary result.

Primitive returns do not become constructor results in the same way.

Derived construction has additional rules because the constructor may not have initialized `this`.

---

### 11.10 `@@species`

The shorthand:

```text
@@species
```

refers to the method/value associated with:

```js
Symbol.species
```

Certain built-in algorithms consult this mechanism when creating derived objects.

A common conceptual algorithm is:

```text
SpeciesConstructor
```

The high-level path is:

```text
receiver
   ↓
determine constructor
   ↓
read constructor[Symbol.species]
   ↓
validate/select constructor
   ↓
construct result
```

The exact algorithm depends on the built-in operation.

---

### 11.11 `SpeciesConstructor`

Conceptually, species construction answers:

> Which constructor should this built-in operation use when it creates a new result from this object?

This is different from asking:

> Which constructor created the receiver?

That distinction is the reason species semantics exist.

---

## 12. Advanced Behavior

### 12.1 Synthesized derived constructor

If:

```js
class Child extends Parent {}
```

does not define its own constructor, JavaScript provides derived constructor behavior equivalent in conceptual effect to forwarding the arguments:

```js
constructor(...args) {
  super(...args);
}
```

This is a useful mental model, while remembering that the actual language specification defines the constructor rather than performing textual rewriting.

---

### 12.2 Explicit derived constructor

```js
class Child extends Parent {
  constructor(value) {
    super(value);
    this.extra = true;
  }
}
```

The derived constructor can customize initialization after base initialization.

---

### 12.3 Derived constructor returning an object

```js
class Base {}

class Child extends Base {
  constructor() {
    return { replaced: true };
  }
}

const value = new Child();

console.log(value.replaced); // true
```

This illustrates why the "must call super" statement is an incomplete teaching shortcut.

The real rule is more precise.

---

### 12.4 Returning a primitive from a derived constructor

```js
class Base {}

class Child extends Base {
  constructor() {
    return 10;
  }
}
```

This does not let the primitive become the result of `new Child()`.

Construction rules reject the primitive-return path.

The exact thrown error and language path should be understood rather than guessed.

---

### 12.5 `super()` more than once

```js
class Base {}

class Child extends Base {
  constructor() {
    super();
    super();
  }
}
```

This is invalid because the derived `this` has already been initialized.

The second `super()` cannot reinitialize the same constructor's `this` state.

---

### 12.6 Calling `super()` conditionally

```js
class Child extends Base {
  constructor(flag) {
    if (flag) {
      super();
    }

    this.ready = true;
  }
}
```

If `flag` is false, `this` remains uninitialized and the constructor fails.

Derived constructors must ensure valid completion semantics.

---

### 12.7 `super()` with a custom base constructor that returns an object

```js
class Base {
  constructor() {
    return { base: true };
  }
}

class Child extends Base {
  constructor() {
    super();
    this.child = true;
  }
}
```

The interaction between base-returned objects and derived field/property initialization becomes subtle.

Do not assume that every object returned by a base constructor behaves like an ordinary newly allocated Child instance.

---

### 12.8 `super.method()` can observe child state

```js
class Base {
  describe() {
    return this.kind;
  }
}

class Child extends Base {
  constructor() {
    super();
    this.kind = "child";
  }

  describe() {
    return super.describe();
  }
}
```

The base method reads:

```js
this.kind
```

from the actual Child receiver.

Result:

```text
"child"
```

This is one of the most important differences between:

```js
super.method()
```

and calling a method on a separately created base instance.

---

### 12.9 Overridden methods invoked from constructors

Consider:

```js
class Base {
  constructor() {
    this.initialize();
  }

  initialize() {
    this.value = "base";
  }
}

class Child extends Base {
  initialize() {
    this.value = "child";
  }
}
```

During:

```js
new Child()
```

the Base constructor can dynamically dispatch `this.initialize()` to the Child override.

This creates constructor-time virtual dispatch.

It can be dangerous because derived fields may not yet be initialized as expected.

This is a major production design concern.

---

### 12.10 Built-in subclassing

```js
class MyMap extends Map {
  getSize() {
    return this.size;
  }
}
```

The object retains Map-specific internal state so that built-in operations can work correctly.

This is possible because JavaScript's built-in constructors and internal slots have defined subclassing behavior.

---

### 12.11 Built-in internal slots

Many built-ins depend on internal slots.

For example, a `Map` has internal state that user-defined ordinary objects do not have.

Calling:

```js
Map.prototype.get.call({})
```

fails because the receiver lacks the required Map internal state.

A real `Map` subclass works because construction establishes the required built-in state.

This distinction is critical:

```text
prototype inheritance
≠
having the required internal slots
```

---

### 12.12 Why built-in subclassing can be surprising

A custom subclass may appear to inherit methods through the prototype chain, but the inherited method can require:

- internal slots,
- brand checks,
- species construction,
- constructor compatibility.

Therefore:

> "It has the prototype" does not mean "it is semantically a valid instance for every inherited built-in operation."

---

## 13. Edge Cases

### Edge Case 1 — Class extending `null`

JavaScript supports unusual inheritance arrangements such as:

```js
class NullProto extends null {
  constructor() {
    return Object.create(null);
  }
}
```

A class extending `null` cannot use ordinary base-constructor initialization through `super()` because there is no callable base constructor.

This is an advanced case that demonstrates why constructor semantics cannot be reduced to a simple "parent class always runs" rule.

---

### Edge Case 2 — Extending non-constructors

```js
class Child extends (() => {}) {}
```

fails because the expression does not provide the required constructor behavior.

The `extends` expression is evaluated and its result must satisfy constructor requirements.

---

### Edge Case 3 — Extending expressions

`extends` can use expressions:

```js
const Base = condition ? A : B;

class Child extends Base {}
```

This is powerful but increases runtime coupling.

---

### Edge Case 4 — Dynamically generated classes

```js
function makeChild(Base) {
  return class extends Base {};
}
```

This is legal and useful for mixin-style patterns.

It can also create many distinct class identities.

---

### Edge Case 5 — Private field brand errors

```js
class Base {
  #x = 1;
}

class Child extends Base {
  read(other) {
    return other.#x;
  }
}
```

This fails at parse/semantic binding because `#x` is not declared in Child's private name environment.

---

### Edge Case 6 — Static field initialization

Static fields and static blocks execute as part of class definition/evaluation and interact with inherited constructor state.

Do not assume static initialization is delayed until `new`.

---

### Edge Case 7 — Species returning an invalid constructor

A species getter can return a non-constructor value.

A built-in method using species semantics can then throw during result construction.

Example concept:

```js
class BrokenArray extends Array {
  static get [Symbol.species]() {
    return {};
  }
}
```

Operations that consult species may fail when they attempt construction.

---

### Edge Case 8 — Species getter side effects

Because species can be implemented as a getter:

```js
static get [Symbol.species]() {
  // side effects
  return Array;
}
```

a seemingly ordinary built-in operation may execute user code.

This matters for:

- security,
- performance,
- debugging,
- reentrancy.

---

### Edge Case 9 — `this.constructor` mutation

Some code assumes:

```js
this.constructor
```

always identifies the intended derived constructor.

That property can be:

- inherited,
- shadowed,
- changed,
- proxied.

Therefore constructor-property assumptions can be fragile.

---

### Edge Case 10 — Cross-realm subclassing

Objects and constructors from different realms complicate `instanceof`, prototypes, and intrinsic identity.

Realm-aware reasoning becomes important when subclassing crosses:

- browser windows,
- iframes,
- workers,
- VM contexts.

---

## 14. Common Misconceptions

### Misconception 1 — "extends just copies methods."

False.

It establishes inheritance relationships and affects construction semantics.

---

### Misconception 2 — "super() is basically Parent.call(this)."

False.

A derived constructor may not yet have an initialized `this`.

`super()` participates in `[[Construct]]` semantics.

---

### Misconception 3 — "super.method() changes this to a Parent instance."

False.

The method is invoked with the actual receiver.

---

### Misconception 4 — "The child class can access all private fields of the parent."

False.

Private names belong to their declaring class.

---

### Misconception 5 — "Every method on a subclass returns the subclass."

False.

Result construction depends on the particular algorithm.

---

### Misconception 6 — "Symbol.species means the object's class."

False.

Species is a constructor-selection protocol used by specific built-in operations.

---

### Misconception 7 — "this.constructor is the same thing as Symbol.species."

False.

They solve different problems and use different semantics.

---

### Misconception 8 — "Built-in subclassing is just prototype inheritance."

False.

Built-ins may require internal slots and special construction semantics.

---

### Misconception 9 — "Calling a parent method means parent behavior is isolated."

False.

`super.method()` still uses the derived object's `this`.

---

### Misconception 10 — "Inheritance automatically means better reuse."

False.

Inheritance introduces coupling and can create surprising lifecycle dependencies.

---

## 15. Common Mistakes

### Mistake 1 — Accessing `this` before `super()`

```js
class Child extends Base {
  constructor() {
    this.value = 1;
    super();
  }
}
```

Fails.

---

### Mistake 2 — Forgetting `super()` in a derived constructor

```js
class Child extends Base {
  constructor() {
    // no super()
  }
}
```

Returning normally without initializing the derived instance is invalid.

---

### Mistake 3 — Assuming the base constructor runs automatically after a custom constructor

If you define a derived constructor, constructor behavior becomes explicit.

Use:

```js
super(...)
```

when base construction is required.

---

### Mistake 4 — Calling `super()` twice

This attempts repeated derived initialization and fails.

---

### Mistake 5 — Treating `super.method()` like static dispatch

JavaScript's `super` lookup is special but does not freeze the receiver to the base class.

---

### Mistake 6 — Using `this.constructor` as a universal factory strategy

This can interact badly with:

- subclassing,
- mutation,
- proxies,
- species semantics.

---

### Mistake 7 — Overusing `Symbol.species`

Species can make result construction implicit and difficult to reason about.

Use inheritance intentionally.

---

### Mistake 8 — Assuming built-in methods are generic

Methods such as:

```js
Map.prototype.get
```

require valid Map receiver state.

---

### Mistake 9 — Calling overridable methods from base constructors

Derived overrides may execute before derived initialization is complete.

---

### Mistake 10 — Building deep inheritance hierarchies

Deep hierarchies make:

- construction order,
- override behavior,
- debugging,
- future change,

harder.

---

## 16. Comparison With Related Concepts

| Technique | Main mechanism | Coupling | State initialization | Best use |
|---|---|---:|---|---|
| Inheritance | `extends` + prototype/constructor relationships | High | Base + derived | True subtype relationship |
| Composition | Object contains collaborators | Lower | Explicit | Flexible system design |
| Delegation | Forwarding behavior to another object | Medium | Explicit | Shared capabilities |
| Mixin | Copy/compose behavior patterns | Medium | Custom | Reusable orthogonal behavior |
| Private field | Class-local state | Lower external coupling | Class-defined | Encapsulation |
| WeakMap | External object-keyed state | Lower prototype coupling | Explicit | Hidden metadata |

### Inheritance versus composition

Inheritance says:

```text
Child is a Parent
```

Composition says:

```text
Object has a Parent-like capability
```

The second is often easier to evolve.

---

### `super()` versus direct base call

```js
super.method()
```

is:

- base-oriented lookup,
- current receiver.

A manually stored base function:

```js
Parent.prototype.method.call(this)
```

is a different mechanism.

The latter bypasses some of the special `super` reference behavior and is generally more brittle.

---

### `Symbol.species` versus `this.constructor`

`this.constructor`:

```js
this.constructor
```

is a property lookup.

Species:

```js
this.constructor[Symbol.species]
```

when used by a species-aware built-in algorithm is a defined protocol path.

Neither should be treated as an automatic synonym for:

```text
"class identity"
```

---

## 17. Performance Considerations

Inheritance itself is not automatically slow.

Actual performance depends on:

- object shapes,
- prototype depth,
- method dispatch patterns,
- engine optimizations,
- polymorphism,
- Proxy usage,
- built-in internal paths.

### Potential costs of deep inheritance

Deep hierarchies can increase:

- lookup complexity in conceptual reasoning,
- polymorphic behavior,
- hidden coupling,
- optimization complexity,
- debugging time.

Even where engines optimize lookup well, the human maintenance cost can dominate.

### Species overhead

Species-aware built-in operations may involve:

- constructor lookup,
- property access,
- getter execution,
- constructor validation,
- allocation.

A custom species getter can also introduce user code into operations.

Do not micro-optimize species blindly. Measure actual workloads.

---

## 18. Memory Considerations

Subclass objects still contain ordinary object state plus any built-in internal state established by construction.

Memory questions include:

- instance fields,
- captured closures,
- inherited object graph references,
- collection internal storage,
- static caches,
- retained constructor references.

### Static inheritance and retained state

If a base constructor has:

```js
class Base {
  static cache = new Map();
}
```

a subclass accessing that static property may share the inherited object unless the property is shadowed.

This can create accidental shared mutable state.

---

### Subclass proliferation

Dynamically generating classes can create many constructor/prototype objects.

Patterns such as:

```js
function makeType(Base) {
  return class extends Base {};
}
```

can become expensive when used repeatedly without a stable caching strategy.

---

## 19. Security Considerations

### Overridable constructor methods

A base constructor that calls virtual methods can execute derived code during partially initialized state.

This may violate assumptions around:

- authentication,
- validation,
- invariants,
- object integrity.

---

### Species as user-controlled code

Species access can invoke user-defined getters.

Library code should not assume:

```js
array.map(...)
```

is always a side-effect-free internal operation when custom subclasses are involved.

---

### Constructor property tampering

Do not trust:

```js
this.constructor
```

as a security boundary.

It is a mutable property relationship rather than a cryptographic identity.

---

### Subclassing built-ins

Security-sensitive code should be careful when accepting subclass instances from untrusted callers.

A subclass can override methods and protocol hooks.

Example:

```js
class DangerousArray extends Array {
  static get [Symbol.species]() {
    // arbitrary behavior
    return SomeConstructor;
  }
}
```

A built-in method invoked on the object may cross a user-code boundary.

---

## 20. Production Usage

### Good use case — Stable domain subtype

```js
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
}

class TaxedMoney extends Money {
  withTax(rate) {
    return new TaxedMoney(
      this.amount * (1 + rate),
      this.currency,
    );
  }
}
```

Inheritance is reasonable when the subtype genuinely preserves the base abstraction's semantic contract.

---

### Good use case — Built-in collection specialization

```js
class UniqueIds extends Set {
  add(value) {
    if (typeof value !== "string") {
      throw new TypeError("Expected a string");
    }

    return super.add(value);
  }
}
```

This is useful when the subclass preserves Set semantics.

---

### Risky use case — Constructor-time polymorphism

Avoid:

```js
class Base {
  constructor() {
    this.initialize();
  }
}
```

when `initialize()` is intentionally overridable.

Prefer explicit post-construction initialization or factories when invariants are complex.

---

### Risky use case — Deep framework hierarchies

A 6–10 level hierarchy may encode too many assumptions.

Consider composition when:

```text
behavior changes independently
```

or:

```text
subtypes need incompatible lifecycles
```

---

### Species production guidance

Use custom `Symbol.species` only when a concrete API contract requires non-default derived-result behavior.

Document:

- which operations are species-aware,
- which constructor is expected,
- what invariants the result must preserve,
- whether species can return `null`,
- whether custom constructors are supported.

---

## 21. Implementation From Scratch

### 21.1 Mini inheritance model

A simplified educational model:

```js
function extend(Base, Child) {
  Object.setPrototypeOf(Child, Base);
  Object.setPrototypeOf(Child.prototype, Base.prototype);

  return Child;
}
```

This demonstrates the two prototype relationships.

It does **not** implement constructor semantics.

---

### 21.2 Manual derived-construction simulator

```js
function constructDerived(Child, Base, args) {
  let receiver;

  receiver = Reflect.construct(Base, args, Child);

  return receiver;
}
```

The important lesson is:

```text
constructor identity
+
newTarget
+
base construction
```

all matter.

---

### 21.3 Custom subclassed collection

```js
class PositiveSet extends Set {
  add(value) {
    if (typeof value !== "number" || value <= 0) {
      throw new TypeError("Value must be positive");
    }

    return super.add(value);
  }
}

const values = new PositiveSet();

values.add(10);

console.log([...values]);
```

---

### 21.4 Species experiment

```js
class MyArray extends Array {}

const values = new MyArray(1, 2, 3);

const mapped = values.map((value) => value * 2);

console.log(mapped instanceof MyArray);
```

Then override species:

```js
class PlainArrayResult extends Array {
  static get [Symbol.species]() {
    return Array;
  }
}

const values = new PlainArrayResult(1, 2, 3);
const mapped = values.map((value) => value * 2);

console.log(mapped instanceof PlainArrayResult);
console.log(mapped instanceof Array);
```

The exercise should be run and traced rather than memorized.

---

### 21.5 Production-grade species test

Build tests covering:

```text
default species
custom species
null species where supported by the algorithm
invalid species
species getter side effects
subclassed result identity
cross-realm constructors
frozen constructors
Proxy constructors
```

---

### Implementation progression

**Guided**

- Implement simple `extends`-style prototype links.
- Implement a collection subclass.

**Partially Guided**

- Simulate constructor forwarding.
- Build a constructor factory.

**No Reference**

- Create a custom subclassable collection with result-producing methods.

**Edge-Case Hardened**

Support:

- custom species,
- invalid constructors,
- overridden methods,
- object-returning constructors.

**Production-Grade**

Add:

- invariants,
- tests,
- documentation,
- benchmark suite,
- compatibility matrix,
- explicit composition alternative.

---

## 22. Debugging Exercises

### Exercise 1 — `this` before `super`

```js
class Base {}

class Child extends Base {
  constructor() {
    this.value = 1;
    super();
  }
}
```

Diagnose the failure.

---

### Exercise 2 — Missing `super()`

```js
class Base {}

class Child extends Base {
  constructor() {
    console.log("start");
  }
}

new Child();
```

Explain the failure based on derived constructor completion.

---

### Exercise 3 — `super.method()` receiver

```js
class Base {
  show() {
    return this.kind;
  }
}

class Child extends Base {
  constructor() {
    super();
    this.kind = "child";
  }

  show() {
    return super.show();
  }
}

console.log(new Child().show());
```

Predict without running.

---

### Exercise 4 — Virtual dispatch during construction

```js
class Base {
  constructor() {
    this.init();
  }

  init() {
    this.value = "base";
  }
}

class Child extends Base {
  value = "field";

  init() {
    this.value = "child";
  }
}

console.log(new Child().value);
```

Determine the initialization order and final value.

---

### Exercise 5 — Species

```js
class A extends Array {}

const a = new A(1, 2, 3);
const b = a.map((x) => x * 2);

console.log(b instanceof A);
```

Then repeat after overriding `Symbol.species`.

---

## 23. Code Review Exercise

Review:

```js
class BaseService {
  constructor(logger) {
    this.logger = logger;
    this.initialize();
  }

  initialize() {
    this.logger.info("base initialize");
  }
}

class PaymentService extends BaseService {
  config = {
    retries: 3,
  };

  initialize() {
    this.logger.info(this.config.retries);
  }
}
```

### Questions

1. Is `config` guaranteed to be initialized when `initialize()` runs?
2. What does virtual dispatch do here?
3. Could `config` be `undefined`?
4. What object state is safe to rely on inside a base constructor?
5. Would a factory or explicit `start()` phase be safer?
6. Does inheritance express the real lifecycle relationship?
7. What observability would you add to diagnose initialization-order bugs?

---

## 24. Interview Questions

### Fundamentals

1. What does `extends` do?
2. What is a derived constructor?
3. Why must `super()` normally be called before `this`?
4. What does `super()` actually do?
5. Why is `super.method()` different from `this.method()`?
6. Is `super()` equivalent to `Parent.call(this)`?
7. How does static inheritance work?
8. What happens when a derived class has no explicit constructor?
9. Can a derived constructor return an object without calling `super()`?
10. What happens if a derived constructor returns a primitive?

### Prototypes and construction

11. What are the two inheritance relationships created by `extends`?
12. What is the difference between `Child.prototype` and `Child` inheritance?
13. How does `new` interact with a derived constructor?
14. What is `[[Construct]]`?
15. What is the role of `newTarget`?
16. Why can a base constructor influence the actual instance produced?
17. What does `GetPrototypeFromConstructor` conceptually do?
18. Why is built-in subclassing different from simply sharing a prototype?

### Species

19. What is `Symbol.species`?
20. Why does species exist?
21. Is species the object's class?
22. Is every built-in method species-aware?
23. What is the conceptual role of `SpeciesConstructor`?
24. Why might a species getter introduce side effects?
25. Why is `this.constructor` not equivalent to species semantics?
26. When is custom species design appropriate?

### Principal-level

27. When should composition replace inheritance?
28. Why is constructor-time virtual dispatch dangerous?
29. How would you design a subclassable library API?
30. What invariants must a subclass preserve?
31. What risks arise when subclassing built-ins?
32. How would you document species semantics?
33. How do proxies complicate subclass/constructor reasoning?
34. How does cross-realm behavior affect constructor identity?
35. Would you allow users to subclass your library's core class? Why?
36. How would you prevent a subclass from breaking lifecycle invariants?
37. How would you decide between:
    - subclass,
    - wrapper,
    - adapter,
    - composition,
    - factory?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
class A {
  constructor() {
    this.name = "A";
  }
}

class B extends A {
  constructor() {
    super();
    this.name = "B";
  }
}

console.log(new B().name);
```

---

### Exercise B

```js
class A {
  method() {
    return this.value;
  }
}

class B extends A {
  constructor() {
    super();
    this.value = 10;
  }

  method() {
    return super.method();
  }
}

console.log(new B().method());
```

---

### Exercise C

```js
class A {
  static value() {
    return "A";
  }
}

class B extends A {}

console.log(B.value());
```

---

### Exercise D

```js
class A {}

class B extends A {
  constructor() {
    return { x: 1 };
  }
}

console.log(new B().x);
```

---

### Exercise E

```js
class A {
  constructor() {
    this.init();
  }

  init() {
    this.value = "A";
  }
}

class B extends A {
  value = "field";

  init() {
    this.value = "B";
  }
}

console.log(new B().value);
```

---

### Exercise F

```js
class A extends Array {}

const a = new A(1, 2);
const b = a.map(x => x * 2);

console.log(b instanceof A);
console.log(b instanceof Array);
```

---

### Exercise G

Modify Exercise F:

```js
class A extends Array {
  static get [Symbol.species]() {
    return Array;
  }
}
```

Predict both `instanceof` results.

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
base constructor
derived constructor
super()
new
```

without using the phrase "parent method call" as the entire explanation.

### Level 2 — Explain

Teach why:

```js
this.x = 1;
super();
```

fails in a derived constructor.

### Level 3 — Predict

Predict initialization order for classes containing:

- base fields,
- base constructor body,
- derived fields,
- derived constructor body.

### Level 4 — Implement

Build:

```js
class ValidatedArray extends Array {}
```

that enforces a domain invariant.

### Level 5 — Debug

Repair a constructor hierarchy with unsafe virtual dispatch.

### Level 6 — Compare

Compare:

```text
inheritance
composition
delegation
mixins
```

on:

- coupling,
- reuse,
- testing,
- lifecycle,
- extension,
- performance,
- future changes.

### Level 7 — Apply

Build a subclassable collection API and document its subclassing contract.

### Level 8 — Defend

Decide whether your custom collection should support `Symbol.species`.

### Level 9 — Principal Judgment

Given a library API consumed by hundreds of teams, decide whether allowing subclassing is worth the semantic contract it creates.

Document:

```text
constructors
overrides
super expectations
private state
species
error behavior
future compatibility
```

---

## 27. Key Takeaways

1. `extends` establishes both instance and static inheritance relationships.
2. A derived constructor has different `this` initialization semantics from a base constructor.
3. `super()` participates in construction; it is not simply `Parent.call(this)`.
4. `this` cannot normally be used before `super()` in a derived constructor.
5. `super.method()` starts lookup from the base side while preserving the actual receiver.
6. Derived constructors can explicitly return objects, which affects construction semantics.
7. Base constructors can return objects and influence the resulting instance.
8. Class fields have defined initialization order.
9. Private fields are class-scoped brands, not inherited names.
10. Built-in subclasses can rely on internal slots established by built-in constructors.
11. Prototype inheritance alone does not reproduce built-in internal state.
12. `Symbol.species` is a result-constructor protocol for selected built-in operations.
13. Species is not the same as object class identity.
14. `this.constructor` is a property lookup, not a universal semantic identity mechanism.
15. Constructor-time virtual dispatch can execute derived code too early.
16. Deep inheritance increases semantic coupling.
17. Subclassing should express a stable substitutability relationship.
18. Composition often provides a lower-coupling alternative.
19. Species and subclassing are powerful but can introduce hidden control flow.
20. Principal-level JavaScript engineering requires reasoning about constructor semantics, prototypes, internal slots, and result construction together.

---

## 28. Concept Connections

### Depends On

- Chapter 14 — `this`
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols

### Builds Toward

- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 43 — Ordinary Object Internal Methods
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 47 — JavaScript Engine Architecture
- Chapter 57 — JavaScript Security Engineering
- Chapter 75 — OOP
- Chapter 76 — Composition / Abstraction Design
- Chapter 80 — Library Authoring
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs

### Related Concepts

- `new`
- `[[Construct]]`
- `newTarget`
- `super`
- `[[HomeObject]]`
- `GetPrototypeFromConstructor`
- `OrdinaryCreateFromConstructor`
- `SpeciesConstructor`
- `Symbol.species`
- private brands
- built-in internal slots
- virtual dispatch

### Concepts Revisited

**Chapter 14 — `this`:**  
Derived construction explains why `this` initialization is not identical across base and derived constructors.

**Chapter 17 — Prototypes:**  
`extends` creates a dual inheritance structure on constructor and prototype objects.

**Chapter 18 — Classes:**  
This chapter turns class syntax into construction and internal semantics.

**Chapter 19 — Proxy / Reflect:**  
Construction and custom constructors interact with `Reflect.construct`, proxies, and traps.

**Chapter 20 — Symbols:**  
`Symbol.species` demonstrates how well-known Symbols expose standardized extensibility hooks.

### Why This Chapter Matters Later

Without this chapter, built-in subclassing and species can look like magic.

After this chapter, the learner should recognize:

```text
extends
   ↓
prototype relationships
   +
constructor relationships
   +
derived this initialization
   +
super construction
   +
built-in internal state
   +
species-aware result construction
```

This is the foundation for advanced work with:

- Arrays,
- Maps,
- Sets,
- typed arrays,
- framework base classes,
- extensible libraries,
- result-producing built-in methods.

---

## Track A — Core Theory

### Level 1 — Intuition

> A derived class is a specialized constructor/prototype relationship that also changes how construction initializes `this`.

### Level 2 — Syntax

Know:

```js
class Child extends Parent {}
super()
super.method()
static get [Symbol.species]()
```

### Level 3 — Practical

Implement:

- a simple subclass,
- a built-in subclass,
- a species override.

### Level 4 — Edge Cases

Understand:

- object-returning constructors,
- missing/duplicate `super`,
- constructor-time virtual dispatch,
- invalid species,
- null species,
- `extends null`,
- dynamic base classes.

### Level 5 — Runtime/Internal

Understand:

- base versus derived constructor kind,
- `[[Construct]]`,
- `newTarget`,
- receiver establishment,
- internal slots.

### Level 6 — Specification Semantics

Be comfortable with:

- `[[ConstructorKind]]`,
- `OrdinaryCreateFromConstructor`,
- `GetPrototypeFromConstructor`,
- super property references,
- `SpeciesConstructor`.

### Level 7 — Performance/Security

Reason about:

- constructor dispatch,
- hidden side effects,
- species getter execution,
- subclass proliferation,
- built-in specialization.

### Level 8 — Production Engineering

Design:

- subclass contracts,
- constructor invariants,
- safe initialization,
- result types,
- species policy.

### Level 9 — Interview/Reasoning

Answer:

> Why does JavaScript distinguish prototype inheritance from constructor/result construction?

### Level 10 — Principal Judgment

Evaluate:

> Should a public class be subclassable at all?

The correct answer must examine:

- substitutability,
- invariants,
- private state,
- lifecycle,
- override points,
- testing,
- future compatibility,
- operational risk.

---

## Track B — Implementation

The implementation ladder is:

```text
1. Simple class inheritance
        ↓
2. Explicit constructor + super()
        ↓
3. super method dispatch
        ↓
4. Static inheritance
        ↓
5. Built-in subclassing
        ↓
6. Species experiments
        ↓
7. Constructor simulator
        ↓
8. Edge-case hardening
        ↓
9. Subclassable library API
        ↓
10. Production compatibility contract
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain why:

```js
super.method()
```

does not mean:

```text
execute with a Parent instance
```

### Drill 2

Explain why:

```text
prototype inheritance
```

is insufficient to reproduce:

```text
Map
Set
Array
TypedArray
```

internal semantics.

### Drill 3

Explain:

```text
receiver constructor
        vs
result constructor
```

and connect it to `Symbol.species`.

### Drill 4

Analyze constructor-time virtual dispatch and propose a safer design.

### Drill 5

Choose between:

```text
extends
composition
adapter
factory
mixin
```

for a production API and defend your decision.

---

## 29. Completion Criteria

Mark Chapter 21 `[+] Completed` only when the learner can:

- [ ] Explain base versus derived constructors.
- [ ] Explain why derived `this` is uninitialized before `super()`.
- [ ] Explain the semantic role of `super()`.
- [ ] Explain `super.method()` receiver behavior.
- [ ] Explain static inheritance.
- [ ] Explain the dual prototype relationships created by `extends`.
- [ ] Predict constructor initialization order.
- [ ] Explain synthesized derived constructors.
- [ ] Explain object-returning constructor behavior.
- [ ] Explain built-in subclassing.
- [ ] Explain built-in internal slots.
- [ ] Explain why prototype inheritance alone is insufficient for built-in semantics.
- [ ] Explain `Symbol.species`.
- [ ] Explain the purpose of species-aware construction.
- [ ] Distinguish species from `this.constructor`.
- [ ] Identify which operations actually use species semantics instead of assuming all do.
- [ ] Explain private-field behavior across inheritance.
- [ ] Debug constructor-time virtual dispatch.
- [ ] Debug missing/duplicate `super()`.
- [ ] Implement and test a built-in subclass.
- [ ] Implement and explain a custom species.
- [ ] Compare inheritance with composition and delegation.
- [ ] Evaluate subclassability as an API contract.
- [ ] Complete output-prediction exercises.
- [ ] Defend a production inheritance decision.

### Mastery Gate

Mastery requires:

```text
Understand
   ↓
Explain
   ↓
Predict
   ↓
Implement
   ↓
Debug
   ↓
Apply
   ↓
Compare
   ↓
Defend
```

The final defense should answer:

> When should a JavaScript library expose a class as subclassable, what invariants must it preserve, and how should constructor, private state, built-in behavior, species, and future compatibility be designed?

---

## Chapter 21 Retrieval Set

### Retrieval 1

Why is:

```js
this.value = 1;
```

illegal before:

```js
super();
```

in a normal derived constructor?

### Retrieval 2

Explain why:

```js
super.method()
```

uses the child instance as the receiver.

### Retrieval 3

What are the two prototype relationships created by:

```js
class Child extends Parent {}
```

### Retrieval 4

Why does:

```js
class MyMap extends Map {}
```

work differently from:

```js
const fake = Object.create(Map.prototype);
```

### Retrieval 5

What problem does `Symbol.species` solve?

### Retrieval 6

Why is:

```js
this.constructor
```

not equivalent to:

```js
Symbol.species
```

### Retrieval 7

Why is calling overridable methods from a base constructor dangerous?

### Retrieval 8

When would you reject subclassing and choose composition instead?

---

## Chapter 21 Final Mental Model

Remember:

```text
                extends
                   |
        +----------+----------+
        |                     |
   static side           instance side
        |                     |
 Child → Parent        Child.prototype
                              ↓
                       Parent.prototype
```

Construction:

```text
new Child()
    ↓
derived constructor
    ↓
super(...)
    ↓
base construction
    ↓
instance established
    ↓
derived initialization
    ↓
final object
```

Species:

```text
receiver
   ↓
constructor selection
   ↓
constructor[Symbol.species]
   ↓
result construction
```

And the central distinction:

> Inheritance answers "what behavior and construction relationship does this subtype have?" Species answers a narrower question: "when this particular built-in operation creates a derived result, which constructor should it use?"

---