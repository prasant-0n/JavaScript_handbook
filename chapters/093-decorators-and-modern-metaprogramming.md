# Chapter 93 — Decorators

> **JavaScript Mastery — Part XVII: Modern ECMAScript & Language Evolution**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Current verification:** 2026-09-10
>
> **Current standards state:** The official TC39 proposals repository currently lists **Decorators at Stage 2.7**. The official proposal repository identifies Kristen Hewell Garrett as champion and describes the current proposal as work in progress; its history spans multiple years of design iteration. citeturn946931search0turn946931search1

---

# 0. Chapter Mission

Decorators are not “just annotations.”

They are a metaprogramming facility for participating in class definition and class-element initialization. The central engineering questions are:

```text
What value is being decorated?
What is its semantic kind?
What context is supplied?
Can the decorator replace the value?
Can it schedule initialization work?
What execution-time effects occur?
What information crosses the decorator boundary?
What is standardized versus transpiler-specific?
```

This chapter develops decorators from first principles:

```text
Motivation
→ language model
→ decorator expressions
→ evaluation order
→ context objects
→ return semantics
→ replacement
→ addInitializer
→ fields/accessors/methods/classes
→ private elements
→ static elements
→ composition
→ inheritance
→ errors
→ metadata ecosystem
→ transpilers
→ production architecture
→ specification semantics
```

The official proposal says decorators are functions called on classes, class elements, and related class-definition forms; they can replace decorated values with matching-semantic replacements, add initialization behavior, and observe contextual information. citeturn946931search1

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- Explain what the current ECMAScript Decorators proposal is.
- Explain why decorators exist.
- Distinguish decorators from comments and annotations.
- Explain decorators as metaprogramming rather than magic syntax.
- Explain how decorator expressions are evaluated.
- Explain class-definition timing versus instance-construction timing.
- Explain the decorator context object.
- Explain `kind`.
- Explain `name`.
- Explain `static`.
- Explain `private`.
- Explain `access`.
- Explain `addInitializer()`.
- Explain decorator return semantics.
- Explain replacing a method.
- Explain replacing a field initializer.
- Explain decorating accessors.
- Explain decorating private elements.
- Explain decorating static elements.
- Explain class decorators.
- Explain decorator ordering.
- Explain multiple decorators and composition.
- Explain why decorator order is observable.
- Explain what happens when decorators throw.
- Explain why an `addInitializer` callback is not simply a constructor wrapper.
- Explain the distinction between definition-time effects and initialization-time effects.
- Explain how decorators interact with inheritance.
- Explain the distinction between standard decorators and historical decorator proposals.
- Explain why Babel/TypeScript legacy decorators are not automatically equivalent to standard decorators.
- Explain why decorator metadata is a separate proposal.
- Evaluate decorator performance and memory costs.
- Evaluate decorator security risks.
- Design a production decorator policy.
- Implement useful decorators without hiding core application behavior.
- Debug decorator execution-order problems.
- Read decorator proposal specification text.
- Answer senior and principal-level decorator interview questions.

---

# 2. Prerequisites

Especially important previous chapters:

```text
Chapter 09 — Functions / First-Class Behavior
Chapter 10 — Scope / Lexical Environments
Chapter 13 — Closures
Chapter 14 — this / Invocation / Binding
Chapter 15 — Objects / Property Semantics
Chapter 17 — Prototypes / Prototype Chains
Chapter 18 — Classes / OOP JavaScript
Chapter 19 — Proxy / Reflect / Metaprogramming
Chapter 20 — Symbols
Chapter 41 — Spec Architecture
Chapter 42 — Abstract Operations
Chapter 43 — Ordinary Object Internal Methods
Chapter 65 — CommonJS / Interoperability
Chapter 68 — Transpilation / Compilation
Chapter 77 — Design Patterns
Chapter 89 — Code Review / Refactoring
Chapter 91 — TC39 Proposal Tracking

# 3. What Are Decorators?

A decorator is a function participating in the definition of a class or class element.

Canonical conceptual shape:

```js
function decorate(value, context) {
  // inspect context
  // optionally return replacement
  // optionally call context.addInitializer(...)
}
```

The current proposal describes decorators as functions invoked with the value being decorated and a context object describing that decoration. citeturn946931search1

The key point is that the decorator is not merely an annotation stored in a class. It participates in semantics during class definition.


# 4. Why Do Decorators Exist?

Decorators address recurring metaprogramming problems around class-based APIs. Common examples include:

- wrapping methods,
- changing initialization behavior,
- registering classes,
- adding validation,
- implementing reactivity,
- method binding,
- instrumentation,
- dependency-injection integration.

The proposal README specifically calls out capabilities such as replacing decorated values and adding initialization behavior. citeturn946931search1

The important design goal is to provide these capabilities without forcing libraries to rely on broad prototype mutation or post-hoc class surgery.


# 5. Mental Model

Use this pipeline:

```text
class source
  ↓
decorator expressions evaluated
  ↓
class elements created / prepared
  ↓
decorators invoked in specified order
  ↓
replacement values returned where provided
  ↓
initializers collected
  ↓
class definition completes
  ↓
static initialization runs
  ↓
instance construction later runs instance initializers
```

Do not think:

```text
@decorator = comment
```

Think:

```text
@decorator = executable language semantics
```


# 6. Standard Decorators vs Legacy Decorators

This distinction is mandatory.

The modern proposal is the result of years of TC39 iteration. The current proposal README explicitly says it is a work in progress and has evolved from previous proposals. citeturn946931search1

Historical TypeScript/Babel decorator systems may expose:

- different arguments,
- different timing,
- different metadata behavior,
- descriptor-based mutation patterns,
- legacy class/parameter decorator forms.

Therefore:

```text
"We use decorators in TypeScript"
≠
"We use standardized ECMAScript decorators"
```

Before migrating, inspect the emitted semantics rather than relying on similar syntax.


# 7. Basic Syntax

Illustrative syntax:

```js
function logged(value, context) {
  console.log(context.kind, context.name);
  return value;
}

class Example {
  @logged
  method() {
    return 42;
  }
}
```

A production codebase should first use a small number of decorators whose behavior is easy to explain and test.


# 8. Decorator Context

The context object is central. The current proposal describes information such as:

```text
kind
name
access
static
private
addInitializer
metadata (in related metadata proposal context)
```

The exact shape differs by decoration target. The proposal README shows context information for class elements including `kind`, `name`, `access`, `isPrivate`, `isStatic`, and `addInitializer`. citeturn946931search1

Treat the context as a typed capability surface, not as a generic object full of accidental properties.


# 9. `kind`

The decorator must understand what it is decorating. Typical kinds include:

```text
class
method
getter
setter
field
accessor
```

A method decorator should not casually assume the value behaves like a field initializer. The value and context work together.


# 10. `name`

`name` identifies the element where the language semantics make a name meaningful. It may be a string or symbol for class elements, and the exact representation must be respected for symbol-keyed members.

Never build a decorator that assumes:

```js
typeof context.name === "string"
```

A robust decorator considers symbol names and private elements separately.


# 11. `static` and `private`

A decorator may need to distinguish:

```text
instance method
static method
public field
private field
static private field
```

For security and encapsulation, never assume a private name can be turned into an ordinary property key. The decorator context exposes controlled access capabilities rather than handing over arbitrary private syntax.


# 12. `access`

The context can expose controlled accessors for decorated elements. This can be used to build abstractions around an element while respecting the language's encapsulation model.

Conceptually:

```text
context.access.get()
context.access.set(value)
```

Not every decoration kind provides every access operation. Check the kind and the specific proposal semantics.


# 13. Returning a Replacement

A decorator can return a replacement value when the decorated element kind permits it.

Example:

```js
function logged(value, context) {
  if (context.kind !== "method") return;

  return function (...args) {
    console.log(`calling ${String(context.name)}`);
    return value.apply(this, args);
  };
}
```

The replacement should preserve the semantic contract expected by the language for that element kind. The proposal describes decorators as able to replace values with matching-semantic values. citeturn946931search1


# 14. `addInitializer()`

Decorators can register initializer functions.

Conceptually:

```js
function bound(value, context) {
  if (context.kind === "method") {
    context.addInitializer(function () {
      this[context.name] = this[context.name].bind(this);
    });
  }
}
```

The exact production implementation should be derived from the current proposal/spec semantics, but the architectural distinction is important:

```text
decorator execution
≠
initializer execution
```

An initializer is scheduled to run at the appropriate initialization phase.


# 15. Definition Time vs Initialization Time

This distinction explains most decorator bugs.

```text
Class evaluation / definition
        │
        ├── evaluate decorator expressions
        ├── call decorators
        ├── collect replacement values
        └── collect initializers

