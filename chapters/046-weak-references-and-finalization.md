# Chapter 46 — Weak References and Finalization

## Chapter Metadata

```text
Chapter: 46
Title: Weak References and Finalization
Part: VIII — JavaScript Engine
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```

---

# 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain strong reachability.
- Explain weak reachability.
- Explain why weak references exist.
- Distinguish `WeakMap`, `WeakSet`, `WeakRef`, and `FinalizationRegistry`.
- Explain how weak references interact with garbage collection.
- Explain why weak references do not create normal strong reachability.
- Explain why WeakMap keys do not keep objects alive in the same way as Map keys.
- Explain why WeakSet values do not keep objects alive in the same way as Set values.
- Explain the purpose of `WeakRef`.
- Explain the purpose of `FinalizationRegistry`.
- Explain why `WeakRef` and finalization are nondeterministic.
- Explain why finalization is not a destructor.
- Explain why finalization is not a reliable resource-cleanup mechanism.
- Understand the difference between:
  - object lifetime;
  - reachability;
  - collection;
  - finalization notification.
- Explain why weak collections are not normally enumerable.
- Understand how weak associations help metadata follow object lifetime.
- Understand why a WeakMap cannot be used to determine how many objects it contains.
- Understand why WeakSet does not expose deterministic membership enumeration.
- Explain the semantics and risks of `WeakRef.prototype.deref()`.
- Explain why a value returned by `deref()` may disappear after the reference is no longer held strongly.
- Understand the concept of liveness versus reachability.
- Understand why an object can remain alive for reasons that are not obvious from local code.
- Understand the relationship between GC roots and weak references.
- Understand the effect of microtasks/jobs and host scheduling on weak-reference observability.
- Understand why finalizer execution timing is intentionally unspecified/nondeterministic.
- Understand why an engine may collect an object later than expected.
- Understand why an engine may keep an object alive longer than a human expects.
- Understand why optimization can make weak-reference reasoning subtle.
- Understand cleanup obligations for external resources.
- Explain safe and unsafe WeakMap use cases.
- Explain safe and unsafe WeakRef use cases.
- Explain safe and unsafe FinalizationRegistry use cases.
- Implement a teaching model of weak references.
- Implement a WeakMap-like metadata structure conceptually.
- Simulate reachability and weak cleanup.
- Build tests that do not depend on exact GC timing.
- Diagnose accidental strong references that defeat intended weak behavior.
- Analyze cache designs that use weak references.
- Design object-associated metadata without changing object ownership/lifetime.
- Explain identity-keyed metadata patterns.
- Explain ephemeron-style reasoning at a conceptual level.
- Distinguish language semantics from engine-specific GC implementation behavior.
- Evaluate weak-reference designs using correctness, performance, memory, security, observability, and reliability criteria.
- Defend weak-reference and finalization decisions at principal-engineer depth.

### Mastery Gate

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# 2. Prerequisites

Required:

- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations
- Chapter 44 — Realms, Agents, and Execution Isolation
- Chapter 45 — JavaScript Memory and Garbage Collection

Strongly related:

- Chapter 13 — Closures
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 43 — Ordinary Object Internal Methods

Builds toward:

- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 108 — Cache System
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

---

# 3. What Is It?

JavaScript supports **weak references**: relationships that do not keep an object alive in the same way as ordinary strong references.

The core abstractions are:

```text
WeakMap
WeakSet
WeakRef
FinalizationRegistry
```

A strong reference looks conceptually like:

```text
Root
 ↓
Object
```

A weak relationship can be modeled conceptually as:

```text
Root
 ↓
Weak reference ──X──> Object
```

The weak relationship does not by itself make the target reachable through ordinary strong reachability.

This enables patterns such as:

```text
object
  ↕
metadata
```

where metadata should disappear naturally when the key object becomes otherwise unreachable.

It also enables advanced observation of object liveness through `WeakRef` and eventual cleanup notifications through `FinalizationRegistry`.

However:

> Weak references deliberately expose nondeterministic behavior around object lifetime.

Therefore they are specialized tools, not replacements for explicit ownership and cleanup.

---

# 4. Why Does It Exist?

Some data should be associated with an object without changing that object's lifetime.

Example:

```js
const metadata = new WeakMap();

function attachMetadata(object, value) {
  metadata.set(object, value);
}
```

The desired policy is:

```text
object alive
→ metadata available

object no longer otherwise reachable
→ metadata association becomes irrelevant
```

A normal Map would retain the key:

```js
const map = new Map();

map.set(object, value);
```

The Map itself keeps:

```text
object
```

strongly reachable.

That can defeat the desired lifetime relationship.

Weak collections solve a different ownership requirement:

> Associate information with an object without making the key's lifetime depend on the association.

`WeakRef` and `FinalizationRegistry` provide additional advanced capabilities, but their nondeterminism makes them inappropriate for critical lifecycle control.

---

# 5. Mental Model

Use two separate graphs.

### Strong graph

```text
Root
 ↓
A
 ↓
B
 ↓
C
```

All are reachable.

### Weak relationship

```text
Root
 ↓
WeakRef ───────────> B
                      ↑
                 no strong path
```

If there is no strong path to `B`, the weak reference does not rescue it.

A WeakMap can be viewed conceptually as:

```text
Strong:
application → WeakMap

Weak:
WeakMap - - -→ key object
              ↓
           value data
```

The critical point is:

> The WeakMap itself is strongly reachable, but its key does not become strongly reachable merely because it is registered in the WeakMap.

---

# 6. Core Rules

### Rule 1 — Weak references do not create ordinary strong reachability

### Rule 2 — WeakMap keys are weakly held

The key object is not kept alive merely because it is a key in a WeakMap.

### Rule 3 — WeakSet values are weakly held

Membership does not establish the same strong retention relationship as a Set.

### Rule 4 — WeakRef is nondeterministic

You cannot predict exactly when a referent becomes unavailable.

### Rule 5 — `deref()` may return an object or `undefined`

A weakly referenced object can disappear from the set of reachable objects.

### Rule 6 — Holding the result of `deref()` strongly matters

A strong local reference can keep the object alive while that reference remains reachable.

### Rule 7 — Finalization is nondeterministic

Do not rely on exact timing.

### Rule 8 — Finalization is not a destructor

