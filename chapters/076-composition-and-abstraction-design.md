# Chapter 76 — Composition and Abstraction Design

> **Curriculum position:** Part XIV — Programming Paradigms  
> **Previous chapter:** Chapter 75 — Object-Oriented Programming  
> **Next chapter:** Chapter 77 — Design Patterns  
> **Primary environment:** Modern JavaScript / TypeScript, Node.js, browsers, backend services, libraries, and large-scale platforms.

---

# Chapter Mission

Master **composition and abstraction design** as the layer that connects:

```text
functions
+
objects
+
modules
+
data structures
+
algorithms
```

into systems that can change without becoming fragile.

The core problem is not:

> “How many classes should this system have?”

It is:

```text
Where should responsibility live?
What should depend on what?
What should be stable?
What should be replaceable?
Where should variation exist?
Where should state be owned?
How much should a caller know?
```

The central design transformation is:

```text
large behavior
    ↓
small responsibilities
    ↓
explicit contracts
    ↓
composition
    ↓
stable boundary
    ↓
replaceable implementation
```

A good abstraction hides **irrelevant complexity** without hiding **important behavior**.

A bad abstraction:

```text
adds names
adds indirection
adds files
adds configuration
```

without making the system easier to reason about.

This chapter develops a principal-level vocabulary for:

- composition;
- abstraction;
- cohesion;
- coupling;
- dependency direction;
- dependency injection;
- inversion of control;
- ports and adapters;
- strategy objects;
- policy versus mechanism;
- delegation;
- adapters;
- facades;
- decorators;
- wrappers;
- factories;
- builders;
- module boundaries;
- capability boundaries;
- stable interfaces;
- structural contracts;
- dependency graphs;
- substitution;
- change isolation;
- coupling metrics;
- architecture fitness;
- refactoring toward composition.

The principal-level goal is:

> **Design boundaries around stable responsibilities and compose replaceable pieces so that local change has limited blast radius.**


# 1. Learning Objectives

By completion you should be able to:

## Core concepts

- define abstraction;
- define composition;
- explain cohesion;
- explain coupling;
- explain dependency direction;
- explain separation of concerns;
- explain encapsulation;
- distinguish mechanism from policy;
- distinguish interface from implementation;
- distinguish delegation from inheritance;
- explain inversion of control;
- explain dependency injection;
- explain explicit versus implicit dependencies.

## Composition techniques

- compose functions;
- compose objects;
- delegate behavior;
- inject dependencies;
- use strategy objects;
- use adapters;
- use facades;
- use decorators;
- use factories;
- use builders;
- compose middleware;
- compose validators;
- compose parsers;
- compose domain rules.

## Architecture

- design module boundaries;
- identify stable versus volatile concepts;
- keep dependency direction intentional;
- avoid circular dependencies;
- design ports and adapters;
- separate domain from infrastructure;
- design capability boundaries;
- isolate third-party libraries.

## Trade-offs

- recognize abstraction tax;
- identify over-abstraction;
- identify under-abstraction;
- compare composition with inheritance;
- compare dependency injection approaches;
- reason about runtime cost;
- reason about memory and object graph size;
- preserve observability;
- preserve debuggability.

## Principal judgment

- identify the correct abstraction boundary;
- explain why a boundary exists;
- determine where variation should live;
- predict change blast radius;
- refactor a tangled module graph;
- defend a composition architecture.


# 2. Prerequisites

Recommended:

- Chapter 09 — Functions and First-Class Behavior
- Chapter 13 — Closures
- Chapter 15 — Objects and Property Semantics
- Chapter 17 — Prototypes and Prototype Chains
- Chapter 18 — Classes and OOP
- Chapter 24 — Map and Set
- Chapter 45 — Memory and Garbage Collection
- Chapter 47 — JavaScript Engine Architecture
- Chapter 71 — Fundamental Data Structures
- Chapter 72 — Complexity
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 75 — Object-Oriented Programming


# 3. What Is Composition?

Composition builds a larger behavior from smaller behaviors.

Functional composition:

```js
const normalize = value => value.trim();
const parse = value => Number(value);

const parseNormalized =
  value => parse(normalize(value));
```

Object composition:

```js
class Service {
  constructor(logger, repository) {
    this.logger = logger;
    this.repository = repository;
  }
}
```

The larger component delegates to smaller components.

The important idea is:

```text
behavior emerges from collaboration
```

rather than:

```text
behavior must all be inherited
```


# 4. What Is Abstraction?

Abstraction exposes what a consumer needs while hiding details that should not matter to that consumer.

Example:

```js
paymentGateway.authorize(request);
```

The caller should not need to know:

```text
HTTP details
authentication headers
retry policy
provider-specific response formats
```

A good abstraction reduces unnecessary knowledge.

A bad abstraction merely relocates details behind:

```text
another function
another class
another interface
```

without reducing conceptual complexity.


# 5. Abstraction Is a Boundary

Think:

```text
consumer
   │
   ▼
stable contract
   │
   ▼
implementation
```

The implementation can change if the contract remains valid.

That creates:

```text
change isolation
```

The boundary should contain the details most likely to vary.

This is more useful than simply grouping code by file type.


# 6. Cohesion

Cohesion asks:

> Do the responsibilities inside this component belong together?

High cohesion:

```text
InvoiceCalculator
→ invoice calculation
```

Low cohesion:

```text
OrderManager
→ database
→ email
→ PDF generation
→ authentication
→ caching
→ metrics
→ scheduling
```

High cohesion usually makes:

```text
testing
understanding
change
reuse
```

easier.


# 7. Coupling

Coupling asks:

> How strongly does one component depend on another component's details?

Tight coupling:

```text
Service
  ↓
specific provider
  ↓
provider-specific response shape
  ↓
provider-specific error codes
```

Looser coupling:

```text
Service
  ↓
PaymentPort
  ↓
provider adapter
```

Lower coupling does not mean:

```text
zero dependency
```

It means:

```text
dependency is constrained through a useful boundary
```


# 8. Coupling vs Cohesion

A strong module tends toward:

```text
high cohesion
+
controlled coupling
```

Bad:

```text
low cohesion + high coupling
```

Also possible:

```text
high cohesion + excessive abstraction
```

which can still be difficult to operate.

Architecture is a balance.


# 9. Dependency Direction

Dependency direction describes:

```text
A depends on B
```

A useful architecture makes dependencies flow toward stable concepts.

Example:

```text
HTTP adapter
    ↓
application use case
    ↓
domain policy
```

The domain should not need to know about:

```text
HTTP framework
database driver
cloud vendor
```

unless that coupling is intentionally part of the domain.


# 10. Volatility and Stability

When designing a boundary, ask:

```text
What changes frequently?
What changes rarely?
```

Potentially volatile:

```text
payment provider
database driver
HTTP framework
cloud SDK
UI library
```

Potentially stable:

```text
order rules
pricing policy
domain invariants
business vocabulary
```

Place volatile implementation behind stable concepts when the business actually benefits from substitution.


# 11. Mechanism vs Policy

Policy:

```text
What should happen?
```

Mechanism:

```text
How is it physically done?
```

Example:

```text
Policy:
"retry a transient payment failure up to 3 times"

Mechanism:
HTTP client implementation
```