Later instance construction
        │
        └── run instance initializers at the defined point
```

Static decorators can have initialization behavior associated with class initialization; instance decorators can register work for each instance initialization.


# 16. Decorator Evaluation Order

Decorator ordering is observable.

Example:

```js
@outer
@inner
class C {}
```

You must distinguish:

```text
evaluation of decorator expressions
vs
invocation of decorators
vs
application/replacement ordering
vs
initializer execution ordering
```

Never teach ordering using a vague “top to bottom” rule. Track each phase separately. The specification is the authority.


# 17. Multiple Decorators and Composition

Decorators compose. A useful mathematical analogy is function composition:

```text
outer(inner(value))
```

But real decorators can also register initializers and inspect metadata/context, so composition is richer than ordinary pure function composition.

For maintainability, prefer decorators whose composition behavior is explicit and documented.


# 18. Field Decorators

Fields have an initialization value rather than a method body. A field decorator can participate in transforming or replacing field initialization behavior, depending on the current semantics.

The critical reasoning question is:

```text
Does the decorator receive the current field value?
What replacement shape is legal?
When does the replacement run?
Is it per instance or static?
```

Do not assume field decoration behaves exactly like method wrapping.


# 19. Method Decorators

Method decoration is often easier to visualize:

```js
function trace(value, context) {
  return function (...args) {
    console.log("enter", context.name);
    const result = value.apply(this, args);
    console.log("exit", context.name);
    return result;
  };
}
```

Be careful about:

- `this`,
- async return values,
- generator semantics,
- function `.name`,
- function `.length`,
- replacement identity,
- stack traces,
- source maps,
- performance.


# 20. Getter and Setter Decorators

Accessor decorators are sensitive to the difference between:

```text
read
write
combined accessor
```

A decorator that logs setters should not accidentally change getter semantics.

Review:

```text
value type
access direction
receiver semantics
exceptions
property descriptors
private access
```


# 21. Auto Accessors

Auto-accessors are important because they expose a higher-level abstraction than a simple public field.

A decorator may interact with an accessor pair and its initializer semantics differently from a method or field.

This is a specification-reading checkpoint:

```text
syntax surface
→ internal representation
→ decorator value
→ context
→ returned accessor object
→ initialization
```

Study the proposal repository's current examples before implementing sophisticated accessor decorators.


# 22. Class Decorators

A class decorator decorates the class value itself.

A conceptual shape:

```js
function register(value, context) {
  registry.set(context.name, value);
}
```

A class decorator can also replace the class with a compatible class value where permitted.

Principal concern:

```text
class identity
static initialization
subclassing
instanceof expectations
registry behavior
serialization/tooling
```


# 23. Static Elements

Static elements belong to the class rather than individual instances.

For a decorator, distinguish:

```text
static initialization
instance initialization
class value replacement
```

A decorator that captures state at the wrong phase can accidentally create shared state where per-instance state was intended, or vice versa.


# 24. Private Elements

Private class elements introduce an important security/encapsulation constraint.

A decorator does not simply receive a string like `"#secret"` and then access the private slot through bracket syntax. The proposal exposes controlled access information.

This protects the semantic distinction between:

```text
private name
vs
public property key
```

Do not design decorators that attempt to bypass private-field encapsulation.


# 25. Symbols

A class element can use a symbol key.

Therefore code like:

```js
String(context.name)
```

may be useful for logs, but should not be used as a canonical identity key. Preserve the symbol when identity matters.

Connection:

```text
Chapter 20 — Symbols
        ↓
Decorator context.name
```


# 26. `this` and Decorators

Wrapping a method changes the call path.

Correct wrapper:

```js
return function (...args) {
  return value.apply(this, args);
};
```

Incorrect patterns can accidentally capture the decorator's `this` or use an arrow function when dynamic receiver binding is required.

This is directly connected to Chapter 14.


# 27. Async Methods

When decorating an async method, preserving the returned promise is essential.

```js
return async function (...args) {
  try {
    return await value.apply(this, args);
  } finally {
    // trace end
  }
};
```

Be aware that `await` changes stack/exception timing and can create extra scheduling behavior. A decorator should not add an `await` merely to log.


# 28. Generators

A generator method returns an iterator object. A decorator that blindly assumes a synchronous return value can break semantics.

Test:

```text
next()
throw()
return()
iterator identity
````

Wrapping should preserve the generator contract or explicitly document the new behavior.


# 29. Error Propagation

Decorator execution can throw during class definition.

That means the failure can prevent the class from being defined at all.

Similarly, initializer execution can throw during class/static initialization or instance construction.

Production rule:

```text
decorator errors are class-definition/initialization errors
not “just logging errors.”
```


# 30. Initializer Errors

An initializer may fail later than the decorator call. This can create confusing stack traces if the team assumes all decorator work occurs once.

Debug by logging:

```text
decorator invoked
initializer registered
initializer executed
initializer instance/class identity
```

Use this only in development diagnostics, not as a permanent noisy production pattern.


# 31. Inheritance

Decorators interact with class inheritance indirectly through the classes/elements they modify and through initializers.

Questions:

- Is the decorator applied to the base class only?
- Is the resulting method inherited?
- Does an initializer run for subclass instances?
- Does a class replacement affect `extends`?
- Does state accidentally become shared?

Test base and subclass behavior separately.


# 32. Decorators Are Not Inheritance Hooks

A decorator can participate in class definition, but it should not be used as an excuse to hide all inheritance behavior.

If a design requires complicated subclass lifecycle rules, consider whether plain composition, explicit class methods, or a framework lifecycle API would be clearer.


# 33. Decorators and Composition

Decorator-heavy architecture can become difficult to reason about because behavior is distributed around declarations.

Compare:

```js
class Service {
  @transactional
  @authorized
  create() {}
}
```

with explicit composition:

```js
const create = authorized(transactional(coreCreate));
```

Neither is universally better. The right choice depends on whether cross-cutting behavior is fundamentally declaration-oriented and whether developers can trace it.


# 34. Decorators and Dependency Injection

Dependency injection is a common decorator use case in frameworks.

A decorator may register:

```text
provider metadata
constructor/class
property/method hooks
initialization work
````

But standard ECMAScript decorators do not automatically provide a complete DI container. The container remains an application/framework responsibility.


# 35. Decorators and Metadata

Decorator metadata is a separate TC39 proposal. The current repository lists **Decorator Metadata at Stage 3**. It proposes a metadata object available through the decorator context and a `Symbol.metadata` property on the class after decoration. citeturn946931search6

Therefore:

```text
decorators
≠
decorator metadata
```

Track the proposals separately in a proposal intelligence system.


# 36. Why Metadata Is Separate

A decorator can transform a value without needing a permanent metadata registry. Metadata introduces a different capability: allowing external code to inspect information associated with decorated values.

That raises separate questions:

- ownership,
- lifetime,
- inheritance,
- visibility,
- serialization,
- collisions,
- security.

Do not invent custom global metadata semantics where the standard proposal already defines a future direction.


# 37. Legacy Parameter Decorators

The current TC39 proposal for class method/constructor parameter decorators remains Stage 1. The repository explicitly describes it as an extension to the current decorators design. citeturn946931search9

This is an excellent example of proposal separation:

```text
class/method/field decorators
= current core decorators proposal

parameter decorators
= separate proposal
```

Never imply that Stage 2.7 Decorators automatically includes parameter decorators.


# 38. Decorator Expression Evaluation

Example:

```js
@factory()
class C {}
```

The expression `factory()` itself is evaluated as part of the language's decorator evaluation semantics. That can have side effects.

Therefore:

```text
Decorator expression evaluation
can be observable
```

Avoid decorators with surprising global side effects in module initialization.


# 39. Evaluation Side Effects

A declaration can look declarative while executing arbitrary JavaScript through decorator expressions.

Bad pattern:

```js
@registerIntoGlobalRegistryThatTalksToNetwork()
class Service {}
```

This makes module evaluation depend on infrastructure.

Prefer deterministic registration and explicit application bootstrap.


# 40. Decorator Factories

Factories can make decorators configurable:

```js
function retry(times) {
  return function (value, context) {
    // ...
  };
}
```

Use factories when configuration materially changes behavior.

Avoid deep factory nesting that makes stack traces and debugging difficult.


# 41. Decorator Composition Order

Suppose:

```js
@cache
@trace
method() {}
```

Possible intent:

```text
cache(trace(original))
````

Or the reverse.

