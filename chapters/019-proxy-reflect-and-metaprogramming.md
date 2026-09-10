
# Chapter 19 — Proxy, Reflect, and Metaprogramming

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 15–18 — Objects, Property Semantics, Property Keys, Prototypes, and Classes
>
> **Next:** Chapter 20 — Symbols and Well-Known Symbols

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- define metaprogramming in JavaScript;
- explain what a `Proxy` is;
- distinguish a Proxy object from its target object;
- explain traps such as `get`, `set`, `has`, `deleteProperty`, `defineProperty`, `getOwnPropertyDescriptor`, `ownKeys`, `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `preventExtensions`, `apply`, and `construct`;
- explain that Proxy traps intercept object internal operations rather than arbitrary source expressions;
- connect common JavaScript syntax to underlying internal operations;
- explain why `Reflect` methods are useful when forwarding Proxy behavior;
- distinguish `Reflect.get(target, key, receiver)` from `target[key]`;
- explain receiver propagation through getters, setters, inheritance, and proxies;
- understand Proxy invariants;
- explain why some proxy trap results are constrained by non-configurable or non-extensible target properties;
- understand why `ownKeys` must respect important target invariants;
- explain how proxies affect `this`, prototype lookup, property descriptors, and method calls;
- explain Proxy identity and why `proxy !== target`;
- understand that Proxy interception does not transparently preserve all target behavior;
- explain revoked proxies;
- understand callable and constructable proxies;
- explain how proxying functions differs from proxying ordinary objects;
- reason about proxy effects on arrays, classes, and built-in objects;
- identify performance costs and optimization implications;
- identify security uses such as membranes, capability wrappers, validation, and isolation patterns;
- distinguish useful metaprogramming from unnecessary abstraction;
- implement a simplified Proxy-like object layer;
- implement forwarding traps with `Reflect`;
- debug recursive trap calls, invariant violations, receiver bugs, and identity mismatches;
- design a safe validation or observation proxy;
- reason at principal level about metaprogramming, API transparency, correctness, performance, and maintainability.

---

# 2. Prerequisites

You should understand:

```text
Chapter 15
objects and property internal behavior

Chapter 16
property keys and enumeration

Chapter 17
prototype chains and receiver-aware lookup

Chapter 18
classes, private fields, constructors, inheritance
```

The progression is:

```text
object syntax
   ↓
internal object operations
   ↓
Proxy interception
   ↓
Reflect forwarding
   ↓
metaprogramming
```

---

# 3. What Is Metaprogramming?

Metaprogramming means writing code that operates on:

```text
program structure
language behavior
object operations
functions
metadata
```

Examples include:

```text
reflection
property descriptor manipulation
Proxy
Reflect
dynamic function behavior
decorators
custom protocols
```

Metaprogramming can be powerful because it lets software modify or observe behavior at a higher level than ordinary business logic.

It can also make systems significantly harder to reason about.

---

# 4. What Is a Proxy?

A Proxy is an object that wraps another object and can intercept certain operations performed on that object.

Example:

```js
const target = {
  value: 1
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    console.log("reading:", key);
    return Reflect.get(target, key, receiver);
  }
});
```

Then:

```js
proxy.value;
```

can invoke the Proxy's `get` trap.

Conceptually:

```text
proxy
  ↓
trap
  ↓
target operation
```

---

# 5. Proxy Is Not a Transparent Alias

Consider:

```js
const target = {};
const proxy = new Proxy(target, {});
```

Then:

```js
proxy === target
```

is:

```text
false
```

The Proxy is a distinct object identity.

This matters for:

```text
Map keys
Set membership
WeakMap keys
=== comparisons
caches
framework identity
memoization
```

---

# 6. Mental Model

Think:

```text
ordinary object

caller
  ↓
internal operation
  ↓
target

proxy object

caller
  ↓
internal operation on proxy
  ↓
proxy trap
  ↓
target operation if forwarded
```

The Proxy sits at the internal-operation boundary.

---

# 7. Traps

A Proxy handler can define traps for different internal operations.

Important groups:

### Property access

```text
get
set
has
deleteProperty
```

### Property definition/reflection

```text
defineProperty
getOwnPropertyDescriptor
ownKeys
```

### Prototype operations

```text
getPrototypeOf
setPrototypeOf
```

### Extensibility

```text
isExtensible
preventExtensions
```

### Function operations

```text
apply
construct
```

Each trap corresponds conceptually to one or more object internal operations.

---

# 8. `get` Trap

Example:

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    console.log("GET", key);

    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.value);
```

Conceptual sequence:

```text
proxy.value
→ proxy [[Get]]
→ get trap
→ Reflect.get
→ target lookup
→ 10
```

---

# 9. `set` Trap

Example:

```js
const target = {};

const proxy = new Proxy(target, {
  set(target, key, value, receiver) {
    console.log("SET", key, value);

    return Reflect.set(target, key, value, receiver);
  }
});

proxy.value = 10;
```

The assignment becomes an intercepted internal operation.

---

# 10. `has` Trap

For:

```js
key in proxy
```

the Proxy can intercept property existence checking:

```js
const proxy = new Proxy(
  { value: 10 },
  {
    has(target, key) {
      console.log("HAS", key);
      return Reflect.has(target, key);
    }
  }
);
```

Then:

```js
"value" in proxy;
```

triggers the trap.

---

# 11. `deleteProperty`

For:

```js
delete proxy.value
```

the Proxy can intercept the delete operation.

Example:

```js
const proxy = new Proxy(
  { value: 10 },
  {
    deleteProperty(target, key) {
      console.log("DELETE", key);
      return Reflect.deleteProperty(target, key);
    }
  }
);
```

---

# 12. `defineProperty`

Operations such as:

```js
Object.defineProperty(proxy, "x", descriptor);
```

can trigger the `defineProperty` trap.

This is important because:

```text
property assignment
```

and:

```text
explicit property definition
```

are related but distinct operations.

---

# 13. `getOwnPropertyDescriptor`

Reflection:

```js
Object.getOwnPropertyDescriptor(proxy, "x")
```

can trigger:

```text
getOwnPropertyDescriptor
```

The trap lets metaprogramming systems virtualize property metadata.

---

# 14. `ownKeys`

Operations including:

```js
Reflect.ownKeys(proxy)
Object.keys(proxy)
Object.getOwnPropertyNames(proxy)
Object.getOwnPropertySymbols(proxy)
```

can involve the `ownKeys` trap.

Example:

```js
const proxy = new Proxy(
  { a: 1, b: 2 },
  {
    ownKeys(target) {
      return ["b", "a"];
    }
  }
);
```

This demonstrates why Proxy can alter reflection behavior.

---

# 15. `getPrototypeOf`

