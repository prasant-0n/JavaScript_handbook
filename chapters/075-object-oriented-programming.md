# Chapter 75 — Object-Oriented Programming

> **Curriculum position:** Part XIV — Programming Paradigms  
> **Previous chapter:** Chapter 74 — Functional Programming  
> **Next chapter:** Chapter 76 — Composition and Abstraction Design  
> **Primary environment:** Modern JavaScript / TypeScript, Node.js, browsers, libraries, and production systems.

---

# Chapter Mission

Master **object-oriented programming (OOP) in JavaScript** from the language's actual object model rather than from assumptions imported from class-first languages.

JavaScript supports:

```text
objects
prototypes
prototype chains
constructors
classes
private fields
getters/setters
inheritance
composition
polymorphism
encapsulation
```

But JavaScript is fundamentally a prototype-based language with class syntax layered over the object/prototype model.

The central question is not:

> “How do I write classes?”

It is:

```text
What responsibilities belong to an object?
What state does it own?
What invariants does it maintain?
What behavior does it expose?
How is identity represented?
How does delegation work?
Should behavior be inherited or composed?
Where should mutation occur?
```

A principal engineer should be able to choose among:

```text
class
prototype delegation
factory function
closure
composition
pure functions
```

instead of assuming one paradigm is universally correct.

The principal-level goal is:

> **Use object-oriented design when identity, encapsulated state, lifecycle, polymorphism, and behavioral contracts make it the clearest model—and reject inheritance when composition or simpler functions are better.**


# 1. Learning Objectives

By completion you should be able to:

## JavaScript object model

- explain objects as property collections;
- explain prototype delegation;
- explain prototype chains;
- distinguish own and inherited properties;
- distinguish constructor functions from classes;
- explain `new`;
- explain class syntax as JavaScript language semantics;
- explain instance methods;
- explain static methods;
- explain private class fields;
- explain getters/setters;
- explain derived classes;
- explain `super`;
- explain constructor behavior;
- explain object identity.

## OOP principles

- explain encapsulation;
- explain abstraction;
- explain polymorphism;
- explain inheritance;
- explain composition;
- explain substitutability;
- explain contracts and invariants.

## Design

- choose inheritance versus composition;
- design stable public interfaces;
- protect internal invariants;
- manage object lifecycle;
- model state transitions;
- avoid mutable shared state where dangerous;
- design extension points deliberately.

## Production

- understand object allocation costs;
- reason about hidden runtime behavior without treating it as a language guarantee;
- avoid inheritance hierarchies that become brittle;
- benchmark object-heavy hot paths when relevant;
- design APIs that remain stable while implementation changes;
- combine OOP with functional techniques appropriately.


# 2. Prerequisites

Recommended:

- Chapter 09 — Functions and First-Class Behavior
- Chapter 10 — Scope and Lexical Environments
- Chapter 13 — Closures
- Chapter 14 — `this`, Invocation, and Binding
- Chapter 15 — Objects and Property Semantics
- Chapter 17 — Prototypes and Prototype Chains
- Chapter 18 — Classes and OOP in JavaScript
- Chapter 19 — Proxy and Reflect
- Chapter 45 — Memory and Garbage Collection
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 71 — Fundamental Data Structures
- Chapter 74 — Functional Programming


# 3. What Is Object-Oriented Programming?

OOP organizes software around objects that combine:

```text
state
+
behavior
+
identity
```

Example:

```js
class BankAccount {
  #balance = 0;

  deposit(amount) {
    if (amount <= 0) {
      throw new RangeError("Amount must be positive");
    }

    this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}
```

The object owns an invariant:

```text
balance is never negative
```

assuming every mutating operation enforces that contract.

OOP is useful when:

```text
state has identity
state and behavior belong together
lifecycle matters
invariants need protection
multiple implementations share an interface
```


# 4. Why Does It Exist?

OOP can control complexity by encapsulating:

```text
state
mutation
invariants
lifecycle
polymorphic behavior
```

Instead of allowing any module to change:

```js
account.balance = -1000;
```

a class can hide state and expose:

```js
account.deposit(...)
account.withdraw(...)
```

The important benefit is not syntax.

It is:

```text
controlled state ownership
```


# 5. JavaScript Is Not Class-First

JavaScript's object model is based on objects and prototype delegation.

Class syntax provides a structured way to define constructor behavior, methods, fields, and inheritance.

Conceptually:

```text
object
  ↓
[[Prototype]]
  ↓
another object
  ↓
...
```

Therefore:

```text
class syntax
```

does not mean:

```text
JavaScript internally becomes Java/C++/C#.
```

Understanding the prototype system is necessary for advanced OOP reasoning.


# 6. Objects, Identity, and State

Objects have identity.

```js
const a = { value: 1 };
const b = { value: 1 };

console.log(a === b);
```

Prediction:

```text
false
```

Even when state looks identical:

```text
a ≠ b
```

OOP often uses identity deliberately:

```text
user instance
connection
job
cache entry
service
transaction
```

Functional designs may instead emphasize values and transformations.


# 7. Encapsulation

Encapsulation means controlling access to internal state and behavior.

Modern JavaScript class private fields:

```js
class Counter {
  #value = 0;

  increment() {
    this.#value++;
  }

  get value() {
    return this.#value;
  }
}
```

External code cannot directly access:

```js
counter.#value
```

The field is private by language semantics.

Encapsulation is valuable when object invariants would otherwise be easy to violate.


# 8. Public vs Private State

Public:

```js
class User {
  name;
}
```

Private:

```js
class User {
  #name;
}
```

Do not choose private fields merely because they look “more OOP.”

Use them when:

```text
representation should be hidden
invariants need protection
future implementation changes should not break callers
```


# 9. Getters and Setters

Example:

```js
class Temperature {
  #celsius;

  constructor(celsius) {
    this.celsius = celsius;
  }

  get celsius() {
    return this.#celsius;
  }

  set celsius(value) {
    if (!Number.isFinite(value)) {
      throw new TypeError("Invalid temperature");
    }

    this.#celsius = value;
  }
}
```

Getters/setters allow:

```text
property-like syntax
+
behavioral validation
```

But avoid turning every method into a getter/setter.

Property syntax can hide significant work.


# 10. Constructor

A constructor establishes an instance's initial valid state.

```js
class User {
  #id;

  constructor(id) {
    if (!id) {
      throw new TypeError("id required");
    }

    this.#id = id;
  }
}
```

A well-designed constructor should establish the object's initial invariants.

Avoid creating instances that are temporarily invalid when possible.


# 11. `new` Mental Model

Conceptually, a `new` call involves:

```text
1. allocate a new object
2. connect it to the constructor's prototype
3. call constructor with that object as `this`
4. use the appropriate constructor return behavior
```

The exact specification semantics are more precise than this summary.

Understand:

```text
new
+
constructor
+
prototype
```

as one connected mechanism.


# 12. Constructor Functions