A production team must define an ordering policy. For example:

```text
security → authorization → tracing → caching
```

if that matches the framework semantics.

Do not rely on visual proximity alone.


# 42. Idempotency

A decorator may be applied more than once accidentally. Ask whether repeated application should:

```text
be harmless
stack behavior
replace previous wrapper
throw
````

Idempotency is especially important for build tooling and code generation.


# 43. Decorator Identity and Replacement

If a decorator replaces a method with another function, identity changes. That can affect:

- equality comparisons,
- reflection,
- stack traces,
- instrumentation,
- function names,
- test spies,
- caching keyed by function identity.

The principal engineer asks:

> What observable identity changes does this decorator introduce?


# 44. Performance

Decorator overhead can come from:

```text
wrapper calls
closure allocation
initializer allocation
per-instance setup
metadata structures
reflection
logging
````

A method decorator that adds a wrapper to a million hot calls can cost more than the syntax suggests.

Benchmark:

```text
raw method
vs
one decorator
vs
multiple decorators
````

Use realistic workloads.


# 45. Memory

A decorator can accidentally retain objects through closures.

Example failure shape:

```text
decorator closure
   ↓
large registry/configuration object
   ↓
class/method lifetime
   ↓
unexpected retention
```

Review captured variables carefully.


# 46. Security

Decorator code executes with the authority of the module/class context.

Risks include:

- hidden authorization changes,
- method replacement that bypasses validation,
- registration of secrets,
- unintended data exposure via metadata,
- global side effects during module load,
- supply-chain risks through decorator packages.

Decorators are metaprogramming. Treat them as executable policy.


# 47. Supply Chain

A dependency can introduce decorators without making the code visibly look different to every reviewer.

Security review should trace:

```text
source
→ decorator package
→ factory
→ replacement/initializer
→ runtime effects
````

Pin dependencies and review decorator libraries like any other executable dependency.


# 48. Debugging Workflow

When decorated behavior is wrong:

```text
1. Identify target element kind.
2. Log decorator invocation.
3. Inspect context.
4. Inspect returned replacement.
5. Record initializers registered.
6. Determine when initializer executes.
7. Compare base/subclass behavior.
8. Remove decorators one at a time.
9. Reproduce with minimal class.
10. Read the exact proposal/spec rule involved.
```


# 49. Debugging Exercise — Wrapper `this`

Bug:

```js
function broken(value) {
  return (...args) => value(...args);
}
```

Question:

> What can this break?

Answer direction:
The arrow function captures lexical `this`; a method replacement may need the caller's receiver. The wrapper should normally preserve receiver semantics explicitly.


# 50. Debugging Exercise — Async Logging

Bug:

```js
function log(value) {
  return function (...args) {
    const result = value.apply(this, args);
    console.log("done");
    return result;
  };
}
```

Question:

> Does “done” mean the async operation completed?

No. It only means the promise-returning function returned. A correct async lifecycle logger must define whether it tracks invocation or eventual completion.


# 51. Debugging Exercise — Initialization Phase

Suppose an initializer expects an instance property to already exist but the initializer actually runs before the property has been initialized.

The bug is a timing assumption.

Fix by consulting the exact initialization order and rearranging either the decorator or the class element.


# 52. Code Review Exercise

Review:

```js
@cache
@auth
@log
class UserService {}
```

Ask:

- What does each decorator return?
- Which one wraps the class?
- What is the ordering?
- Is authorization applied before caching?
- Does cache key identity change?
- Is logging occurring at class definition or request time?
- Are side effects happening during module load?
- How are errors handled?
- What happens to subclasses?

If the team cannot answer these, the decorator stack is too opaque.


# 53. Production Usage — Good Candidates

Good candidates include:

- instrumentation,
- narrowly scoped validation,
- declarative registration,
- framework integration,
- cross-cutting class lifecycle behavior,
- method wrappers with clear contracts.

Good decorators are:

```text
small
predictable
composable
testable
well documented
````


# 54. Production Usage — Bad Candidates

Avoid decorators for:

- core business rules hidden from callers,
- network calls during class definition,
- hidden database access,
- global mutable registries without lifecycle policy,
- complex authorization logic that reviewers cannot see,
- behavior where explicit composition is clearer.


# 55. Architecture Rule

Use decorators at architectural boundaries, not everywhere.

A practical rule:

```text
If the behavior is about “what this declaration means”
→ decorator may fit.

If the behavior is the core business algorithm
→ explicit code usually fits better.
``


# 56. Testing Strategy

Decorator tests should cover:

```text
correct target kind
correct context
replacement behavior
initializer timing
multiple decorators
error propagation
static behavior
instance behavior
private behavior
inheritance
symbol names
async methods
generators
``

Also test the undecorated implementation so that the decorator is clearly isolated as a transformation.


# 57. Snapshot Testing Pitfall

Snapshot tests can hide semantic changes in decorator-generated structures. Prefer behavioral tests for:

- invocation,
- initialization,
- errors,
- return values,
- identity,
- ordering.


# 58. TypeScript Considerations

TypeScript's type system can help describe decorator signatures, but type support does not prove runtime semantics are standard.

Track separately:

```text
TypeScript syntax support
TypeScript emitted output
Runtime native support
TC39 stage
``


# 59. Babel Considerations

Babel may transform decorator syntax for environments that do not implement it natively. The transform mode and version matter.

Do not assume:

```text
Babel decorator plugin
=
current ECMAScript semantics
``

Inspect the exact compiler configuration.


# 60. Source Maps and Debugging

Transformed decorators can complicate debugging because source code and runtime code differ.

Check:

- stack traces,
- source maps,
- breakpoint locations,
- wrapper names,
- generated helper functions.


# 61. Runtime Detection

For native standard decorators, syntax itself may require parser support. A runtime `typeof` check is not enough when older environments cannot parse the source.

This is a build-time compatibility problem as well as a runtime problem.


# 62. Polyfill Limits

Syntax cannot be fully “polyfilled” by ordinary runtime code if the parser cannot understand the syntax.

Therefore support options may be:

```text
native runtime
transpilation
alternate source build
``

The distinction connects directly to Chapter 68.


# 63. Specification Perspective

The proposal repository contains specification-oriented material and extensive design history. citeturn946931search1

For advanced mastery, read the formal algorithms to understand:

```text
decorator expression evaluation
call timing
context construction
replacement validation
initializer queues
class definition sequencing
error propagation
``

Then map each algorithm back to the observable examples.


# 64. Abstract-Operation Lens

Ask the Chapter 42 questions:

```text
What value is obtained?
What type/record is created?
What abstract operation is called?
Where can user code run?
Where can abrupt completion occur?
What state is stored?
What is observable?
``

Decorators are a powerful exercise because they combine evaluation, callable values, class definitions, and initialization.


# 65. Internal-State Lens

Think in terms of records and queues:

```text
decorator result
replacement value
initializer list
class elements
static initialization
instance initialization
``

The implementation details differ by engine, but the specification model defines the required observable behavior.


# 66. Edge Case — Decorator Returns Wrong Shape

If a decorator returns a value that does not satisfy the requirements for the target kind, class definition can fail.

Rule:

```text
replacement contracts are semantic contracts
``

Do not return arbitrary objects because they are “close enough.”


# 67. Edge Case — Decorator Throws

A decorator can throw. This can abort class definition.

Test:

```js
try {
  class C {
    @throws
    method() {}
  }
} catch (error) {
  // class definition failed
}
```

Do not assume `C` exists after the failed definition.


# 68. Edge Case — Multiple Initializers

If multiple decorators register initializers, ordering matters. Document the expected order and test it.

A useful test records:

```text
initializer A
initializer B
initializer C
``

and verifies the specified order.


# 69. Edge Case — Subclass Construction

If a base class decorator registers instance initialization behavior, subclass construction may still trigger base-class initialization as part of the inherited constructor process.

Do not infer behavior from “decorator applied only to base” without checking actual initialization semantics.


# 70. Implementation — Guided Decorator

Implement a simple method logger:

```js
function trace(value, context) {
  if (context.kind !== "method") return;

  return function (...args) {
    console.log("enter", String(context.name));
    try {
      return value.apply(this, args);
    } finally {
      console.log("exit", String(context.name));
    }
  };
}
```

Requirements:

- preserve `this`,
- preserve thrown errors,
- explain async limitations.


# 71. Implementation — Partially Guided

Create a `timed` decorator that:

- measures synchronous execution,
- handles rejected promises correctly,
- reports method name,
- does not alter the return type intentionally,
- has configurable sampling.

Document whether the timer is monotonic.


# 72. Implementation — No Reference

Build a `retry` decorator that supports:

```text
max attempts
backoff strategy
allowed errors
async methods only
``