Separating them allows:

```text
policy to remain stable
mechanism to change
```


# 12. Example — Pricing Policy

Pure policy:

```js
function calculateDiscount(order, customer) {
  if (customer.tier === "gold") {
    return order.total * 0.1;
  }

  return 0;
}
```

Mechanism:

```text
how customer data was loaded
how order was persisted
how money was represented
```

Keep these decisions separate where the boundary has real value.


# 13. Delegation

Delegation means one component asks another component to perform behavior.

```js
class CheckoutService {
  constructor(paymentGateway) {
    this.paymentGateway = paymentGateway;
  }

  authorize(request) {
    return this.paymentGateway.authorize(request);
  }
}
```

The service delegates payment-specific behavior.

Delegation often provides reuse without inheriting implementation.


# 14. Composition vs Inheritance

Inheritance:

```text
subtype → base type
```

Composition:

```text
component → collaborator
```

Inheritance couples:

```text
behavior
lifecycle
override points
constructor structure
```

Composition usually couples:

```text
explicit contract
```

Use inheritance when subtype semantics are real.

Use composition when capabilities can vary independently.


# 15. Dependency Injection

Dependency injection means dependencies are supplied rather than constructed implicitly.

Bad:

```js
class OrderService {
  constructor() {
    this.repository = new PostgresOrderRepository();
  }
}
```

Better:

```js
class OrderService {
  constructor(repository) {
    this.repository = repository;
  }
}
```

The service now depends on:

```text
repository behavior
```

rather than:

```text
construction details
```


# 16. Constructor Injection

```js
class OrderService {
  constructor(repository, paymentGateway, clock) {
    this.repository = repository;
    this.paymentGateway = paymentGateway;
    this.clock = clock;
  }
}
```

Benefits:

```text
dependencies visible
object valid after construction
tests easy to configure
```

Cost:

```text
constructor can become too large
```

If there are 20 dependencies, the problem may be poor cohesion rather than insufficient dependency-injection technology.


# 17. Function Injection

Dependency injection does not require classes.

```js
function createInvoiceService({
  save,
  now,
  calculateTax,
}) {
  return async invoice => {
    const tax = calculateTax(invoice);
    return save({
      ...invoice,
      tax,
      createdAt: now(),
    });
  };
}
```

Functions can be excellent dependency boundaries.

Use the smallest abstraction that communicates intent.


# 18. Explicit Dependencies

Good dependency:

```js
function createMailer({ transport }) {}
```

Bad hidden dependency:

```js
function sendEmail(message) {
  return globalMailClient.send(message);
}
```

Explicit dependencies improve:

```text
testability
reasoning
security
deployment
```


# 19. Inversion of Control

Inversion of control means a component does not fully control when/how its collaborators are invoked.

Examples:

```text
event handlers
callbacks
middleware
dependency injection
framework lifecycle hooks
```

Instead of:

```text
service constructs everything
```

the composition root supplies:

```text
dependencies
```

and the runtime coordinates execution.


# 20. Composition Root

The composition root is the place where concrete implementations are assembled.

Example:

```js
const db = createDatabase(config);
const repository = new PostgresOrderRepository(db);
const gateway = new StripeGateway(config);
const service = new OrderService(repository, gateway);
```

Domain/application code should not need to know how these concrete objects are assembled.

This creates one visible place for infrastructure configuration.


# 21. Ports and Adapters

A port is a stable boundary representing required behavior.

An adapter implements that boundary for a specific technology.

Example:

```text
Application
    ↓
PaymentPort
    ↑
 ┌──┴───────────────┐
 │                  │
StripeAdapter   FakePaymentAdapter
```

The application owns the required behavior.

Infrastructure adapts external systems to it.


# 22. Hexagonal Thinking

A conceptual layout:

```text
        HTTP Adapter
             ↓
      ┌─────────────┐
      │ Application │
      └──────┬──────┘
             ↓
          Domain
             ↑
      ┌──────┴──────┐
      │             │
    DB Adapter   API Adapter
```

The point is not folder structure.

The point is:

```text
domain/application dependence
```

does not flow unnecessarily into external technologies.


# 23. Adapters

An adapter translates one contract to another.

```js
class StripeGatewayAdapter {
  constructor(stripe) {
    this.stripe = stripe;
  }

  async authorize(request) {
    const response =
      await this.stripe.paymentIntents.create({
        amount: request.amount,
        currency: request.currency,
      });

    return {
      id: response.id,
      approved: response.status === "succeeded",
    };
  }
}
```

The application sees:

```text
authorize(request)
```

not provider-specific API details.


# 24. Facades

A facade presents a simpler interface over a more complex subsystem.

Example:

```js
class MediaService {
  constructor(storage, transcoder, metadata) {
    this.storage = storage;
    this.transcoder = transcoder;
    this.metadata = metadata;
  }

  async publish(file) {
    // coordinate several subsystems
  }
}
```

A facade is useful when:

```text
the subsystem is genuinely complex
callers need a stable simplified workflow
```

It becomes harmful when it becomes a god object.


# 25. Strategy Pattern as Composition

Strategy means behavior is supplied as a replaceable collaborator.

```js
class CheckoutService {
  constructor(pricingStrategy) {
    this.pricingStrategy = pricingStrategy;
  }

  total(order) {
    return this.pricingStrategy.calculate(order);
  }
}
```

Strategies:

```text
StandardPricing
WholesalePricing
PromoPricing
```

The service does not need to inherit from every pricing policy.


# 26. Function Strategy

For simple behavior, a function may be enough:

```js
function calculateTotal(order, pricingPolicy) {
  return pricingPolicy(order);
}
```

Usage:

```js
calculateTotal(order, standardPricing);
calculateTotal(order, wholesalePricing);
```

Do not create classes merely because the word “strategy” exists.


# 27. Decorator as Composition

A decorator adds behavior around an existing component.

```js
function withLogging(service, logger) {
  return {
    async execute(input) {
      logger.info("start");
      const result = await service.execute(input);
      logger.info("done");
      return result;
    },
  };
}
```

This can add:

```text
logging
metrics
caching
authorization
retry
timing
```

without modifying the core component.


# 28. Decorator Ordering

Composition order matters.

```js
withRetry(
  withMetrics(
    withLogging(service)
  )
);
```

may behave differently from:

```js
withLogging(
  withMetrics(
    withRetry(service)
  )
);
```

For example:

```text
do metrics around every retry
```

versus:

```text
one metric around the entire retry operation
```

Define semantics explicitly.


# 29. Middleware Composition

Middleware often follows:

```text
request
 ↓
middleware A
 ↓
middleware B
 ↓
handler
```

Each layer can:

```text
inspect
modify
short-circuit
delegate
```

This is composition over a chain of responsibilities.

Potential risks:

```text
order dependence
hidden control flow
duplicate work
latency
```


# 30. Pipeline Composition

A pipeline can be:

```js
const pipeline = [
  validate,
  normalize,
  authorize,
  process,
];
```

Then execute:

```js
for (const step of pipeline) {
  input = await step(input);
}
```

Benefits:

```text
explicit sequence
replaceable steps
testable units
```

Risks:

```text
shared mutable context
unclear error semantics
order sensitivity
```