Example:

```js
const proxy = new Proxy(
  {},
  {
    getPrototypeOf(target) {
      return null;
    }
  }
);
```

Prototype reflection can be intercepted.

But invariants restrict inconsistent results under certain target states.

---

# 16. `setPrototypeOf`

Likewise:

```js
Object.setPrototypeOf(proxy, proto);
```

can be trapped.

Proxy does not mean:

```text
ignore all target constraints
```

The language maintains important invariants.

---

# 17. Extensibility Traps

Two relevant traps:

```text
isExtensible
preventExtensions
```

These interact closely with:

```js
Object.isExtensible()
Object.preventExtensions()
```

The Proxy must not report impossible target states.

---

# 18. Function Proxies

Functions can be proxied.

Example:

```js
function greet(name) {
  return `Hello ${name}`;
}

const proxy = new Proxy(greet, {
  apply(target, thisArg, args) {
    console.log("CALL", args);
    return Reflect.apply(target, thisArg, args);
  }
});
```

Then:

```js
proxy("A");
```

can trigger:

```text
apply trap
```

---

# 19. Construct Proxies

If the target is constructable:

```js
function User(name) {
  this.name = name;
}
```

a Proxy can intercept:

```js
new proxy("A");
```

through:

```text
construct
```

Example:

```js
const proxy = new Proxy(User, {
  construct(target, args, newTarget) {
    console.log("CONSTRUCT", args);

    return Reflect.construct(
      target,
      args,
      newTarget
    );
  }
});
```

---

# 20. Callable vs Constructable

A function can be:

```text
callable
constructable
```

or in some cases callable but not constructable.

Arrow functions are callable but not constructable.

This distinction matters when creating function proxies.

---

# 21. Reflect

`Reflect` provides standard functions corresponding closely to fundamental object operations.

Examples:

```js
Reflect.get(obj, key, receiver);
Reflect.set(obj, key, value, receiver);
Reflect.has(obj, key);
Reflect.deleteProperty(obj, key);
Reflect.defineProperty(obj, key, descriptor);
Reflect.getOwnPropertyDescriptor(obj, key);
Reflect.ownKeys(obj);
Reflect.getPrototypeOf(obj);
Reflect.setPrototypeOf(obj, proto);
Reflect.isExtensible(obj);
Reflect.preventExtensions(obj);
Reflect.apply(fn, thisArg, args);
Reflect.construct(Ctor, args, newTarget);
```

Reflect is especially valuable for forwarding Proxy operations.

---

# 22. Why Use `Reflect` in Proxy Handlers?

Consider:

```js
get(target, key, receiver) {
  return Reflect.get(target, key, receiver);
}
```

This preserves important semantics, especially:

```text
receiver propagation
getters
prototype delegation
setter behavior
```

A naive:

```js
return target[key];
```

can differ from:

```js
Reflect.get(target, key, receiver);
```

when inheritance/accessors/receiver semantics matter.

---

# 23. Receiver Example

Consider:

```js
const base = {
  get value() {
    return this._value;
  }
};

const target = Object.create(base);
target._value = 10;

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.value);
```

The getter's `this` can correspond to:

```text
proxy
```

because the receiver is preserved.

This is a critical Proxy concept.

---

# 24. Why `target[key]` Can Be Wrong

A naive trap:

```js
get(target, key) {
  return target[key];
}
```

uses the target as the receiver for internal getter execution.

A forwarding trap:

```js
get(target, key, receiver) {
  return Reflect.get(target, key, receiver);
}
```

preserves the caller's receiver.

This difference matters for:

```text
accessors
inheritance
super-related behavior
private fields in some call paths
reactive proxies
```

---

# 25. Proxy + Class Getter

Example:

```js
class User {
  constructor(name) {
    this.name = name;
  }

  get label() {
    return `User: ${this.name}`;
  }
}

const user = new User("A");

const proxy = new Proxy(user, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.label);
```

The getter can observe:

```text
this = proxy
```

rather than necessarily:

```text
this = target
```

This is important for transparent wrappers.

---

# 26. Private Fields and Proxies

Private fields are not ordinary property lookups.

Example:

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

If:

```js
const proxy = new Proxy(user, {});
proxy.getName();
```

the method can encounter private-brand behavior involving the receiver.

The Proxy is a distinct object and does not automatically possess the target's private brand.

Therefore:

```text
proxying class instances
+
private fields
```

can produce surprising failures.

This is a major design consideration for transparent proxies.

---

# 27. Method Calls Through Proxies

Consider:

```js
proxy.method();
```

Conceptually:

```text
get proxy "method"
→ trap
→ function obtained
→ invoke with receiver proxy
```

Therefore inside the method:

```text
this may be proxy
```

not:

```text
target
```

This affects:

```text
getters
setters
private fields
identity checks
method calls
```

---

# 28. Transparent Proxy Is Difficult

A wrapper that claims:

```text
"the proxy behaves exactly like the target"
```

has a difficult contract.

Differences can remain in:

```text
identity
private fields
reflective behavior
function identity
prototype relationships
receiver
WeakMap keys
WeakSet membership
error stacks
```

Therefore:

> Perfect transparency is a strong requirement, not the default property of Proxy.

---

# 29. Proxy Identity

Consider:

```js
const target = {};
const proxy = new Proxy(target, {});
```

Then:

```js
proxy === target
```

is:

```text
false
```

This affects:

```js
map.get(target);
```

versus:

```js
map.get(proxy);
```

They are different keys.

---

# 30. Identity in Frameworks

Reactive systems sometimes use proxies to track access.

For example:

```js
const state = reactive({
  count: 0
});
```

Internally:

```text
state
→ proxy
→ target
```

Framework code must carefully manage:

```text
proxy identity
raw target identity
caching
dependency tracking
```

Otherwise:

```text
raw === proxy
```

falsehood can produce subtle bugs.

---

# 31. `WeakMap` and Proxy Identity

Consider:

```js
const target = {};
const proxy = new Proxy(target, {});

const map = new WeakMap();

map.set(target, "target");
map.set(proxy, "proxy");
```

These are separate entries.

This matters for:

```text
metadata registries
memoization
caches
framework state
```

---

# 32. Proxy Invariants

Proxy traps are powerful, but ECMAScript enforces invariants.

The purpose is to prevent proxies from reporting impossible object states.

Important areas include:

```text
non-configurable properties
non-extensible targets
prototype relationships
property descriptors
own keys
```

Think:

```text
customization
+
invariants
=
controlled metaprogramming
```

---

# 33. `get` Invariant

A Proxy cannot arbitrarily lie about certain non-configurable, non-writable data properties.

If the target has a fixed property descriptor requiring a specific value, the `get` trap must not produce an incompatible result.