Then ask whether decorators are actually the best abstraction for your retry system. Defend the decision.


# 73. Implementation — Edge-Case Hardened

Harden the decorator against:

- symbol method names,
- private elements where allowed,
- getters/setters,
- async methods,
- repeated decoration,
- thrown errors,
- non-function replacements.


# 74. Implementation — Production Grade

Package a decorator toolkit with:

```text
src/
  decorators/
    trace.js
    metrics.js
    retry.js
    validate.js
  policy.js
  errors.js

tests/
  decorator-order.test.js
  initializer.test.js
  async.test.js
  inheritance.test.js
``

Requirements:

- API documentation,
- compatibility policy,
- benchmark suite,
- security review,
- migration notes from legacy decorators.


# 75. Principal Decision Framework

Evaluate a decorator system using:

```text
Correctness
Performance
Memory
Security
Reliability
Maintainability
Scalability
Observability
Developer Experience
Operational Complexity
Future Change
```

A decorator that saves 15 lines of code but makes debugging 2 hours slower is not automatically an improvement.


# 76. Interview Questions — Fundamentals

1. What is an ECMAScript decorator?
2. Why do decorators exist?
3. What is a decorator context?
4. What does `kind` mean?
5. What is `addInitializer()`?
6. Can a decorator replace a method?
7. Can a decorator replace a class?
8. What is the difference between static and instance decoration?
9. Why are private elements special?
10. Why is decorator order observable?


# 77. Interview Questions — Senior

1. Why are standard decorators different from legacy TypeScript decorators?
2. How would you preserve `this` in a method wrapper?
3. How do decorators affect method identity?
4. When does decorator code execute?
5. When do initializers execute?
6. How would you test decorator order?
7. How can decorators create memory leaks?
8. How can decorators increase security risk?
9. How do decorators interact with async methods?
10. Why is metadata a separate proposal?


# 78. Interview Questions — Principal

1. Should a large Node.js platform standardize on decorators?
2. How would you migrate a legacy decorator ecosystem?
3. How would you prevent decorator stacks from becoming untraceable?
4. How would you benchmark decorator overhead?
5. What governance would you create around framework-level decorators?
6. How would you handle a proposal change before standardization?
7. How would you design decorator security review?
8. When would explicit composition beat decorators?
9. How would you debug an inheritance bug caused by initializers?
10. How would you communicate decorator risks to application teams?


# 79. Predict-the-Result Exercise

Before executing, predict the order:

```text
@A
@B
class C {}
```

Then instrument both decorator expression evaluation and decorator invocation. Record both orders separately.

The lesson is methodological: never collapse multiple specification phases into one “decorators run in this order” sentence.


# 80. Predict-the-Result Exercise — Method Wrapper

Predict:

```js
class A {
  @trace
  method() {
    return this.value;
  }
}

const a = new A();
a.value = 10;
console.log(a.method());
```

Then test the same method when detached:

```js
const f = a.method;
console.log(f());
```

Use the result to explain receiver binding rather than blaming decorators generically.


# 81. Mastery Exercise — Decorator Trace

Build a reusable tracing decorator that records:

```text
class/element name
kind
static/private flags
invocation count
execution duration
errors
``

Keep the instrumentation out of the core business method.


# 82. Mastery Exercise — Initializer Laboratory

Create a class with:

```text
static element
instance field
instance method
private field
auto-accessor where supported by your environment
``

Decorate each where legal and record:

```text
decorator invocation order
initializer registration order
initializer execution order
``


# 83. Mastery Exercise — Legacy Migration

Take a legacy decorator example from a real TypeScript/Babel codebase. Document:

```text
old arguments
old timing
old descriptor semantics
metadata assumptions
emitted code
new standard-decorator model
compatibility gaps
``

Do not assume syntactic similarity means semantic compatibility.


# 84. Mastery Exercise — Framework Review

Pick a framework that uses decorators heavily. Analyze:

```text
why decorators are useful
what becomes implicit
what becomes observable
what debugging cost is introduced
what metadata model is used
what compiler assumptions exist
``

Then propose one place where explicit APIs would be clearer.


# 85. Mastery Exercise — Security Review

Design a threat model for a decorator package used by many services. Include:

```text
package compromise
code execution at module load
method replacement
authorization bypass
metadata exfiltration
initializer side effects
``

Produce mitigations.


# 86. Spaced Retrieval Schedule

### Day 0
Explain the decorator context.

### Day 1
Explain decorator invocation vs initializer execution.

### Day 3
Explain standard vs legacy decorators.

### Day 7
Reproduce ordering rules from memory.

### Day 14
Build a method decorator without notes.

### Day 30
Review a real framework decorator stack.

### Day 60
Design an organizational decorator policy.

### Day 90
Defend or reject decorators for a production platform.


# 87. Dependency Graph

```text
Chapter 14 — this / Invocation / Binding
          │
          └────→ method decorator wrappers

Chapter 18 — Classes / OOP
          │
          └────→ class/element decoration

Chapter 19 — Proxy / Reflect / Metaprogramming
          │
          └────→ decorator mental model

Chapter 42 — Abstract Operations
          │
          └────→ specification reasoning

Chapter 68 — Transpilation / Compilation
          │
          └────→ decorator tooling

Chapter 91 — TC39 Proposal Tracking
          │
          └────→ proposal maturity reasoning

Chapter 93 — Decorators
          │
          ├────→ Chapter 94 Compatibility Engineering
          └────→ framework/metaprogramming architecture
```


# 88. Concept Connections

## Depends On

- functions and closures,
- `this`,
- classes,
- metaprogramming,
- abstract operations,
- transpilation,
- proposal tracking.

## Builds Toward

- compatibility engineering,
- production architecture,
- library authoring,
- framework design,
- security engineering.

## Related Concepts

- Proxy,
- Reflect,
- annotations,
- metadata,
- dependency injection,
- aspect-oriented programming.

## Concepts Revisited

- class definition semantics,
- closures,
- `this`,
- property identity,
- proposal maturity.

## Why This Chapter Matters Later

Decorators are where language-level metaprogramming meets framework architecture. Their power is real; so is their ability to hide control flow.


# 89. Production Checklist

```text
[ ] Standard vs legacy semantics explicitly identified
[ ] Proposal stage recorded
[ ] Runtime support verified
[ ] Toolchain support verified
[ ] Decorator ordering documented
[ ] Initializer timing tested
[ ] `this` semantics tested
[ ] Error propagation tested
[ ] Inheritance tested
[ ] Async methods tested
[ ] Symbol/private cases considered
[ ] Benchmark completed for hot paths
[ ] Security review completed
[ ] Dependency/package risk assessed
[ ] Migration/rollback plan exists
```


# 90. Final Mental Model

```text
Decorator
= executable metaprogramming during class definition

Context
= controlled information about the decoration target

Replacement
= a new semantically valid value

Initializer
= scheduled initialization work

Decorator stack
= ordered composition with observable semantics

Standard decorator
≠ legacy TypeScript/Babel decorator

Decorator Metadata
= separate proposal, separate maturity

Production decorator policy
= explicit scope + tests + compatibility + observability
```

The principal engineer's question is:

> “What declaration-time capability are we introducing, and is the resulting implicit behavior worth the governance and debugging cost?”


# 91. Canonical References

Primary sources:

1. TC39 current proposal tracker
   https://github.com/tc39/proposals

2. ECMAScript Decorators proposal repository
   https://github.com/tc39/proposal-decorators

3. ECMAScript Decorators proposal specification
   https://tc39.es/proposal-decorators/

4. Decorator Metadata proposal
   https://github.com/tc39/proposal-decorator-metadata

5. Class Method / Constructor Parameter Decorators proposal
   https://github.com/tc39/proposal-class-method-parameter-decorators

6. TC39 process
   https://tc39.es/process-document/

Current-source notes:
- The official proposals tracker lists Decorators at Stage 2.7 as of this verification. citeturn946931search0
- The official Decorators proposal README states Stage 2.7 and describes the current proposal as work in progress. citeturn946931search1
- Decorator Metadata is currently Stage 3 and is a separate proposal. citeturn946931search6
- Parameter decorators remain a separate Stage 1 proposal. citeturn946931search9

Because proposal status can change, re-check the official sources before production adoption.


# 92. Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Can I explain decorators without notes? [ ]
- Can I explain context fields? [ ]
- Can I explain replacement semantics? [ ]
- Can I explain initializer timing? [ ]
- Can I explain ordering? [ ]
- Can I distinguish standard and legacy decorators? [ ]
- Can I explain metadata as separate proposal? [ ]
- Can I debug a decorator stack? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```