Before class syntax, constructor-style patterns used functions:

```js
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return `Hello ${this.name}`;
};

const user = new User("A");
```

The instance delegates to:

```text
User.prototype
```

Class syntax provides a more structured form of modern constructor/prototype programming.


# 13. Prototype Delegation

Consider:

```js
const parent = {
  greet() {
    return "hello";
  },
};

const child = Object.create(parent);

console.log(child.greet());
```

The child does not need its own `greet`.

Property lookup can delegate through:

```text
child
 ↓
parent
 ↓
parent's prototype
```

This is the core model behind JavaScript's prototype chains.


# 14. Instance Methods

Example:

```js
class User {
  greet() {
    return "hello";
  }
}
```

Conceptually, method behavior is associated with the prototype rather than requiring a new function object to be created for each instance through ordinary class-method semantics.

This is one reason prototype methods differ from per-instance arrow functions.


# 15. Per-Instance Arrow Methods

Example:

```js
class User {
  name = "A";

  greet = () => {
    return this.name;
  };
}
```

This places an arrow function on each instance.

Benefits can include:

```text
lexical this
convenient callback passing
```

Costs can include:

```text
per-instance function allocation
different object shape behavior
memory overhead
```

Measure in hot object-heavy workloads.


# 16. Static Methods

Static methods belong to the class constructor rather than an instance.

```js
class User {
  constructor(name) {
    this.name = name;
  }

  static fromJSON(json) {
    return new User(json.name);
  }
}
```

Usage:

```js
const user = User.fromJSON({ name: "A" });
```

Not:

```js
user.fromJSON()
```

Static methods are useful for:

```text
factories
parsers
class-level utilities
construction policies
```


# 17. Static Initialization

Modern class syntax can define static fields and static initialization blocks.

Example:

```js
class Config {
  static version = "1";
  static {
    Config.loadedAt = Date.now();
  }
}
```

Use static initialization for class-level setup when it is clearer than external initialization.

Be cautious about:

```text
hidden startup effects
import-time work
test isolation
```


# 18. Inheritance

Class inheritance expresses:

```text
is-a
```

relationships.

Example:

```js
class Animal {
  speak() {
    return "sound";
  }
}

class Dog extends Animal {
  speak() {
    return "woof";
  }
}
```

The derived class inherits/delegates behavior from the base class.

Inheritance is useful when the subtype genuinely conforms to the base contract.


# 19. `super`

Example:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name);
  }
}
```

`super(...)` invokes the parent constructor in a derived constructor.

In methods, `super.method()` performs inherited method access through the appropriate prototype relationship.

Do not treat `super` as a generic parent-object pointer; it has defined language semantics around home objects and prototype lookup.


# 20. Derived Constructors

Derived classes have special initialization rules.

A derived constructor cannot use `this` before the base constructor initialization has occurred through:

```js
super(...)
```

This is a common source of bugs.

The language enforces initialization order because a derived instance is established through the base constructor process.


# 21. Method Overriding

A subclass can override a method:

```js
class Dog extends Animal {
  speak() {
    return "woof";
  }
}
```

This enables polymorphic dispatch:

```js
function makeSound(animal) {
  return animal.speak();
}
```

The caller depends on behavior:

```text
speak()
```

rather than concrete subtype.


# 22. Polymorphism

Polymorphism means different objects can respond to the same operation according to their own behavior.

JavaScript's dynamic nature permits structural-style polymorphism:

```js
function render(view) {
  return view.render();
}
```

Any object with an appropriate `render()` method can participate.

This is often called:

```text
duck typing
```

The important property is behavioral compatibility.


# 23. Structural vs Nominal Thinking

JavaScript commonly supports structural behavior:

```text
"Can this object perform the required operation?"
```

rather than strict nominal hierarchy:

```text
"Does this object inherit from class X?"
```

Example:

```js
const logger = {
  write(message) {
    console.log(message);
  },
};

function useWriter(writer) {
  writer.write("hello");
}
```

The object does not need to extend a `Writer` class.


# 24. Liskov-Style Substitutability

A subtype should be usable anywhere the base contract is expected without violating that contract.

Bad design:

```text
Base:
withdraw(amount)

Subtype:
withdraw() always throws
```

The subtype technically inherits the method but violates the behavioral expectation.

Inheritance creates a contract, not merely code reuse.


# 25. Composition

Composition combines smaller objects or functions rather than inheriting behavior.

Example:

```js
class CheckoutService {
  constructor(paymentGateway, logger) {
    this.paymentGateway = paymentGateway;
    this.logger = logger;
  }
}
```

The service receives capabilities:

```text
paymentGateway
logger
```

instead of extending:

```text
BaseCheckoutService
```

This can reduce coupling.


# 26. Composition vs Inheritance

Prefer inheritance when:

```text
true subtype relationship
stable shared contract
substitutability
shared lifecycle semantics
```

Prefer composition when:

```text
behavior changes independently
capabilities combine
multiple implementations are interchangeable
inheritance hierarchy would become deep
```

Principal engineers should prefer the simplest stable relationship.


# 27. Deep Inheritance Warning

A hierarchy such as:

```text
Entity
 ↓
BusinessEntity
 ↓
FinancialEntity
 ↓
PaymentEntity
 ↓
CreditCardPaymentEntity
 ↓
PremiumCreditCardPaymentEntity
```

can become fragile.

Problems:

```text
implicit coupling
base-class changes affect many subclasses
override complexity
testing difficulty
constructor ordering
diamond-like conceptual reuse
```

Composition often scales better for independently changing capabilities.


# 28. Mixins

JavaScript can compose behavior using mixin functions.

Example:

```js
const Timestamped = Base => class extends Base {
  createdAt = Date.now();
};

const Identified = Base => class extends Base {
  id = crypto.randomUUID();
};
```

Then:

```js
class Model {}
class User extends Identified(Timestamped(Model)) {}
```

Mixins can compose behavior but may create:

```text
complex inheritance chains
method conflicts
debugging difficulty
```

Use deliberately.


# 29. Prototype Composition

You can also compose delegation directly:

```js
const printable = {
  print() {
    return this.value;
  },
};

const identifiable = {
  getId() {
    return this.id;
  },
};

const object = Object.assign(
  {},
  printable,
  identifiable
);
```

The result is an object with combined behavior.

This is different from class inheritance.


# 30. Factory Functions

A factory returns an object.

```js
function createUser(name) {
  return {
    name,

    greet() {
      return `Hello ${name}`;
    },
  };
}
```

Factories can be excellent when:

```text
identity is simple
private state is closure-based
inheritance is unnecessary
composition is preferred
```

They also naturally support dependency injection.


# 31. Closures vs Private Fields

Closure:

```js
function createCounter() {
  let count = 0;

  return {
    increment() {
      count++;
    },

    get value() {
      return count;
    },
  };
}
```

Class private field:

```js
class Counter {
  #count = 0;

