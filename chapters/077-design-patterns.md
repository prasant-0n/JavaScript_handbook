# Chapter 77 — Design Patterns

> **Curriculum position:** Part XIV — Programming Paradigms  
> **Previous chapter:** Chapter 76 — Composition and Abstraction Design  
> **Next chapter:** Chapter 78 — Production JavaScript Architecture  
> **Primary environment:** Modern JavaScript / TypeScript, Node.js, browsers, libraries, backend services, and large-scale platforms.

---

# Chapter Mission

Master **design patterns** as reusable design vocabulary—not as a catalog to memorize and mechanically apply.

A pattern is a recurring solution shape to a recurring design problem.

The important sequence is:

```text
problem
  ↓
forces / constraints
  ↓
design pressure
  ↓
pattern
  ↓
implementation
  ↓
trade-offs
```

The pattern is not the goal.

The goal is:

```text
lower coupling
clear responsibility
controlled variation
stable contracts
explicit lifecycle
testability
maintainability
```

JavaScript already gives you powerful language primitives:

```text
functions
closures
objects
prototypes
classes
modules
iterators
generators
Promises
async functions
Map
Set
symbols
private fields
```

Many classic patterns therefore appear in JavaScript in lighter forms.

For example:

```text
Strategy
→ function

Factory
→ function/module

Iterator
→ built-in iterable protocol

Observer
→ EventTarget / callbacks

Decorator
→ function/object wrapper

Module
→ ES module

Singleton
→ module-scoped state
```

That does **not** mean the pattern disappeared.

It means:

> **The language can provide the mechanism while the pattern remains the design idea.**

A principal engineer should be able to:

```text
recognize a pattern
name the problem it solves
identify its forces
implement it naturally in JavaScript
reject it when unnecessary
spot accidental pattern usage
remove harmful ceremony
```

The principal-level goal is:

> **Use patterns as communication and design leverage, never as architecture theater.**


# 1. Learning Objectives

By completion you should be able to:

## Foundations

- define design pattern;
- explain pattern versus implementation;
- explain pattern versus framework;
- explain pattern versus architecture;
- explain forces and trade-offs;
- recognize accidental patterns;
- distinguish problem-driven design from pattern-driven design.

## Creational patterns

- Factory Function;
- Factory Method;
- Abstract Factory;
- Builder;
- Prototype;
- Singleton;
- Object Pool.

## Structural patterns

- Adapter;
- Facade;
- Decorator;
- Proxy;
- Composite;
- Bridge;
- Flyweight.

## Behavioral patterns

- Strategy;
- Observer;
- Pub/Sub;
- Command;
- Chain of Responsibility;
- State;
- Template Method;
- Iterator;
- Mediator;
- Visitor;
- Memento;
- Interpreter.

## JavaScript-native pattern forms

- module pattern;
- closure-based encapsulation;
- function strategy;
- function composition;
- event emitter / EventTarget;
- iterator protocol;
- generator;
- middleware;
- dependency injection;
- registry;
- async pipeline;
- reducer/state transition.

## Principal judgment

- determine whether a pattern is justified;
- identify pattern overuse;
- compare multiple implementations;
- understand coupling introduced by patterns;
- account for runtime/memory cost;
- preserve debuggability;
- protect security boundaries;
- evolve patterns safely.


# 2. Prerequisites

Recommended:

- Chapter 09 — Functions and First-Class Behavior
- Chapter 13 — Closures
- Chapter 15 — Objects and Property Semantics
- Chapter 17 — Prototypes and Prototype Chains
- Chapter 18 — Classes and OOP
- Chapter 24 — Map and Set
- Chapter 25 — Iterables and Iterators
- Chapter 26 — Generators
- Chapter 31–38 — Asynchronous JavaScript
- Chapter 71 — Fundamental Data Structures
- Chapter 72 — Complexity
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 75 — Object-Oriented Programming
- Chapter 76 — Composition and Abstraction Design


# 3. What Is a Design Pattern?

A design pattern is a named, reusable design solution to a recurring problem under recurring constraints.

A pattern normally communicates:

```text
context
problem
forces
solution shape
consequences
```

A pattern is not:

```text
copy this exact code
```

It is:

```text
recognize this recurring design pressure
```

and:

```text
use a known solution shape if its trade-offs fit.
```


# 4. Why Do Patterns Exist?

Patterns provide:

```text
shared vocabulary
reusable design knowledge
trade-off visibility
communication
```

Instead of writing:

```text
"We have a wrapper around the service that adds metrics and forwards all calls."
```

you can say:

```text
"We are decorating the service with metrics."
```

The name is useful only because the team understands the consequences.


# 5. Pattern vs Algorithm vs Data Structure

Algorithm:

```text
procedure for computation
```

Data structure:

```text
representation for organizing data
```

Pattern:

```text
recurring design arrangement
```

Architecture:

```text
system-level organization
```

Framework:

```text
technology/library that structures application execution
```

These concepts overlap, but they operate at different levels.


# 6. Pattern vs Framework

A pattern is an idea.

A framework is executable infrastructure that constrains or coordinates application code.

Example:

```text
Middleware pipeline
```

can be a pattern.

A web framework implementing:

```text
middleware
routing
lifecycle
HTTP server integration
```

is a framework.

Do not confuse:

```text
pattern vocabulary
```

with:

```text
vendor technology
```


# 7. Pattern vs Architecture

A pattern such as:

```text
Adapter
```

usually affects one boundary.

An architecture such as:

```text
hexagonal architecture
```

defines broader dependency direction and organization.

Use:

```text
micro pattern
→ local problem

architectural pattern
→ system-wide structural problem
```

Do not let a local pattern dictate the whole system.


# 8. Pattern Forces

Before applying a pattern, identify forces:

```text
What varies?
What must remain stable?
What is coupled?
What must be extended?
What must be isolated?
What lifecycle exists?
What performance constraint exists?
What security boundary exists?
```

If there is no meaningful force:

```text
no pattern needed
```


# 9. Pattern Selection Mental Model

Ask:

```text
1. What problem exists?
2. Why is the current design insufficient?
3. What is changing?
4. What should remain stable?
5. Which pattern addresses that exact force?
6. What coupling does the pattern introduce?
7. What runtime cost does it add?
8. What happens if requirements change?
9. Is simpler code enough?
```


# 10. Creational Pattern Family

Creational patterns address:

```text
how objects/values are created
```

Common examples:

```text
Factory
Abstract Factory
Builder
Prototype
Singleton
Object Pool
```

Modern JavaScript often uses:

```text
functions
modules
object literals
class constructors
```

instead of formal pattern-heavy implementations.


# 11. Factory Function

A factory function returns an object/value.

```js
function createUser(id, name) {
  return {
    id,
    name,

    greet() {
      return `Hello ${name}`;
    },
  };
}
```

Use when:

```text
construction needs logic
implementation should be hidden
different variants may be selected
closure state is useful
```

Factory functions are one of the most natural JavaScript patterns.


# 12. Factory Method