Conceptually:

```text
frozen target property
→ proxy cannot freely pretend it has another value
```

This maintains semantic consistency.

---

# 34. `set` Invariant

Similarly, a `set` trap cannot claim success for an operation that would violate certain fixed target property constraints.

For example:

```text
non-writable, non-configurable target property
```

cannot be arbitrarily reported as successfully changed.

A trap returning `true` does not grant permission to violate target invariants.

---

# 35. `deleteProperty` Invariant

A Proxy cannot report successful deletion of a property that cannot legally be deleted due to target invariants.

Especially important:

```text
non-configurable own property
```

must continue to be represented consistently.

---

# 36. `defineProperty` Invariants

A proxy cannot freely report descriptor changes that violate fixed target property constraints.

This matters when virtualizing:

```text
writable
configurable
enumerable
value
get/set
```

metadata.

---

# 37. `ownKeys` Invariants

A proxy cannot arbitrarily omit or duplicate certain target keys when doing so would violate invariants.

Important cases involve:

```text
non-configurable own properties
non-extensible target objects
```

The own-key result must remain consistent with those constraints.

---

# 38. `getPrototypeOf` Invariants

If the target is non-extensible, a proxy cannot freely report an arbitrary prototype that contradicts the target's actual prototype.

This prevents:

```text
target is fixed
proxy claims different prototype
```

from becoming an impossible object state.

---

# 39. `setPrototypeOf` Invariants

Likewise, a proxy cannot claim to successfully change a non-extensible target's prototype to an incompatible prototype.

This reinforces a general principle:

> Proxy traps customize behavior but do not abolish the invariants of the underlying object model.

---

# 40. Revocable Proxies

You can create a revocable proxy:

```js
const { proxy, revoke } = Proxy.revocable(
  target,
  {}
);
```

Then:

```js
revoke();
```

causes later operations through the proxy to fail.

This is useful for:

```text
capability lifetime
resource revocation
membranes
temporary authority
sandbox boundaries
```

---

# 41. Revocation as Capability Control

A useful pattern:

```text
give proxy capability
        ↓
use capability
        ↓
revoke
        ↓
future operations fail
```

This can be powerful in systems that need explicit authority lifetimes.

It is not equivalent to OS-level security isolation.

---

# 42. Proxy Membranes

A membrane can wrap objects crossing a trust or ownership boundary.

Conceptually:

```text
side A object
   ↓
proxy
   ↓
side B representation
```

Nested objects can be wrapped as they cross the boundary.

A membrane can enforce:

```text
access rules
identity translation
revocation
logging
validation
```

This is advanced metaprogramming.

---

# 43. Validation Proxy

Example:

```js
const user = {};

const validated = new Proxy(user, {
  set(target, key, value, receiver) {
    if (key === "age" && !Number.isInteger(value)) {
      throw new TypeError("age must be an integer");
    }

    return Reflect.set(target, key, value, receiver);
  }
});
```

Then:

```js
validated.age = 30;
```

works.

But:

```js
validated.age = "30";
```

throws.

---

# 44. Observation Proxy

Logging:

```js
const observed = new Proxy(target, {
  get(target, key, receiver) {
    console.log("GET", String(key));
    return Reflect.get(target, key, receiver);
  }
});
```

Useful for:

```text
debugging
telemetry
dependency tracking
reactivity
performance investigation
```

But logging every access can itself change timing and performance.

---

# 45. Default-Value Proxy

A Proxy can expose defaults:

```js
const config = new Proxy(
  {},
  {
    get(target, key, receiver) {
      return Reflect.has(target, key)
        ? Reflect.get(target, key, receiver)
        : getDefault(key);
    }
  }
);
```

This can be convenient.

But it can hide missing configuration errors.

Be careful when absence itself is semantically meaningful.

---

# 46. Negative-Index Array Proxy

A pedagogical example:

```js
function withNegativeIndexes(array) {
  return new Proxy(array, {
    get(target, key, receiver) {
      const index = Number(key);

      if (
        Number.isInteger(index) &&
        index < 0
      ) {
        return target[target.length + index];
      }

      return Reflect.get(target, key, receiver);
    }
  });
}
```

Now:

```js
const values = withNegativeIndexes([10, 20, 30]);

values[-1];
```

can return:

```text
30
```

This demonstrates how Proxy can create virtual property semantics.

---

# 47. Why Virtual Semantics Can Be Dangerous

When:

```js
values[-1]
```

looks like normal property access but means something custom, developers must learn another hidden rule.

Metaprogramming should be used when:

```text
the abstraction is worth the semantic cost
```

not merely because interception is possible.

---

# 48. Proxy and Arrays

Arrays are objects with specialized indexed and length behavior.

Proxying arrays can intercept:

```text
get
set
defineProperty
delete
ownKeys
```

But correct forwarding must respect:

```text
array index semantics
length invariants
non-configurable properties
```

Naive array proxies can easily create confusing or inefficient behavior.

---

# 49. Proxy and `length`

When an array index changes:

```js
proxy[5] = 10;
```

the target's:

```text
length
```

may also change.

A proxy trap that manually emulates array behavior must account for these relationships.

Using:

```js
Reflect.set(target, key, value, receiver);
```

is usually safer than reimplementing array semantics from scratch.

---

# 50. Proxy and Functions

Function proxies can intercept:

```text
calls
construction
property access
property writes
reflection
```

This enables:

```text
decorators
logging
authorization
mocking
instrumentation
dependency injection
RPC-like wrappers
```

But each wrapper changes function identity.

---

# 51. Function Proxy and `this`

Example:

```js
const proxy = new Proxy(
  function () {
    return this;
  },
  {
    apply(target, thisArg, args) {
      return Reflect.apply(target, thisArg, args);
    }
  }
);
```

The forwarding trap preserves the supplied receiver.

This is important for transparent function wrappers.

---

# 52. Function Proxy and `new`

For:

```js
new proxy();
```

the `construct` trap can customize constructor behavior.

The trap receives:

```text
target
arguments
newTarget
```

`newTarget` matters for inheritance and constructor semantics.

Do not implement a constructor proxy without understanding:

```text
prototype
newTarget
construct result rules
```

---

# 53. `Reflect.apply` vs `.apply`

Compare:

```js
Reflect.apply(fn, thisArg, args);
```

with:

```js
fn.apply(thisArg, args);
```

Both can invoke a function, but `Reflect.apply` is often more direct for metaprogramming because it performs the corresponding reflective operation without relying on the target function's inherited `.apply` property.

This is especially convenient in Proxy handlers.

---

# 54. `Reflect.construct`

Use:

```js
Reflect.construct(Target, args, NewTarget);
```

to perform constructor-style invocation with explicit control over:

```text
target
arguments
newTarget
```

This is valuable for proxies and advanced subclassing.

---

# 55. Why `Reflect` Is More Than Convenience

Reflect APIs make internal-operation semantics explicit.

Compare:

```js
target[key]
```

with:

```js
Reflect.get(target, key, receiver)
```

The second communicates:

```text
I am intentionally performing reflective [[Get]]-like behavior
with an explicit receiver.
```

That clarity is valuable in framework/runtime code.

---

# 56. Proxy Trap Recursion

Dangerous:

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return receiver[key];
  }
});
```

This can recursively trigger the same `get` trap.

Instead:

```js
return Reflect.get(target, key, receiver);
```

or explicitly access the target when appropriate.

Metaprogramming code must understand the boundary between:

```text
intercepted operation
```

and:

```text
forwarded operation
```

---

# 57. Infinite Trap Recursion Example

Similarly:

```js
set(target, key, value, receiver) {
  receiver[key] = value;
  return true;
}
```

can recursively invoke:

```text
set trap
→ set trap
→ set trap
...
```

A safe forwarding implementation:

```js
return Reflect.set(target, key, value, receiver);
```

---

# 58. Proxy and `Object.defineProperty`

A set trap can indirectly lead to target property-definition behavior when forwarding through:

```js
Reflect.set(...)
```

depending on the receiver and property descriptor situation.

This is why the following may trigger more than one internal operation in a complex object graph:

```text
source-level assignment
→ Proxy trap
→ Reflect operation
→ target/receiver semantics
→ possible define-property behavior
```

Do not assume one source statement maps to exactly one internal hook.

---

# 59. Proxy and `Object.keys`

Calling:

```js
Object.keys(proxy)
```

involves more than:

```text
ownKeys
```

Conceptually:

```text
own-key discovery
→ descriptor checks
→ enumerable filtering
→ result
```

Thus a proxy can participate in a chain of internal operations.

This is another reason metaprogramming requires careful tracing.

---

# 60. Proxy and Descriptors

A handler can virtualize descriptors:

```js
getOwnPropertyDescriptor(target, key) {
  return Reflect.getOwnPropertyDescriptor(target, key);
}
```

But it cannot use descriptor virtualization to escape target invariants.

This gives a useful pattern:

```text
forward by default
customize only the intended behavior
```

---

# 61. Forwarding Handler Pattern

A robust baseline:

```js
const handler = {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  },

  set(target, key, value, receiver) {
    return Reflect.set(target, key, value, receiver);
  },

  has(target, key) {
    return Reflect.has(target, key);
  },

  deleteProperty(target, key) {
    return Reflect.deleteProperty(target, key);
  },

  ownKeys(target) {
    return Reflect.ownKeys(target);
  }
};

const proxy = new Proxy(target, handler);
```

Then add deliberate behavior only where required.

---

# 62. Proxy Transparency Principle

For a mostly transparent wrapper:

```text
trap
→ inspect / modify
→ Reflect forwarding
```

Example:

```js
get(target, key, receiver) {
  audit(key);
  return Reflect.get(target, key, receiver);
}
```

The more traps you customize, the more ways behavior can diverge from the target.

---

# 63. Security Validation

Proxies can enforce runtime validation:

```js
const secure = new Proxy(config, {
  set(target, key, value, receiver) {
    validateConfigField(key, value);
    return Reflect.set(target, key, value, receiver);
  }
});
```

But validation proxies have limitations.

Code may still access:

```text
raw target
other aliases
external copies
nested unwrapped objects
```

A proxy only controls operations that pass through it.

---

# 64. Security Boundary Warning

This is unsafe as a claim:

> “Wrapping an object in a Proxy makes it secure.”

False.

If:

```js
const target = {};
const proxy = new Proxy(target, validator);
```

and another component still has:

```js
target
```

then it can bypass the proxy.

Therefore:

> A Proxy can enforce a boundary only when references crossing the boundary are controlled.

This is central to capability design.

---

# 65. Proxy Membrane Identity

A real membrane may need to maintain:

```text
target → proxy
proxy → target
```

mappings.

Usually:

```js
WeakMap
```

is useful for this.

Why?

Because if the same target crosses the boundary twice, you generally want the same corresponding proxy identity to preserve stable relationships.

---

# 66. Membrane Design

A robust membrane may need:

```text
wrap(target)
unwrap(proxy)
identity cache
revocation
nested object wrapping
function wrapping
error handling
promise/async propagation
prototype handling
descriptor handling
```

This is significantly more complex than:

```js
new Proxy(target, handler)
```

---

# 67. Reactive Systems

A Proxy can track dependency reads:

```js
const state = new Proxy(
  { count: 0 },
  {
    get(target, key, receiver) {
      track(key);
      return Reflect.get(target, key, receiver);
    }
  }
);
```

and writes:

```js
set(target, key, value, receiver) {
  const result = Reflect.set(
    target,
    key,
    value,
    receiver
  );

  trigger(key);

  return result;
}
```

This is one reason Proxy became important to modern reactive libraries.

---

# 68. Reactive-System Correctness Issues

A reactive proxy implementation must define:

```text
identity
dependency granularity
nested object wrapping
arrays
iteration
symbols
non-enumerable properties
destructuring
method calls
computed access
effect scheduling
cleanup
```

A `get` trap alone does not produce a complete reactivity system.

---

# 69. Logging Proxies and Observability

A logging proxy can be useful for:

```text
debug sessions
development diagnostics
access tracing
contract testing
```

But production usage must consider:

```text
volume
PII
secrets
timing
performance
log recursion
```

Never blindly log every property access on sensitive objects.

---

# 70. Proxy and Performance

Proxy operations can inhibit or complicate engine optimization.

Potential costs include:

```text
trap dispatch
dynamic behavior
identity indirection
optimization barriers
reduced inline-cache effectiveness
additional function calls
```

The exact impact is engine- and workload-dependent.

Do not claim:

```text
Proxy is always slow
```

or:

```text
Proxy has zero overhead
```

without measurement.

---

# 71. Proxy on Hot Paths

If a proxy sits on a hot loop:

```js
for (...) {
  total += proxy.value;
}
```

the trap may execute repeatedly.

Ask:

```text
Is the interception needed on this path?
Can instrumentation occur at boundaries?
Can development-only proxies be disabled in production?
```

This is often a better architectural optimization than micro-optimizing trap code.

---

# 72. Memory Considerations

A proxy introduces an additional object identity.

Large systems may maintain:

```text
target
proxy
metadata
identity maps
nested wrappers
```

Membranes can create substantial wrapper graphs.

Use `WeakMap`/`WeakSet` where appropriate so metadata does not unintentionally extend object lifetimes.

---

# 73. Security Considerations

Important Proxy risks:

```text
confused deputy behavior
identity mismatch
incomplete validation
raw-target bypass
trap recursion
invariant violations
unexpected code execution
performance denial-of-service
```

Remember:

```text
property access
→ user-defined trap
→ arbitrary code
```

A metaprogrammed object can execute code where consumers expected a simple operation.

---

# 74. Production Usage

Good Proxy use cases include:

```text
reactivity
observation
validation
virtualization
compatibility adapters
membranes
revocable capabilities
debugging
RPC-like object facades
```

Risky use cases include:

```text
replacing straightforward functions
hiding core business rules
deeply nesting proxies
using proxy magic as a default architecture
```

The semantic cost should be justified.

---

# 75. When Not to Use Proxy

Prefer direct code when:

```text
normal method calls are sufficient
explicit validation is clearer
a wrapper function communicates behavior better
compile-time types can enforce the contract
instrumentation can happen at boundaries
performance is sensitive
debuggability matters more than transparency
```

Proxy is a tool for cases where interception itself provides significant architectural value.

---

# 76. Proxy and Type Systems

A runtime Proxy can change behavior dynamically.

Static type systems can describe the apparent surface, but the runtime behavior may still be more dynamic than the type declaration suggests.

For TypeScript-oriented systems, clearly document:

```text
runtime semantics
identity
mutability
unknown keys
private behavior
```

Do not let static types hide runtime metaprogramming complexity.

---

# 77. Proxy and Serialization

Serialization can trigger:

```text
property discovery
property reads
getters
proxy traps
```

Therefore a Proxy can affect serialization behavior.

This matters for:

```text
logging
API responses
caching
structured data
```

Treat serialization as a behavioral interaction, not merely data extraction.

---

# 78. Proxy and JSON

A proxy can influence values observed by `JSON.stringify` through intercepted property operations.

But JSON serialization has its own rules.

Therefore:

```text
Proxy
+
serialization
=
composition of semantics
```

not:

```text
proxy controls JSON completely
```

---

# 79. Proxy and Object Spread

Object spread may trigger proxy-mediated:

```text
ownKeys
descriptor/metadata
property reads
```

depending on the source object and algorithm.

This means:

```js
const copy = { ...proxy };
```

is not necessarily equivalent to:

```js
const copy = { ...target };
```

even if the proxy forwards most operations.

---

# 80. Proxy and Private State

As shown earlier:

```text
private fields
```

use private-brand semantics rather than ordinary property lookup.

Therefore a Proxy wrapper can break code that expects:

```text
method called with the original instance as receiver
```

when the method is invoked through the proxy.

This is one of the strongest reasons to be cautious with “transparent” proxies around class instances that use private fields.

---

# 81. Proxy and Built-ins

Some built-ins depend on internal slots.

For example, a Proxy around a `Map` is not necessarily interchangeable with the `Map` itself when invoking methods.

A method may expect:

```text
this has [[MapData]]
```

but:

```text
this = proxy
```

does not necessarily have the target's internal slot.

Thus:

```text
Proxy(target)
```

does not magically copy target internal slots.

This is a crucial advanced rule.

---

# 82. Internal Slots vs Properties

Ordinary property access can be intercepted.

Internal slots are not generic object properties.

Conceptually:

```text
Map
├── public properties
└── internal slot [[MapData]]
```

A Proxy around the Map does not automatically acquire:

```text
[[MapData]]
```

This explains why certain built-in methods can fail when invoked with a Proxy receiver.

---

# 83. Example: Map Proxy Caveat

Conceptually:

```js
const map = new Map();
const proxy = new Proxy(map, {});