  increment() {
    this.#count++;
  }

  get value() {
    return this.#count;
  }
}
```

Both can encapsulate state.

Choose based on:

```text
identity
inheritance
prototype reuse
API style
testing
ergonomics
```


# 32. Class Fields

Modern classes can define fields:

```js
class User {
  role = "user";

  constructor(name) {
    this.name = name;
  }
}
```

Field initialization occurs as part of instance construction.

Private fields:

```js
#role = "user";
```

provide stronger encapsulation through language-level private names.


# 33. Object Invariants

Suppose:

```text
balance >= 0
```

is an invariant.

Do not allow external code to mutate:

```js
account.balance
```

without validation.

The class should control:

```text
deposit
withdraw
transfer
```

This is where OOP provides value:

```text
state ownership
+
invariant enforcement
```


# 34. Lifecycle Modeling

Some objects have explicit lifecycle:

```text
created
initialized
connected
active
closing
closed
```

Examples:

```text
database connection
WebSocket
stream
transaction
worker
resource handle
```

A class can encode legal transitions.

But the lifecycle should be explicit rather than relying on callers to remember undocumented state rules.


# 35. State Machines in Classes

Example:

```js
class Connection {
  #state = "idle";

  connect() {
    if (this.#state !== "idle") {
      throw new Error("Already connected or connecting");
    }

    this.#state = "connected";
  }

  close() {
    if (this.#state !== "connected") {
      throw new Error("Not connected");
    }

    this.#state = "closed";
  }
}
```

The state machine is:

```text
idle → connected → closed
```

Methods enforce legal transitions.


# 36. OOP and Functional Programming Together

These paradigms are not mutually exclusive.

A class can own lifecycle/state:

```text
PaymentService
```

while pure functions implement calculations:

```text
calculateTax()
validatePayment()
computeFee()
```

Example:

```text
class service
   ↓
pure domain functions
   ↓
effect adapters
```

This hybrid style is often highly effective in production JavaScript.


# 37. OOP and Dependency Injection

Constructor injection:

```js
class OrderService {
  constructor(repository, clock, logger) {
    this.repository = repository;
    this.clock = clock;
    this.logger = logger;
  }
}
```

This makes dependencies:

```text
explicit
replaceable
testable
```

Avoid hidden global dependencies where practical.


# 38. Interface Thinking Without Interfaces

JavaScript does not require a class declaration to express a behavioral interface.

Instead define:

```text
required methods
required semantics
error behavior
lifecycle
```

Example:

```text
PaymentGateway:
  authorize(request)
  capture(id)
  refund(id)
```

Implementations can be:

```text
StripeGateway
MockGateway
LocalGateway
```

The caller depends on behavior.


# 39. TypeScript Interfaces

TypeScript can formalize structural contracts:

```ts
interface PaymentGateway {
  authorize(request: PaymentRequest): Promise<Authorization>;
}
```

Then classes or objects can satisfy the contract.

Remember:

```text
TypeScript interface
```

is primarily a compile-time concept.

It does not create a runtime interface object automatically.


# 40. Abstract Classes

TypeScript supports abstract classes:

```ts
abstract class PaymentGateway {
  abstract authorize(
    request: PaymentRequest
  ): Promise<Authorization>;
}
```

This can encode a nominal inheritance hierarchy at the type level.

Do not use abstract classes merely to satisfy a design-pattern checklist.

Use them when shared state/behavior and a meaningful base contract actually exist.


# 41. Encapsulation vs Abstraction

Encapsulation:

```text
hide/control internal state
```

Abstraction:

```text
expose useful model while hiding irrelevant detail
```

Example:

```js
account.withdraw(100);
```

Abstraction hides:

```text
ledger rules
balance checks
transaction logic
```

Encapsulation prevents external code from directly corrupting those internals.


# 42. OOP and Modules

Classes should not automatically become global abstractions.

A module can expose:

```js
export class UserService {}
```

or:

```js
export function createUserService() {}
```

Choose the public abstraction based on behavior.

Modules are useful for:

```text
dependency boundaries
encapsulation
testing
deployment
```


# 43. Object Identity vs Value Semantics

OOP often emphasizes:

```text
same object over time
```

Functional programming often emphasizes:

```text
new value represents new state
```

Example:

```text
BankAccount object:
balance changes

immutable account value:
new account returned
```

Neither is universally superior.

Use identity when lifecycle matters.
Use values when replacement and deterministic transformation are more useful.


# 44. Equality in OOP

JavaScript's object equality is reference-based:

```js
new User("A") === new User("A")
```

is:

```text
false
```

If domain equality matters, define it explicitly:

```js
class User {
  constructor(id) {
    this.id = id;
  }

  equals(other) {
    return other instanceof User &&
           other.id === this.id;
  }
}
```

But equality semantics should be domain-specific.


# 45. Serialization

Class instances may serialize only their enumerable data properties through ordinary JSON serialization.

Methods/prototype behavior are not automatically reconstructed by:

```js
JSON.stringify()
JSON.parse()
```

Therefore:

```text
wire data
≠
class instance
```

A production boundary should reconstruct domain objects deliberately if needed.


# 46. ORM / Entity Objects

Database systems sometimes map records to class instances.

This can be convenient.

But beware:

```text
entity object
+
lazy database access
+
hidden queries
```

can make simple property access perform I/O.

Prefer explicit data-access behavior where hidden work would surprise callers.


# 47. OOP and Resource Management

A class is useful for a resource that has:

```text
acquire
use
release
```

Examples:

```text
file-like resource
connection
transaction
lock-like abstraction
stream
```

Modern JavaScript also provides explicit resource-management syntax covered in Chapter 30.

Use classes to model ownership when they make lifecycle semantics clearer.


# 48. Error Types as Objects

Custom error classes:

```js
class ValidationError extends Error {
  constructor(message, details) {
    super(message);
    this.name = "ValidationError";
    this.details = details;
  }
}
```

This allows:

```js
error instanceof ValidationError
```

and can establish domain-specific behavior/data.

Keep error hierarchies shallow unless the distinctions are operationally useful.


# 49. Polymorphic Errors

Callers can handle a shared interface:

```js
function classify(error) {
  if (error instanceof ValidationError) {
    return "client";
  }

  return "server";
}
```

But error classification should not rely on brittle inheritance alone.

Stable error codes can be useful for distributed boundaries:

```text
VALIDATION_FAILED
PAYMENT_DECLINED
DEPENDENCY_TIMEOUT
```


# 50. Method Binding and `this`

A method reference can lose its receiver:

```js
class Counter {
  value = 0;

  increment() {
    this.value++;
  }
}

const counter = new Counter();
const fn = counter.increment;