Factory Method delegates creation to an overridable or replaceable creator.

In JavaScript, this can be represented through:

```text
class methods
subclasses
function callbacks
configuration
registries
```

Do not force the classical inheritance-heavy form.

Use the underlying idea:

```text
creation policy is replaceable
```


# 13. Abstract Factory

Abstract Factory creates families of related objects.

Example:

```text
DatabaseFactory
├─ createClient()
├─ createRepository()
└─ createTransaction()
```

A PostgreSQL family:

```text
PostgresClient
PostgresRepository
PostgresTransaction
```

and another provider can supply a compatible family.

This is useful when objects must remain compatible as a set.


# 14. Abstract Factory Trade-Off

Benefits:

```text
family consistency
centralized creation
provider substitution
```

Costs:

```text
many abstractions
larger interfaces
more indirection
```

Do not create an abstract factory when only:

```text
one object
one implementation
```

exists.


# 15. Builder

Builder separates complex construction from final representation.

Example:

```js
class RequestBuilder {
  #method = "GET";
  #headers = {};
  #body;

  method(value) {
    this.#method = value;
    return this;
  }

  header(name, value) {
    this.#headers[name] = value;
    return this;
  }

  body(value) {
    this.#body = value;
    return this;
  }

  build() {
    return {
      method: this.#method,
      headers: { ...this.#headers },
      body: this.#body,
    };
  }
}
```

Use only when construction complexity justifies it.


# 16. Builder Pitfalls

Builder problems:

```text
mutable intermediate state
invalid intermediate states
long chains
hidden defaults
configuration order dependence
```

A builder should define:

```text
required fields
defaults
validation
build-time invariants
```

For simple objects, use an object literal instead.


# 17. Prototype Pattern

Prototype-based creation can clone or derive behavior from an existing object.

JavaScript naturally supports:

```js
const base = {
  greet() {
    return "hello";
  },
};

const child = Object.create(base);
```

This creates delegation through:

```text
child → base
```

The JavaScript object model itself makes Prototype a first-class concept.


# 18. Prototype Cloning vs Deep Copy

Prototype delegation:

```text
new object
↓
delegates behavior
```

is not the same as:

```text
deep copy all properties
```

Do not call:

```js
Object.create(existing)
```

a deep clone.

The new object has a prototype relationship.


# 19. Singleton

A Singleton restricts construction to one logical instance.

In JavaScript, module caching and module-scoped state often provide simpler singleton-like behavior:

```js
const connection = createConnection();

export function getConnection() {
  return connection;
}
```

But Singleton introduces global-like state.

Consider:

```text
test isolation
lifecycle
concurrency
configuration
hidden dependency
```


# 20. Singleton Critique

Singletons can create:

```text
global coupling
implicit dependencies
difficult tests
unclear lifecycle
```

A dependency injected from a composition root is often easier to reason about.

Use a singleton when:

```text
one process-wide resource truly has one logical ownership/lifecycle
```

not simply because only one instance happens to be created today.


# 21. Object Pool

An object pool reuses resources.

Examples:

```text
database connections
worker resources
buffers
specialized objects
```

Concept:

```text
available
   ↓
acquired
   ↓
released
   ↓
available
```

Pooling is useful when resource creation is expensive and reuse is safe.


# 22. Object Pool Risks

Pooling can cause:

```text
stale state
resource retention
pool exhaustion
leaks
complex lifecycle
```

Reset every reusable resource to a valid state.

Do not pool lightweight objects merely to avoid allocations unless profiling demonstrates benefit.


# 23. Structural Pattern Family

Structural patterns address:

```text
how components are combined
```

Common patterns:

```text
Adapter
Facade
Decorator
Proxy
Composite
Bridge
Flyweight
```

These connect directly to Chapter 76.


# 24. Adapter

Adapter translates one interface into another.

```js
class StripeAdapter {
  constructor(client) {
    this.client = client;
  }

  async charge(request) {
    const response = await this.client.paymentIntents.create({
      amount: request.amount,
      currency: request.currency,
    });

    return {
      id: response.id,
      status: response.status,
    };
  }
}
```

The application consumes:

```text
charge(request)
```

instead of provider-specific API details.


# 25. Adapter Use Cases

Adapters are useful for:

```text
third-party SDKs
legacy code
provider migration
version migration
protocol translation
database migration
```

A good adapter absorbs external vocabulary.

A bad adapter merely re-exports it.


# 26. Facade

Facade exposes a simpler interface over a subsystem.

```js
class CheckoutFacade {
  async checkout(order) {
    // validate
    // reserve inventory
    // charge payment
    // persist
    // publish event
  }
}
```

The facade is useful when callers should not coordinate the subsystem themselves.

Avoid a facade becoming a god object.


# 27. Decorator

Decorator adds behavior without modifying the wrapped component.

```js
function withTiming(service, record) {
  return {
    async execute(input) {
      const start = performance.now();

      try {
        return await service.execute(input);
      } finally {
        record(performance.now() - start);
      }
    },
  };
}
```

Typical uses:

```text
logging
metrics
caching
authorization
retry
tracing
```


# 28. Decorator Ordering

Decorator order changes semantics.

```js
withRetry(
  withMetrics(service)
);
```

versus:

```js
withMetrics(
  withRetry(service)
);
```

can mean:

```text
metrics per attempt
```

versus:

```text
metrics for entire retry operation
```

Make ordering explicit.


# 29. Proxy

Proxy controls interaction with another object.

JavaScript has a language-level `Proxy` object:

```js
const proxy = new Proxy(target, {
  get(target, property, receiver) {
    return Reflect.get(target, property, receiver);
  },
});
```

Uses can include:

```text
validation
logging
virtual properties
access control
instrumentation
reactivity
```

Proxy is powerful but can complicate:

```text
debugging
performance
semantics
tooling
```


# 30. Composite

Composite treats individual objects and groups uniformly.

Example:

```text
File
Directory
```

Both can implement:

```text
size()
```

A directory delegates to children:

```text
directory.size()
→ sum child sizes
```

This is useful for recursive structures.


# 31. Composite Trade-Off

Composite simplifies callers:

```text
leaf or collection
```

share an interface.

But it can blur important distinctions.

If:

```text
leaf
```

cannot support an operation meaningfully, forcing the same contract may create bad abstractions.


# 32. Bridge

Bridge separates:

```text
abstraction
```

from:

```text
implementation
```

so both can vary independently.

Conceptual example:

```text
Notification
   ↓
Delivery
   ├─ Email
   └─ SMS

Template
   ├─ Alert
   └─ Marketing
```

Composition is usually a natural JavaScript implementation.


# 33. Flyweight

Flyweight shares reusable state to reduce memory.

Example:

```text
millions of objects
```

share:

```text
common metadata
```

while unique state remains external.

Concept:

```text
intrinsic shared state
+
extrinsic per-instance state
```

Useful in memory-constrained object-heavy systems.


# 34. Behavioral Pattern Family

Behavioral patterns address:

```text
how objects/functions communicate
```

Common examples:

```text
Strategy
Observer
Command
Chain of Responsibility
State
Template Method
Iterator
Mediator
Visitor
Memento
Interpreter
```


# 35. Strategy

Strategy extracts varying behavior.

Class form:

```js
class Checkout {
  constructor(pricing) {
    this.pricing = pricing;
  }

  total(order) {
    return this.pricing.calculate(order);
  }
}
```

Functional form:

```js
const total = (order, pricing) =>
  pricing(order);
```

The core pattern is:

```text
replaceable algorithm/policy
```


# 36. Observer

Observer creates:

```text
subject
 ↓
subscribers
```

Example:

```js
class ObservableValue {
  #listeners = new Set();
  #value;

  subscribe(listener) {
    this.#listeners.add(listener);

    return () => {
      this.#listeners.delete(listener);
    };
  }

  set(value) {
    this.#value = value;

    for (const listener of this.#listeners) {
      listener(value);
    }
  }
}
```

The unsubscribe function is essential for lifecycle safety.


# 37. Observer Risks

Observer can create:

```text
memory leaks
hidden control flow
event storms
ordering ambiguity
reentrancy
error propagation problems
```

Always define:

```text
subscription lifecycle
error behavior
delivery ordering
synchronous/asynchronous delivery
```


# 38. Pub/Sub

Pub/Sub separates:

```text
publisher
```

from:

```text
specific subscribers
```

through a topic/channel.

Example:

```text
OrderCreated
   ↓
event bus
 ├→ inventory handler
 ├→ analytics handler
 └→ notification handler
```

Pub/Sub increases decoupling but can reduce discoverability and traceability.


# 39. Observer vs Pub/Sub

Observer:

```text
subject knows observers
```

Pub/Sub:

```text
publisher and subscriber communicate through intermediary
```

The distinction is useful because coupling differs.

Use direct observation when:

```text
relationship is local and explicit
```

Use Pub/Sub when:

```text
multiple independent consumers
```

need event distribution.


# 40. Command

Command represents an action as data/object behavior.

```js
class ChargeCustomer {
  constructor(gateway, payment) {
    this.gateway = gateway;
    this.payment = payment;
  }

  execute() {
    return this.gateway.charge(this.payment);
  }
}
```

Commands are useful for:

```text
queues
retries
undo
audit
scheduling
authorization
```


# 41. Command as Data

A command can be serialized:

```json
{
  "type": "ChargeCustomer",
  "paymentId": "p-42",
  "amount": 1000
}
```

This allows:

```text
producer
→ durable queue
→ worker
→ execution
```

The important distinction is:

```text
command intent
```

versus:

```text
implementation object
```


# 42. Chain of Responsibility

A request passes through handlers:

```text
A → B → C → D
```

Each may:

```text
handle
or
delegate
```

Middleware is a natural JavaScript example.

Use for:

```text
HTTP middleware
validation pipeline
authorization layers
processing stages
```


# 43. Chain Risks

Risks:

```text
order dependence
hidden short-circuiting
duplicate processing
error swallowing
latency accumulation
```

Document:

```text
ordering
termination
mutation
error propagation
```


# 44. State Pattern

State pattern moves state-dependent behavior into explicit state representations.

Instead of:

```js
if (state === "idle") ...
if (state === "running") ...
if (state === "closed") ...
```

you can model:

```text
IdleState
RunningState
ClosedState
```

or simply use a state machine with a transition table.

For small state spaces, an enum plus switch may be clearer.


# 45. State Machine Before State Pattern

Before creating classes for every state, define:

```text
states
events
transitions
illegal transitions
effects
```

Then decide representation:

```text
switch
transition table
state objects
state machine library
```

The pattern should follow the domain, not the other way around.


# 46. Template Method

Template Method defines a stable algorithm skeleton with customizable steps.

Classical form:

```js
class Importer {
  run(input) {
    const parsed = this.parse(input);
    const valid = this.validate(parsed);
    return this.save(valid);
  }

  parse(input) {}
  validate(value) {}
  save(value) {}
}
```

JavaScript often replaces this with:

```text
higher-order functions
composition
pipeline arrays
```

when inheritance is unnecessary.


# 47. Iterator

Iterator abstracts sequential traversal.

JavaScript has standardized iterator protocols:

```js
const iterator = values[Symbol.iterator]();
```

and:

```js
for (const value of values) {
  // ...
}
```

Therefore the Iterator pattern is built into the language.

You should understand the pattern even when using the language mechanism directly.


# 48. Generator as Iterator Pattern

Generators provide a natural iterator implementation:

```js
function* values() {
  yield 1;
  yield 2;
  yield 3;
}
```

Consumers can use:

```js
for (const value of values()) {
  console.log(value);
}
```

The design pattern survives as:

```text
lazy traversal abstraction
```

while syntax becomes simpler.


# 49. Mediator

Mediator centralizes interactions among components.

Instead of:

```text
A ↔ B
A ↔ C
B ↔ C
```

use:

```text
A → Mediator ← B
       ↑
       C
```

Useful when direct peer communication becomes too complex.

Risk:

```text
mediator becomes god object
```


# 50. Visitor

Visitor separates:

```text
operation
```

from:

```text
data structure
```

Common in:

```text
AST processing
compilers
interpreters
tree analysis
```

JavaScript can implement visitor dispatch through:

```text
type tags
methods
maps
pattern matching libraries
```

Trade-off:

```text
easy to add operations
harder to add new node types
```

That trade-off matters.


# 51. Memento

Memento captures state for restoration.

Use cases:

```text
undo
snapshots
checkpointing
rollback
```

Example:

```text
current state
→ snapshot
→ mutate
→ restore snapshot
```

The cost depends on:

```text
copying
structural sharing
state size
snapshot frequency
```


# 52. Interpreter

Interpreter represents and evaluates a small language or rule system.

Example:

```text
expression
→ AST
→ evaluator
```

Useful for:

```text
query languages
rules
filters
configuration DSLs
policy engines
```

Do not build an interpreter when a simple parser/evaluator or existing language is enough.


# 53. Module Pattern

JavaScript's modules naturally provide encapsulation:

```js
const secret = 42;

export function getSecret() {
  return secret;
}
```

This is often simpler than the historical IIFE module pattern.

The underlying pattern is:

```text
private implementation
+
public API
```


# 54. Registry Pattern

A registry maps names to implementations:

```js
const handlers = new Map();

handlers.set("created", handleCreated);
handlers.set("paid", handlePaid);
```

Use when:

```text
behavior is extensible
lookup is natural
registration is explicit
```

Risk:

```text
hidden initialization
global mutable registry
ordering dependency
```


# 55. Middleware Pattern

Middleware is a chain-based composition pattern:

```text
request
 ↓
auth
 ↓
validation
 ↓
metrics
 ↓
handler
```

It combines ideas from:

```text
Chain of Responsibility
Decorator
composition
```

Middleware order is part of semantics.


# 56. Dependency Injection Pattern

Dependency injection moves construction outside the dependent component.