It does not provide deterministic resource release.

### Rule 9 — Weak collections are intentionally not general-purpose enumerable collections

Deterministic enumeration would expose unstable GC-dependent membership.

### Rule 10 — Weak references are not cache eviction policy by themselves

A cache should still have explicit policy when correctness/performance requires predictable behavior.

### Rule 11 — External resources require explicit cleanup

Use:

```text
close
dispose
abort
unsubscribe
terminate
```

where appropriate.

### Rule 12 — Weakness is about reachability, not “importance”

An object can be weakly referenced and still be strongly reachable elsewhere.

### Rule 13 — Strong aliases defeat intended weak lifetime

A hidden strong reference means the target can remain alive.

### Rule 14 — Finalization callback timing is not a scheduling contract

Do not build correctness around when it runs.

### Rule 15 — Weak references increase complexity

Use them only when their lifecycle semantics solve a real problem.

---

# 7. Syntax

## WeakMap

```js
const metadata = new WeakMap();

metadata.set(object, value);

metadata.get(object);
metadata.has(object);
metadata.delete(object);
```

## WeakSet

```js
const seen = new WeakSet();

seen.add(object);
seen.has(object);
seen.delete(object);
```

## WeakRef

```js
const ref = new WeakRef(object);

const value = ref.deref();
```

## FinalizationRegistry

```js
const registry = new FinalizationRegistry(heldValue => {
  // notification may happen later
});

registry.register(object, heldValue);
```

Optional unregister token:

```js
registry.register(object, heldValue, unregisterToken);
registry.unregister(unregisterToken);
```

These APIs are standardized language features, but their exact garbage-collection timing is intentionally not deterministic.

---

# 8. Basic Examples

## Example 1 — WeakMap metadata

```js
const metadata = new WeakMap();

let object = {};

metadata.set(object, {
  createdAt: Date.now()
});

console.log(metadata.has(object));

object = null;
```

The WeakMap does not by itself retain the former key object.

## Example 2 — Map versus WeakMap

```js
let key = {};

const strong = new Map();
strong.set(key, "value");
```

The Map itself strongly retains the key.

Compare:

```js
let weakKey = {};

const weak = new WeakMap();
weak.set(weakKey, "value");

weakKey = null;
```

The WeakMap does not create the same strong-retention path.

## Example 3 — WeakSet

```js
const visited = new WeakSet();

const node = {};

visited.add(node);

console.log(visited.has(node));
```

WeakSet is useful when membership should follow object lifetime.

## Example 4 — WeakRef

```js
let object = {
  value: 10
};

const ref = new WeakRef(object);

console.log(ref.deref()?.value);

object = null;
```

You must not assume the later `deref()` will deterministically return `undefined` at a specific point.

## Example 5 — FinalizationRegistry

```js
const registry = new FinalizationRegistry(token => {
  console.log("eventually finalized", token);
});

let object = {};

registry.register(object, "object-token");

object = null;
```

The callback is not guaranteed to run at a predictable time.

---

# 9. Execution Walkthrough

Consider:

```js
const metadata = new WeakMap();

let user = {
  id: 1
};

metadata.set(user, {
  role: "admin"
});
```

### Step 1

`user` strongly references the object.

```text
root
 ↓
user
 ↓
UserObject
```

### Step 2

The WeakMap contains a weak key relationship:

```text
WeakMap - - -→ UserObject
```

### Step 3

The metadata value is associated with the key.

### Step 4

The application releases its strong reference:

```js
user = null;
```

### Step 5

If no other strong path reaches `UserObject`, the object can become unreachable.

### Step 6

The WeakMap does not create a strong root that keeps it alive merely because it is a key.

### Step 7

The key/object association can therefore become collectible.

The application should not attempt to observe an exact moment of removal.

---

# 10. Internal Mechanics

## 10.1 WeakMap identity

WeakMap membership is based on object identity.

```js
const a = {};
const b = {};

map.set(a, 1);

console.log(map.has(a)); // true
console.log(map.has(b)); // false
```

## 10.2 Object-only keys

WeakMap keys are objects or symbols according to the current language semantics; primitive support must always be checked against the ECMAScript version being studied.

For foundational reasoning, the important distinction is:

```text
weakly held identity-capable keys
```

versus ordinary primitive values.

## 10.3 WeakSet

WeakSet provides weak membership tracking for supported values.

## 10.4 WeakRef target

A WeakRef stores a weak relationship to its target.

Conceptually:

```text
WeakRef
  │
  └ - - - → target
```

## 10.5 `deref`

```js
const target = ref.deref();
```

can produce:

```text
target object
```

or:

```text
undefined
```

depending on whether the referent is still available.

## 10.6 Temporary strong reference

If:

```js
const value = ref.deref();
```

returns the target, then:

```text
value
→ target
```

is a strong reference for the relevant lifetime.

This is why access patterns should be carefully structured.

## 10.7 FinalizationRegistry registration

A registry associates:

```text
target object
held value
```

without treating the held value as another strong owner of the target.

## 10.8 Held values

The held value is what the finalizer callback receives.

It should contain enough information for meaningful post-finalization handling without unnecessarily retaining the target through another strong reference.

## 10.9 Unregister token

A token can allow explicit removal of a registration.

## 10.10 Cleanup callback

The callback may run only after collection/finalization conditions established by the engine/runtime.

The exact timing is intentionally nondeterministic.

## 10.11 Weak collections and reachability

Weakness is meaningful only with respect to strong reachability.

If another path exists:

```text
root → target
```

then:

```text
WeakMap - - -→ target
```

does not make the target collectible.

## 10.12 Ephemeron-style model

A useful conceptual model for WeakMap is an ephemeron:

```text
(key, value)
```

where the value relationship becomes relevant when the key is reachable.

A simple implementation must not naïvely treat all WeakMap values as ordinary strong references from the WeakMap itself.

## 10.13 Why ephemeron reasoning matters

Consider:

```text
WeakMap
 └── weak key → object
              └── value graph
```

The collector must reason about whether the key is otherwise reachable before allowing the associated value to remain live through the weak relationship.

## 10.14 Weak value retention

A WeakMap value can reference other objects.

Those objects can become reachable if the key is strongly reachable and the WeakMap association is semantically active.

## 10.15 Cycles