# 31. Builders

A builder separates construction from final product creation.

Example:

```js
const request = new RequestBuilder()
  .method("POST")
  .header("content-type", "application/json")
  .body(payload)
  .build();
```

Builders are useful when:

```text
construction has many optional combinations
intermediate validity needs control
readability improves materially
```

Do not use builders for trivial object literals.


# 32. Factories

Factories centralize object selection/creation:

```js
function createPaymentGateway(config) {
  switch (config.provider) {
    case "stripe":
      return new StripeGateway(config);

    case "mock":
      return new FakeGateway();

    default:
      throw new Error("Unknown provider");
  }
}
```

Benefits:

```text
construction centralized
implementation hidden
substitution easy
```

Potential cost:

```text
indirection
```


# 33. Factory Placement

Factories should live near the composition boundary when they decide infrastructure implementation.

Avoid:

```text
domain entity
  ↓
creates database adapter
```

Prefer:

```text
composition root
  ↓
creates adapter
  ↓
injects adapter
```

This preserves dependency direction.


# 34. Module Boundaries

A module boundary is an abstraction boundary.

Expose:

```js
export {
  createOrderService,
};
```

Keep implementation details internal.

A module should have:

```text
clear responsibility
clear public API
minimal accidental exports
```

The number of files is not a quality metric.


# 35. Stable Interfaces

A stable interface changes less frequently than the underlying implementation.

Example:

```text
PaymentPort.authorize(request)
```

can remain stable while:

```text
Stripe
Adyen
mock
local emulator
```

change.

Do not freeze an interface too early.

A premature abstraction can be harder to change than duplicated code.


# 36. The Rule of Three Intuition

A practical heuristic:

```text
first implementation → understand
second variation → compare
third repeated variation → consider abstraction
```

The first repetition may reveal no stable commonality.

Wait until the invariant structure becomes visible.

This is a heuristic, not a law.


# 37. Duplication vs Wrong Abstraction

Duplication is sometimes cheaper than the wrong abstraction.

Bad abstraction:

```text
three implementations
→ forced through one interface
→ dozens of conditionals
```

This creates:

```text
semantic coupling
```

Sometimes:

```text
small duplication
```

is better until the common behavior becomes clear.


# 38. Abstraction Tax

Every abstraction adds possible cost:

```text
naming
indirection
configuration
documentation
mental model
debugging path
runtime objects
```

The abstraction must return value through:

```text
change isolation
reuse
correctness
testability
clarity
```

If it does not, remove it.


# 39. Indirection Budget

Ask:

```text
How many hops must an engineer follow to understand one operation?
```

Bad:

```text
controller
→ facade
→ manager
→ service
→ coordinator
→ executor
→ strategy
→ helper
```

if each layer adds little value.

A principal engineer protects:

```text
conceptual locality
```


# 40. Abstraction Leakage

An abstraction leaks when callers must understand details it was supposed to hide.

Example:

```js
paymentGateway.authorize({
  stripePaymentIntentMode: ...
});
```

The abstraction is leaking provider-specific concepts.

A better contract:

```js
paymentGateway.authorize({
  amount,
  currency,
});
```

The adapter translates provider specifics internally.


# 41. Temporal Coupling

Temporal coupling means:

```text
A must happen before B
```

Example:

```js
service.initialize();
service.start();
```

Callers must know the lifecycle order.

Better designs can sometimes make illegal orderings impossible through:

```text
construction
state machines
factories
scoped APIs
```

or explicitly document the lifecycle when ordering is unavoidable.


# 42. Spatial Coupling

Spatial coupling means understanding:

```text
which internal object/field must be updated elsewhere
```

For example:

```text
service.cache.entries
service.cache.index
```

both must be updated manually.

Encapsulate related state behind one abstraction to reduce spatial coupling.


# 43. Knowledge Coupling

Knowledge coupling asks:

```text
How many implementation facts must one component know about another?
```

Low knowledge coupling:

```text
save(order)
```

High knowledge coupling:

```text
begin transaction
set driver flag
format SQL
inspect dialect
flush cache
```

The latter leaks mechanism.


# 44. Semantic Coupling

Semantic coupling occurs when components depend on unstated meaning.

Example:

```text
repository returns null
```

and callers silently interpret:

```text
null = deleted
```

while another caller interprets:

```text
null = temporarily unavailable
```

A stable abstraction must document:

```text
success
absence
failure
retryability
```


# 45. Dependency Graphs

Model modules as a directed graph:

```text
A → B
A → C
B → D
```

A healthy dependency graph tends toward:

```text
clear direction
few cycles
stable lower layers
```

Circular dependency:

```text
A → B
↑   ↓
└── C
```

can make:

```text
initialization
testing
reasoning
```

harder.


# 46. Dependency Cycles

If:

```text
A imports B
B imports A
```

ask whether there is a missing concept.

Possible fix:

```text
extract shared stable contract
```

or:

```text
move orchestration upward
```

Avoid blindly adding another abstraction module solely to break the cycle.


# 47. Dependency Inversion

The high-level policy should not depend on low-level implementation details.

Instead:

```text
high-level policy
      ↓
stable abstraction
      ↑
low-level detail
```

This is useful when:

```text
details vary independently
policy must remain stable
```

Do not apply it mechanically to every function.


# 48. Ports Belong Near Their Consumers

A useful architectural heuristic:

```text
the component that needs behavior
defines the behavior contract
```

Example:

```text
OrderService needs:
loadOrder(id)
saveOrder(order)
```

The application can define this required capability.

The database adapter then conforms to it.

This prevents infrastructure from dictating domain terminology.


# 49. Capability-Oriented Composition

Instead of injecting a giant object:

```js
new Service(platform)
```

inject only capabilities:

```js
new Service({
  saveOrder,
  charge,
  now,
});
```

This reduces:

```text
authority
coupling
test surface
```

and can improve security.


# 50. Principle of Least Knowledge

A component should know only what it needs to perform its responsibility.

Bad:

```text
controller knows database table schema
controller knows payment provider fields
controller knows cache eviction internals
```

Better:

```text
controller → application operation
```

This reduces change propagation.


# 51. Encapsulation Through Composition

Composition can isolate details:

```js
function createCachedRepository(repository, cache) {
  return {
    async findById(id) {
      if (cache.has(id)) {
        return cache.get(id);
      }

      const value = await repository.findById(id);
      cache.set(id, value);
      return value;
    },
  };
}
```

The caller sees:

```text
findById
```

not:

```text
cache policy
```

This is composition as encapsulation.


# 52. Composition and Testing

Small components can be tested independently.

Example:

```text
OrderService
  ↓
FakeOrderRepository
FakePaymentGateway
FakeClock
```

This allows deterministic tests without:

```text
database
network
real time
```

But avoid fake implementations that have behavior unrelated to production semantics.


# 53. Test Double Selection

Use:

```text
stub → controlled return
fake → lightweight working implementation
mock → interaction verification
spy → observe calls
```

Do not overuse interaction-based tests.

Prefer testing:

```text
observable behavior
```

over:

```text
exact internal call sequence
```

unless sequence is itself part of the contract.


# 54. Composition and Observability

Composed systems can make tracing harder because one logical action crosses many layers.