fn();
```

This can fail because:

```text
method call syntax was lost
```

Use deliberate binding:

```js
const fn = counter.increment.bind(counter);
```

or an appropriate arrow-function instance method when the trade-off is understood.


# 51. Callback Binding

Common pattern:

```js
button.addEventListener(
  "click",
  counter.increment.bind(counter)
);
```

Every `.bind()` call creates a bound function.

For long-lived listeners:

```text
store the bound function
```

so it can later be removed correctly.

Functional and OOP designs share this identity concern.


# 52. Private Fields and Inheritance

Private fields are not inherited as ordinary public properties.

A derived class cannot directly access a base class's private field by writing:

```js
this.#baseField
```

unless that private name was declared in the relevant class lexical scope.

The parent class must expose appropriate behavior.

This is strong encapsulation.


# 53. `instanceof`

Example:

```js
class User {}

const user = new User();

console.log(user instanceof User);
```

Result:

```text
true
```

`instanceof` follows JavaScript's prototype-based semantics and can be influenced by custom behavior such as `Symbol.hasInstance`.

Therefore:

```text
instanceof
≠
universal proof of semantic compatibility
```


# 54. Method Dispatch

A method call:

```js
object.run()
```

involves:

```text
property lookup
+
call with receiver
```

If `run` is inherited, lookup walks the prototype chain.

If the property exists on the object itself, it can shadow the inherited method.

This connects OOP directly to Chapter 17.


# 55. Shadowing

Example:

```js
const parent = {
  greet() {
    return "parent";
  },
};

const child = Object.create(parent);

child.greet = function () {
  return "child";
};
```

Now:

```js
child.greet()
```

returns:

```text
"child"
```

The own property shadows the inherited method.

Accidental shadowing can cause difficult bugs in large hierarchies.


# 56. Overriding vs Shadowing

In class inheritance:

```text
subclass method
```

overrides/delegates through the prototype relationship.

At the object level:

```text
own property
```

can shadow an inherited property.

These are related but should not be treated as exactly the same mechanism.


# 57. OOP and Object Shapes

JavaScript engines can optimize objects when their property layout is predictable.

Therefore consistent construction:

```js
class User {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }
}
```

is generally easier for engines to optimize than arbitrary mutation that creates wildly different property layouts.

Do not treat this as a universal performance guarantee; use profiling for hot code.


# 58. Hidden Classes / Shapes Caveat

Engines such as V8 use internal object representations and optimization strategies.

Terms such as:

```text
hidden class
shape
inline cache
```

describe implementation techniques, not ECMAScript language requirements.

Use these concepts to reason about possible performance behavior, not to make portable semantic claims.


# 59. Object Allocation

Creating many objects can increase:

```text
allocation
memory
GC pressure
```

Example:

```js
for (let i = 0; i < 10_000_000; i++) {
  const point = {
    x: i,
    y: i + 1,
  };
}
```

Whether this is problematic depends on:

```text
lifetime
escape
allocation rate
runtime optimization
```

Measure before redesigning.


# 60. OOP Performance — Method Style

Potentially different representations:

```text
prototype method
per-instance arrow method
closure method
```

All can implement similar behavior.

Trade-offs include:

```text
function identity
memory
allocation
callback convenience
prototype sharing
```

There is no universal winner.


# 61. Object Pooling

Object pooling reuses objects rather than allocating repeatedly.

It can help in specialized workloads with:

```text
very high allocation rate
predictable object lifecycle
measurable GC pressure
```

But pooling can introduce:

```text
state-reset bugs
retention
complexity
stale references
```

Do not add pools without evidence.


# 62. Mutability and Thread/Worker Boundaries

JavaScript workers normally communicate through message passing/structured data rather than arbitrary shared object identity.

OOP objects therefore do not automatically preserve identity across worker boundaries.

Design explicit serialization:

```text
domain object
→ message DTO
→ worker
→ reconstructed state
```

Shared memory introduces additional synchronization concerns.


# 63. OOP and Concurrency

An object with mutable state can become a coordination boundary.

Example:

```text
ConnectionPool
```

must coordinate:

```text
active
available
closing
closed
```

In asynchronous JavaScript, concurrent callers can interleave between `await` points.

Methods must preserve invariants across asynchronous transitions.


# 64. Async Methods and State

Example:

```js
class Worker {
  #busy = false;

  async run() {
    if (this.#busy) {
      throw new Error("Already running");
    }

    this.#busy = true;

    try {
      await doWork();
    } finally {
      this.#busy = false;
    }
  }
}
```

The `finally` block is essential for restoring the invariant after:

```text
success
failure
cancellation
```

This is OOP plus resource/state discipline.


# 65. Inheritance and Constructor Side Effects

Base constructors execute during derived construction.

Therefore base constructor behavior can create:

```text
I/O
registration
events
subclass coupling
```

and make object construction expensive or surprising.

Prefer constructors that establish state.

Perform substantial asynchronous initialization explicitly rather than pretending a constructor can `await`.


# 66. Async Factory Pattern

If initialization requires asynchronous work:

```js
class Client {
  constructor(config) {
    this.config = config;
  }

  static async connect(config) {
    const connection = await openConnection(config);
    return new Client(connection);
  }
}
```

This cleanly separates:

```text
synchronous instance construction
```

from:

```text
asynchronous setup
```


# 67. OOP API Stability

Public class APIs become contracts.

Changing:

```js
user.getName()
```

to:

```js
user.name
```

may affect:

```text
callers
tests
subclasses
mocks
documentation
serialization
```

Design public methods/fields deliberately.

Hide representation that you may need to change.


# 68. Fragile Base Class Problem

A base class can unintentionally expose behavior that subclasses depend on.

A change such as:

```js
Base.process()
```

changing internal call order can break derived classes that overrode:

```text
validate()
transform()
save()
```

Deep inheritance increases this risk.

Composition reduces some forms of implicit coupling.


# 69. Inheritance and Open/Closed Thinking

A system is easier to extend when new behavior can be added without constantly modifying stable core logic.

Polymorphic interfaces can help:

```text
PaymentGateway
├─ GatewayA
├─ GatewayB
└─ GatewayMock
```

But composition can provide the same extensibility:

```text
service
+
injected gateway
```

Choose based on coupling, not slogans.


# 70. OOP Design Smell — God Object

A god object does too much:

```text
database
HTTP
validation
business logic
logging
caching
scheduling
```

This destroys:

```text
cohesion
testability
change isolation
```

Split responsibilities around meaningful domain boundaries.


# 71. OOP Design Smell — Anemic Domain Model

An object that contains only:

```text
data fields
```

while all behavior lives elsewhere may be appropriate for DTOs.

But domain entities with important invariants can benefit from behavior attached to state.

Question:

```text
Does this object own meaningful behavior or merely transport data?
```


# 72. DTO vs Domain Object

DTO:

```text
data transfer
serialization
API boundary
```

Domain object:

```text
behavior
invariants
identity
business rules
```

Do not force a DTO to become a domain object.

Keeping boundary representations simple can improve interoperability.


# 73. Repository Object

A repository can encapsulate:

```text
persistence behavior
```

Example:

```js
class UserRepository {
  constructor(db) {
    this.db = db;
  }