Weak structures are especially useful when cycles would otherwise retain entire graphs.

## 10.16 No deterministic size

You cannot safely ask:

```text
How many weak entries currently exist?
```

because membership can depend on GC/liveness.

## 10.17 No deterministic enumeration

Enumeration would expose GC timing and violate the intended weak abstraction.

## 10.18 Finalization ordering

Do not assume a particular order among finalizers.

## 10.19 Finalization delay

An object can become unreachable and remain physically uncollected for an arbitrary period.

## 10.20 Object resurrection

A weak/finalization design must not be based on assuming the object can simply be recovered or resurrected through a predictable finalizer sequence.

## 10.21 Cleanup callback scheduling

The callback does not run synchronously at the moment the last strong reference disappears.

## 10.22 GC heuristics

Collection frequency depends on engine heuristics and runtime conditions.

## 10.23 Optimization interaction

JIT/runtime optimization can affect physical reachability opportunities without changing the language's weak-reference contract.

---

# 11. ECMAScript / Specification Semantics

## 11.1 WeakMap

WeakMap provides key-value associations where supported keys do not gain ordinary strong reachability merely from membership.

## 11.2 WeakSet

WeakSet provides weak membership tracking.

## 11.3 WeakRef

WeakRef provides a controlled mechanism for observing whether an object remains accessible through the weak reference.

The language deliberately avoids promising exact collection timing.

## 11.4 FinalizationRegistry

A FinalizationRegistry can register objects and receive cleanup notifications after the runtime determines that the target is no longer reachable in the relevant way.

The callback timing is not deterministic.

## 11.5 No deterministic collection guarantee

The specification does not give application code a portable command:

```text
collect this object now
```

nor does it provide a portable timestamp for collection.

## 11.6 Weakness versus strong semantics

Ordinary references participate in reachability.

Weak references intentionally do not create the same strong relationship.

## 11.7 Non-enumerability

Weak collections avoid exposing membership through general iteration.

This is fundamental to the abstraction.

## 11.8 Held-value semantics

FinalizationRegistry's held value is not automatically a strong reference back to the target.

But an application can accidentally construct a strong retention path through related objects, so ownership analysis remains necessary.

## 11.9 Unregister

Explicit unregistering changes the registration set and can prevent a future finalization notification for that registration.

## 11.10 Host interaction

Actual GC implementation, scheduling, and process memory behavior remain engine/runtime concerns.

---

# 12. Advanced Behavior

## 12.1 WeakMap metadata pattern

A classic architecture:

```text
object
  ↓ identity
WeakMap
  ↓
metadata
```

Useful for:

```text
DOM-node metadata
library private state
object annotations
framework bookkeeping
```

## 12.2 Why Map can leak

With:

```js
const metadata = new Map();

metadata.set(object, metadataValue);
```

the Map may become a global owner of the object.

## 12.3 WeakMap ownership model

With:

```js
const metadata = new WeakMap();
metadata.set(object, metadataValue);
```

the metadata relationship can follow the object's lifetime.

## 12.4 WeakMap versus private fields

Private class fields attach state directly to the object.

WeakMap metadata can associate external metadata with objects you do not control.

## 12.5 WeakMap versus Symbol properties

Symbol properties mutate the target object's property space.

WeakMap keeps metadata outside that property namespace.

## 12.6 WeakRef cache

A weak-reference cache can avoid some strong retention:

```js
const cache = new Map();
cache.set(key, new WeakRef(value));
```

But this is not sufficient as a complete cache policy.

The cache entries themselves can grow without bound.

## 12.7 WeakRef + Map hazard

This:

```text
Map
→ key strongly retained
→ WeakRef(value)
```

may solve value retention but not key retention.

## 12.8 WeakRef-only lookup

A cache may return `undefined` because the target was collected.

Callers must tolerate misses.

## 12.9 WeakRef and correctness

Never make correctness depend on:

```text
“the object should still be alive.”
```

Only use it for opportunistic optimization/observation.

## 12.10 Memoization with WeakMap

WeakMap can be useful for identity-based memoization:

```js
const memo = new WeakMap();

function expensive(object) {
  if (memo.has(object)) {
    return memo.get(object);
  }

  const result = compute(object);
  memo.set(object, result);

  return result;
}
```

This can allow results to follow object lifetime.

## 12.11 Memoization limitation

If the value strongly references the key:

```text
WeakMap
key → result
       ↓
      key
```

the conceptual graph becomes more subtle.

The exact collector semantics matter.

Avoid designing cycles casually.

## 12.12 DOM metadata

A library can associate state with DOM nodes without writing custom properties onto the nodes.

## 12.13 Plugin metadata

Plugins can maintain private external state associated with host objects.

## 12.14 Framework bookkeeping

Frameworks can use weak identity associations for optional metadata.

## 12.15 WeakSet visitation

WeakSet can record objects already seen during graph traversal without permanently retaining them.

## 12.16 Graph algorithms

For object graphs that may outlive traversal contexts, weak tracking can reduce accidental retention.

## 12.17 Finalization for auxiliary cleanup

Finalization can support best-effort cleanup of non-critical auxiliary structures.

It must remain secondary to explicit cleanup.

## 12.18 Finalization token

Use held values carefully.

A token that contains:

```text
large object graph
```

can itself create retention or memory-pressure problems.

## 12.19 Registration lifetime

A FinalizationRegistry can itself remain alive because some root retains the registry.

Its registrations therefore need to be understood in application lifecycle terms.

## 12.20 Registry retention

A global registry is intentionally long-lived.

Be careful about what metadata it stores.

## 12.21 Finalizer exceptions

A cleanup callback throwing does not turn finalization into ordinary synchronous application control flow.

Do not rely on finalizers for user-visible error handling.

## 12.22 Finalizer reentrancy

Cleanup callbacks execute in a separate scheduling context and can interact with application state.

Design them as minimal and defensive.

## 12.23 Finalizer storms

Large numbers of finalized objects can result in many callbacks.

Do not assume one callback is an inexpensive operation.

## 12.24 Finalizer ordering

Never depend on:

```text
A finalizes before B
```

unless a separate guarantee exists.

## 12.25 Finalizer timing across runtimes

Timing differences across browsers, Node versions, and engines are expected.

## 12.26 Process exit