```js
class UserService {
  constructor(repository) {
    this.repository = repository;
  }
}
```

Benefits:

```text
testability
substitution
explicit dependencies
```

Do not confuse DI with:

```text
large framework container
```

Passing a dependency is sufficient.


# 57. Dependency Injection vs Service Locator

Dependency injection:

```text
dependency visible in constructor/function
```

Service locator:

```text
component asks container at runtime
```

The latter hides dependencies.

For core business logic, explicit injection usually improves reasoning.

A framework container can still be used at the composition root.


# 58. Null Object

Null Object replaces:

```text
null / missing behavior
```

with:

```text
safe no-op object
```

Example:

```js
const noopLogger = {
  info() {},
  error() {},
};
```

Then callers can do:

```js
logger.info("hello");
```

without repeated:

```js
if (logger) ...
```

Use carefully: silent no-op behavior can hide configuration failures.


# 59. Specification Pattern

Specification represents a business predicate as an object/function.

```js
const isEligible = user =>
  user.age >= 18 &&
  user.active === true;
```

Specifications can be composed:

```js
const and = (a, b) => value =>
  a(value) && b(value);
```

This is especially useful for:

```text
business rules
filtering
authorization
validation
```


# 60. Policy Object

A policy object encapsulates a decision rule:

```js
const retryPolicy = {
  shouldRetry(error, attempt) {
    return error.retryable && attempt < 3;
  },
};
```

This separates:

```text
decision
```

from:

```text
mechanism
```

and connects directly to Strategy.


# 61. Repository Pattern

Repository provides collection/domain-oriented persistence access.

```js
class UserRepository {
  async findById(id) {}
  async save(user) {}
}
```

Use when:

```text
domain/application should not depend directly on persistence mechanics
```

Avoid generic repositories that erase important domain semantics.


# 62. Unit of Work

Unit of Work tracks a set of changes and commits them together.

Conceptually:

```text
read/change/change
      ↓
commit
```

Useful when:

```text
multiple related changes
transaction semantics
```

are important.

Do not invent an in-memory unit-of-work abstraction when the underlying persistence model already provides clearer transactions.


# 63. Saga / Process Manager

Distributed workflows cannot usually rely on one global transaction.

A Saga coordinates:

```text
step A
→
step B
→
compensation if B/C fails
```

Example:

```text
reserve inventory
→ charge payment
→ create shipment
```

Compensation might be:

```text
release inventory
```

This is a higher-level distributed pattern and belongs to system architecture, but the design-pattern vocabulary helps recognize it.


# 64. Command Query Separation

CQS separates:

```text
command → changes state
query   → returns information
```

This improves reasoning when operations are clearly classified.

Avoid pretending a method is a pure query when it secretly:

```text
writes
publishes events
warms caches
```

unless those effects are part of the documented contract.


# 65. Event Sourcing Connection

Event sourcing stores:

```text
events
```

as the canonical history.

State is derived:

```text
initial state
+
events
→
current state
```

Patterns involved:

```text
Command
Event
Reducer
Observer/PubSub
Memento/snapshot
```

Do not adopt event sourcing merely because these patterns fit together.


# 66. CQRS Connection

CQRS separates:

```text
write model
read model
```

when different access patterns justify it.

Potential benefits:

```text
read optimization
write optimization
independent scaling
```

Costs:

```text
consistency
duplication
operational complexity
```

It is an architectural pattern, not a default API pattern.


# 67. Pattern Combinations

Real systems combine patterns:

```text
Factory
  ↓
Strategy
  ↓
Decorator
  ↓
Adapter
```

or:

```text
Command
  ↓
Queue
  ↓
Worker
  ↓
Repository
```

or:

```text
Facade
  ↓
Repository
  ↓
Adapter
```

Pattern combinations should arise from independent forces.

Do not stack patterns because they look architecturally impressive.


# 68. Pattern Interaction Example

Payment system:

```text
Factory
→ choose provider

Adapter
→ normalize provider API

Strategy
→ choose routing policy

Decorator
→ metrics/retry

Repository
→ persistence

Command
→ queue durable intent
```

This is reasonable only when each pattern corresponds to a real concern.


# 69. Pattern Overuse

Warning signs:

```text
Factory for one constructor
Strategy for one fixed algorithm
Observer for two local callbacks
Facade for a tiny module
Builder for a five-field object
Repository around a simple in-memory array
Singleton for convenience
Abstract Factory before multiple implementations exist
```

Ask:

```text
What force requires this pattern?
```


# 70. Pattern Underuse

The opposite problem exists.

Signs:

```text
provider conditionals everywhere
duplicate adapters
business logic tightly coupled to SDKs
repeated construction rules
cross-cutting logic duplicated in 20 handlers
```

A pattern may reduce the repetition and coupling.

The solution is not “more patterns.”

It is:

```text
one appropriate pattern
```


# 71. Pattern Cargo Culting

Cargo-cult design says:

```text
good architecture
=
many patterns
```

False.

A small system may need:

```text
functions
objects
modules
```

and nothing more.

Pattern quality is measured by:

```text
problem solved
```

not:

```text
pattern count
```


# 72. Pattern Vocabulary in Code Review

A useful review comment:

> “This appears to be a Strategy because pricing varies independently. Keep it.”

is better than:

> “Use Strategy because Strategy is a best practice.”

Pattern names communicate the reason only when the underlying problem is real.


# 73. JavaScript-Native Pattern Mapping

| Classical idea | Natural JavaScript form |
|---|---|
| Strategy | function/object |
| Factory | function |
| Iterator | iterable protocol |
| Prototype | prototype delegation |
| Module | ES module |
| Observer | callback/EventTarget |
| Decorator | wrapper function/object |
| Command | object/function/data record |
| State | state machine / object |
| Dependency Injection | function/class argument |
| Chain of Responsibility | middleware |
| Registry | Map |
| Memoization | closure + Map |
| Facade | module/class function |


# 74. Pattern and Language Evolution

As languages evolve, pattern implementations can become smaller.

Example:

```text
manual iterator class
```

can become:

```text
generator
```

and:

```text
manual module closure
```

can become:

```text
ES module
```

The pattern concept remains useful even when language features eliminate boilerplate.

Good engineers ask:

```text
What is the design idea?
What does the language already provide?
```


# 75. Performance Considerations

Patterns can affect:

```text
function calls
object allocations
indirection
memory
branching
closures
subscription lists
serialization
```

Examples:

```text
Decorator chain → more calls
Observer → subscriber iteration
Proxy → intercepted operations
Builder → intermediate mutable state
Flyweight → lookup overhead
DI → composition/runtime object graph
```

These costs are usually acceptable.

In hot paths, measure.


# 76. Memory Considerations

Potential memory risks:

```text
Observer subscriptions retained
Singleton state retained globally
Registry entries never removed
Decorator graph retained
Command queue growth
Object pools retaining resources
Memoization caches
Flyweight tables
```

Every pattern that stores references creates a lifecycle responsibility.


# 77. Security Considerations