Useful observability:

```text
operation name
component name
duration
result
error
trace ID
```

Do not add layers that cannot be understood or observed in production.


# 55. Composition and Performance

Composition adds potential runtime overhead:

```text
function calls
wrapper objects
closures
allocation
indirection
```

Often this is negligible.

In hot paths it can matter.

Benchmark before replacing clear composition with tightly coupled optimization.


# 56. Composition and Memory

More composed objects can create larger object graphs.

Example:

```text
Service
 ├─ Strategy
 ├─ LoggerDecorator
 ├─ CacheDecorator
 ├─ MetricsDecorator
 └─ Adapter
```

This is acceptable when each layer earns its place.

Avoid:

```text
wrapper explosion
```

that increases retention and debugging complexity.


# 57. Wrapper Explosion

If every feature creates:

```text
wrapper
wrapper
wrapper
wrapper
```

the final runtime path can become difficult to trace.

Ask:

```text
Can two concerns be combined?
Should they be infrastructure-level middleware?
Is this concern worth a layer?
```

Composition should create modularity, not a maze.


# 58. Error Boundaries

Composed components should define:

```text
which errors pass through
which errors translate
which errors are retried
which errors are wrapped
```

Bad:

```text
catch everything
throw new Error("failed")
```

This destroys diagnostic context.

Good adapters preserve:

```text
cause
code
retryability
original context
```

where appropriate.


# 59. Error Translation

Provider-specific:

```text
StripeCardError
```

may become:

```text
PaymentDeclined
```

at the domain boundary.

The adapter should preserve enough cause/detail for diagnostics.

This avoids leaking provider-specific semantics into the domain while keeping observability.


# 60. Configuration as Composition

Configuration can select behavior:

```js
const gateway = createGateway(config.paymentProvider);
```

Avoid configuration that creates hundreds of conditionals inside core logic.

Prefer:

```text
configuration
→ implementation selection
→ composed graph
```

rather than:

```text
every method checks config
```


# 61. Feature Flags

Feature flags can also select behavior.

Bad:

```js
if (flags.newFlow) {
  // 500 lines
} else {
  // 500 lines
}
```

Better:

```js
const flow = flags.newFlow
  ? newFlow
  : oldFlow;

flow.execute(input);
```

Selection occurs at the composition boundary.

This reduces branching inside business logic.


# 62. Composition and Conditional Complexity

When code contains:

```js
if (type === "A") ...
if (type === "B") ...
if (type === "C") ...
```

ask whether the variation is stable.

If new types arrive frequently, strategy/registry composition may reduce conditional growth.

But if there are only:

```text
two stable cases
```

an `if`/`switch` can be clearer.


# 63. Registries

A registry maps a stable key to behavior:

```js
const handlers = new Map([
  ["created", handleCreated],
  ["paid", handlePaid],
  ["cancelled", handleCancelled],
]);

const handler = handlers.get(event.type);
```

Registries are useful when:

```text
handlers are dynamic/extensible
lookup is natural
```

But they can hide control flow.

Document where registrations happen.


# 64. Plugin Architecture

A plugin architecture composes externally supplied behavior.

Conceptually:

```text
core
  ↓
plugin contract
  ↑
plugin A
plugin B
plugin C
```

Important concerns:

```text
versioning
isolation
security
lifecycle
failure handling
observability
```

Do not expose unrestricted internal capabilities to plugins.


# 65. Extension Points

An extension point should answer:

```text
what can be customized?
when is it called?
what can it return?
what errors are allowed?
what resources can it access?
```

An extension point without a stable contract is an invitation to accidental coupling.


# 66. Public API Surface

Every exported function/class/type is a potential compatibility commitment.

Minimize unnecessary public surface.

A good module can have:

```text
many private helpers
few public exports
```

This makes future refactoring easier.


# 67. Abstraction Boundary Review

When reviewing a boundary, ask:

```text
What knowledge crosses?
What state crosses?
What errors cross?
What timing assumptions cross?
What implementation details cross?
What can change independently?
```

If too many details cross:

```text
boundary is weak
```

If almost nothing crosses:

```text
abstraction may be useless
```


# 68. Decision Matrix — Composition vs Inheritance vs Function

| Situation | Strong candidate |
|---|---|
| stateless transformation | function |
| behavior composition | function/object composition |
| mutable identity + lifecycle | class/object |
| subtype contract | inheritance |
| interchangeable capability | composition/strategy |
| infrastructure adapter | object/function adapter |
| simple configuration selection | factory |
| many optional construction steps | builder |
| cross-cutting behavior | decorator/middleware |
| complex subsystem simplification | facade |


# 69. Decision Matrix — Injection Style

| Style | Strength | Risk |
|---|---|---|
| constructor | visible dependencies | large constructors |
| function arguments | very explicit | noisy signatures |
| factory closure | convenient private configuration | hidden captured state |
| module singleton | simple access | global coupling |
| service locator | flexible | invisible dependencies |
| framework DI | standardized composition | framework complexity |

Prefer explicit dependencies unless the runtime/framework provides a strong reason otherwise.


# 70. Refactoring Toward Composition

A practical sequence:

```text
1. identify responsibility clusters
2. identify shared mutable state
3. identify unstable dependencies
4. extract stable contracts
5. inject collaborators
6. move infrastructure outward
7. replace condition-heavy variation with composition where justified
8. delete obsolete abstractions
9. test behavior
10. measure change impact
```


# 71. Refactoring Example

Before:

```js
class OrderManager {
  async process(order) {
    // validate
    // calculate
    // query DB
    // call payment provider
    // log
    // email
    // cache
  }
}
```

After:

```text
pure pricing
pure validation
OrderRepository
PaymentGateway
NotificationPort
Logger
Cache
OrderService
```

Composition root wires them together.

The goal is not more classes.

The goal is:

```text
smaller change boundaries
```


# 72. Avoiding Service Fragmentation

Over-refactoring can produce:

```text
OrderValidator
OrderNormalizer
OrderCalculator
OrderMapper
OrderCoordinator
OrderExecutor
```

when each contains five lines.

Ask:

```text
Does this component own a meaningful concept?
Does it change independently?
Can it be tested meaningfully?
Does the boundary reduce coupling?
```

Do not split merely to make files smaller.


# 73. Stable Core, Volatile Edge

A useful architectural shape:

```text
        volatile edge
 ┌──────────────────────────┐
 │ HTTP │ DB │ Cloud │ SDK │
 └────────────┬─────────────┘
              ↓
        stable application
              ↓
         stable domain
```

This does not mean the domain never changes.

It means:

```text
external technology volatility
```

is prevented from infecting every layer.


# 74. Anti-Corruption Boundary

External systems may use concepts that do not belong in your domain.

Example provider field:

```text
payment_intent_status = "requires_action"
```

Translate it:

```text
PaymentState.RequiresUserAction
```

at the boundary.

This prevents third-party vocabulary from becoming your internal domain model accidentally.


# 75. Anti-Pattern — Leaky Adapter

Bad:

```js
paymentGateway.rawStripeClient
```

The supposed adapter exposes the provider.

Now callers depend on:

```text
Stripe API
```

The boundary no longer isolates change.