At process shutdown, finalization may not behave like a normal deterministic cleanup phase.

Critical resources must not depend on it.

## 12.27 Crashes

On abrupt process termination:

```text
finalization may never happen
```

This is another reason external resources need explicit lifecycle handling.

## 12.28 Worker termination

Worker shutdown can occur before a finalization callback would be observed.

## 12.29 Cross-Agent boundaries

Weak references are local semantic/runtime relationships; do not assume they magically coordinate object lifetime across separate Agents/processes.

## 12.30 Realms

Realm-specific object identities remain relevant to weak associations.

## 12.31 Proxy objects

A Proxy can be the identity key rather than its target.

Do not assume:

```text
proxy === target
```

or that WeakMap lookup by target automatically matches a Proxy key.

## 12.32 Identity is exact

WeakMap uses identity semantics:

```text
same object reference
```

not structural equality.

## 12.33 Primitive keys

Weak collections have specific key/value constraints. Check the exact ECMAScript version when relying on newly standardized primitive-key behavior.

## 12.34 Symbol keys

Modern ECMAScript semantics have evolved around weak collection key capabilities; always align production assumptions with the language edition/runtime actually supported.

## 12.35 Debugger effects

Debugging/devtools can keep objects reachable longer than production code.

Do not assume development observations define production GC timing.

## 12.36 Logging effects

Logging an object can temporarily retain or expose it through diagnostic tooling.

This can complicate memory experiments.

## 12.37 Testing GC-dependent behavior

Tests should not assert:

```text
object is definitely collected after X ms
```

because that is not a portable language guarantee.

---

# 13. Edge Cases

- A WeakMap being reachable does not make all its keys strongly reachable.
- A WeakMap value can still be strongly reachable through other paths.
- A key can remain alive through an unrelated alias.
- A WeakRef can return an object long after the code that originally created it stopped using it.
- A WeakRef can return `undefined` later without an application-visible deterministic collection event.
- Calling `deref()` and retaining the result creates a strong reference while that result remains reachable.
- Finalization can be delayed.
- Finalization may never run before process termination.
- Finalization order is not a reliable application contract.
- A finalizer can itself allocate memory/work.
- A global FinalizationRegistry can accumulate registrations.
- Held values can accidentally retain large state.
- Unregister tokens can be strongly held by application state.
- WeakMap values can form graphs back to their keys, creating subtle lifetime relationships.
- Map keys strongly retain objects; WeakMap keys are designed not to do so.
- WeakSet is not a general-purpose Set replacement.
- Weak collections do not provide deterministic size/iteration.
- WeakRef is not a deterministic cache.
- A weak cache can still grow because the cache's key structure is strong.
- Cross-Realm identity matters.
- Proxy identity differs from target identity.
- Debuggers can alter reachability.
- Devtools heap inspection can perturb timing.
- A finalizer is not guaranteed to run before a resource becomes unusable.
- A finalizer cannot safely be treated as a transaction boundary.
- Worker/process termination can occur before weak cleanup is observed.

---

# 14. Common Misconceptions

### Misconception 1 — “WeakMap immediately removes dead entries.”

No. GC timing is nondeterministic.

### Misconception 2 — “WeakRef lets me know exactly when an object dies.”

No.

### Misconception 3 — “FinalizationRegistry is a destructor.”

No.

### Misconception 4 — “Finalization always happens eventually before program exit.”

Do not rely on that.

### Misconception 5 — “WeakMap is just Map but garbage-collectable.”

It has different semantic constraints, especially around enumeration and key reachability.

### Misconception 6 — “WeakRef makes a cache memory-safe automatically.”

No. Strong keys, metadata, indexes, and cache policy still matter.

### Misconception 7 — “Using WeakMap means the value cannot leak.”

The value may still be retained through other strong paths.

### Misconception 8 — “WeakMap can be iterated if I really need the size.”

No. Its design intentionally avoids deterministic enumeration.

### Misconception 9 — “Calling `deref()` never changes liveness.”

The returned value can become a strong reference if retained.

### Misconception 10 — “If the variable is null, the object is dead.”

Other references may keep it alive.

### Misconception 11 — “A finalizer is guaranteed to execute before cleanup is needed.”

No.

### Misconception 12 — “Finalization is a safe way to close files.”

No.

### Misconception 13 — “Weak references are an optimization with no semantic cost.”

They add nondeterminism and complexity.

### Misconception 14 — “Finalization callback order is predictable.”

No.

### Misconception 15 — “Worker termination will always run finalizers.”

No.

### Misconception 16 — “Devtools prove exactly when production GC occurs.”

No.

### Misconception 17 — “WeakMap uses structural equality.”

No. Object identity is central.

### Misconception 18 — “Proxy and target are the same WeakMap key.”

No.

---

# 15. Common Mistakes

1. Using WeakRef for correctness-critical state.
2. Using FinalizationRegistry for deterministic resource cleanup.
3. Building unbounded weak-reference caches.
4. Accidentally retaining keys elsewhere.
5. Storing huge held values in a registry.
6. Assuming a finalizer will run promptly.
7. Writing timing-dependent GC tests.
8. Treating weak collection APIs as ordinary Map/Set replacements.
9. Ignoring object identity.
10. Ignoring cross-Realm identity.
11. Ignoring Proxy/target identity.
12. Forgetting that `deref()` can create a temporary strong reference.
13. Using finalization to coordinate transactions.
14. Closing sockets/files only in a finalizer.
15. Assuming process exit guarantees cleanup callbacks.
16. Testing GC behavior only under debugger/devtools.
17. Assuming one engine's weak-reference timing is universal.
18. Ignoring registry lifecycle.
19. Ignoring held-value retention.
20. Assuming weak references solve every memory leak.

---

# 16. Comparison With Related Concepts

| Concept | Strongly retains target? | Enumeration | Deterministic lifetime? | Typical purpose |
|---|---:|---:|---:|---|
| `Map` | Yes | Yes | Yes, while reachable by map | General key-value storage |
| `Set` | Yes | Yes | Yes, while reachable by set | General membership |
| `WeakMap` | No, for weak keys | No | No | Identity metadata |
| `WeakSet` | No, for weak members | No | No | Identity membership |
| `WeakRef` | No | N/A | No | Opportunistic observation |
| `FinalizationRegistry` | No target retention by registration itself | N/A | No | Best-effort notification |