# 93. Completion Snapshot

```text
Track A — Core Theory
[ ] Explain decorator motivation
[ ] Explain context
[ ] Explain replacement
[ ] Explain initializer semantics
[ ] Explain ordering
[ ] Explain classes/elements/static/private
[ ] Explain spec perspective
[ ] Explain standard vs legacy

Track B — Implementation
[ ] Build trace decorator
[ ] Build async-safe decorator
[ ] Test ordering
[ ] Test initialization
[ ] Harden edge cases
[ ] Build production decorator package

Track C — Interview / Reasoning
[ ] Explain when decorators fit
[ ] Reject decorators when inappropriate
[ ] Evaluate performance
[ ] Evaluate memory
[ ] Evaluate security
[ ] Design governance policy
[ ] Design migration policy

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```


# 94. Completion Criteria

Do not mark this chapter mastered because you know the `@` syntax. You are ready to move forward when you can independently:

1. Explain the current TC39 Decorators status and verify it from primary sources.
2. Explain what a decorator receives and what it can return.
3. Explain decorator evaluation versus decorator invocation.
4. Explain initializer registration versus initializer execution.
5. Explain static versus instance decoration.
6. Explain private-element constraints.
7. Explain composition and ordering.
8. Preserve `this` in a method wrapper.
9. Handle async and generator methods without breaking contracts.
10. Diagnose errors during class definition and initialization.
11. Distinguish standard decorators from legacy transpiler systems.
12. Explain why decorator metadata is separate.
13. Build a decorator test matrix.
14. Evaluate performance and memory costs.
15. Perform a security review.
16. Decide when explicit composition is better.
17. Defend an organizational decorator policy.

---

# 95. Principal Challenge

Design a decorator policy for a 500-engineer Node.js platform.

Your policy must define:

```text
Allowed decorator categories
Naming rules
Ordering rules
Initializer restrictions
Async rules
Security review
Performance budget
Testing requirements
Legacy compatibility
Tooling baseline
Migration policy
Observability policy
Rollback strategy
Exception process
```

Then answer:

```text
Which decorators become platform-approved?
Which require architecture review?
Which are prohibited?
Why?
```

---

# 96. Principal Review Memo Template

```md
# Decorator Architecture Review

## Purpose
-

## Proposal / Runtime State
-

## Decorator Contract
-

## Execution Model
-

## Ordering
-

## Initialization
-

## Performance
-

## Memory
-

## Security
-

## Tooling
-

## Compatibility
-

## Debuggability
-

## Recommendation
- Adopt / Pilot / Restrict / Reject

## Preconditions
-

## Revisit Trigger
-
```

---

# 97. Compact Reference Card

```text
Decorator
= metaprogramming capability used during class definition

Context
= information/capabilities describing target

Replacement
= semantically valid new value

Initializer
= work scheduled for a later initialization phase

Order
= observable; distinguish evaluation, invocation, replacement, initialization

Private
= controlled access, not ordinary property-key access

Standard
≠ legacy TypeScript/Babel semantics

Metadata
= separate proposal

Production rule
= use where declaration-oriented cross-cutting behavior is clearer than explicit composition
```

---

# 98. Appendix — Decorator Test Matrix

| Case | Test | Expected question |
|---|---|---|
| Method | sync | Is `this` preserved? |
| Method | async | Is promise behavior preserved? |
| Generator | iterator | Is generator contract preserved? |
| Field | instance | When does replacement initialize? |
| Static field | static | When does initialization run? |
| Getter | read | Is access behavior preserved? |
| Setter | write | Is assignment behavior preserved? |
| Symbol name | symbol | Is identity preserved? |
| Private | private | Is access capability respected? |
| Multiple decorators | stack | Is order correct? |
| Initializers | multiple | Is execution order correct? |
| Error | throw | Does class definition/initialization abort correctly? |
| Inheritance | subclass | What initializes for base/subclass? |

# Chapter 93 — Revision / Retrieval Record

```text
Current status: [ ] Not Started
Verification date: 2026-09-10
Primary current status: Stage 2.7
Next review trigger: any TC39 stage change, major specification change, or runtime/tooling baseline change
```

---

# Chapter 93 — Canonical References and Source Discipline

The proposal status claims in this chapter are date-stamped and should be reverified. Primary sources:

- https://github.com/tc39/proposals
- https://github.com/tc39/proposal-decorators
- https://tc39.es/proposal-decorators/
- https://github.com/tc39/proposal-decorator-metadata
- https://github.com/tc39/proposal-class-method-parameter-decorators
- https://tc39.es/process-document/

Status verification used the official TC39 proposals tracker and official proposal repositories. citeturn946931search0turn946931search1

---

# Chapter 93 — Completion Snapshot

```text
Chapter 91 — TC39 Proposal Tracking  [ ]
Chapter 92 — Temporal               [ ]
Chapter 93 — Decorators             [ ]
Chapter 94 — Compatibility Engineering [ ]
```

Reading is not completion. Demonstrated reasoning, implementation, debugging, and defense are required.

---

# 99. Advanced Semantics Drill — Value, Context, Replacement

For every decorator, write this table before coding:

| Question | Answer |
|---|---|
| What value arrives? | |
| What `kind` arrives? | |
| Is it static? | |
| Is it private? | |
| What access capability exists? | |
| Can a replacement be returned? | |
| What replacement shape is legal? | |
| Are initializers registered? | |
| When do those initializers execute? | |
| What exceptions can escape? | |

This forces the design away from syntax-first thinking.

---

# 100. Advanced Semantics Drill — Separate the Phases

Take this source:

```js
@outer(makeLogger("class"))
class Example {
  @inner(makeLogger("method"))
  run() {}
}
```

Create four separate timelines:

```text
Timeline A — decorator expression evaluation
Timeline B — decorator invocation
Timeline C — replacement composition
Timeline D — initializer execution
```

Do not merge them.

A correct explanation should be able to state which operations occur during class definition and which occur only when instances are created.

---

# 101. Advanced Semantics Drill — Why “Annotation” Is an Incomplete Model

An annotation model usually suggests:

```text
source
 ↓
metadata
```

A decorator model can instead be:

```text
source
 ↓
execute decorator logic
 ↓
transform definition
 ↓
collect initialization behavior
 ↓
produce final class
```

Therefore decorators can alter execution behavior, not merely describe it.

The implementation and specification must be treated accordingly.

---

# 102. Advanced Semantics Drill — Declaration-Time Side Effects

A decorator expression can execute while a module/class definition is being evaluated.

Therefore this:

```js
@connectToDatabase()
class Repository {}
```

can accidentally make import evaluation depend on infrastructure.

Preferred architecture:

```text
module evaluation
  ↓
deterministic declaration
  ↓
explicit application bootstrap
  ↓
connect infrastructure
```

Use decorators for declarative transformation, not as a hidden dependency-initialization engine.

---

# 103. Advanced Semantics Drill — Wrapper Contracts

A decorator wrapper can accidentally change:

```text
this
arguments behavior
return type
error timing
stack traces
function identity
function name
function length
async behavior
generator behavior
```

A safe wrapper starts from a contract.

Example contract:

```text
Input: synchronous method
Output: synchronous method
`this`: preserved
return value: preserved
thrown errors: preserved
side effect: one log before and after
```

Then implement only what the contract requires.

---

# 104. Advanced Semantics Drill — Async Contract

For:

```js
async method() {}
```

the observable contract includes promise behavior.

A decorator that turns:

```js
return value.apply(this, args);
```

into:

```js
return await value.apply(this, args);
```

may change stack/exception timing and adds a semantic boundary.

Do not add `async`/`await` to a wrapper unless you need that behavior.

---

# 105. Advanced Semantics Drill — Generator Contract

For:

```js
g *items() {}
```

the result is an iterator/generator object.

A decorator must not accidentally convert the method into:

```text
ordinary function returning an unrelated value
```

without documenting the behavioral change.

Test:

```text
next()
return()
throw()
for...of
spread
```

---

# 106. Advanced Semantics Drill — Promise Transparency

If a decorator is meant only to measure duration, decide whether measurement means:

```text
function invocation duration
```

or:

```text
asynchronous completion duration
```

For an async method, those are different.

Use a contract such as:

```text
start timer before invocation
await returned promise
stop timer after settlement
rethrow original rejection
```

Then test success and failure.

---

# 107. Advanced Semantics Drill — Error Transparency