Pattern misuse can create security problems.

Examples:

```text
global singleton
→ excessive authority

plugin registry
→ unsafe extension

Proxy
→ false assumption that it enforces security everywhere

Decorator
→ authentication layer placed after privileged operation

Command
→ untrusted command deserialization

Observer
→ sensitive data leaked to subscribers
```

Always model:

```text
who can construct
who can call
who can observe
what capabilities are exposed
```


# 78. Pattern and Error Handling

Patterns should preserve error semantics.

Example adapter:

```text
provider timeout
→ TimeoutError
```

Decorator:

```text
metrics failure
```

should not necessarily break the business operation.

Command:

```text
retryable vs permanent
```

should be explicit.

A pattern that hides error behavior creates operational ambiguity.


# 79. Pattern and Observability

Patterns increase indirection.

Therefore preserve:

```text
component name
pattern role
operation name
trace context
error cause
```

Example:

```text
CheckoutFacade
→ RetryDecorator
→ PaymentAdapter
→ Provider
```

A production trace should make these boundaries understandable.


# 80. Pattern and Testing

Test the contract:

```text
Strategy
→ correct result

Adapter
→ correct translation

Decorator
→ behavior preserved + additional behavior

Observer
→ subscription lifecycle

Factory
→ correct implementation selection
```

Avoid tests that prove only:

```text
"class Strategy exists"
```

Patterns are implementation means, not outcomes.


# 81. Code Walkthrough — Strategy + Factory

```js
const strategies = {
  standard: order => order.subtotal,
  premium: order => order.subtotal * 0.9,
};

function createPricing(type) {
  const strategy = strategies[type];

  if (!strategy) {
    throw new RangeError(`Unknown pricing: ${type}`);
  }

  return strategy;
}

const pricing = createPricing("premium");

console.log(
  pricing({ subtotal: 100 })
);
```

Prediction:

```text
90
```

Patterns:

```text
Factory
+
Strategy
+
Registry
```

The code stays small because JavaScript functions already provide the necessary mechanism.


# 82. Code Walkthrough — Decorator + Adapter

```js
function withTiming(service, record) {
  return {
    async charge(input) {
      const started = performance.now();

      try {
        return await service.charge(input);
      } finally {
        record(performance.now() - started);
      }
    },
  };
}

function createProviderAdapter(client) {
  return {
    async charge({ amount, currency }) {
      const result =
        await client.charge({ amount, currency });

      return {
        id: result.id,
        status: result.status,
      };
    },
  };
}
```

Composition:

```text
provider client
→ adapter
→ decorator
→ application
```


# 83. Code Walkthrough — Observer

```js
function createObservable() {
  const listeners = new Set();

  return {
    subscribe(listener) {
      listeners.add(listener);

      return () => {
        listeners.delete(listener);
      };
    },

    emit(value) {
      for (const listener of listeners) {
        listener(value);
      }
    },
  };
}

const observable = createObservable();

const unsubscribe = observable.subscribe(
  value => console.log(value)
);

observable.emit("A");
unsubscribe();
observable.emit("B");
```

Prediction:

```text
A
```

The second event produces no output because the subscription was removed.


# 84. Code Walkthrough — Command

```js
function createChargeCommand(gateway, payment) {
  return {
    async execute() {
      return gateway.charge(payment);
    },
  };
}
```

The command separates:

```text
intent
```

from:

```text
execution
```

It can now be passed to:

```text
queue
scheduler
retry mechanism
audit system
```


# 85. Code Walkthrough — Chain

```js
const middleware = [
  validate,
  authorize,
  process,
];

async function run(input) {
  let current = input;

  for (const step of middleware) {
    current = await step(current);
  }

  return current;
}
```

This is a simple chain.

Before adding a framework, define:

```text
error behavior
short-circuit behavior
ordering
shared context
```


# 86. Debugging Exercise — Observer Leak

An event bus retains:

```text
50,000 listeners
```

after associated requests finish.

Investigate:

```text
missing unsubscribe
long-lived publisher
closure retention
subscription lifecycle
```

The likely issue is lifecycle, not GC.


# 87. Debugging Exercise — Singleton State

Tests pass individually but fail when run together.

A module-level singleton stores:

```text
currentUser
```

Possible cause:

```text
state leaks between tests
```

Options:

```text
reset state
create per-test instance
inject dependency
avoid singleton
```

Choose based on actual ownership requirements.


# 88. Debugging Exercise — Decorator Order

You have:

```text
retry
metrics
authorization
```

Compare:

```text
metrics(retry(authorization(service)))
```

with:

```text
retry(metrics(authorization(service)))
```

Determine:

```text
what gets measured
what gets retried
what gets authorized
```

Pattern ordering is semantics.


# 89. Debugging Exercise — Adapter Leakage

An interface exposes:

```js
paymentProvider.rawResponse
```

Now application code depends on provider-specific fields.

The adapter has leaked.

Fix:

```text
normalize response
keep raw response internal
provide explicit diagnostic escape hatch if truly required
```


# 90. Debugging Exercise — God Facade

A facade has methods for:

```text
users
orders
payments
inventory
shipping
email
analytics
```

The facade is probably a god object.

Split by cohesive workflows or bounded responsibilities.

A facade should simplify a subsystem, not become the subsystem.


# 91. Code Review Exercise

Review:

```js
class UserManager {
  static instance = new UserManager();

  constructor() {
    this.repository = new UserRepository();
    this.logger = new Logger();
  }

  static getInstance() {
    return UserManager.instance;
  }

  async create(user) {
    return this.repository.save(user);
  }
}
```

Find:

```text
singleton coupling
hidden dependencies
construction coupling
test isolation
lifecycle
extension difficulty
```

Propose a composition-root design instead.


# 92. Code Review Exercise — Pattern Explosion

Review:

```text
UserFactory
UserBuilder
UserStrategy
UserAdapter
UserDecorator
UserFacade
UserMediator
UserRepository
UserManager
```

There is one:

```text
create user
```

use case.

The review question is:

> What problem is each pattern solving?

Delete everything that cannot answer that question.


# 93. Production Scenario — Payment Providers

Need:

```text
Provider A
Provider B
Provider Fake
```

Recommended concepts:

```text
Factory → choose implementation
Adapter → normalize provider API
Strategy → route based on business policy
Decorator → metrics/retry
```

But the exact composition should remain minimal.

Document:

```text
failure semantics
retry rules
provider capabilities
```


# 94. Production Scenario — Notification System

Need:

```text
email
SMS
push
```

Potential:

```text
Strategy → delivery choice
Adapter → provider-specific SDK
Decorator → metrics
Observer/Event → event-driven trigger
```

Do not let:

```text
notification event
```

implicitly expose:

```text
provider credentials
```

Maintain capability boundaries.


# 95. Production Scenario — File Import

Potential pipeline:

```text
Command
 ↓
Factory
 ↓
Parser Strategy
 ↓
Validation Chain
 ↓
Repository
 ↓
Event
```

This is a good example of composition.

It is also a warning:

```text
pattern stack
```

can become complex.

Keep each layer only when it protects a real variation or boundary.


# 96. Production Scenario — Caching

Potential composition:

```text
Repository
 ↓
Caching Decorator
 ↓
Application
```

Need to define:

```text
TTL
invalidation
stale data
cache failures
metrics
```

The decorator hides mechanics while the cache semantics remain part of the contract.


# 97. Production Scenario — Authorization

Use:

```text
Policy / Strategy
```

for authorization rules.

Example:

```js
const canRefund = ({ user, payment }) =>
  user.role === "finance" &&
  payment.status === "captured";
```

Then:

```text
HTTP identity
+
domain data
+
pure policy
```

compose into the use case.

This keeps the rule testable.


# 98. Production Scenario — Background Jobs

Potential design:

```text
Command
→ queue
→ worker
→ handler registry
→ strategy
→ repository
```

Requirements:

```text
idempotence
retry policy
dead-letter behavior
observability
timeout
cancellation
```

The patterns help organize the workflow, but reliability semantics dominate.


# 99. Production Scenario — Plugin System

Potential patterns:

```text
Factory
Registry
Adapter
Strategy
Observer
```

Security requirements:

```text
least authority
versioned contracts
failure isolation
resource limits
```

Do not let a plugin receive the whole application object if it needs only:

```text
registerHandler
log
config
```


# 100. Pattern Refactoring Workflow

When introducing a pattern:

```text
1. characterize current behavior
2. identify variation
3. define stable contract
4. introduce smallest abstraction
5. move one variation behind it
6. test
7. measure
8. migrate remaining variants
9. remove obsolete coupling
```

Do not rewrite everything around a pattern in one step.


# 101. Pattern Removal Workflow

Patterns should be removable.

If a strategy now has:

```text
one implementation
```

and no expected variation:

```text
inline it
```

If a factory only does:

```text
return new User(...)
```

consider deleting it.

Simplification is a valid architectural improvement.


# 102. Decision Table

| Design pressure | Pattern candidate |
|---|---|
| implementation selection | Factory |
| algorithm variation | Strategy |
| provider translation | Adapter |
| complex subsystem simplification | Facade |
| add behavior around component | Decorator |
| controlled interception | Proxy |
| recursive tree uniformity | Composite |
| shared expensive resources | Pool |
| object creation complexity | Builder |
| one logical process-wide resource | Singleton, cautiously |
| event subscribers | Observer |
| topic-based event distribution | Pub/Sub |
| request pipeline | Chain / Middleware |
| action as data | Command |
| state-specific behavior | State |
| stable algorithm skeleton | Template Method |
| traversal abstraction | Iterator |
| peer interaction mediation | Mediator |
| tree-wide operations | Visitor |
| snapshots/restore | Memento |
| rule evaluation | Specification / Interpreter |


# 103. Pattern Decision Questions

Before implementing, answer:

```text
What varies?
What is stable?
Who owns construction?
Who owns state?
Who owns the invariant?
Who should know about the dependency?
How many implementations exist?
How often do they change?
What happens on failure?
What is the runtime path?
What is the memory cost?
What security capability crosses the boundary?
What happens at 10× scale?
```


# 104. Performance Considerations

Pattern-related costs can include:

```text
extra function calls
extra object allocations
wrapper traversal
Map lookups
subscription iteration
proxy traps
serialization
indirection
```

Most business applications can afford these costs.

Hot paths may not.

Use:

```text
profile
→ benchmark
→ optimize
```

rather than:

```text
pattern = slow
```


# 105. Memory Considerations

Audit:

```text
singleton lifetime
registry lifetime
subscriber lifetime
decorator references
command queue size
pool resources
memoization
cached strategies
```

Every long-lived pattern structure creates an ownership question:

```text
Who removes it?
When?
Under what failure?
```


# 106. Security Considerations

Patterns can increase or reduce security.

Helpful:

```text
Adapter → isolates vendor capabilities
DI → limits authority
Facade → narrows subsystem access
Policy → centralizes authorization
Command → explicit intent validation
```

Dangerous:

```text
global singleton
untrusted plugin registry
deserialized commands
overpowered service locator
observer with broad data visibility
```

Security depends on capability boundaries, not pattern names.


# 107. Common Misconceptions

### “Every problem should use a design pattern.”
No.

### “A design pattern makes code cleaner.”
Only if it addresses a real design force.

### “Strategy requires a class.”
No. A JavaScript function is often enough.

### “Factory requires an interface.”
No.

### “Singleton is always bad.”
No. But global state has costs.

### “Observer is harmless.”
No. Subscription lifecycle can create leaks and hidden control flow.

### “Decorator and Proxy are the same.”
No. They overlap conceptually but operate through different mechanisms.

### “Middleware is not a design pattern.”
It can embody Chain/Decorator/composition ideas.

### “Pattern-rich code is more senior.”
No.

### “Deleting a pattern is regression.”
No. Simplification can be an improvement.


# 108. Common Mistakes

```text
[ ] pattern-first thinking
[ ] copying textbook class diagrams
[ ] factory for one constructor
[ ] singleton for convenience
[ ] strategy for stable code
[ ] generic adapter with leaked provider types
[ ] observer without unsubscribe
[ ] decorator without order semantics
[ ] command without idempotence
[ ] state classes for trivial switches
[ ] facade becoming god object
[ ] registry becoming global mutable state
[ ] patterns with no tests of the actual contract
[ ] ignoring runtime/memory cost
[ ] ignoring security capability boundaries
```


# 109. Mastery Project — Pattern Catalog

Implement from scratch in JavaScript:

```text
Factory
Strategy
Adapter
Facade
Decorator
Proxy
Composite
Observer
Pub/Sub
Command
Chain of Responsibility
State
Iterator
Mediator
Registry
Dependency Injection
```

For each include:

```text
problem
forces
implementation
invariant
test
trade-offs
when not to use
```


# 110. Mastery Project — Refactor Without Patterns

Start with a deliberately tangled service.

Refactor it twice:

```text
Version A:
minimal composition

Version B:
pattern-rich
```

Then compare:

```text
readability
change isolation
testability
performance
memory
```

The goal is to discover whether the patterns actually improved the design.


# 111. Mastery Project — Pattern Removal

Take a legacy pattern-heavy module.

Find:

```text
unused factories
single strategies
redundant facades
global registries
unnecessary builders
```

Remove them safely.

Preserve behavior.

Measure whether complexity decreases.


# 112. Mastery Project — Production Payment Platform

Design:

```text
Payment API
Provider A
Provider B
retries
metrics
audit
fraud policy
idempotence
```

Possible patterns:

```text
Factory
Adapter
Strategy
Decorator
Command
Repository
Policy
```

Defend every pattern.

If a pattern does not solve a concrete pressure, remove it.


# 113. Interview Questions — Foundation

1. What is a design pattern?
2. Why use patterns?
3. Pattern vs framework?
4. Pattern vs architecture?
5. What are pattern forces?
6. What is a Factory?
7. What is Strategy?
8. What is Adapter?
9. What is Decorator?
10. What is Observer?