### Map vs WeakMap

```text
Map:
map → key

WeakMap:
map - - -→ key
```

### Set vs WeakSet

```text
Set:
set → value

WeakSet:
set - - -→ value
```

### WeakRef vs WeakMap

WeakMap:

```text
object identity → metadata
```

WeakRef:

```text
reference → possibly reachable object
```

### WeakRef vs strong reference

```text
strong:
readable while reference exists

weak:
may become unavailable
```

### FinalizationRegistry vs explicit cleanup

```text
finalization:
nondeterministic

explicit cleanup:
application-controlled
```

### WeakMap vs private fields

```text
WeakMap:
external metadata

private field:
state attached to object definition
```

### WeakMap vs Symbol property

```text
WeakMap:
metadata outside property namespace

Symbol:
metadata is still an object property
```

### Weak cache vs bounded cache

```text
weak cache:
opportunistic lifetime behavior

bounded cache:
explicit size/TTL/eviction policy
```

They solve different problems.

---

# 17. Performance Considerations

### 17.1 Weakness has bookkeeping cost

Weak collections require special GC-aware processing.

### 17.2 Ephemeron processing

Collectors may need additional tracing passes or special treatment for weak key/value relationships.

### 17.3 WeakRef access

`deref()` introduces a liveness-sensitive operation that can complicate optimization and reasoning.

### 17.4 Finalization callbacks

Large numbers of finalizers can create extra scheduling and CPU work.

### 17.5 WeakMap lookup

WeakMap operations are optimized in engines but should not be assumed to have the same physical representation as Map.

### 17.6 Allocation

WeakMap/WeakRef/registry objects themselves consume memory.

### 17.7 Registry growth

A registry can grow if registrations accumulate and targets remain reachable.

### 17.8 Cleanup bursts

A large set of collected targets can lead to bursts of cleanup work.

### 17.9 Weak caches

A weak-value cache may frequently experience cache misses due to collected values.

### 17.10 Cache efficiency

Weak lifetime does not imply high hit rates.

### 17.11 GC interaction

Weak structures add work to collection algorithms.

### 17.12 Benchmarking

Benchmarks involving weak references are highly sensitive to:

```text
GC heuristics
heap pressure
engine
runtime version
debugging tools
workload
```

### 17.13 Performance rule

Use weak references because their semantics are useful—not because they are assumed to be faster.

---

# 18. Memory Considerations

### 18.1 WeakMap metadata

Can reduce accidental retention compared with Map in appropriate designs.

### 18.2 WeakRef cache

Can reduce strong retention of large values, but only if the surrounding cache architecture is also bounded.

### 18.3 Registry metadata

Held values should remain compact.

### 18.4 Strong references elsewhere

One overlooked alias can make the entire weak design ineffective.

### 18.5 Cycles

Cycles are not inherently leaks for tracing GC, but weak/strong relationships can alter liveness in non-obvious ways.

### 18.6 Cache bounds

Use explicit bounds when memory limits matter:

```text
max entries
max bytes
TTL
eviction
```

### 18.7 Queues

Do not assume weak references compensate for unbounded queues.

### 18.8 Devtools

Diagnostics may retain objects.

### 18.9 Process memory

Weak-reference behavior does not directly map to immediate RSS reduction.

### 18.10 Memory policy

Use:

```text
weak association
+
explicit lifecycle
+
bounded storage
```

when predictable memory behavior is required.

---

# 19. Security Considerations

### 19.1 Hidden retention

WeakMap can protect against some unintended lifetime extension, but it is not an access-control mechanism.

### 19.2 Sensitive metadata

WeakMap metadata can still contain sensitive data while the key remains alive.

### 19.3 Finalization timing

Do not use finalization for security-critical cleanup.

### 19.4 Timing sensitivity

WeakRef/finalization behavior can expose runtime-dependent liveness signals.

Threat models should account for this where relevant.

### 19.5 Denial of service

Attackers can create many:

```text
WeakRefs
registrations
objects
callbacks
```

and increase GC/cleanup pressure.

### 19.6 Registry abuse

A global registry with attacker-controlled registrations can become a resource-management problem.

### 19.7 Weak cache poisoning

A weak cache still needs validation, eviction, authorization, and tenant isolation.

### 19.8 Capability retention

Weak references do not revoke capabilities.

If a strong reference to a privileged object exists, the object remains usable.

### 19.9 Finalizer side effects

Avoid security-sensitive state transitions from nondeterministic finalizers.

---

# 20. Production Usage

## 20.1 External metadata

Use WeakMap when:

```text
you do not control the target object's shape
+
metadata should follow target lifetime
```

## 20.2 DOM metadata

Suitable for libraries attaching private bookkeeping to DOM nodes.

## 20.3 Object identity memoization

WeakMap can memoize object-based computations without forcing permanent retention of keys.

## 20.4 Graph visitation

WeakSet can track visited object identity during long-lived processes.

## 20.5 Opportunistic cache

WeakRef can support caches where:

```text
cache hit → optimization
cache miss → recompute/reload
```

is semantically acceptable.

## 20.6 Finalization as fallback signal

Finalization can support auxiliary cleanup/metrics where cleanup is non-critical.

## 20.7 Explicit cleanup remains primary

For:

```text
socket
file
database connection
lock
worker
subscription
stream
```

use explicit lifecycle management.

## 20.8 Production testability

Do not make functionality depend on:

```text
GC happened
finalizer ran
WeakRef returned undefined
```

at a specific time.

## 20.9 Observability

Track:

```text
cache hits
cache misses
registry size
resource lifecycle
explicit cleanup
memory
GC pressure
```

rather than attempting to infer exact object death.

## 20.10 API design

Document when weak references are:

```text
best-effort
optional
nondeterministic
```

## 20.11 Service architecture

Prefer predictable policies:

```text
owner
lifetime
cleanup
capacity
```

before adding weak-reference mechanisms.

---

# 21. Implementation From Scratch

These are teaching models, not engine-equivalent GC implementations.

## Stage 1 — Weak Relationship Model

Represent:

```text
strong references
weak references
```

separately in a simulated graph.

## Stage 2 — WeakMap Simulator

Implement:

```js
class WeakMapModel {
  set(key, value) {}
  get(key) {}
  has(key) {}
  delete(key) {}
}
```

Track keys separately from the strong graph.

## Stage 3 — Mark-and-Sweep Integration

Extend the simulator:

```text
mark strong graph
inspect weak relationships
remove unreachable weak entries
```

## Stage 4 — Ephemeron Processing

Model:

```text
WeakMap
(key → value)
```

where the value's reachability depends on key reachability.

## Stage 5 — WeakRef Model

Implement:

```js
class WeakRefModel {
  constructor(target) {
    this.targetId = target.id;
  }

  deref() {
    // return target only if still simulated-live
  }
}
```

## Stage 6 — FinalizationRegistry Model

Implement:

```js
class FinalizationRegistryModel {
  register(target, heldValue, token) {}
  unregister(token) {}
  collectFinalizationCandidates() {}
}
```

Keep finalization asynchronous/nondeterministic in the teaching model.

## Stage 7 — No-Reference Exercise

Design a weak metadata system without using the native WeakMap API.

Then compare its ownership model against a normal Map.

## Stage 8 — Edge-Case Hardening

Handle:

- duplicate registrations;
- unregister tokens;
- multiple keys;
- object identity;
- registry lifecycle;
- held-value retention;
- simulated finalizer scheduling.

## Stage 9 — Production-Oriented Model

Add metrics:

```text
weak associations
live keys
collected associations
finalization candidates
callback queue
memory
```

---

# 22. Debugging Exercises

### Exercise 1 — Accidental Strong Map

```js
const metadata = new Map();

function remember(object) {
  metadata.set(object, {
    expensive: true
  });
}
```

Determine how this changes object lifetime.

### Exercise 2 — Replace Map With WeakMap

Refactor and explain the ownership difference.

### Exercise 3 — Strong Alias

Create a WeakMap entry but retain the key in another array.

Explain why the object does not become collectible.

### Exercise 4 — WeakRef Dereference

Create:

```js
let object = {};
const ref = new WeakRef(object);
```

Observe that `deref()` can succeed while the object remains strongly reachable.

### Exercise 5 — Held-Value Retention

Register an object using a held value that references a large object.

Analyze the retention graph.

### Exercise 6 — Global Registry

Create a global registry with many registrations.

Design a strategy for lifecycle control.

### Exercise 7 — Weak Cache Growth

Build:

```text
Map<string, WeakRef<Value>>
```

with unbounded keys.

Show why the cache can still grow.

### Exercise 8 — Finalizer Timing

Write a test that would incorrectly assume:

```text
object = null
→ finalizer within 100ms
```

Explain why the test is invalid.

### Exercise 9 — Debugger Retention

Compare object lifetime observations with and without devtools/debugger references.

### Exercise 10 — Proxy Identity

Create:

```js
const target = {};
const proxy = new Proxy(target, {});
```

Store one in a WeakMap and test the other.

Explain identity semantics.

---

# 23. Code Review Exercise

Review:

```js
const cache = new Map();

export function getResource(id) {
  const existing = cache.get(id);

  if (existing) {
    return existing.deref();
  }

  const resource = createResource();

  cache.set(id, new WeakRef(resource));

  return resource;
}
```

Identify:

```text
unbounded cache key growth
weak value semantics
cache-miss behavior
resource ownership
whether resource recreation is acceptable
concurrency races
stale entries
observability
```

Then design an explicit cache policy.

---

# 24. Interview Questions

## Foundational

1. What is a weak reference?
2. Why does WeakMap exist?
3. What is the difference between Map and WeakMap?
4. What is the difference between Set and WeakSet?
5. What is WeakRef?
6. What is FinalizationRegistry?
7. Why are weak collections not enumerable?
8. What does `deref()` return?
9. Why is finalization nondeterministic?
10. Is FinalizationRegistry a destructor?

## Intermediate

11. Why can WeakMap reduce accidental retention?
12. What is strong reachability?
13. What is weak reachability?
14. What is an ephemeron-like relationship?
15. Why can't WeakMap expose deterministic size?
16. Why can a WeakRef cache still grow?
17. Why can `deref()` affect practical liveness?
18. What happens if another strong reference remains?
19. What is an unregister token?
20. Why should held values be small/carefully designed?

## Advanced

21. Explain WeakMap GC semantics conceptually.
22. Explain WeakRef nondeterminism.
23. Explain FinalizationRegistry limitations.
24. Explain why finalization cannot replace explicit cleanup.
25. Explain weak cache design.
26. Explain WeakMap versus private fields.
27. Explain WeakMap versus Symbol-keyed metadata.
28. Explain debugger effects on GC observations.
29. Explain cross-Realm identity with WeakMap.
30. Explain Proxy identity with WeakMap.

## Principal-Level

31. Design an identity metadata system for a large framework.
32. Design a weak memoization system.
33. Design a weak-value cache with bounded strong metadata.
34. Explain when WeakRef should never be used.
35. Design a production lifecycle policy that uses finalization only as a fallback.
36. Diagnose a memory leak in a supposed WeakMap-based system.
37. Diagnose a registry that grows despite objects being short-lived.
38. Design GC-independent tests for weak-reference functionality.
39. Compare WeakMap, private fields, Symbols, and external caches.
40. Defend:

> “Weak references solve a specific ownership problem; they do not solve application lifetime, resource management, or cache-policy problems.”

---

# 25. Predict-the-Output Exercises

For each:

```text
Predict
→ Run
→ Inspect reachability
→ Avoid assumptions about exact GC timing
→ Explain
```

### Exercise A

```js
const weak = new WeakMap();

const object = {};

weak.set(object, 10);

console.log(weak.get(object));
```

### Exercise B

```js
let object = {};

const weak = new WeakMap();
weak.set(object, 10);

console.log(weak.has(object));

object = null;
```

What can you conclude, and what can you not conclude?

### Exercise C

```js
const weak = new WeakSet();

const object = {};

weak.add(object);

console.log(weak.has(object));
```

### Exercise D

```js
let object = {
  value: 10
};

const ref = new WeakRef(object);

console.log(ref.deref()?.value);
```

Why is this result predictable while `object` remains strongly reachable?

### Exercise E

```js
let object = {
  value: 10
};

const ref = new WeakRef(object);

const strong = ref.deref();

object = null;

console.log(strong?.value);
```