proxy.set("x", 1);
```

This can fail in common runtimes because the Map method expects a receiver with the appropriate internal slot.

The safe pattern may involve binding/wrapping methods deliberately, depending on the intended API.

This is a concrete reminder:

> Proxy transparency is limited by internal-slot semantics.

---

# 84. Function Methods and Proxies

Similarly, built-in methods may require:

```text
specific internal slots
```

and not work correctly when:

```text
this = Proxy
```

This is especially relevant for:

```text
Map
Set
WeakMap
WeakSet
Date
TypedArray
Promise-related objects
```

Careful testing is required.

---

# 85. `Proxy.revocable` and Resource Lifetimes

Revocation gives a clear lifecycle:

```text
capability active
      ↓
operations allowed
      ↓
revoke
      ↓
operations fail
```

This is useful when designing temporary authority.

But the target object itself still exists if other references remain.

Revocation controls the proxy capability, not object destruction.

---

# 86. Debugging Proxy Code

When debugging a Proxy:

```text
1. Identify proxy and target.
2. Determine exact trap triggered.
3. Determine receiver.
4. Determine whether target is accessed directly.
5. Check Reflect forwarding.
6. Check invariant requirements.
7. Check recursion.
8. Check function identity.
9. Check private/internal-slot assumptions.
10. Check raw-target aliases.
```

This workflow should become automatic.

---

# 87. Debugging Trap Recursion

Bug:

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return receiver[key];
  }
});
```

Trace:

```text
proxy[key]
→ get trap
→ receiver[key]
→ get trap
→ receiver[key]
→ ...
```

Fix:

```js
return Reflect.get(target, key, receiver);
```

or deliberately use the target when receiver semantics are not desired.

---

# 88. Debugging Receiver Bug

Bug:

```js
get(target, key) {
  return target[key];
}
```

Suppose the target has an inherited accessor.

Potentially:

```text
getter this = target
```

instead of the desired:

```text
getter this = proxy
```

Fix:

```js
get(target, key, receiver) {
  return Reflect.get(target, key, receiver);
}
```

when transparent receiver behavior is intended.

---

# 89. Debugging Private Field Failure

Given:

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}

const user = new User();
const proxy = new Proxy(user, {});
```

Calling:

```js
proxy.getName();
```

can fail because:

```text
this = proxy
```

and:

```text
proxy
```

does not automatically carry the private brand of:

```text
user
```

Possible designs:

```text
bind methods to target
avoid proxying such instances
use explicit wrapper API
carefully preserve receiver semantics
```

Each has trade-offs.

---

# 90. Code Review Exercise — Incomplete Forwarding

Review:

```js
const proxy = new Proxy(target, {
  get(target, key) {
    return target[key];
  },

  set(target, key, value) {
    target[key] = value;
    return true;
  }
});
```

Problems:

```text
receiver omitted
descriptor/prototype semantics simplified
Reflect forwarding not used
set success is assumed
accessors can behave differently
invariants are easier to violate
```

A better baseline:

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  },

  set(target, key, value, receiver) {
    return Reflect.set(target, key, value, receiver);
  }
});
```

---

# 91. Code Review Exercise — Logging Trap