  findById(id) {
    return this.db.query(...);
  }
}
```

The repository's value is not “it is a class.”

It is:

```text
persistence boundary
```

The same abstraction could also be implemented as functions.


# 74. Service Object

A service object can coordinate:

```text
domain rules
dependencies
effects
```

Keep business logic from becoming an unstructured “service class” containing every operation.

Prefer cohesive responsibilities.


# 75. Domain Entity Example

```js
class BankAccount {
  #balance;

  constructor(initialBalance = 0) {
    if (initialBalance < 0) {
      throw new RangeError("Invalid opening balance");
    }

    this.#balance = initialBalance;
  }

  deposit(amount) {
    if (amount <= 0) {
      throw new RangeError("Amount must be positive");
    }

    this.#balance += amount;
  }

  withdraw(amount) {
    if (amount <= 0) {
      throw new RangeError("Amount must be positive");
    }

    if (amount > this.#balance) {
      throw new RangeError("Insufficient funds");
    }

    this.#balance -= amount;
  }

  get balance() {
    return this.#balance;
  }
}
```

The class owns:

```text
state
identity
invariant
behavior
```

This is a strong OOP use case.


# 76. Functional Domain Rule Inside OOP

Separate pure rule:

```js
function calculateFee(amount, rate) {
  if (amount < 0) {
    throw new RangeError("amount");
  }

  return amount * rate;
}
```

Then object orchestration:

```js
class Payment {
  constructor(amount) {
    this.amount = amount;
  }

  fee(rate) {
    return calculateFee(this.amount, rate);
  }
}
```

The class provides identity/domain structure while pure logic remains easy to test.


# 77. Debugging Exercise — Lost `this`

Given:

```js
class Counter {
  value = 0;

  increment() {
    this.value++;
  }
}

const counter = new Counter();
const increment = counter.increment;

increment();
```

Explain why:

```text
method extraction
```

changes call semantics.

Possible solutions:

```text
bind
arrow-field method
wrapper callback
```

Choose according to lifecycle and allocation considerations.


# 78. Debugging Exercise — Broken Invariant

Given:

```js
class Account {
  #balance = 0;

  withdraw(amount) {
    this.#balance -= amount;
  }
}
```

Identify the invariant failure.

Fix:

```text
validate amount
validate sufficient funds
preserve nonnegative balance
```

Then write tests for:

```text
zero
negative
exact balance
overdraw
```


# 79. Debugging Exercise — Base-Class Coupling

A base class changes:

```js
process() {
  this.validate();
  return this.save();
}
```

to:

```js
process() {
  return this.save();
}
```

A subclass depended on validation.

Why did a seemingly internal refactor break it?

Because the base class's behavior was an implicit extension contract.

This is the fragile-base-class problem.


# 80. Debugging Exercise — Shared Mutable State

Two services receive the same mutable configuration object:

```js
const config = {
  retries: 3,
};

serviceA(config);
serviceB(config);
```

If `serviceA` mutates:

```js
config.retries = 100;
```

then `serviceB` observes the changed state.

Solutions:

```text
immutable config
copy
encapsulation
read-only API
```

Choose based on ownership semantics.


# 81. Debugging Exercise — Async State

Given:

```js
class Worker {
  #running = false;

  async run() {
    if (this.#running) {
      throw new Error("busy");
    }

    this.#running = true;
    await doWork();
    this.#running = false;
  }
}
```

What happens if `doWork()` rejects?

The state may remain:

```text
running = true
```

Fix with:

```js
try {
  await doWork();
} finally {
  this.#running = false;
}
```

State invariants must survive failure paths.


# 82. Code Review Exercise

Review:

```js
class OrderManager extends BaseManager {
  constructor(db, logger, cache, http, clock) {
    super(db, logger);
    this.cache = cache;
    this.http = http;
    this.clock = clock;
  }

  async process(order) {
    this.logger.info(order);

    const user = await this.db.findUser(order.userId);

    if (!user) {
      throw new Error("missing user");
    }

    const total = order.items.reduce(
      (sum, item) => sum + item.price * item.qty,
      0
    );

    this.cache.set(order.id, order);

    await this.http.post("/audit", {
      order,
      total,
      time: this.clock.now(),
    });

    return this.db.save({
      ...order,
      total,
    });
  }
}
```

Review:

```text
class responsibility
inheritance necessity
dependency count
pure logic extraction
logging/privacy
cache semantics
remote side effects
transaction boundaries
error behavior
testability
observability
```


# 83. Improved Architecture

Potential separation:

```text
Order entity/value
    ↓
pure calculateOrderTotal()
    ↓
OrderApplicationService
    ├─ UserRepository
    ├─ OrderRepository
    ├─ AuditGateway
    └─ Clock
```

This keeps:

```text
domain calculation
```

separate from:

```text
I/O orchestration
```

and makes the inheritance question smaller.

The right design can still use classes—but the classes now represent meaningful boundaries.


# 84. Performance Considerations

Potential OOP costs include:

```text
object allocation
per-instance methods
prototype lookup
property access
closure retention
GC
deep object graphs
```

Potential benefits include:

```text
prototype method sharing
encapsulation
stable object layout
cohesive lifecycle
```

These are tendencies, not universal guarantees.

Benchmark hot paths under the target runtime.


# 85. Memory Considerations

Ask:

```text
How many instances exist?
How long do they live?
What do they reference?
Do closures capture large objects?
Do caches retain instances?
Do listeners retain instances?
```

An object can remain reachable through:

```text
event listener
Map
timer
promise callback
global registry
```

Even after the application thinks it is “done.”


# 86. Security Considerations

OOP security concerns include:

```text
mutable shared state
unsafe deserialization
prototype pollution
confused-deputy capabilities
method exposure
lifecycle bypass
```

Never deserialize untrusted JSON and assume it becomes a safe domain object.

Validate boundary data before creating privileged objects.


# 87. Capability-Oriented Design

Instead of giving an object access to a giant global service:

```js
new OrderService(globalEverything);
```

inject only what it needs:

```js
new OrderService({
  orders,
  payments,
  clock,
});
```

This reduces authority.

Encapsulation is also a security boundary when it limits capabilities.


# 88. Testing OOP Code

Test:

```text
constructor invariants
public behavior
state transitions
failure paths
dependency interactions
```

Avoid tests that couple tightly to:

```text
private representation
prototype internals
property ordering
```

Tests should preserve the public contract.


# 89. Mocking and OOP

Dependency injection makes substitution easier:

```js
const fakeRepository = {
  findById: async () => ({ id: "1" }),
};
```

JavaScript often does not require a class-based mock.

Behavioral compatibility is usually enough.

Avoid giant mocking frameworks when a small fake object expresses the contract clearly.


# 90. Composition Root

A composition root is the place where concrete dependencies are assembled.

Example:

```text
main
 ├─ create database
 ├─ create repository
 ├─ create gateway
 ├─ create service
 └─ start server