If a provider-specific escape hatch is truly required, mark it as an explicit escape hatch rather than pretending the abstraction is complete.


# 76. Anti-Pattern — Generic Repository

A generic:

```js
repository.save(entity);
repository.find(id);
repository.delete(id);
```

may look reusable.

But if domain behavior actually differs:

```text
OrderRepository
PaymentRepository
InventoryRepository
```

forcing them into one generic abstraction can erase useful semantics.

Reuse should follow genuine commonality.


# 77. Anti-Pattern — Generic Utility Layer

Beware:

```text
utils/
  helper1
  helper2
  helper3
  common
```

Generic utilities often become dumping grounds.

Prefer helpers near the concept they support until reuse is clear.

Location should follow ownership.


# 78. Anti-Pattern — Service Locator

Example:

```js
container.resolve("paymentGateway");
```

inside business code.

The dependency is hidden.

Benefits:

```text
easy replacement
```

Costs:

```text
implicit dependencies
runtime resolution failures
harder reasoning
test complexity
```

Prefer explicit injection in core logic.


# 79. Anti-Pattern — Global Singleton

Globals can create:

```text
shared mutable state
test pollution
order dependence
lifecycle ambiguity
```

Singletons can be appropriate for truly process-wide resources, but make ownership and lifecycle explicit.


# 80. Anti-Pattern — Interface Explosion

A type such as:

```text
IUserRepository
IUserRepositoryReader
IUserRepositoryWriter
IUserRepositoryFactory
IUserRepositoryProvider
```

may exist without meaningful behavioral distinction.

Interfaces should correspond to actual substitution boundaries.

Do not create them because a style guide says every class requires one.


# 81. Anti-Pattern — Premature Generalization

Bad:

```js
function createGenericProcessor({
  strategy,
  mapper,
  resolver,
  policy,
  adapter,
  transformer,
}) {}
```

for a problem with one concrete implementation.

Start simple.

Generalize when variation becomes real.


# 82. Anti-Pattern — Conditional Abstraction

An abstraction that mostly contains:

```js
if (provider === "A") ...
if (provider === "B") ...
if (provider === "C") ...
```

may simply have relocated conditionals.

Better:

```text
factory
→ provider-specific implementation
→ common contract
```

when provider variation is an actual ongoing concern.


# 83. Anti-Pattern — Abstraction by Naming

Renaming:

```text
function → service
```

does not create architecture.

Adding:

```text
Manager
Coordinator
Processor
Handler
```

does not establish responsibility.

The behavior and dependency boundary must justify the abstraction.


# 84. Security — Capability Boundaries

Composition lets you limit authority.

Instead of:

```js
service(platform)
```

provide:

```js
service({
  readUser,
  saveOrder,
});
```

The component cannot automatically access:

```text
filesystem
secrets
admin APIs
unrelated databases
```

Least authority is both an architectural and security principle.


# 85. Security — Plugin Boundaries

Plugins are dangerous if they receive unrestricted objects.

Do not expose:

```js
plugin.initialize(globalApp);
```

when the plugin only needs:

```text
registerRoute
log
readConfig
```

Provide a constrained capability object.

Also define:

```text
time limits
failure isolation
memory limits
versioning
```

when plugins can be untrusted or semi-trusted.


# 86. Reliability — Failure Isolation

Composition can isolate failures.

Example:

```text
notification failure
```

should not necessarily invalidate:

```text
order persistence
```

unless the domain contract requires atomicity.

Define:

```text
critical path
best-effort side effect
retryable effect
compensatable effect
```

at the boundary.


# 87. Reliability — Retry Placement

Where retry is composed matters.

Retry:

```text
HTTP adapter
```

may be appropriate for transient transport failures.

Retrying:

```text
entire business workflow
```

may duplicate side effects.

Compose retries at the layer where:

```text
failure semantics
```

are understood.


# 88. Observability — Boundary Naming

Name important boundaries:

```text
OrderService.process
PaymentGateway.authorize
OrderRepository.save
```

Then traces become understandable.

A thousand anonymous wrapper functions create poor operational visibility.

Abstraction should be observable.


# 89. Performance — Boundary Placement

A good boundary can reduce work by enabling:

```text
caching
batching
memoization
connection reuse
```

A bad boundary can force:

```text
repeated serialization
repeated allocation
unnecessary conversion
```

Therefore measure where data crosses abstractions.


# 90. Memory — Data Copies

Composition should not automatically imply copying.

Bad:

```js
adapter → copies
service → copies
validator → copies
repository → copies
```

For a large object graph, this can create significant memory/GC cost.

Prefer:

```text
clear ownership
immutable values where useful
views/references where safe
boundary copying only when isolation demands it
```


# 91. Performance — Function Calls

Function-call overhead is usually not a reason to flatten a well-designed architecture.

But in a hot inner loop:

```text
billions of calls
```

abstraction overhead can matter.

The correct sequence is:

```text
profile
→ identify hot path
→ benchmark
→ optimize
```

not:

```text
assume abstraction is slow
→ inline everything
```


# 92. Debuggability

Composition creates more frames.

That can be good:

```text
clear responsibility
```

or bad:

```text
deep wrappers
```

Use:

```text
meaningful names
structured logs
trace spans
source maps
good errors
```

to maintain observability through abstraction layers.


# 93. API Compatibility

When changing an abstraction:

```text
contract
```

should remain stable where possible.

Use adapters for migration:

```text
old API
  ↓
compatibility adapter
  ↓
new API
```

This can enable incremental migration rather than a risky rewrite.


# 94. Strangler Refactoring

A large legacy component can be replaced gradually:

```text
legacy system
   ↓
new boundary
   ↓
new implementation
```

Move one capability at a time.

Composition enables coexistence:

```text
old implementation
+
new implementation
```

behind a common boundary during migration.


# 95. Architecture Fitness Questions

Ask:

```text
1. Can a dependency be replaced without editing business logic?
2. Can the domain be tested without infrastructure?
3. Can a new provider be added locally?
4. Can failure semantics be observed?
5. Can the module be understood without reading the whole system?
6. Does the public API hide volatile details?
7. Are there circular dependencies?
8. Are there unnecessary wrappers?
9. Is the composition graph visible?
10. Can the design evolve incrementally?
```


# 96. Code Walkthrough — Strategy Composition

```js
function createCheckout({ pricing }) {
  return {
    total(order) {
      return pricing(order);
    },
  };
}

const standard = order => order.subtotal;

const discounted = order =>
  order.subtotal * 0.9;

const checkout = createCheckout({
  pricing: discounted,
});

console.log(
  checkout.total({ subtotal: 100 })
);
```

Prediction:

```text
90
```

The design uses:

```text
composition
+
function injection
```

without a class hierarchy.


# 97. Code Walkthrough — Decorator Composition

```js
function withMetrics(service, record) {
  return {
    async run(input) {
      const started = performance.now();

      try {
        return await service.run(input);
      } finally {
        record(performance.now() - started);
      }
    },
  };
}
```

The decorator adds:

```text
timing
```

without changing the core service.

Production concern:

```text
What happens if metric recording itself fails?
```

Observability code should usually avoid breaking the business operation unintentionally.


# 98. Code Walkthrough — Adapter