A wrapper should not accidentally turn:

```js
throw error;
```

into:

```js
throw new Error(String(error));
```

unless you intentionally change the error contract.

Preserve:

```text
identity
cause
stack where practical
classification
```

---

# 108. Advanced Semantics Drill — Method Identity

Before decoration:

```js
const original = C.prototype.run;
```

After replacement decoration:

```js
const decorated = C.prototype.run;
```

Potentially:

```text
original !== decorated
```

This affects code such as:

```js
spyOn(C.prototype, "run");
registry.set(original, metadata);
```

Your decorator library documentation should explicitly state identity behavior.

---

# 109. Advanced Semantics Drill — Constructor Identity

A class decorator that returns a replacement class can affect expectations around:

```text
constructor identity
instanceof
static members
subclassing
reflection
stack traces
serialization
framework registration
```

Replacing a class is therefore a much stronger operation than merely attaching metadata.

Use class replacement only when the semantic gain is clear.

---

# 110. Advanced Semantics Drill — `addInitializer` Is a Lifecycle Tool

Think of:

```js
context.addInitializer(fn);
```

as:

```text
“run this callback at the specified initialization phase”
```

not:

```text
“call this right now”
```

This makes initializer queues a lifecycle abstraction.

Document the lifecycle phase in framework APIs.

---

# 111. Advanced Semantics Drill — Initializer `this`

An instance initializer generally runs with an instance-oriented receiver context appropriate to the class initialization phase.

This is useful for behaviors like binding:

```js
context.addInitializer(function () {
  this.method = this.method.bind(this);
});
```

But it also means initializer code is part of the object's construction lifecycle and can affect startup performance.

---

# 112. Advanced Semantics Drill — Instance Cost

A decorator that registers an initializer for every instance can introduce:

```text
O(number of instances)
```

work.

If a class is instantiated one million times, a tiny initializer is not tiny anymore.

Measure:

```text
construction time
allocation
property writes
hidden-class effects
GC pressure
```

---

# 113. Advanced Performance — Shape Stability

A decorator may add or replace properties during initialization.

In optimizing engines, object shape/layout stability can influence optimization.

Do not assert exact V8 behavior without measuring the target engine.

The safe principal-level statement is:

```text
instance initialization patterns can affect engine optimization
```

Benchmark the actual application.

---

# 114. Advanced Performance — Wrapper Depth

Multiple decorators can produce nested wrappers:

```text
A
 ↓
B
 ↓
C
 ↓
original
```

Each layer can add:

```text
function call overhead
closure state
error boundaries
logging
metrics
```

For hot methods, benchmark one wrapper versus several.

---

# 115. Advanced Performance — Observability Decorators

Tracing is a classic decorator use case.

But logging every invocation can destroy throughput.

Prefer policies such as:

```text
sampling
rate limits
aggregation
asynchronous export
structured events
```

Do not make the instrumentation itself the bottleneck.

---

# 116. Advanced Performance — Caching Decorators

A caching decorator requires a key policy.

Questions:

```text
What identifies the call?
Are arguments serializable?
Are object identities meaningful?
Is `this` part of the key?
How long does cache state live?
Is cache invalidation defined?
```

Decorator syntax does not solve cache correctness.

---

# 117. Advanced Memory — Closure Capture Review

Review every decorator closure:

```js
function decorate(value) {
  const hugeConfig = loadConfig();

  return function (...args) {
    use(hugeConfig, args);
    return value.apply(this, args);
  };
}
```

If the returned wrapper lives as long as the class, `hugeConfig` may remain reachable for the same lifetime.

Minimize captures.

---

# 118. Advanced Memory — Registry Design

A global registry can create permanent retention:

```js
const registry = new Map();
```

A decorator that stores every class forever can prevent garbage collection.

Consider:

```text
lifecycle
WeakMap where appropriate
explicit unregister
bounded registry
service-scoped registry
```

The right choice depends on whether registry entries must survive independently.

---

# 119. Advanced Security — Authorization Decorators

Authorization decorators can look elegant:

```js
@requiresRole("admin")
removeUser(id) {}
```

But security review must verify:

```text
what exactly is wrapped?
what happens if the method is accessed another way?
can an alternate code path bypass it?
does inheritance preserve policy?
are framework hooks consistent?
```

Never treat a decorator as the only security boundary unless the architecture proves that it is.

---

# 120. Advanced Security — Validation Decorators

A validation decorator may be useful at an application boundary, but validation logic should remain understandable.

Risky:

```text
validation decorator
  ↓
transform input
  ↓
normalize
  ↓
default values
  ↓
authorization
  ↓
side effect
```

Better:

```text
input validation
→ explicit domain object
→ business method
```

Use decorators where they improve declaration-level structure, not where they obscure the business algorithm.

---

# 121. Advanced Security — Metadata Exposure

Metadata can leak architectural information such as:

```text
routes
roles
serialization rules
dependency relationships
internal names
```

When decorator metadata is exposed to other libraries, classify it as data with a lifecycle and trust model.

Do not assume metadata is harmless merely because it is “just an object.”

---

# 122. Advanced Security — Module Load Side Effects

A decorator factory that reads files, opens sockets, or performs network requests during module evaluation expands the module's attack/availability surface.

Prefer:

```text
pure factory
+
explicit runtime initialization
```

for infrastructure-heavy concerns.

---

# 123. Advanced Compatibility — Parser Support

Decorator compatibility is not only runtime capability.

An older parser can fail before the program starts.

Therefore compatibility includes:

```text
parser
transpiler
bundler
linter
formatter
editor
runtime
```

Create a matrix for every supported environment.

---

# 124. Advanced Compatibility — Native vs Transformed Builds

A platform may have:

```text
modern build → native decorators
legacy build → transformed decorators
```

Now you have two execution paths.

They must be tested for behavioral equivalence.

Do not assume source-level equality means semantic equality.

---

# 125. Advanced Compatibility — Version Pinning

For proposal-stage syntax, pin the exact toolchain configuration.

Example:

```text
Node version
compiler version
parser version
plugin version
bundler version
```

Because a toolchain upgrade can change the generated or accepted semantics.

---

# 126. Advanced Compatibility — Documentation Drift

A common migration failure is:

```text
code migrated
but docs still describe legacy decorator semantics
```

Update:

```text
team guidelines
examples
architecture docs
code templates
lint rules
review checklists
```

at the same time.

---

# 127. Architecture — Decorators as Policy

A mature organization should classify decorators:

```text
Category A — approved infrastructure
Category B — application-level cross-cutting
Category C — experimental
Category D — prohibited
```

Example policy:

```text
approved:
trace
metrics
narrow registration
framework lifecycle

restricted:
authorization
caching
mutation-heavy transforms
class replacement

prohibited:
network I/O during module evaluation
hidden DB writes
secret extraction
global unbounded registries
```

The categories are organization-specific.

---

# 128. Architecture — Explicitness Budget

Every decorator spends some of the team's “implicit behavior budget.”

If a class contains:

```js
@A
@B
@C
@D
@E
class Service {}
```

the declaration may be short while behavior is distributed across five modules.

Principal review asks:

> Can a new engineer reconstruct the effective behavior without specialized tribal knowledge?

If not, reduce decorator density.

---

# 129. Architecture — Decorator Naming

A good decorator name communicates behavior.

Prefer:

```text
@trace
@memoize
@registerRoute
```

over:

```text
@enhance
@process
@magic
```

Names should describe the semantic effect, not implementation excitement.

---

# 130. Architecture — Decorator Documentation Standard

Every production decorator should document:

```text
Target kinds
Arguments
Context requirements
Return behavior
Initializer behavior
Ordering expectations
Errors
Performance cost
Memory implications
Security implications
Examples
Anti-examples
```

This turns decorator usage into a governed API.

---

# 131. Architecture — Decorator Review Checklist

During code review:

```text
[ ] Is a decorator the clearest abstraction?
[ ] Is its target kind correct?
[ ] Is order intentional?
[ ] Is `this` preserved?
[ ] Are return/error semantics preserved?
[ ] Are initializers necessary?
[ ] Is per-instance work acceptable?
[ ] Is state lifetime explicit?
[ ] Is security behavior visible?
[ ] Is tooling support confirmed?
```

---

# 132. Architecture — When Explicit Composition Wins

Prefer explicit composition when:

```text
behavior is central business logic
ordering is critical and complex
runtime configuration dominates
debugging needs are high
decorator count is large
team familiarity is low
```

Example:

```js
const handler = authorize(
  validate(
    trace(
      createUser
    )
  )
);
```