```

Business classes receive:

```text
interfaces/capabilities
```

rather than constructing every dependency internally.

This improves testability and deployment flexibility.


# 91. OOP and Modules

A module can hide:

```text
class
factory
private helper
registry
```

Export only what callers need.

Example:

```js
export function createPaymentService(deps) {
  return new PaymentService(deps);
}
```

The class can remain an implementation detail.

This reduces public coupling.


# 92. When Not to Use a Class

Prefer a simple function/value when:

```text
no identity needed
no lifecycle
no mutable private state
single deterministic transformation
stateless utility
```

Example:

```js
function calculateVat(amount, rate) {
  return amount * rate;
}
```

A class would add ceremony without adding a useful abstraction.


# 93. When a Class Is Appropriate

A class is a strong candidate when:

```text
state evolves over time
identity matters
invariants belong to the object
lifecycle exists
many instances share behavior
polymorphism is useful
resource ownership matters
```

Example:

```text
ConnectionPool
Transaction
WebSocketSession
OrderAggregate
Cache
```


# 94. When Composition Is Better

Composition is often better when:

```text
capabilities vary independently
behavior needs runtime replacement
you need several collaborators
inheritance would express accidental relationships
testing benefits from small dependencies
```

Example:

```text
PaymentService
+
PaymentGateway
+
FraudChecker
+
Clock
+
Logger
```

No inheritance hierarchy is required.


# 95. Design Heuristics

Use these heuristics:

```text
prefer small cohesive objects
prefer explicit dependencies
prefer shallow inheritance
prefer composition for independently changing behavior
protect invariants
keep constructors predictable
avoid hidden I/O
avoid god objects
avoid unnecessary wrappers
keep public contracts narrow
```

These are heuristics, not absolute laws.


# 96. Anti-Patterns

### Inheritance for code reuse
Use inheritance only when the subtype relationship is valid.

### God object
One class owns unrelated responsibilities.

### Service locator
Hidden dependency lookup creates invisible coupling.

### Mutable DTO
Transport data becomes uncontrolled shared state.

### Deep hierarchy
Changes propagate unpredictably.

### Constructor work explosion
Construction performs I/O or complex initialization.

### Per-instance everything
Every object allocates large behavior structures unnecessarily.

### Fake abstraction
Class exists only because “OOP is required.”


# 97. Principal Comparison — OOP vs Functional

| Concern | OOP tendency | Functional tendency |
|---|---|---|
| identity | strong | often de-emphasized |
| mutable state | encapsulated | minimized |
| state transitions | methods | new values/reducers |
| composition | objects/interfaces | function/value composition |
| lifecycle | natural | can be explicit functions |
| deterministic calculations | possible | often excellent |
| polymorphism | inheritance/behavior | higher-order functions/behavior |
| effects | methods/dependencies | effect boundaries |
| resource ownership | strong fit | possible through scoped functions |
| shared state | can encapsulate | often avoided |

The best production systems frequently combine both.


# 98. Mastery Project — Domain Entity

Implement:

```text
BankAccount
```

with:

```text
deposit
withdraw
transfer
balance
```

Requirements:

```text
private state
invariants
custom errors
tests
```

Then create:

```text
pure fee calculation
```

outside the class.


# 99. Mastery Project — Polymorphic Gateway

Define behavior:

```text
authorize()
capture()
refund()
```

Create:

```text
GatewayA
GatewayB
FakeGateway
```

Do not make callers depend on concrete classes.

Then implement the same system using:

```text
composition + injected object
```

Compare the designs.


# 100. Mastery Project — Resource Lifecycle

Implement a class representing a resource:

```text
open
use
close
```

Requirements:

```text
invalid transition protection
async operation
failure-safe cleanup
observability
```

Test:

```text
close before open
double close
use after close
failure during use
concurrent use
```


# 101. Mastery Project — OOP + Functional Core

Create:

```text
Order entity
pure pricing functions
OrderService
Repository
PaymentGateway
Clock
Logger
```

Separate:

```text
domain rules
from
effects
```

Then test each layer independently.


# 102. Mastery Project — Replace Inheritance

Take a five-level class hierarchy.

Refactor it into:

```text
composition
small capabilities
explicit dependencies
```

Then compare:

```text
coupling
testability
change impact
complexity
```

Document what became better and what became worse.


# 103. Interview Questions — Foundation

1. What is OOP?
2. What is encapsulation?
3. What is abstraction?
4. What is inheritance?
5. What is polymorphism?
6. What is composition?
7. What is object identity?
8. What is a prototype?
9. What does `new` do conceptually?
10. What is a constructor?


# 104. Interview Questions — Intermediate

11. How are class methods represented?
12. What is `super`?
13. Why can a method lose `this`?
14. What are private class fields?
15. What is `instanceof`?
16. What is structural polymorphism?
17. When should composition replace inheritance?
18. What is a factory function?
19. What is a mixin?
20. How do closures compare with private fields?


# 105. Interview Questions — Advanced

21. Explain the prototype chain behind a class instance.
22. Explain derived constructor initialization.
23. Explain method overriding versus property shadowing.
24. Design an immutable value and a mutable entity for the same domain.
25. Design a resource lifecycle object.
26. Explain constructor side effects.
27. Explain fragile base classes.
28. Explain object-shape/performance considerations without overclaiming engine internals.
29. Design dependency injection without a framework.
30. Decide whether a class or function should implement a given domain requirement.


# 106. Interview Questions — Principal

31. How would you evaluate an inheritance hierarchy in a large codebase?
32. When is OOP the wrong abstraction?
33. How would you combine OOP and functional programming in a backend?
34. How would you design stateful services safely under asynchronous concurrency?
35. How would you prevent a domain object from exposing invalid states?
36. How would you evolve a class API without breaking consumers?
37. How would you review a base class used by 100 subclasses?
38. How would you reduce GC pressure in an object-heavy hot path?
39. When should behavior be structural rather than nominal?
40. How would you design an extensibility model that avoids both deep inheritance and abstraction explosion?


# 107. Predict-the-Output Exercises

## Exercise 1

```js
class User {
  greet() {
    return "hello";
  }
}

const user = new User();

console.log(user.greet());
```

Predict.

---

## Exercise 2

```js
const parent = {
  value: 10,
};

const child = Object.create(parent);

console.log(child.value);

child.value = 20;

console.log(child.value);
console.log(parent.value);
```

Trace ownership and shadowing.

---

## Exercise 3

```js
class Counter {
  #value = 0;

  increment() {
    return ++this.#value;
  }
}

const counter = new Counter();

console.log(counter.increment());
console.log(counter.increment());
```

Predict.

---

## Exercise 4

```js
class Animal {
  speak() {
    return "animal";
  }
}

class Dog extends Animal {
  speak() {
    return "dog";
  }
}

const animal = new Animal();
const dog = new Dog();