```js
class UserApiAdapter {
  constructor(client) {
    this.client = client;
  }

  async findUser(id) {
    const response =
      await this.client.get(`/users/${id}`);

    return {
      id: response.user_id,
      name: response.display_name,
    };
  }
}
```

The domain does not need to know:

```text
user_id
display_name
```

This is a boundary translation.


# 99. Code Walkthrough — Capability Injection

```js
function createAuditService({
  appendAuditEntry,
}) {
  return {
    record(event) {
      return appendAuditEntry({
        type: event.type,
        id: event.id,
      });
    },
  };
}
```

The service gets exactly one capability.

It does not receive:

```text
database
application container
network client
global logger
```

This reduces authority and coupling.


# 100. Debugging Exercise — Hidden Dependency

```js
class InvoiceService {
  async create(invoice) {
    return globalDb.save(invoice);
  }
}
```

Find the problem.

Possible concerns:

```text
hidden dependency
test pollution
global lifecycle
hard replacement
security authority
```

Refactor toward explicit injection.


# 101. Debugging Exercise — Wrong Abstraction

Three providers have:

```text
charge()
```

but their failure semantics are fundamentally different.

A team creates:

```js
interface ChargeProvider {
  charge(): Promise<boolean>;
}
```

What important information may have been erased?

Consider:

```text
declined
retryable
authentication required
provider timeout
partial result
```

An abstraction that is too weak can be worse than no abstraction.


# 102. Debugging Exercise — Wrapper Explosion

Runtime path:

```text
Controller
→ LoggingDecorator
→ MetricsDecorator
→ RetryDecorator
→ CacheDecorator
→ AuthorizationDecorator
→ Service
→ RepositoryDecorator
→ TracingDecorator
→ Database
```

Questions:

```text
Which layers are actually needed?
Can some concerns move to middleware/interceptors?
Which ordering is required?
Which errors are translated?
Where should tracing begin/end?
```


# 103. Debugging Exercise — Circular Dependency

```text
OrderService → PaymentService
PaymentService → OrderService
```

Ask:

```text
Is there a shared domain concept?
Should orchestration move upward?
Should one service emit an event instead?
Is the dependency truly mutual?
```

Do not break the cycle with a random “CommonService.”


# 104. Code Review Exercise

Review:

```js
class UserManager {
  constructor() {
    this.db = new Postgres();
    this.mailer = new SendGrid();
    this.cache = new Redis();
  }

  async register(user) {
    if (!user.email.includes("@")) {
      throw new Error("invalid");
    }

    await this.db.insert(user);
    await this.cache.set(user.id, user);
    await this.mailer.send(user.email, "Welcome");

    return user;
  }
}
```

Identify:

```text
construction coupling
vendor leakage
business/infrastructure mixing
validation placement
failure semantics
transaction concerns
cache consistency
email reliability
testability
security
```


# 105. Improved Design

Potential structure:

```text
RegisterUser
  ↓
pure validation
  ↓
UserRepository
  ↓
UserCache
  ↓
WelcomeNotifier
```

Composition root:

```text
Postgres adapter
Redis adapter
SendGrid adapter
```

Domain logic remains independent of vendors.

The exact workflow still requires deciding:

```text
what happens if email fails
what happens if cache fails
whether registration is committed before notification
```


# 106. Production Scenario — Provider Migration

A payment service must migrate:

```text
Provider A
→
Provider B
```

A strong composition design:

```text
PaymentPort
 ├─ ProviderAAdapter
 └─ ProviderBAdapter
```

Then selection can happen by:

```text
feature flag
configuration
routing policy
```

The domain logic should not duplicate:

```text
payment business rules
```

per provider.


# 107. Production Scenario — Database Migration

Suppose:

```text
MongoDB
→
PostgreSQL
```

Do not make every business service know both.

Use:

```text
Repository contract
 ├─ MongoAdapter
 └─ PostgresAdapter
```

during migration.

But make sure the contract is actually supported by both data models.

Do not force unrelated semantics through a fake common interface.


# 108. Production Scenario — Caching

Compose caching around a repository:

```text
CachedRepository
   ↓
Repository
   ↓
Database
```

Define:

```text
cache hit
cache miss
stale data
invalidation
negative cache
failure behavior
```

A cache abstraction that hides these semantics can become dangerous.

Important behavior must remain visible in the contract.


# 109. Production Scenario — Authorization

Authorization can be modeled as a policy:

```js
function canRefund(user, payment) {
  return user.role === "finance" &&
         payment.status === "captured";
}
```

Infrastructure provides:

```text
user identity
payment data
```

Composition combines:

```text
identity adapter
policy
use case
```

The policy remains deterministic and testable.


# 110. Production Scenario — Observability

Cross-cutting concerns can be composed:

```text
request
→ trace
→ auth
→ metrics
→ business operation
```

But define order.

For example:

```text
trace first
auth second
business operation third
```

can differ from:

```text
auth first
trace only after authorization
```

which affects visibility and security auditing.


# 111. Mastery Project — Plugin Platform

Build a plugin system with:

```text
plugin contract
registration
lifecycle
capability object
error isolation
logging
versioning
```

Requirements:

```text
plugin cannot access arbitrary application internals
plugin failure is observable
plugin can be disabled
plugin API is versioned
```


# 112. Mastery Project — Provider Abstraction

Design a provider-neutral service supporting:

```text
Provider A
Provider B
Fake provider
```

Requirements:

```text
common business semantics
provider-specific error translation
metrics
retry policy
feature-flag routing
```

Document where each concern belongs.


# 113. Mastery Project — Legacy Refactor

Take a monolithic:

```text
1,000-line service
```

and refactor without a rewrite.

Sequence:

```text
characterize behavior
extract pure logic
identify dependencies
inject infrastructure
create boundary
introduce adapter
migrate one path
delete legacy path
measure
```

Preserve behavior throughout.


# 114. Mastery Project — Composition Benchmark

Compare:

```text
direct function
single wrapper
five wrappers
ten wrappers
```

Measure:

```text
throughput
latency
allocation
memory
```

Use realistic workloads.

The lesson is not:

```text
wrappers are bad
```

but:

```text
abstraction overhead is measurable
```


# 115. Interview Questions — Foundation

1. What is abstraction?
2. What is composition?
3. What is cohesion?
4. What is coupling?
5. What is delegation?
6. What is dependency injection?
7. What is inversion of control?
8. What is a factory?
9. What is an adapter?
10. What is a facade?


# 116. Interview Questions — Intermediate

11. Compare composition and inheritance.
12. Explain strategy using JavaScript functions.
13. What is constructor injection?
14. What is a composition root?
15. What are ports and adapters?
16. Why can generic repositories be harmful?
17. What is abstraction leakage?
18. What is temporal coupling?
19. What is a decorator?
20. How can feature flags be composed?


# 117. Interview Questions — Advanced

21. Design a provider migration without changing domain logic.
22. Design a database migration abstraction.
23. Design a plugin system with capabilities.
24. Refactor a god service using composition.
25. Break a circular dependency.
26. Explain abstraction tax.
27. Decide between duplication and a new abstraction.
28. Design error translation at an adapter boundary.
29. Design retries without duplicating side effects.
30. Design observable cross-cutting concerns.


# 118. Interview Questions — Principal