Explain the importance of `strong`.

### Exercise F

```js
const registry = new FinalizationRegistry(value => {
  console.log("finalized", value);
});

let object = {};

registry.register(object, "x");

object = null;

console.log("done");
```

What can you conclude about the ordering of `"done"` and the eventual finalizer?

### Exercise G

```js
const target = {};
const proxy = new Proxy(target, {});

const weak = new WeakMap();

weak.set(target, "target");
weak.set(proxy, "proxy");

console.log(weak.get(target));
console.log(weak.get(proxy));
```

What does identity imply?

### Exercise H

```js
const cache = new Map();

let value = {};

cache.set("key", new WeakRef(value));

value = null;
```

Why does the Map remain a potentially growing structure even though the value is weakly referenced?

---

# 26. Mastery Exercises

### Exercise 1 — WeakMap Teaching Model

Implement a weak metadata store over a simulated GC graph.

### Exercise 2 — Ephemeron Collector

Implement:

```text
strong marking
→ weak-key checking
→ value marking
→ repeat until stable
→ sweep
```

This should be a conceptual model, not a production collector.

### Exercise 3 — WeakRef Simulator

Implement:

```text
create
deref
collect
```

and demonstrate nondeterministic availability.

### Exercise 4 — Finalization Simulator

Implement:

```text
register
unregister
collect candidate
schedule callback
```

with deliberately nondeterministic scheduling.

### Exercise 5 — Weak Memoization

Implement:

```js
memoizeObject(fn)
```

using WeakMap.

### Exercise 6 — Weak-Value Cache

Build:

```text
strong bounded key index
+
WeakRef values
+
explicit eviction
```

and explain why both mechanisms are required.

### Exercise 7 — Identity Metadata Library

Implement:

```text
attach
get
has
remove
```

without modifying the target object.

### Exercise 8 — Leak Diagnosis

Create a supposed WeakMap design that still leaks because some other data structure strongly retains the keys.

Find and remove the strong path.

### Exercise 9 — GC-Independent Test Suite

Create tests that validate:

```text
weak association API behavior
```

without asserting exact collection/finalization timing.

### Exercise 10 — Principal Architecture

Design an object-metadata subsystem for:

```text
100 million transient objects
large metadata
multiple tenants
tight memory budget
long-lived service
```

Choose among:

```text
Map
WeakMap
private fields
Symbols
external cache
bounded cache
```

and explain:

```text
ownership
memory
performance
lifecycle
security
observability
```

---

# 27. Key Takeaways

1. Weak references do not create ordinary strong reachability.
2. WeakMap supports weak object-identity associations.
3. WeakSet supports weak object-identity membership.
4. WeakRef provides nondeterministic access to a potentially collected object.
5. FinalizationRegistry provides nondeterministic cleanup notifications.
6. Weak references exist primarily to model relationships that should not control object lifetime.
7. WeakMap is ideal for many forms of external object metadata.
8. WeakRef is appropriate only when stale/missing values are acceptable.
9. Finalization is a best-effort notification mechanism, not deterministic destruction.
10. Weak collections are not enumerable in the general Map/Set sense.
11. Exact weak-entry counts are intentionally unavailable.
12. GC timing is not a portable application scheduling primitive.
13. `deref()` can produce a strong reference if the result is retained.
14. Strong aliases elsewhere can defeat an intended weak-lifetime design.
15. Weak-reference caches still require explicit capacity policy.
16. Held values in FinalizationRegistry need careful ownership analysis.
17. External resources still need explicit cleanup.
18. Weak references add semantic and implementation complexity.
19. The central principle is:

> Weak references let an application associate information with object identity without making that association the owner of the object's lifetime; they are powerful precisely because they do not provide deterministic lifetime control.

---

# 28. Concept Connections

## Depends On

- Chapter 13 — Closures
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 30 — Resource Management / Cleanup
- Chapter 39 — Concurrency / Parallelism
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations
- Chapter 43 — Ordinary Object Internal Methods
- Chapter 44 — Realms, Agents, and Execution Isolation
- Chapter 45 — JavaScript Memory and Garbage Collection

## Builds Toward

- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 108 — Cache System
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

## Related Concepts

- Strong reachability
- Weak reachability
- WeakMap
- WeakSet
- WeakRef
- FinalizationRegistry
- Finalization
- GC roots
- Reachability
- Ephemeron
- Object identity
- Metadata
- Memoization
- Cache
- Resource cleanup
- Explicit ownership
- Lifetime
- Dereference
- Unregister
- Held value
- Garbage collection
- Heap retention
- Cross-Realm identity
- Proxy identity

## Concepts Revisited

This chapter revisits:

- objects;
- Map/Set;
- garbage collection;
- reachability;
- object identity;
- memory leaks;
- caching;
- closures;
- Realms;
- Agents;
- explicit resource cleanup.

## Why This Chapter Matters Later

Chapter 45 explains:

```text
how unreachable memory becomes collectible
```

Chapter 46 explains:

```text
how JavaScript exposes limited language-level mechanisms
for relationships that should not keep objects alive
```

That distinction becomes critical for large applications.

The correct mental model is:

```text
GC
→ reclaims unreachable memory

Weak references
→ avoid creating certain strong retention paths

Finalization
→ provides nondeterministic notification

Application lifecycle
→ still owns correctness and resources
```

---

# 29. Completion Criteria

## Conceptual Understanding

- [ ] Define weak reference.
- [ ] Define strong reachability.
- [ ] Explain WeakMap.
- [ ] Explain WeakSet.
- [ ] Explain WeakRef.
- [ ] Explain FinalizationRegistry.
- [ ] Explain nondeterministic GC timing.
- [ ] Explain why weak collections are not enumerable.
- [ ] Explain why WeakMap has no deterministic size.
- [ ] Explain `deref()`.
- [ ] Explain temporary strong retention.
- [ ] Explain finalization.
- [ ] Explain held values.
- [ ] Explain unregister tokens.
- [ ] Explain ephemeron-style reasoning.
- [ ] Explain weak cache design.
- [ ] Explain explicit cleanup vs finalization.

## Predictive Mastery