console.log(animal.speak());
console.log(dog.speak());
```

Predict.

---

## Exercise 5

```js
class User {
  constructor(id) {
    this.id = id;
  }
}

const a = new User(1);
const b = new User(1);

console.log(a === b);
```

Predict and explain identity.


# 108. Debugging / Reasoning Exercises

### Exercise 1
A callback loses `this`.

Find three correct fixes and compare their allocation/lifecycle implications.

### Exercise 2
A private field cannot be accessed from a subclass.

Explain why and redesign the base-class API.

### Exercise 3
A subclass breaks when the parent constructor changes.

Identify the hidden coupling.

### Exercise 4
A class instance serializes to plain data and loses methods after JSON parsing.

Explain why and design explicit reconstruction.

### Exercise 5
A service class contains 15 injected dependencies.

Determine whether it is:

```text
cohesive service
or
god object
```

and propose boundaries.


# 109. Code Review Exercise

Review:

```js
class BaseRepository {
  constructor(db) {
    this.db = db;
  }

  async save(entity) {
    this.validate(entity);
    await this.db.insert(entity);
  }

  validate(entity) {}
}

class PaymentRepository extends BaseRepository {
  validate(entity) {
    if (entity.amount <= 0) {
      throw new Error("invalid amount");
    }
  }
}

class PaymentService extends PaymentRepository {
  constructor(db, http, cache, logger, clock) {
    super(db);
    this.http = http;
    this.cache = cache;
    this.logger = logger;
    this.clock = clock;
  }
}
```

Find:

```text
inheritance misuse
repository/service boundary problems
base-class coupling
validation placement
dependency count
testability
responsibility confusion
```


# 110. Production Checklist

```text
[ ] object identity is intentional
[ ] state ownership is explicit
[ ] invariants are documented
[ ] private state is protected when needed
[ ] constructor establishes valid state
[ ] constructors avoid hidden I/O
[ ] lifecycle is explicit
[ ] dependencies are injected
[ ] public API is narrow
[ ] inheritance is shallow
[ ] composition considered
[ ] method binding is understood
[ ] async state transitions restore invariants
[ ] serialization boundaries are explicit
[ ] security capabilities are limited
[ ] performance hotspots are measured
[ ] tests target public behavior
```


# 111. Spaced Retrieval Plan

### Day 0
Explain:

```text
object
prototype
class
encapsulation
inheritance
composition
polymorphism
```

### Day 2
Implement:

```text
private-field class
factory function
composition-based service
```

### Day 7
Refactor an inheritance hierarchy into composition.

### Day 14
Build a resource lifecycle class with async failure handling.

### Day 30
Defend a principal-level decision between:

```text
class
factory
pure functions
composition
inheritance
```

for a real domain.


# 112. Principal Decision Framework

Evaluate an OOP design across:

| Dimension | Question |
|---|---|
| Correctness | Does the object enforce its invariants? |
| Performance | What is the allocation/dispatch/GC impact? |
| Memory | What does each instance retain? |
| Security | What capabilities are exposed? |
| Reliability | Are lifecycle transitions failure-safe? |
| Maintainability | Is responsibility cohesive? |
| Scalability | Does the object model survive growth in features/instances? |
| Observability | Can lifecycle and behavior be measured? |
| Developer Experience | Is the public API obvious? |
| Operational Complexity | Does the abstraction add lifecycle/configuration burden? |
| Future Change | Can implementations evolve without breaking callers? |


# 113. Common Misconceptions

### “JavaScript is class-based.”
JavaScript's core object model uses prototype delegation; class syntax provides structured language support around that model.

### “Inheritance is the main way to reuse code.”
Composition, functions, modules, and delegation are often better.

### “Encapsulation means private fields only.”
Encapsulation is about controlling state/behavior boundaries. Private fields are one mechanism.

### “A class always makes code more maintainable.”
A class can also add ceremony and coupling.

### “Polymorphism requires inheritance.”
Behavioral/structural polymorphism can work without inheritance.

### “Functional programming and OOP cannot coexist.”
They routinely coexist in production systems.

### “Private fields are automatically faster.”
Performance is runtime/workload dependent.

### “A constructor should initialize everything.”
Synchronous construction should not hide complex asynchronous I/O.


# 114. Common Mistakes

```text
[ ] inheritance only for code reuse
[ ] deep hierarchy
[ ] god class
[ ] hidden dependencies
[ ] mutable public state
[ ] constructor side effects
[ ] lost this
[ ] async invariant failures
[ ] exposing internal representation
[ ] confusing DTO and domain entity
[ ] class for a stateless function
[ ] method-level abstraction without domain value
[ ] trusting instanceof as semantic proof
[ ] coupling tests to implementation
[ ] assuming engine optimizations are language guarantees
```


# 115. Track A — Core Theory

Study:

```text
objects
prototypes
constructors
classes
encapsulation
identity
inheritance
polymorphism
composition
substitutability
lifecycle
invariants
```

Be able to explain the underlying JavaScript object model rather than only class syntax.


# 116. Track B — Implementation

Progress:

```text
guided
→ partially guided
→ no-reference
→ edge-case hardened
→ production-grade
```

Implement:

```text
domain entity
factory
dependency-injected service
resource lifecycle object
polymorphic gateway
composition-based replacement for inheritance
```


# 117. Track C — Interview / Reasoning

Practice:

```text
prototype tracing
this tracing
inheritance review
composition selection
invariant design
lifecycle design
API stability
performance reasoning
security boundary reasoning
```

For every design defend:

```text
why object?
why this state ownership?
why inheritance/composition?
why this API?
what changes later?
```


# 118. Completion Criteria

```text
[ ] Explain JavaScript's object/prototype model
[ ] Explain object identity
[ ] Explain encapsulation
[ ] Explain private fields
[ ] Explain constructors
[ ] Explain new
[ ] Explain class methods
[ ] Explain static methods
[ ] Explain getters/setters
[ ] Explain inheritance
[ ] Explain super
[ ] Explain derived constructors
[ ] Explain overriding
[ ] Explain shadowing
[ ] Explain polymorphism
[ ] Explain structural behavior
[ ] Explain composition
[ ] Explain mixins
[ ] Explain factory functions
[ ] Explain closure encapsulation
[ ] Explain state invariants
[ ] Explain lifecycle modeling
[ ] Explain dependency injection
[ ] Explain OOP/functional combination
[ ] Analyze object memory/performance
[ ] Analyze async mutable state
[ ] Design stable class APIs
[ ] Identify OOP anti-patterns
[ ] Refactor inheritance to composition
[ ] Defend class vs function
```


# 119. Mastery Gate

### Understand
You can explain OOP in terms of JavaScript's actual object model.

### Explain
You can teach encapsulation, identity, inheritance, composition, and polymorphism.

### Predict
You can trace `this`, prototype lookup, overriding, shadowing, and initialization.

### Implement
You can build stateful objects with explicit invariants.

### Debug
You can diagnose lost `this`, broken lifecycle state, subclass coupling, and leaked representation.

### Apply
You can design production services/entities/resources with appropriate object boundaries.

### Compare
You can defend classes, factories, closures, pure functions, inheritance, and composition.

### Defend
You can justify an OOP architecture across:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
observability
future change
```