31. How do you identify a stable abstraction boundary?
32. When is duplication better than abstraction?
33. How do you predict change blast radius?
34. How do you control wrapper explosion?
35. How should domain/application dependencies point?
36. How do you design capability-limited plugin APIs?
37. How would you refactor a large legacy service incrementally?
38. How do you balance abstraction quality against runtime cost?
39. How do you review a proposed “generic” platform abstraction?
40. What evidence tells you an abstraction is earning its maintenance cost?


# 119. Predict-the-Output Exercises

## Exercise 1

```js
const addTax = amount => amount * 1.1;
const round = amount => Math.round(amount);

const calculate = value =>
  round(addTax(value));

console.log(calculate(100));
```

Predict.

## Exercise 2

```js
function createService(strategy) {
  return {
    run(value) {
      return strategy(value);
    },
  };
}

const service = createService(x => x * 2);

console.log(service.run(5));
```

Predict.

## Exercise 3

```js
const a = x => x + 1;
const b = x => x * 2;

const composed = x => b(a(x));

console.log(composed(3));
```

Predict.

## Exercise 4

```js
function withLogging(service, log) {
  return {
    run(value) {
      log("start");
      return service.run(value);
    },
  };
}

const service = withLogging(
  { run: x => x * 2 },
  console.log
);

console.log(service.run(4));
```

Trace output order.


# 120. Mastery Exercises

1. Implement `compose`.
2. Implement `pipe`.
3. Implement strategy through functions.
4. Implement object strategy.
5. Implement an adapter.
6. Implement a decorator.
7. Implement a facade.
8. Build a composition root.
9. Build a capability-limited plugin boundary.
10. Refactor a global dependency into injection.
11. Break a circular dependency.
12. Build a provider-neutral contract.
13. Build an incremental migration adapter.
14. Benchmark abstraction overhead.
15. Document an abstraction's stability assumptions.


# 121. Track A — Core Theory

Study:

```text
abstraction
composition
cohesion
coupling
dependency direction
stability
volatility
policy
mechanism
delegation
dependency injection
inversion of control
ports and adapters
capabilities
change isolation
abstraction leakage
```

You should be able to explain why each exists.


# 122. Track B — Implementation

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
strategy
adapter
decorator
factory
builder
facade
composition root
plugin boundary
migration adapter
```

For each record:

```text
contract
dependency direction
failure semantics
tests
performance considerations
```


# 123. Track C — Interview / Reasoning

Practice:

```text
find responsibility
find volatility
find coupling
find boundary
find leakage
find unnecessary abstraction
design composition
predict blast radius
defend trade-offs
```

The key interview skill is not naming a pattern.

It is explaining:

```text
why the boundary exists
```


# 124. Principal Decision Framework

Evaluate an abstraction/composition design across:

| Dimension | Question |
|---|---|
| Correctness | Does the boundary preserve required semantics? |
| Performance | What runtime/dispatch/allocation cost does it add? |
| Memory | What object graphs/caches/wrappers are retained? |
| Security | What capabilities cross the boundary? |
| Reliability | How are failures/retries/partial success modeled? |
| Maintainability | Does the abstraction reduce change cost? |
| Scalability | Does it survive more implementations and callers? |
| Observability | Can the composed path be understood in production? |
| Developer Experience | Is the abstraction obvious to users? |
| Operational Complexity | How many moving parts does it add? |
| Future Change | Which likely changes stay local? |


# 125. Production Checklist

```text
[ ] responsibility is clear
[ ] public contract is explicit
[ ] implementation details are hidden where useful
[ ] dependencies are visible
[ ] dependency direction is intentional
[ ] volatile details are isolated
[ ] abstraction is not premature
[ ] common behavior is genuinely common
[ ] error semantics are explicit
[ ] retry semantics are explicit
[ ] capability scope is minimal
[ ] observability crosses boundaries
[ ] memory impact is understood
[ ] hot-path performance is measured
[ ] tests target contracts
[ ] migration strategy exists
```


# 126. Common Misconceptions

### “More abstraction is better.”
No. Abstraction has a maintenance and cognitive cost.

### “Composition always beats inheritance.”
No. Valid subtype relationships can make inheritance appropriate.

### “Dependency injection means a framework container.”
No. Passing a dependency as an argument is dependency injection.

### “Interfaces must be generic.”
No. A useful interface should model real substitution.

### “Duplication is always bad.”
Sometimes duplicated code is clearer than an unstable abstraction.

### “A facade should hide everything.”
No. It should simplify the parts callers should not need to manage.

### “Adapters should expose provider details for flexibility.”
That defeats the isolation purpose unless explicitly intended.

### “Composition is free.”
Wrappers, objects, functions, and conversions can have runtime costs.

### “A switch statement means poor design.”
Not necessarily. Simple stable variation can be clearer than an abstraction hierarchy.


# 127. Common Mistakes

```text
[ ] abstracting before variation is understood
[ ] creating generic interfaces with no stable meaning
[ ] injecting giant objects instead of capabilities
[ ] hiding dependencies in globals
[ ] using service locators everywhere
[ ] leaking third-party types across boundaries
[ ] over-wrapping simple functions
[ ] creating circular dependency workarounds
[ ] forcing unrelated implementations into one interface
[ ] ignoring failure semantics
[ ] ignoring observability
[ ] optimizing theoretical purity over practical clarity
```


# 128. Spaced Retrieval Plan

### Day 0
Explain:

```text
abstraction
composition
cohesion
coupling
dependency injection
delegation
adapter
strategy
decorator
ports/adapters
```

### Day 2
Refactor one global dependency into explicit injection.

### Day 7
Refactor a conditional provider selector into composition.

### Day 14
Design a safe plugin boundary.

### Day 30
Review a real architecture and predict which changes would have the largest blast radius.


# 129. Concept Connections

## Depends On

- Functions
- Closures
- Objects
- Prototypes
- Data structures
- Complexity
- Core algorithms
- Functional programming
- Object-oriented programming

## Builds Toward

- Chapter 77 — Design Patterns
- Chapter 78 — Production JavaScript Architecture
- Chapter 79 — API Design
- Chapter 80 — Library Authoring
- Chapter 82 — API Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 89 — Code Review and Refactoring
- Chapters 101–111 — Real-world production scenarios/projects

## Related Concepts

```text
SOLID
dependency inversion
hexagonal architecture
clean architecture
domain-driven design
plugin architecture
middleware
API design
module boundaries
event-driven systems
```

## Why This Chapter Matters Later

Design patterns are named forms of recurring composition decisions.

Before memorizing:

```text
Strategy
Adapter
Decorator
Factory
Facade
```

you should understand:

```text
why composition works
where boundaries belong
what coupling costs
```

That foundation makes Chapter 77 useful instead of mnemonic.


# 130. Completion Criteria

```text
[ ] Define abstraction
[ ] Define composition
[ ] Explain cohesion
[ ] Explain coupling
[ ] Explain dependency direction
[ ] Explain stability/volatility
[ ] Explain policy vs mechanism
[ ] Explain delegation
[ ] Explain dependency injection
[ ] Explain inversion of control
[ ] Explain composition root
[ ] Explain ports and adapters
[ ] Implement adapters
[ ] Implement strategies
[ ] Implement decorators
[ ] Implement facades
[ ] Implement factories
[ ] Explain builders
[ ] Explain capability injection
[ ] Explain module boundaries
[ ] Explain abstraction leakage
[ ] Explain temporal coupling
[ ] Explain semantic coupling
[ ] Analyze abstraction tax
[ ] Analyze runtime/memory impact
[ ] Analyze security capability boundaries
[ ] Refactor toward composition
[ ] Predict change blast radius
[ ] Defend architecture
```


# 131. Mastery Gate

### Understand
You can explain composition and abstraction without relying on pattern names.

### Explain
You can identify cohesion, coupling, volatility, and stable contracts.

### Predict
You can predict how a change propagates through a dependency graph.

### Implement
You can build adapters, strategies, decorators, factories, and composition roots.

### Debug
You can locate abstraction leakage, hidden dependencies, cycles, and wrapper explosion.

### Apply
You can refactor real systems toward explicit boundaries.

### Compare
You can defend composition, inheritance, functions, and simpler alternatives.

### Defend
You can explain an architecture in terms of:

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


# 132. Status

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


# Chapter 76 — Revision / Retrieval Record

| Date | Retrieval task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain abstraction vs composition | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Identify cohesion/coupling | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Refactor hidden dependency | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement adapter | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement strategy | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Review wrapper stack | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Break circular dependency | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal architecture review | ✅ / ❌ | ... | ... |

### Retrieval Prompts

```text
1. What problem does abstraction solve?
2. What problem does composition solve?
3. What is the difference between coupling and cohesion?
4. How do you identify a stable boundary?
5. When is duplication better?
6. What makes an adapter useful?
7. What makes a strategy abstraction useful?
8. Where should dependencies be assembled?
9. How can composition improve security?
10. How can composition hurt performance?
11. How would you break a circular dependency?
12. How would you reduce wrapper explosion?
13. How would you design a plugin boundary?
14. How would you predict change blast radius?
15. What evidence tells you an abstraction is earning its cost?
```


# Chapter 76 — Canonical References and Source Discipline

## Primary JavaScript references

- ECMAScript Language Specification  
  https://tc39.es/ecma262/
- MDN Functions  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Functions
- MDN Modules  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Modules
- MDN Classes  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Classes

Use these for language-level semantics.

## Software design references

Use established software-engineering literature for:

```text
cohesion
coupling
dependency inversion
composition
separation of concerns
design patterns
architecture boundaries
```

The terminology is broader than JavaScript and should not be treated as a language feature.

## Source discipline

1. Prefer the smallest abstraction that solves the actual boundary problem.
2. Treat composition, inheritance, and functions as design tools rather than ideology.
3. State the reason a dependency is replaceable.
4. Keep volatile third-party details out of stable domain contracts where practical.
5. Do not create generic interfaces without genuine common semantics.
6. Treat runtime performance claims as workload-dependent and benchmark hot paths.
7. Treat memory/object-graph growth as part of architecture review.
8. Design capability boundaries explicitly for security-sensitive components.
9. Keep failure and retry semantics part of the contract.
10. Test observable behavior rather than internal wiring unless wiring itself is the contract.
11. Prefer incremental refactoring over risky rewrites when production constraints require continuity.
12. Delete abstractions that stop earning their maintenance cost.


# Chapter 76 — Completion Snapshot

## Core Theory

```text
[ ] abstraction
[ ] composition
[ ] cohesion
[ ] coupling
[ ] dependency direction
[ ] stability
[ ] volatility
[ ] policy/mechanism
[ ] delegation
```

## Composition Techniques

```text
[ ] dependency injection
[ ] composition root
[ ] strategy
[ ] adapter
[ ] decorator
[ ] facade
[ ] factory
[ ] builder
[ ] middleware
[ ] registry
```

## Architecture

```text
[ ] ports/adapters
[ ] module boundaries
[ ] capability injection
[ ] anti-corruption boundary
[ ] dependency graph
[ ] circular dependency analysis
[ ] change isolation
```

## Production

```text
[ ] error semantics
[ ] retry semantics
[ ] observability
[ ] performance
[ ] memory
[ ] security
[ ] migration
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