Review:

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    console.log("get", key);
    return Reflect.get(target, key, receiver);
  }
});
```

Questions:

```text
Can key be a Symbol?
Can logging expose secrets?
Can logging itself trigger user code?
Can logging create huge output?
Will hot-path reads become expensive?
```

Instrumentation must be designed like production observability.

---

# 92. Code Review Exercise — Validation

Review:

```js
const proxy = new Proxy(config, {
  set(target, key, value, receiver) {
    if (key === "port" && value < 0) {
      return false;
    }

    return Reflect.set(target, key, value, receiver);
  }
});
```

Questions:

```text
Should failure throw or return false?
What happens in strict mode?
What about non-integer ports?
What about inherited setters?
What about raw config aliases?
```

Runtime validation is a contract, not just a trap.

---

# 93. Code Review Exercise — Raw Target Escape

Review:

```js
function createProtected(target) {
  return {
    proxy: new Proxy(target, validator),
    target
  };
}
```

If:

```text
target
```

is exposed, callers can bypass:

```text
validator
```

The security boundary is therefore broken.

This is a general capability-design lesson.

---

# 94. Implementation From Scratch

Build a simplified Proxy model:

```text
ProxyObject
├── target
├── handler
└── operations
```

Implement:

```text
get
set
has
delete
ownKeys
getPrototypeOf
```

Do not attempt exact ECMAScript invariants initially.

---

# 95. Simplified Proxy Model

```js
class SimpleProxy {
  constructor(target, handler = {}) {
    this.target = target;
    this.handler = handler;
  }

  get(key, receiver = this) {
    if (this.handler.get) {
      return this.handler.get(
        this.target,
        key,
        receiver
      );
    }

    return this.target[key];
  }
}
```

This teaches the core interception model.

---

# 96. Add Reflect-Like Forwarding

Implement:

```js
function reflectGet(target, key, receiver) {
  return Reflect.get(target, key, receiver);
}
```

Then use:

```js
get(key, receiver = this) {
  if (this.handler.get) {
    return this.handler.get(
      this.target,
      key,
      receiver
    );
  }

  return reflectGet(
    this.target,
    key,
    receiver
  );
}
```

---

# 97. Trap Dispatch Table

Build a table:

```text
operation
   ↓
trap name
   ↓
default forwarding operation
```

Example:

```text
GET
→ get
→ Reflect.get

SET
→ set
→ Reflect.set

HAS
→ has
→ Reflect.has

DELETE
→ deleteProperty
→ Reflect.deleteProperty

OWN_KEYS
→ ownKeys
→ Reflect.ownKeys
```

This makes Proxy behavior easier to reason about.

---

# 98. Guided Implementation

Implement:

```text
SimpleProxy.get
SimpleProxy.set
SimpleProxy.has
SimpleProxy.delete
SimpleProxy.ownKeys
SimpleProxy.getPrototypeOf
```

Then add:

```text
apply
construct
```

for callable/constructable wrappers.

---

# 99. Partially Guided Invariant Checker

Build checks for:

```text
non-configurable keys cannot disappear
non-extensible target cannot report arbitrary keys
fixed data properties cannot return incompatible values
prototype reports must remain consistent
```

The checker does not need to replicate the complete specification.

Its purpose is to make invariants explicit.

---

# 100. No-Reference Implementation

Build a metaprogramming layer supporting:

```text
object proxy
function proxy
property validation
logging
own-key virtualization
revocation
identity cache
```

Then compare behavior against native Proxy.

---

# 101. Edge-Case Hardening

Test:

```text
Symbols
getters
setters
non-configurable properties
non-extensible targets
arrays
classes
private fields
Map/Set
functions
constructors
prototype changes
revocation
cross-object identity
nested proxies
```

Document every deviation from native behavior.

---

# 102. Production-Grade Exercise

Build a revocable capability membrane for a small object graph.

Requirements:

```text
same target gets stable proxy
nested objects are wrapped
functions are wrapped
revocation works
raw target references are not exposed
identity mappings use WeakMap
```

Then document:

```text
security model
lifetime
performance
limitations
```

---

# 103. Principal-Level Design Exercise

A framework proposes:

```text
Proxy every application object
```

Evaluate the proposal.

Questions:

```text
What value does global interception provide?
Which paths become hot?
How does debugging change?
How does memory usage change?
How are private fields affected?
How do internal-slot built-ins behave?
What happens to identity?
Can users access raw objects?
What is the failure mode?
```

A principal answer should reject blanket metaprogramming unless there is a compelling, measurable benefit.

---

# 104. Performance Benchmark Exercise

Compare:

```js
target.value
```

and:

```js
proxy.value
```

for:

```text
10^5
10^6
10^7
```

iterations.

Measure:

```text
elapsed time
allocation
GC
optimized/deoptimized behavior where tooling exposes it
```

Then test:

```text
empty handler
forwarding handler
logging handler
validation handler
```

The purpose is to learn the actual workload cost.

---

# 105. Debugging Exercise 1

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    console.log("get", key);
    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.value);
```

Trace every step.

---

# 106. Debugging Exercise 2

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {
  get(target, key) {
    return target[key];
  }
});
```

Compare this with a `Reflect.get` implementation using an accessor on the prototype.

---

# 107. Debugging Exercise 3

```js
const target = {
  value: 1
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return receiver[key];
  }
});

console.log(proxy.value);
```

Identify the recursion.

---

# 108. Debugging Exercise 4

```js
const target = {};

Object.defineProperty(target, "x", {
  value: 1,
  writable: false,
  configurable: false
});

const proxy = new Proxy(target, {
  get() {
    return 2;
  }
});
```

Explain why the proxy cannot simply violate the target's fixed-property invariant.

---

# 109. Debugging Exercise 5

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}

const user = new User();

const proxy = new Proxy(user, {});

console.log(proxy.getName());
```

Explain the role of:

```text
receiver
private brand
proxy identity
```

---

# 110. Debugging Exercise 6

```js
const map = new Map();

const proxy = new Proxy(map, {});

proxy.set("x", 1);
```

Explain why built-in internal slots can make naive proxies fail.

---

# 111. Interview Questions

## Beginner

1. What is a Proxy?
2. What is a trap?
3. What is Reflect?
4. Why are Proxy and target not `===`?
5. What does the `get` trap intercept?
6. What does `set` intercept?

## Intermediate

7. Why is Reflect useful inside Proxy handlers?
8. What is the receiver parameter?
9. What does `ownKeys` intercept?
10. What are Proxy invariants?
11. Why can `for...in` and `Object.keys` behave differently through a proxy?
12. What is a revocable Proxy?
13. What is the difference between a function Proxy and an object Proxy?

## Advanced