It may be more verbose but can make the execution pipeline visible.

---

# 133. Architecture — When Decorators Win

Prefer decorators when:

```text
a behavior belongs naturally to a declaration
cross-cutting behavior is small
composition is predictable
team understands the lifecycle
framework integration is declaration-driven
```

The decision is about semantic clarity, not line count.

---

# 134. Architecture — Framework Boundary

Frameworks can use decorators to encode declarations like:

```text
route
controller
injectable
serializable
observable
```

But the framework should expose tooling that lets users answer:

```text
What does this decorator do?
When does it run?
What does it change?
Where can it fail?
```

Good framework ergonomics compensate for metaprogramming complexity.

---

# 135. Architecture — Generated Documentation

For decorator-heavy frameworks, generate a registry such as:

```text
Class
  ↓
Decorators
  ↓
Target kind
  ↓
Configuration
  ↓
Runtime effect
```

This creates a discoverability layer without changing runtime semantics.

---

# 136. Architecture — Observability for Decorators

At development/debug level, instrument:

```text
decorator name
target kind
target name
initializer count
replacement yes/no
```

At production level, observe only what provides operational value.

Do not emit every decorator event on every class/module load by default.

---

# 137. Architecture — Versioning Decorator Behavior

A decorator library can create compatibility risk when its semantics change.

Version semantic behavior explicitly:

```text
v1 decorator contract
v2 decorator contract
```

Then document migration steps.

This is especially important if generated code or metadata is persisted.

---

# 138. Architecture — Testing a Decorator Library

Use three layers:

```text
Unit
  decorator function behavior

Integration
  actual class definitions + lifecycle

Compatibility
  native runtime + supported toolchains
```

Do not rely only on unit tests of the decorator function because class-definition semantics are part of the feature.

---

# 139. Architecture — Golden Tests

For a framework decorator, maintain golden behavioral examples:

```text
input class
→ expected transformed behavior
→ expected initialization
→ expected metadata where applicable
```

Use human-readable examples for migration safety.

---

# 140. Architecture — Fuzzing and Edge Testing

For complex decorator frameworks, vary:

```text
nested decorators
symbol names
private members
static members
inherited classes
throws
async methods
generators
multiple instances
```

This can uncover lifecycle bugs not visible in happy-path tests.

---

# 141. Architecture — Error Taxonomy

Classify decorator-related failures:

```text
ParseError
DecoratorEvaluationError
DecoratorApplicationError
InitializerError
RuntimeWrapperError
ConfigurationError
CompatibilityError
```

A clear taxonomy improves support and observability.

---

# 142. Architecture — Rollout Strategy

For a new decorator library:

```text
prototype
 ↓
internal pilot
 ↓
small service
 ↓
measure
 ↓
review
 ↓
platform recommendation
 ↓
optional default
```

Do not force a new metaprogramming model across a platform in one migration.

---

# 143. Architecture — Migration Strategy From Legacy Decorators

A safe migration plan:

```text
1. Inventory existing decorators.
2. Classify each as legacy/standard/unknown.
3. Capture current behavior tests.
4. Inspect compiler output.
5. Map old lifecycle to new lifecycle.
6. Replace the simplest decorators first.
7. Handle metadata separately.
8. Run native and transformed builds if both exist.
9. Remove legacy configuration.
10. Update documentation.
```

---

# 144. Architecture — Migration Risk Matrix

| Legacy feature | Migration risk | Why |
|---|---|---|
| Simple method wrapper | Low | clear replacement model |
| Field transform | Medium | lifecycle semantics differ |
| Descriptor mutation | High | different model |
| Parameter decorators | High | separate proposal/status |
| Runtime metadata | High | metadata model differs |
| Class replacement | Medium/High | identity/inheritance effects |

---

# 145. Architecture — Legacy Decorator Detection

Search for:

```text
experimentalDecorators
legacy Babel decorator plugins
createDecorator
PropertyDescriptor
__decorate
emitDecoratorMetadata
reflect-metadata
```

These names can reveal legacy decorator assumptions.

Treat them as migration clues, not proof by themselves.

---

# 146. Architecture — `reflect-metadata`

Frameworks may historically use `reflect-metadata` for metadata storage.

Do not assume that existing reflection metadata behavior becomes native decorator metadata automatically.

Track:

```text
legacy metadata
standard decorators
Decorator Metadata proposal
```

as separate concepts.

---

# 147. Architecture — Metadata Namespace Collisions

If metadata becomes a shared object, define ownership rules.

Safer patterns include:

```text
unique symbols
package-specific namespace objects
schema versioning
```

Avoid unrelated libraries writing arbitrary top-level keys without coordination.

---

# 148. Architecture — Metadata Lifetime

Ask:

```text
How long must metadata live?
Does it follow subclasses?
Does it survive serialization?
Does it expose implementation details?
```

Metadata is data architecture, not a decorative afterthought.

---

# 149. Advanced Interview Drill — Explain Decorators Without Syntax

Answer:

> Decorators are a language-level metaprogramming mechanism that lets executable code participate in class and class-element definition, inspect a standardized context, optionally replace supported values, and register initialization work.

If you can explain them without using `@`, you understand the concept rather than the notation.

---

# 150. Advanced Interview Drill — Why Not Proxy?

Question:

> Why use decorators instead of `Proxy`?

Strong reasoning:

```text
Decorator
= declaration/class-definition transformation

Proxy
= runtime interception of object operations
```

They overlap in some use cases but operate at different semantic levels.

---

# 151. Advanced Interview Drill — Why Not Functions?

Question:

> Why not simply wrap functions?

Answer framework:

```text
Function composition is excellent for explicit runtime behavior.
Decorators are useful when the behavior belongs naturally to a declaration and can be expressed as part of class definition/lifecycle.
```

Choose the abstraction based on where the behavior semantically belongs.

---

# 152. Advanced Interview Drill — Why Not Inheritance?

Question:

> Why not create a base class instead of decorating methods?

Inheritance can encode reusable behavior, but it introduces subtype and hierarchy relationships.

Decorators can apply narrow behavior without creating a new inheritance tree.

Again, the trade-off is semantic, not syntactic.

---

# 153. Advanced Interview Drill — Principal Architecture

Question:

> Your team wants 20 decorators in every service class. Approve?

Recommended reasoning:

```text
No automatic approval.

First measure:
- hidden behavior
- ordering complexity
- initialization cost
- debugging burden
- security ambiguity
- team comprehension

Then reduce to a small approved vocabulary if decorators remain justified.
```

---

# 154. Predict-the-Result Exercise — Evaluation vs Invocation

Create:

```js
function make(name) {
  console.log("evaluate", name);
  return function (value, context) {
    console.log("invoke", name, context.kind);
    return value;
  };
}
```

Use:

```js
@make("A")
@make("B")
class C {}
```

Predict:

```text
evaluate A?
evaluate B?
invoke A?
invoke B?
```

Then execute in a standards-supporting environment and compare.

Record the distinction permanently in your notes.

---

# 155. Predict-the-Result Exercise — Initializer Timing

Create a decorator that calls:

```js
context.addInitializer(function () {
  console.log("initializer", this.constructor.name);
});
```

Then instantiate:

```js
new C();
new C();
```

Predict how many times the initializer runs.

Then add a static decorator and compare class initialization behavior.

---

# 156. Predict-the-Result Exercise — Wrapper Receiver

Use:

```js
function trace(value) {
  return function (...args) {
    console.log(this === undefined);
    return value.apply(this, args);
  };
}
```

Compare:

```js
obj.method();
```

and:

```js
const fn = obj.method;
fn();
```

Explain the result using Chapter 14, not decorator terminology alone.

---

# 157. Mastery Exercise — Spec-to-Code Translation

Choose one decorator rule from the current proposal.

Write four representations:

```text
1. Plain-English rule
2. Formal specification rule
3. Minimal code example
4. Observable test
```

This is a direct bridge between standards reading and engineering implementation.

---

# 158. Mastery Exercise — Decorator Order Laboratory

Build:

```text
class-level decorators: 3
method decorators: 3
initializer registrations: 3
```

Record:

```text
expression evaluation
invocation
replacement
initializer registration
initializer execution
```

Represent the result as a timeline diagram.

---

# 159. Mastery Exercise — Decorator Contract Review

For three decorators from a framework, write:

```text
Input
Output
Context
Side effects
Initializer
Error behavior
Identity behavior
Performance
Security
```

If a field is unknown, investigate the source.

Never fill it with assumptions.

---

# 160. Mastery Exercise — Standard vs Legacy Diff

Take one legacy decorator and write:

```text
Legacy syntax
Legacy arguments
Legacy timing
Legacy descriptor behavior
Legacy metadata
Generated output

Modern standard syntax
Modern context
Modern replacement
Modern initializer
Modern metadata relationship
```

Then list incompatible assumptions.

---

# 161. Mastery Exercise — Production Decorator Policy

Write a company-wide policy containing:

```text
Allowed uses
Restricted uses
Prohibited uses
Performance expectations
Testing expectations
Security expectations
Documentation requirements
Review requirements
Migration requirements
```

Make the policy short enough to be enforceable and specific enough to review code against.

---

# 162. Mastery Exercise — Remove Decorators

Take a decorator-heavy class and rewrite it without decorators.

Then compare:

```text
lines
cyclomatic complexity
execution visibility
testability
configuration
runtime overhead
```

Then restore decorators only where they provide a clear architectural advantage.

---

# 163. Mastery Exercise — Benchmark

Benchmark:

```text
plain method
one decorator
three decorators
initializer-heavy class
```

Measure:

```text
calls/second
instance construction time
memory allocations where tooling allows
p95 latency
```

Do not generalize from one machine.

---

# 164. Mastery Exercise — Security Review

Threat-model a decorator package with:

```text
class replacement
method wrapping
metadata
initializers
registry
configuration
```

Find at least five ways a compromised dependency could alter application behavior.

Then design mitigations.

---

# 165. Spaced Retrieval — 24-Hour Test

Without opening notes, answer:

```text
What is a decorator?
What does context provide?
What can a decorator return?
What does addInitializer do?
When do initializers run?
Why is order important?
```

Then verify against the source.

---

# 166. Spaced Retrieval — 7-Day Test

Explain from memory:

```text
standard decorators
vs
legacy decorators
vs
decorator metadata
vs
parameter decorators
```

State the current proposal maturity of each only after checking current primary sources.

---

# 167. Spaced Retrieval — 30-Day Test

Build a decorator framework without notes.

Then conduct your own code review using the production checklist.

---

# 168. Spaced Retrieval — 90-Day Test

Defend one position:

```text
“Decorators should be a standard organization-wide pattern.”
```

Then defend the opposite.

The goal is not ideological consistency.

The goal is trade-off fluency.

---

# 169. Dependency Graph — Expanded

```text
Functions
   ↓
Closures
   ↓
this / invocation
   ↓
Classes
   ↓
Objects / properties
   ↓
Proxy / Reflect / metaprogramming
   ↓
Decorators
   ↓
Decorator metadata
   ↓
Framework architecture
   ↓
Compatibility engineering
   ↓
Production JavaScript architecture
```

Parallel implementation/tooling path:

```text
Transpilation
   ↓
Parser support
   ↓
Decorator transforms
   ↓
Source maps
   ↓
Native runtime support
```

---

# 170. Concept Connections

## Depends On

- functions,
- closures,
- `this`,
- classes,
- objects,
- internal methods,
- metaprogramming,
- compilation,
- TC39 proposal process.

## Builds Toward

- compatibility engineering,
- library authoring,
- framework architecture,
- security engineering,
- production JavaScript architecture.

## Related Concepts

- Proxy,
- Reflect,
- annotations,
- metadata,
- dependency injection,
- aspect-oriented techniques.

## Concepts Revisited

- class initialization,
- lexical closures,
- receiver binding,
- property identity,
- proposal stages,
- source-to-runtime transformations.

## Why This Chapter Matters Later

Decorators sit at the boundary between language evolution and framework design. They are small syntactically but large architecturally.

---

# 171. Final Principal Decision Framework

Before standardizing decorators, ask:

```text
Correctness
→ Does the abstraction preserve the intended semantics?

Performance
→ What is the cost on the hot paths and constructors?

Memory
→ What gets retained and for how long?

Security
→ What hidden authority does the decorator exercise?

Reliability
→ What happens when decoration or initialization fails?

Maintainability
→ Can engineers trace effective behavior?

Scalability
→ Does per-instance/per-call work scale?

Observability
→ Can production behavior be understood?

Developer Experience
→ Can the team learn and use it safely?

Operational Complexity
→ Does the toolchain become harder to maintain?

Future Change
→ What happens if semantics or ecosystem support evolve?
```

Approve decorators only when the answers are stronger than the explicit-code alternative.

---

# 172. Final Mental Model

```text
@classDecorator
class C {}
```

should trigger this mental sequence:

```text
1. Decorator expression exists.
2. Expression is evaluated according to language semantics.
3. A decorator function participates in class definition.
4. The function receives a value and context.
5. It may return a permitted replacement.
6. It may register initialization work.
7. Definition continues according to the specification.
8. Static initialization and later instance initialization occur at their defined lifecycle points.
9. Errors are observable at the phase where they happen.
```

And this engineering sequence:

```text
What does it change?
↓
When does it change it?
↓
What can fail?
↓
What does it cost?
↓
What does it hide?
↓
Should we use it?
```

---

# Chapter 93 — Completion Criteria

Do not mark this chapter mastered because you can write `@trace`.

You are ready to move forward when you can independently:

1. Explain decorators without relying on syntax.
2. Explain decorator expression evaluation.
3. Explain decorator invocation.
4. Explain context fields and target kinds.
5. Explain replacement semantics.
6. Explain `addInitializer()`.
7. Explain definition-time versus initialization-time effects.
8. Explain ordering as multiple separate phases.
9. Explain static, instance, symbol, and private targets.
10. Preserve `this` when wrapping methods.
11. Preserve async/generator contracts.
12. Debug initializer timing.
13. Evaluate method/class identity changes.
14. Evaluate performance and memory implications.
15. Conduct a security review.
16. Distinguish standard decorators from legacy systems.
17. Distinguish decorators from decorator metadata and parameter decorators.
18. Design a production decorator policy.
19. Decide when explicit composition is better.
20. Read the current TC39 proposal status from primary sources.

---

# Chapter 93 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I explain the decorator model without syntax? [ ]
- Could I separate expression evaluation from invocation? [ ]
- Could I explain context? [ ]
- Could I explain replacement? [ ]
- Could I explain initializer timing? [ ]
- Could I explain ordering? [ ]
- Could I preserve `this`? [ ]
- Could I test async/generator behavior? [ ]
- Could I distinguish standard vs legacy? [ ]
- Could I evaluate security/performance? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 93 — Canonical References and Source Discipline

Use primary sources first:

1. TC39 proposal tracker — https://github.com/tc39/proposals
2. Decorators proposal — https://github.com/tc39/proposal-decorators
3. Decorators specification — https://tc39.es/proposal-decorators/
4. Decorator Metadata — https://github.com/tc39/proposal-decorator-metadata
5. Parameter Decorators — https://github.com/tc39/proposal-class-method-parameter-decorators
6. TC39 process — https://tc39.es/process-document/

Current status verification:

- The official TC39 proposals tracker currently lists Decorators at **Stage 2.7**. citeturn946931search0
- The official Decorators proposal repository identifies the proposal as Stage 2.7 and describes the current document as a work in progress. citeturn946931search1
- Decorator Metadata is a separate Stage 3 proposal. citeturn946931search6
- Class method/constructor parameter decorators remain a separate Stage 1 proposal. citeturn946931search9

Do not treat third-party framework behavior as evidence of ECMAScript standard semantics.

---

# Chapter 93 — Completion Snapshot

```text
Track A — Core Theory
[ ] Decorator motivation
[ ] Decorator context
[ ] Replacement semantics
[ ] Initializer semantics
[ ] Ordering
[ ] Class/element/static/private behavior
[ ] Standard vs legacy
[ ] Metadata separation
[ ] Specification interpretation

Track B — Implementation
[ ] Trace decorator
[ ] Async-safe decorator
[ ] Generator-safe decorator
[ ] Initializer laboratory
[ ] Production decorator library
[ ] Compatibility matrix
[ ] Benchmark suite
[ ] Security review

Track C — Interview / Reasoning
[ ] Explain decorators simply
[ ] Explain trade-offs
[ ] Compare with Proxy
[ ] Compare with explicit composition
[ ] Design governance
[ ] Design migration policy
[ ] Defend production adoption

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# File Metadata

```text
Chapter: 93
Title: Decorators
Part: XVII — Modern ECMAScript & Language Evolution
Format: Standalone Markdown chapter
Verification date: 2026-09-10
Current proposal stage at verification: 2.7
Status: [ ] Not Started
```

> **Mastery reminder:** Reading is not mastery. You must retrieve, predict, implement, debug, apply, compare, and defend the concepts before marking the chapter complete.