The purpose of composition is not to make code “modular” in the abstract.

The purpose is:

```text
change isolation
+
clear ownership
+
controlled coupling
+
replaceable behavior
```

A principal engineer sees an architecture as a graph:

```text
components
   ↓
dependencies
   ↓
contracts
   ↓
change propagation
```

The key questions are:

```text
What is stable?
What is volatile?
Who owns this rule?
Who owns this state?
What can vary independently?
What knowledge crosses the boundary?
What failure semantics cross the boundary?
What capabilities are granted?
Can I replace this implementation?
Can I test the core without infrastructure?
Can I observe the composed path?
What is the runtime cost?
What is the memory cost?
Is the abstraction earning its maintenance cost?
```

The strongest design may be:

```text
pure function
```

or:

```text
small object
```

or:

```text
adapter
```

or:

```text
class
```

or:

```text
composition of several capabilities
```

There is no prize for using the most patterns.

There is value in making the system's **change boundaries obvious**.

The deepest lesson is:

> **Abstraction should reduce the amount of system you must understand when making a change. Composition is the mechanism that lets those abstractions remain replaceable.**

The curriculum now moves from design principles to named recurring solutions:

```text
composition
+
abstraction
      ↓
recurring design solutions
      ↓
Chapter 77 — Design Patterns


# 133. Extended Retrieval Bank

### Retrieval Drill 1

Given a system with approximately `10` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 2

Given a system with approximately `20` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 3

Given a system with approximately `30` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 4

Given a system with approximately `40` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 5

Given a system with approximately `50` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 6

Given a system with approximately `60` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 7

Given a system with approximately `70` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 8

Given a system with approximately `80` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 9

Given a system with approximately `90` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 10

Given a system with approximately `100` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 11

Given a system with approximately `110` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 12

Given a system with approximately `120` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 13

Given a system with approximately `130` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 14

Given a system with approximately `140` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 15

Given a system with approximately `150` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 16

Given a system with approximately `160` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 17

Given a system with approximately `170` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 18

Given a system with approximately `180` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 19

Given a system with approximately `190` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 20

Given a system with approximately `200` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 21

Given a system with approximately `210` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 22

Given a system with approximately `220` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 23

Given a system with approximately `230` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 24

Given a system with approximately `240` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 25

Given a system with approximately `250` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 26

Given a system with approximately `260` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 27

Given a system with approximately `270` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 28

Given a system with approximately `280` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 29

Given a system with approximately `290` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 30

Given a system with approximately `300` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 31

Given a system with approximately `310` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 32

Given a system with approximately `320` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 33

Given a system with approximately `330` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 34

Given a system with approximately `340` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 35

Given a system with approximately `350` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 36

Given a system with approximately `360` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```

### Retrieval Drill 37

Given a system with approximately `370` components, identify:

```text
1. the most volatile dependency
2. the most stable concept
3. one inappropriate coupling
4. one useful abstraction boundary
5. one composition opportunity
6. one abstraction that should probably be removed
7. the likely blast radius of changing the volatile dependency
8. one observability boundary
```

Then defend the decision using:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
future change
```