# 114. Interview Questions — Intermediate

11. How does JavaScript simplify classical patterns?
12. What is a Facade?
13. What is a Proxy?
14. What is Composite?
15. What is Command?
16. What is Chain of Responsibility?
17. What is State?
18. What is Builder?
19. What is Singleton?
20. What is Iterator?


# 115. Interview Questions — Advanced

21. Implement Strategy without classes.
22. Design provider adapters.
23. Design a retry decorator.
24. Design observer lifecycle safely.
25. Design a plugin registry.
26. Design an action queue using Command.
27. Replace a deep inheritance hierarchy with composition.
28. Explain how a pattern affects memory.
29. Explain how pattern composition affects observability.
30. Identify an abstraction that should be removed.


# 116. Interview Questions — Principal

31. When should you deliberately avoid a design pattern?
32. How do you distinguish useful abstraction from pattern cargo culting?
33. How would you evaluate pattern cost in a hot path?
34. How do patterns affect dependency direction?
35. How would you design a secure plugin architecture?
36. How would you evolve a provider adapter without leaking vendor details?
37. How do you prevent a facade from becoming a god object?
38. How do you preserve observability through decorator chains?
39. How do you determine whether a Singleton is actually justified?
40. What evidence tells you an existing pattern should be removed?


# 117. Predict-the-Output Exercises

## Exercise 1 — Strategy

```js
const strategies = {
  double: x => x * 2,
  square: x => x * x,
};

console.log(strategies.double(4));
console.log(strategies.square(4));
```

Predict.

## Exercise 2 — Decorator

```js
const service = {
  run(x) {
    return x * 2;
  },
};

const decorated = {
  run(x) {
    return service.run(x) + 1;
  },
};

console.log(decorated.run(4));
```

Predict.

## Exercise 3 — Observer

```js
const listeners = new Set();

const unsubscribe = listeners.add(
  value => console.log(value)
);

for (const listener of listeners) {
  listener("A");
}

console.log(typeof unsubscribe);
```

Explain why `Set.add()` is not an unsubscribe function and design the correct API.

## Exercise 4 — Factory

```js
function create(type) {
  if (type === "a") {
    return () => "A";
  }

  return () => "B";
}

console.log(create("a")());
console.log(create("b")());
```

Predict.

## Exercise 5 — Registry

```js
const handlers = new Map([
  ["x", value => value + 1],
  ["y", value => value * 2],
]);

console.log(handlers.get("x")(5));
console.log(handlers.get("y")(5));
```

Predict.


# 118. Debugging / Code Review Exercises

### Exercise 1
A pattern-heavy module has 14 classes for 2 behaviors. Identify unnecessary abstraction.

### Exercise 2
A strategy is never replaced. Determine whether it should remain a strategy.

### Exercise 3
An observer grows to 100,000 listeners. Diagnose memory ownership.

### Exercise 4
A retry decorator retries a non-idempotent command. Explain the failure.

### Exercise 5
An adapter exposes raw provider responses. Identify abstraction leakage.

### Exercise 6
A singleton contains tenant-specific request state. Explain why this is unsafe.

### Exercise 7
A facade owns every subsystem in the company. Identify the boundary failure.

### Exercise 8
A middleware chain has authorization after the handler. Identify the security failure.


# 119. Track A — Core Theory

Study:

```text
pattern
context
forces
consequences
creational
structural
behavioral
composition
delegation
variation
contracts
lifecycle
```

For every pattern know:

```text
problem
solution shape
benefit
cost
failure mode
when not to use
```