# 120. Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current status:

```text
[ ] Not Started
```

Reading alone does not mark mastery.


# Chapter 75 — Revision / Retrieval Record

| Date | Retrieval task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain prototype lookup | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain `this` loss | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement private-state entity | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Compare inheritance/composition | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design lifecycle object | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Refactor deep hierarchy | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Analyze OOP performance | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal architecture review | ✅ / ❌ | ... | ... |

### Retrieval Prompts

```text
1. What is the relationship between class and prototype?
2. What does new do conceptually?
3. Why can a method lose this?
4. What makes an object invariant valuable?
5. When is inheritance valid?
6. When is composition better?
7. What is structural polymorphism?
8. What does a private field protect?
9. How should async methods preserve object state?
10. When should you use a factory instead of a class?
11. How do OOP and FP complement each other?
12. What makes a class a god object?
13. How would you reduce inheritance coupling?
14. What object behavior belongs in a domain entity?
15. How would you defend the design at principal level?
```


# Chapter 75 — Canonical References and Source Discipline

## ECMAScript

- ECMAScript Language Specification  
  https://tc39.es/ecma262/
- ECMAScript Classes  
  https://tc39.es/ecma262/#sec-class-definitions
- ECMAScript Objects and Internal Methods  
  https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots

Use the specification for:

```text
class semantics
prototype behavior
property lookup
constructors
private names
this
super
instanceof
```

## MDN

- Classes  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Classes
- Inheritance and the prototype chain  
  https://developer.mozilla.org/docs/Web/JavaScript/Inheritance_and_the_prototype_chain
- Private properties  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Classes/Private_elements
- `instanceof`  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/instanceof

## TypeScript

- Classes  
  https://www.typescriptlang.org/docs/handbook/2/classes.html
- Interfaces  
  https://www.typescriptlang.org/docs/handbook/2/objects.html

## Source discipline

1. Treat ECMAScript as the authority for language semantics.
2. Treat class syntax as JavaScript syntax over the JavaScript object model, not as evidence of another language's runtime model.
3. Separate prototype semantics from engine optimizations.
4. Treat TypeScript interfaces and abstract classes as type-system constructs unless runtime behavior is explicitly present.
5. Benchmark object/allocation claims on the target runtime.
6. Treat inheritance as a behavioral contract, not merely code reuse.
7. Prefer composition where independently changing capabilities make inheritance brittle.
8. Keep constructors predictable and avoid hidden asynchronous initialization.
9. Protect invariants through explicit ownership boundaries.
10. Test public behavior rather than internal representation.


# Chapter 75 — Completion Snapshot

## JavaScript OOP Model

```text
[ ] objects
[ ] identity
[ ] prototypes
[ ] prototype chain
[ ] constructors
[ ] class syntax
[ ] new
[ ] this
[ ] super
```

## OOP Concepts

```text
[ ] encapsulation
[ ] abstraction
[ ] inheritance
[ ] polymorphism
[ ] composition
[ ] substitutability
[ ] invariants
[ ] lifecycle
```

## Design

```text
[ ] class
[ ] factory
[ ] closure
[ ] composition
[ ] dependency injection
[ ] structural polymorphism
[ ] shallow inheritance
```

## Production

```text
[ ] memory
[ ] allocation
[ ] GC
[ ] async state
[ ] security
[ ] serialization
[ ] API stability
[ ] observability
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


# Final Principal Perspective

Object-oriented programming in JavaScript is not:

```text
class + extends + methods
```

It is the deliberate modeling of:

```text
identity
state
behavior
invariants
lifecycle
capabilities
```

The most important decision is often not:

```text
"Which class should extend which?"
```

but:

```text
"Should this be an object at all?"
```

Then ask:

```text
Does identity matter?
Does state evolve?
Who owns the invariant?
Who can mutate it?
Does lifecycle matter?
Do implementations need polymorphism?
Is inheritance actually valid?
Would composition be clearer?
Would a pure function be simpler?
What happens under async interleaving?
What does each instance retain?
How stable is this API?
```

The strongest JavaScript systems often combine paradigms:

```text
OOP
→ identity + lifecycle + encapsulated state

Functional programming
→ deterministic calculations + transformations

Composition
→ replaceable capabilities

Modules
→ boundaries

Data structures
→ efficient representation
```

The deepest lesson is:

> **Good OOP is not about creating more classes; it is about assigning state, behavior, ownership, and boundaries so that change remains understandable.**

The curriculum now moves to the next design-level question:

```text
objects
+
functions
+
capabilities
      ↓
composition and abstraction
      ↓
Chapter 76 — Composition and Abstraction Design
```


# 121. Extended Retrieval Bank

### Retrieval Drill 1

Given a domain with approximately `100` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 2

Given a domain with approximately `200` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 3

Given a domain with approximately `300` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 4

Given a domain with approximately `400` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 5

Given a domain with approximately `500` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 6

Given a domain with approximately `600` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 7

Given a domain with approximately `700` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 8

Given a domain with approximately `800` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 9

Given a domain with approximately `900` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 10

Given a domain with approximately `1000` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 11

Given a domain with approximately `1100` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 12

Given a domain with approximately `1200` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 13

Given a domain with approximately `1300` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 14

Given a domain with approximately `1400` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 15

Given a domain with approximately `1500` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 16

Given a domain with approximately `1600` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 17

Given a domain with approximately `1700` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 18

Given a domain with approximately `1800` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 19

Given a domain with approximately `1900` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 20

Given a domain with approximately `2000` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 21

Given a domain with approximately `2100` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 22

Given a domain with approximately `2200` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 23

Given a domain with approximately `2300` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 24

Given a domain with approximately `2400` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 25

Given a domain with approximately `2500` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 26

Given a domain with approximately `2600` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 27

Given a domain with approximately `2700` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 28

Given a domain with approximately `2800` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 29

Given a domain with approximately `2900` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 30

Given a domain with approximately `3000` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 31

Given a domain with approximately `3100` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 32

Given a domain with approximately `3200` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 33

Given a domain with approximately `3300` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 34

Given a domain with approximately `3400` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 35

Given a domain with approximately `3500` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 36

Given a domain with approximately `3600` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 37

Given a domain with approximately `3700` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.

### Retrieval Drill 38

Given a domain with approximately `3800` active objects, decide:

```text
1. class vs factory vs pure function
2. state owner
3. invariant
4. public API
5. inheritance vs composition
6. lifecycle boundary
7. memory/retention risk
8. test strategy
9. security capability boundary
10. future-change risk
```

Write the decision without relying on a design-pattern name.