14. Why can `target[key]` differ from `Reflect.get(target, key, receiver)`?
15. Why can Proxy break private-field access?
16. Why can Proxy break built-in internal-slot expectations?
17. How does `construct` differ from `apply`?
18. Why is a transparent Proxy hard to implement?
19. How do Proxy identity and WeakMap identity interact?
20. How can a Proxy cause recursive traps?
21. What invariants constrain `ownKeys`?
22. Why can proxying arrays be tricky?

## Principal

23. When is Proxy appropriate in production?
24. When should Proxy be rejected in favor of explicit wrappers?
25. How would you design a revocable capability membrane?
26. How would you prevent raw-target bypass?
27. How would you benchmark proxy overhead?
28. How would you design a reactive system on Proxy without turning every access into an unbounded runtime cost?
29. How would you explain Proxy transparency limitations to a framework team?
30. Which Proxy behaviors are ECMAScript guarantees and which performance claims are engine-specific?

---

# 112. Predict-the-Output Exercises

Predict before running.

## Exercise A

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.value);
```

## Exercise B

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {
  get() {
    return 20;
  }
});

console.log(proxy.value);
console.log(target.value);
```

## Exercise C

```js
const target = {
  value: 10
};

const proxy = new Proxy(target, {});

console.log(proxy === target);
```

## Exercise D

```js
const target = {
  value: 1
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return receiver[key];
  }
});

console.log(proxy.value);
```

## Exercise E

```js
const parent = {
  get value() {
    return this._value;
  }
};

const target = Object.create(parent);
target._value = 10;

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});

console.log(proxy.value);
```

## Exercise F

```js
const target = {};

Object.defineProperty(target, "x", {
  value: 1,
  writable: false,
  configurable: false
});

const proxy = new Proxy(target, {
  get() {
    return 2;
  }
});

console.log(proxy.x);
```

Explain why this is constrained by a Proxy invariant.

## Exercise G

```js
const { proxy, revoke } = Proxy.revocable(
  { value: 1 },
  {}
);

console.log(proxy.value);

revoke();

console.log(proxy.value);
```

## Exercise H

```js
function greet() {
  return this.name;
}

const proxy = new Proxy(greet, {
  apply(target, thisArg, args) {
    return Reflect.apply(target, thisArg, args);
  }
});

console.log(proxy.call({ name: "A" }));
```

## Exercise I

```js
const target = {
  a: 1,
  b: 2
};

const proxy = new Proxy(target, {
  ownKeys() {
    return ["b", "a"];
  }
});

console.log(Reflect.ownKeys(proxy));
```

## Exercise J

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}

const user = new User();
const proxy = new Proxy(user, {});

console.log(proxy.getName());
```

Predict the result and explain why.

---

# 113. Mastery Exercises

## Level 1 — Understand

Define:

```text
Proxy
target
handler
trap
Reflect
receiver
invariant
revocation
metaprogramming
```

---

## Level 2 — Explain

Explain why:

```js
Reflect.get(target, key, receiver)
```

is often preferable to:

```js
target[key]
```

inside a transparent `get` trap.

---

## Level 3 — Predict

Predict:

```text
get
set
has
delete
ownKeys
apply
construct
revocation
```

behavior.

---

## Level 4 — Implement

Build the simplified proxy layer.

---

## Level 5 — Debug

Diagnose:

```text
recursive trap
wrong receiver
invariant violation
private-brand failure
internal-slot failure
raw-target bypass
identity mismatch
```

---

## Level 6 — Defend

Defend:

> Proxy is an interception mechanism constrained by the object model's invariants, not a magical replacement for the target object.

---

# 114. Principal-Level Reasoning Problems

## Problem 1 — Transparent proxy promise

A framework promises:

> “All proxied objects behave exactly like the originals.”

Challenge this.

Discuss:

```text
identity
private fields
internal slots
receiver
reflection
function identity
WeakMap keys
performance
errors
```

Explain why “transparent” requires a carefully scoped definition.

---

## Problem 2 — Global reactivity

A framework proxies every object in the application.

Evaluate:

```text
instrumentation value
proxy overhead
identity
debugging
memory
raw object escape
built-in compatibility
development vs production mode
```

Would you approve the architecture?

---

## Problem 3 — Security membrane

You must expose an object graph to untrusted plugins.

Design:

```text
membrane
identity mapping
revocation
nested wrapping
function wrapping
raw-target prevention
error isolation
```

Then identify which guarantees still require stronger isolation than Proxy.

---

# 115. Production Design Framework

Before adding Proxy, answer:

```text
1. What operation must be intercepted?
2. Why is normal code insufficient?
3. Is the interception global or local?
4. Who owns the target reference?
5. Can callers bypass the proxy?
6. What identity should consumers observe?
7. Are private fields involved?
8. Are internal-slot built-ins involved?
9. What are the performance characteristics?
10. What debugging tools will expose the behavior?
11. Can a simpler wrapper express the same contract?
12. Can the proxy be disabled in hot paths or production?
```

---

# 116. Proxy Review Checklist

Review:

```text
Handler
→ Are traps minimal?

Forwarding
→ Is Reflect used correctly?

Receiver
→ Is receiver preserved?

Invariants
→ Can any trap violate target constraints?

Recursion
→ Could handler operations trigger the same trap?

Identity
→ Is proxy !== target acceptable?

Lifecycle
→ Who retains/revokes it?

Security
→ Can raw targets escape?

Internal slots
→ Are built-ins/classes safe to proxy?

Performance
→ Is the trap on a hot path?