# 120. Track B — Implementation

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
Factory
Strategy
Adapter
Facade
Decorator
Proxy
Observer
Command
Chain
State
Iterator
Registry
DI
```

Then implement equivalent versions using JavaScript-native primitives where possible.


# 121. Track C — Interview / Reasoning

Practice:

```text
recognize
name
justify
implement
compare
remove
defend
```

The most important question is:

```text
Why this pattern?
```

followed immediately by:

```text
Why not simpler code?
```


# 122. Principal Decision Framework

Evaluate a pattern across:

| Dimension | Question |
|---|---|
| Correctness | Does the pattern preserve the required contract? |
| Performance | What calls, lookup, proxy, or allocation overhead is introduced? |
| Memory | What references are retained and for how long? |
| Security | What capabilities or data cross the boundary? |
| Reliability | What happens when one participant fails? |
| Maintainability | Does the pattern reduce future change cost? |
| Scalability | Does the pattern remain understandable with more variants? |
| Observability | Can the runtime path be traced clearly? |
| Developer Experience | Does the pattern communicate intent? |
| Operational Complexity | Does it add lifecycle/configuration burden? |
| Future Change | What variation does it isolate? |


# 123. Production Checklist

```text
[ ] concrete problem identified
[ ] forces documented
[ ] pattern justified
[ ] simpler alternative considered
[ ] implementation uses language primitives naturally
[ ] public contract documented
[ ] invariants documented
[ ] failure semantics documented
[ ] lifecycle documented
[ ] security boundary reviewed
[ ] performance measured when relevant
[ ] memory ownership reviewed
[ ] observability preserved
[ ] tests exercise behavior
[ ] pattern can be removed safely
```


# 124. Spaced Retrieval Plan

### Day 0
Define:

```text
Factory
Strategy
Adapter
Decorator
Facade
Observer
Command
Chain
State
Iterator
```

### Day 2
Implement five patterns without references.

### Day 7
Find three places in an existing codebase where a pattern is accidental.

### Day 14
Remove one unnecessary abstraction.

### Day 30
Design a production subsystem using no more patterns than necessary and defend every choice.


# 125. Completion Criteria

```text
[ ] Define design pattern
[ ] Explain pattern forces
[ ] Pattern vs algorithm
[ ] Pattern vs data structure
[ ] Pattern vs framework
[ ] Pattern vs architecture
[ ] Factory
[ ] Abstract Factory
[ ] Builder
[ ] Prototype
[ ] Singleton
[ ] Object Pool
[ ] Adapter
[ ] Facade
[ ] Decorator
[ ] Proxy
[ ] Composite
[ ] Bridge
[ ] Flyweight
[ ] Strategy
[ ] Observer
[ ] Pub/Sub
[ ] Command
[ ] Chain of Responsibility
[ ] State
[ ] Template Method
[ ] Iterator
[ ] Mediator
[ ] Visitor
[ ] Memento
[ ] Interpreter
[ ] Module pattern
[ ] Registry
[ ] Middleware
[ ] Dependency Injection
[ ] Null Object
[ ] Specification/Policy
[ ] Repository
[ ] Unit of Work
[ ] Pattern combinations
[ ] Pattern overuse detection
[ ] Pattern removal
[ ] Pattern security review
[ ] Pattern performance review
```


# 126. Mastery Gate

### Understand
You can explain major pattern families and the problems they address.

### Explain
You can state the forces and trade-offs for a pattern.

### Predict
You can reason about behavior, ordering, lifecycle, and side effects.

### Implement
You can implement patterns naturally in JavaScript without copying classical language-specific templates.

### Debug
You can identify lifecycle leaks, abstraction leakage, ordering bugs, and pattern misuse.

### Apply
You can use patterns in real application architecture.

### Compare
You can compare a pattern against simpler composition or direct code.

### Defend
You can justify every pattern using:

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


# 127. Status

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


# Chapter 77 — Revision / Retrieval Record

| Date | Retrieval task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain pattern forces | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement Strategy | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement Adapter | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement Decorator | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Debug Observer lifecycle | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Review Singleton usage | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Remove unnecessary patterns | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal pattern review | ✅ / ❌ | ... | ... |

### Retrieval Prompts

```text
1. What is the problem a design pattern solves?
2. What are forces?
3. Why should patterns not be memorized as code templates?
4. How does JavaScript simplify Strategy?
5. How does ES modules simplify Module pattern?
6. What makes Observer dangerous?
7. When is Adapter appropriate?
8. When is Facade harmful?
9. Why can Singleton create test problems?
10. What makes a pattern overused?
11. How do pattern combinations affect runtime?
12. How would you remove a pattern safely?
13. How do pattern choices affect security?
14. How do you preserve observability through indirection?
15. What evidence justifies a pattern in production?
```


# Chapter 77 — Canonical References and Source Discipline

## JavaScript language references

- ECMAScript Language Specification  
  https://tc39.es/ecma262/
- MDN Classes  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Classes
- MDN Functions  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Functions
- MDN Modules  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Modules
- MDN Iterators and Generators  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Iterators_and_generators
- MDN Proxy  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Proxy

Use the language/platform documentation for the mechanisms through which patterns are implemented.

## Design pattern references

Classic pattern catalogs and software-design literature are useful for the canonical names, intent, participants, and consequences of patterns.

Use pattern literature for:

```text
pattern vocabulary
problem/force analysis
canonical trade-offs
```

but do not copy language-specific implementation templates blindly into modern JavaScript.

## Source discipline

1. A pattern is a design idea, not a mandatory code template.
2. JavaScript often provides native primitives that simplify classical implementations.
3. Distinguish pattern from framework and architecture.
4. State the concrete design force before introducing a pattern.
5. Document pattern consequences.
6. Evaluate lifecycle and memory ownership.
7. Evaluate performance when indirection affects a hot path.
8. Preserve error semantics through adapters/decorators/chains.
9. Treat security as a capability-boundary question.
10. Remove patterns that no longer earn their maintenance cost.
11. Prefer composition and language-native mechanisms over ceremony.
12. Use pattern names to communicate real design decisions, not to signal sophistication.


# Chapter 77 — Completion Snapshot

## Creational

```text
[ ] Factory
[ ] Abstract Factory
[ ] Builder
[ ] Prototype
[ ] Singleton
[ ] Object Pool
```

## Structural

```text
[ ] Adapter
[ ] Facade
[ ] Decorator
[ ] Proxy
[ ] Composite
[ ] Bridge
[ ] Flyweight
```

## Behavioral

```text
[ ] Strategy
[ ] Observer
[ ] Pub/Sub
[ ] Command
[ ] Chain
[ ] State
[ ] Template Method
[ ] Iterator
[ ] Mediator
[ ] Visitor
[ ] Memento
[ ] Interpreter
```

## JavaScript-Native

```text
[ ] Module
[ ] Function Strategy
[ ] Closure
[ ] Registry
[ ] Middleware
[ ] Dependency Injection
[ ] Generator/Iterator
[ ] Reducer
[ ] Policy
```

## Judgment

```text
[ ] identify force
[ ] choose pattern
[ ] reject unnecessary pattern
[ ] measure cost
[ ] debug lifecycle
[ ] review security
[ ] preserve observability
[ ] remove pattern safely
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

Design patterns are a vocabulary for recurring problems.

They are useful when they answer:

```text
What varies?
What should remain stable?
What should be isolated?
Who owns construction?
Who owns state?
Who owns failure?
Who owns lifecycle?
What should callers know?
```

The wrong way to use patterns is:

```text
problem
 ↓
pattern catalog
 ↓
force problem into pattern
```

The better way is:

```text
problem
 ↓
constraints
 ↓
forces
 ↓
simple design
 ↓
pattern if it clarifies/reduces cost
```

Remember:

```text
Factory
→ creation variation

Strategy
→ algorithm/policy variation

Adapter
→ interface translation

Decorator
→ behavior wrapping

Facade
→ subsystem simplification

Observer
→ subscription relationship

Command
→ action as data

Chain
→ ordered processing

State
→ state-dependent behavior

Iterator
→ traversal abstraction
```

And remember the principal question:

> **What does this pattern let us change independently?**

If the answer is:

```text
nothing
```

the pattern may not be earning its cost.

The strongest JavaScript engineers are not pattern collectors.

They are engineers who can:

```text
recognize
→ simplify
→ compose
→ isolate
→ measure
→ evolve
```

The curriculum now moves from local design vocabulary to system-wide structure:

```text
patterns
+
composition
+
abstraction
      ↓
production architecture
      ↓
Chapter 78 — Production JavaScript Architecture


# 128. Extended Retrieval Bank

### Retrieval Drill 1

Given a production subsystem with `3` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 2

Given a production subsystem with `4` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 3

Given a production subsystem with `5` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 4

Given a production subsystem with `6` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 5

Given a production subsystem with `7` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 6

Given a production subsystem with `8` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 7

Given a production subsystem with `9` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 8

Given a production subsystem with `10` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 9

Given a production subsystem with `11` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 10

Given a production subsystem with `12` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 11

Given a production subsystem with `13` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 12

Given a production subsystem with `14` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 13

Given a production subsystem with `15` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 14

Given a production subsystem with `16` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 15

Given a production subsystem with `17` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 16

Given a production subsystem with `18` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 17

Given a production subsystem with `19` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 18

Given a production subsystem with `20` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 19

Given a production subsystem with `21` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 20

Given a production subsystem with `22` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 21

Given a production subsystem with `23` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 22

Given a production subsystem with `24` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 23

Given a production subsystem with `25` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 24

Given a production subsystem with `26` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 25

Given a production subsystem with `27` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 26

Given a production subsystem with `28` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 27

Given a production subsystem with `29` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 28

Given a production subsystem with `30` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 29

Given a production subsystem with `31` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 30

Given a production subsystem with `32` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 31

Given a production subsystem with `33` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 32

Given a production subsystem with `34` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 33

Given a production subsystem with `35` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 34

Given a production subsystem with `36` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```

### Retrieval Drill 35

Given a production subsystem with `37` moving requirements, decide whether to:

```text
1. keep simple functions/objects
2. introduce a pattern
3. compose existing patterns
4. remove an existing pattern
```

Document:

```text
problem
forces
chosen design
pattern, if any
runtime cost
memory/lifecycle risk
security impact
observability impact
why simpler code was rejected or retained
```