- [ ] Predict WeakMap lookup while key is strongly reachable.
- [ ] Predict what changes when all strong references disappear.
- [ ] Predict WeakSet membership while object is reachable.
- [ ] Predict WeakRef while target is strongly held.
- [ ] Predict the effect of retaining `deref()` result.
- [ ] Reason correctly about finalizer timing.
- [ ] Reason about strong aliases defeating weak lifetime.
- [ ] Reason about weak-value cache growth.
- [ ] Reason about held-value retention.
- [ ] Reason about Proxy/target identity.

## Implementation

- [ ] Implement WeakMap teaching model.
- [ ] Implement weak graph simulation.
- [ ] Implement ephemeron-style marking.
- [ ] Implement WeakRef simulation.
- [ ] Implement FinalizationRegistry simulation.
- [ ] Implement weak memoization.
- [ ] Implement bounded weak-value cache.
- [ ] Implement identity metadata storage.
- [ ] Build GC-independent tests.
- [ ] Build leak-diagnosis laboratory.

## Debugging

- [ ] Diagnose accidental strong Map retention.
- [ ] Diagnose strong aliases.
- [ ] Diagnose unbounded weak-reference caches.
- [ ] Diagnose registry growth.
- [ ] Diagnose held-value retention.
- [ ] Diagnose `deref()` lifetime assumptions.
- [ ] Diagnose Proxy identity issues.
- [ ] Diagnose cross-Realm identity issues.
- [ ] Diagnose debugger-induced retention.
- [ ] Diagnose invalid GC-timing tests.

## Production Engineering

- [ ] Choose WeakMap appropriately.
- [ ] Choose WeakSet appropriately.
- [ ] Use WeakRef only for optional/loss-tolerant behavior.
- [ ] Use FinalizationRegistry only as secondary/best-effort behavior.
- [ ] Define explicit cleanup for resources.
- [ ] Define cache bounds.
- [ ] Define metadata ownership.
- [ ] Define registry lifecycle.
- [ ] Define observability.
- [ ] Define failure behavior.
- [ ] Define security implications.

## Interview Readiness

- [ ] Explain WeakMap vs Map.
- [ ] Explain WeakSet vs Set.
- [ ] Explain WeakRef.
- [ ] Explain `deref()`.
- [ ] Explain FinalizationRegistry.
- [ ] Explain nondeterminism.
- [ ] Explain ephemeron-style semantics.
- [ ] Explain weak cache pitfalls.
- [ ] Diagnose weak-reference memory bugs.
- [ ] Defend a production weak-reference architecture.

## Track A — Core Theory

- [ ] Strong vs weak reachability understood.
- [ ] Weak collections understood.
- [ ] WeakRef understood.
- [ ] Finalization understood.
- [ ] Ephemeron model understood.
- [ ] Nondeterminism understood.
- [ ] Explicit cleanup boundary understood.

## Track B — Implementation

- [ ] Guided weak-reference model completed.
- [ ] Partially guided model completed.
- [ ] No-reference model completed.
- [ ] Edge-case hardened model completed.
- [ ] Weak/cache test suite reviewed.

## Track C — Interview / Reasoning

- [ ] Prediction exercises completed.
- [ ] WeakMap debugging completed.
- [ ] Weak cache review completed.
- [ ] Finalization reasoning completed.
- [ ] GC-independent testing completed.
- [ ] Principal-level lifecycle defense completed.

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

# Chapter 46 — Revision / Retrieval Record

## Retrieval Prompts

1. What is a weak reference?
2. What is strong reachability?
3. Why does WeakMap exist?
4. Why does WeakSet exist?
5. How does WeakMap differ from Map?
6. How does WeakSet differ from Set?
7. What is WeakRef?
8. What does `deref()` return?
9. Why is `deref()` nondeterministic?
10. What is FinalizationRegistry?
11. Why is finalization nondeterministic?
12. Why is finalization not a destructor?
13. Why are weak collections not enumerable?
14. Why can't WeakMap expose deterministic size?
15. What is an ephemeron?
16. How do strong aliases defeat weak retention expectations?
17. How can a weak cache still grow?
18. What is a held value?
19. Why should held values be compact?
20. What is an unregister token?
21. When is WeakMap appropriate?
22. When is WeakRef appropriate?
23. When is FinalizationRegistry appropriate?
24. When should finalization never be used?
25. How does `deref()` interact with strong retention?
26. How can Proxy identity matter?
27. How can Realm identity matter?
28. Why should GC timing not be used in tests?
29. Why can devtools change memory observations?
30. How would you debug a supposed WeakMap memory leak?
31. How would you design identity-based metadata?
32. How would you design a weak-value cache?
33. How would you design explicit cleanup alongside finalization?

## Weak Areas

```text
-
-
-
```

## Revision Queue

```text
- [ ] Revisit strong vs weak reachability
- [ ] Revisit WeakMap
- [ ] Revisit WeakSet
- [ ] Revisit WeakRef
- [ ] Revisit deref semantics
- [ ] Revisit FinalizationRegistry
- [ ] Revisit finalization nondeterminism
- [ ] Revisit ephemeron reasoning
- [ ] Revisit weak cache design
- [ ] Revisit held values
- [ ] Revisit unregister
- [ ] Revisit identity
- [ ] Revisit explicit cleanup
- [ ] Revisit GC-independent testing
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

# Chapter 46 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — primary source for WeakMap, WeakSet, WeakRef, FinalizationRegistry, and language-level weak-reference/finalization semantics.
2. Engine/runtime documentation — GC implementation and scheduling details.
3. V8 documentation/source — V8-specific weak-reference and garbage-collection implementation details.
4. Browser/Node runtime documentation — host-specific lifecycle and memory behavior.
5. Profiling/debugging documentation — heap-retention analysis and diagnostics.

Always distinguish:

```text
weak semantic relationship
vs
engine GC implementation
vs
finalizer scheduling
vs
application lifecycle policy
```

Do not claim:

```text
object becomes unreachable
→ finalizer runs immediately
```

Do not use FinalizationRegistry as a substitute for deterministic cleanup.

Do not use WeakRef as a correctness-critical state mechanism.

Do not infer weak-reference behavior from one engine's GC timing.

---

# Chapter 46 — Completion Snapshot

```text
Chapter: 46
Title: Weak References and Finalization
Part: VIII — JavaScript Engine
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```