Observability
→ Can engineers debug it?
```

---

# 117. Completion Criteria

### Understand

You can define:

- Proxy;
- target;
- handler;
- trap;
- Reflect;
- receiver;
- invariant;
- revocable Proxy;
- metaprogramming.

### Explain

You can explain:

- major trap categories;
- Reflect forwarding;
- receiver propagation;
- Proxy invariants;
- function proxies;
- constructor proxies;
- `ownKeys`;
- revocation;
- membranes;
- identity implications;
- private-field limitations;
- internal-slot limitations.

### Predict

You can correctly predict:

- `get`;
- `set`;
- `has`;
- `deleteProperty`;
- `ownKeys`;
- `apply`;
- `construct`;
- revocation;
- receiver behavior;
- invariant errors.

### Implement

You can build:

```text
Proxy-like interception
Reflect forwarding
trap dispatch
identity cache
revocation
basic invariant checks
```

### Debug

You can diagnose:

```text
recursive trap
wrong receiver
invariant violation
private-brand failure
built-in internal-slot failure
raw target bypass
proxy/target identity mismatch
```

### Principal Judgment

You can decide whether Proxy is justified based on:

```text
architectural value
correctness
performance
security
debuggability
complexity
```

**Evidence of mastery:**

- 90%+ output-prediction accuracy;
- working proxy simulator;
- correct receiver explanation;
- correct invariant reasoning;
- successful analysis of private/internal-slot limitations;
- defensible decision on when not to use Proxy.

---

# 118. Key Takeaways

1. **Proxy is a metaprogramming mechanism that intercepts object internal operations.**
2. **A Proxy and its target are distinct object identities.**
3. **Property access, writes, existence checks, deletion, reflection, prototype operations, and function calls can be intercepted.**
4. **Each Proxy trap corresponds conceptually to one or more internal object operations.**
5. **Reflect provides standard reflective operations that are especially useful for forwarding.**
6. **`Reflect.get(target, key, receiver)` preserves receiver semantics more accurately than naive `target[key]` in transparent traps.**
7. **Receiver propagation matters for getters, setters, inheritance, methods, and wrappers.**
8. **A Proxy does not automatically become equivalent to its target.**
9. **Proxy identity matters for `===`, Map/WeakMap keys, caches, and framework identity.**
10. **Proxy traps are constrained by ECMAScript invariants.**
11. **Non-configurable and non-extensible target state places strong limits on what a proxy may report.**
12. **`ownKeys` is constrained by target key invariants.**
13. **Revocable proxies provide explicit capability lifetime control.**
14. **Membranes use proxies to wrap object graphs across boundaries, but robust membranes require identity and lifecycle management.**
15. **A proxy only controls operations performed through that proxy; raw target aliases bypass it.**
16. **Private fields use private-brand semantics and are not ordinary property operations.**
17. **Proxying class instances with private fields can therefore break method behavior.**
18. **Built-ins with internal slots may not be safely transparent through proxies.**
19. **Function proxies can intercept `apply` and `construct`, but callable and constructable are distinct properties.**
20. **Proxy traps can recursively trigger themselves if handlers access the proxy incorrectly.**
21. **Forwarding through Reflect is the safest baseline for transparent proxy behavior.**
22. **Proxy can enable validation, observation, reactivity, virtualization, membranes, and revocable capabilities.**
23. **Proxy can also hide behavior, complicate debugging, increase runtime cost, and create semantic surprises.**
24. **The performance impact of Proxy is engine- and workload-dependent; measure hot paths.**
25. **Proxy is not inherently a security boundary; controlled references are required.**
26. **Metaprogramming should be used where interception provides real architectural value.**
27. **A principal engineer treats transparency, identity, lifecycle, invariants, performance, and debuggability as first-class Proxy design concerns.**

---

# 119. Final Mastery Drill

For every proxy system, answer:

```text
1. What is the target?
2. What is the proxy?
3. What operation is being intercepted?
4. Which trap runs?
5. What is the receiver?
6. What does Reflect do here?
7. Which target invariant applies?
8. Is the result transparent?
9. Can raw target aliases bypass the proxy?
10. What identity is exposed?
11. What state can be retained?
12. Are private fields involved?
13. Are internal slots involved?
14. Is the path hot?
15. What production contract follows?
```

### Drill 1

```js
const target = {
  value: 1
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  }
});
```

### Drill 2

```js
const target = {};

const proxy = new Proxy(target, {
  set(target, key, value, receiver) {
    validate(key, value);
    return Reflect.set(target, key, value, receiver);
  }
});
```

### Drill 3

```js
const target = {
  get value() {
    return this._value;
  }
};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return target[key];
  }
});
```

Explain the receiver difference.

### Drill 4

```js
const target = {};

Object.defineProperty(target, "x", {
  value: 1,
  writable: false,
  configurable: false
});
```

Design a proxy that observes reads without violating invariants.

### Drill 5

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}
```

Explain why a generic transparent proxy may break:

```js
new Proxy(new User(), {}).getName()
```

### Drill 6

```js
const target = new Map();
const proxy = new Proxy(target, {});
```

Explain the internal-slot limitation.

### Drill 7

```js
const { proxy, revoke } = Proxy.revocable(
  { secret: 42 },
  {}
);
```

Design an API that grants and later revokes access.

### Drill 8

```js
const target = {};

const proxy = new Proxy(target, {
  get(target, key, receiver) {
    return receiver[key];
  }
});
```

Find and explain the recursion.

### Drill 9

```js
const target = {};
const proxy = new Proxy(target, {});

const map = new WeakMap();
map.set(target, "raw");
map.set(proxy, "wrapped");
```

Explain why both keys can coexist.

### Drill 10

A framework proposes:

```text
Proxy every object.
```

Write a principal-level approval/rejection memo using:

```text
correctness
performance
memory
security
reliability
maintainability
observability
developer experience
operational complexity
future change
```

---

# 120. Transition to Chapter 20

Chapter 19 established:

```text
internal object operations
   ↓
Proxy interception
   ↓
Reflect forwarding
   ↓
metaprogramming
   ↓
invariants
```

Chapter 20 will focus on another advanced language-level extension mechanism:

```text
What are Symbols?
Why are they unique?
How are symbol keys different from strings?
What are well-known symbols?
How does Symbol.iterator influence iteration?
How does Symbol.toPrimitive influence coercion?
How does Symbol.hasInstance influence instanceof?
How do symbols participate in protocols without creating string-key collisions?
```

The next progression is:

```text
property keys
   ↓
symbol keys
   ↓
well-known symbols
   ↓
language protocols
   ↓
custom object behavior
```

This will connect directly to:

```text
iteration
coercion
instanceof
async iteration
serialization hooks
subclassing
metaprogramming
```

---

# 121. Chapter Completion Record

**Chapter:** 19 — Proxy, Reflect, and Metaprogramming

**Status:** `[+] Completed`

**Strong Areas Expected:**

- Proxy/target distinction;
- trap categories;
- Reflect forwarding;
- receiver semantics;
- invariants;
- `ownKeys`;
- `apply`/`construct`;
- revocable proxies;
- membranes;
- identity;
- private-field limitations;
- internal-slot limitations;
- reactive/validation/observation patterns.

**Revision Triggers:**

- treating Proxy as a transparent alias;
- ignoring receiver;
- using `target[key]` blindly inside `get`;
- causing recursive traps;
- returning invariant-violating results;
- exposing raw targets through a supposed security boundary;
- assuming private fields are proxied;
- assuming built-in internal slots transfer to proxies;
- assuming Proxy performance is universally bad or good;
- using Proxy where explicit code is clearer.

**Evidence of Mastery:**

- 90%+ prediction accuracy;
- working Proxy-like simulator;
- correct Reflect forwarding;
- correct invariant reasoning;
- successful receiver/private-field/internal-slot debugging;
- defensible decision on when Proxy should or should not be used.
---
