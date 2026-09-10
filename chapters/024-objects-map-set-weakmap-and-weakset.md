
# Chapter 24 — Objects / Map / Set / WeakMap / WeakSet

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** Choosing and using object records, `Map`, `Set`, `WeakMap`, and `WeakSet` based on key identity, ordering, membership, iteration, garbage-collection semantics, and production requirements.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain the difference between:
   - an object used as a record,
   - an object used as a dictionary,
   - `Map`,
   - `Set`,
   - `WeakMap`,
   - `WeakSet`.
2. Explain why JavaScript has multiple key/value collection abstractions.
3. Explain String-keyed object properties versus arbitrary `Map` keys.
4. Explain Symbol-keyed object properties and how they differ from `Map` keys.
5. Explain the identity semantics of `Map` keys.
6. Explain the equality semantics used by `Map` and `Set`.
7. Explain SameValueZero and why:
   ```js
   NaN
   ```
   can be found in a `Map`/`Set`.
8. Explain why:
   ```js
   0
   ```
   and:
   ```js
   -0
   ```
   behave as the same key in `Map`/`Set`.
9. Explain insertion ordering for:
   - `Map`,
   - `Set`.
10. Explain why Object property ordering and `Map` insertion ordering are different abstractions.
11. Explain why `Object` is often better for fixed-shape records.
12. Explain why `Map` is often better for dynamic keyed collections.
13. Explain `Set` as a uniqueness/membership abstraction.
14. Explain why `WeakMap` and `WeakSet` are weak collections.
15. Explain why weak collections are not enumerable.
16. Explain why weak collections do not expose:
   - `.size`,
   - iteration,
   - key enumeration.
17. Explain the garbage-collection implications of weak references.
18. Explain the exact practical requirement for a weak collection key.
19. Understand how:
   - object identity,
   - primitive values,
   - structural equality,
   affect collection lookup.
20. Explain shallow equality versus object identity in collection keys.
21. Explain why two structurally identical objects are different `Map` keys.
22. Explain when to use:
   - `Object`
   - `Map`
   - `Set`
   - `WeakMap`
   - `WeakSet`.
23. Explain conversion patterns between:
   - Object and Map,
   - Array and Set,
   - iterable and Map/Set.
24. Explain common collection methods:
   - `Map.set`
   - `Map.get`
   - `Map.has`
   - `Map.delete`
   - `Map.clear`
   - `Map.keys`
   - `Map.values`
   - `Map.entries`
   - `Set.add`
   - `Set.has`
   - `Set.delete`
   - `Set.clear`
   - `Set.values`
   - `Set.keys`
   - `Set.entries`
25. Explain the unusual fact that:
   ```js
   set.keys() === set.values()
   ```
   for the same Set instance.
26. Explain why `Map.prototype.set()` and `Set.prototype.add()` return the collection.
27. Explain chaining behavior.
28. Explain why `Map` and `Set` are iterable.
29. Explain default iteration behavior for:
   - `Map`
   - `Set`.
30. Explain object null-prototypes:
   ```js
   Object.create(null)
   ```
   as an alternative dictionary representation.
31. Explain prototype-pollution considerations for object dictionaries.
32. Explain memory-retention differences between:
   - `Map`
   - `WeakMap`.
33. Explain the metadata pattern:
   ```text
   object → metadata
   ```
   using `WeakMap`.
34. Explain memoization with WeakMap.
35. Explain why WeakMap should not be used as a generic cache without lifecycle reasoning.
36. Explain why weak collections are intentionally difficult to observe.
37. Explain how collection semantics interact with Proxies.
38. Explain object keys versus primitive keys in `Map`.
39. Implement simplified versions of:
   - Map,
   - Set,
   - WeakMap-like metadata storage,
   where exact ECMAScript internals are not required.
40. Debug:
   - key identity mistakes,
   - object dictionary bugs,
   - accidental strong retention,
   - membership errors.
41. Compare collection abstractions under:
   - lookup,
   - insertion,
   - ordering,
   - memory,
   - security,
   - interoperability.
42. Select the right structure for production workloads.
43. Defend collection choices at principal-engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 02 — Values / Types / Type System
- Chapter 07 — Type Conversion / Coercion / Equality
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 45 — Memory / GC is a future deep-dive, but this chapter introduces the operational model needed for collection choice.

---

## 3. What Is It?

JavaScript provides several collection mechanisms because different problems require different semantics.

### Object

An Object is primarily a record abstraction:

```js
const user = {
  id: 1,
  name: "Asha",
};
```

It can also be used as a dictionary:

```js
const counts = Object.create(null);

counts.apple = 3;
counts.orange = 2;
```

But Objects are still part of the property/prototype system.

---

### Map

A `Map` is a general key/value collection:

```js
const map = new Map();

map.set("name", "Asha");
map.set(42, "answer");
map.set({}, "object key");
```

The key can be any value permitted by Map's key semantics.

---

### Set

A `Set` stores unique values:

```js
const set = new Set();

set.add(1);
set.add(1);
set.add(2);
```

Result:

```text
{1, 2}
```

---

### WeakMap

A `WeakMap` associates object keys with values without keeping those keys strongly reachable solely because of the WeakMap entry.

```js
const metadata = new WeakMap();

const object = {};

metadata.set(object, {
  createdAt: Date.now(),
});
```

If the application otherwise loses the object, the WeakMap entry does not by itself keep that object alive.

---

### WeakSet

A `WeakSet` tracks object membership weakly:

```js
const visited = new WeakSet();

visited.add(object);

visited.has(object);
```

It is useful when the membership relationship should not extend the lifetime of the object.

---

## 4. Why Does It Exist?

No single collection abstraction can optimize every semantic requirement.

### Object answers:

> "What are the named fields of this record?"

### Map answers:

> "What value is associated with this arbitrary key?"

### Set answers:

> "Has this value been seen/registered?"

### WeakMap answers:

> "What metadata belongs to this object without keeping the object alive solely through that metadata table?"

### WeakSet answers:

> "Has this object been marked, without strongly retaining it?"

This distinction is more important than memorizing method names.

---

## 5. Mental Model

### Object

```text
record
 ├── property key → value
 ├── property key → value
 └── prototype relationship
```

### Map

```text
collection
 ├── key identity → value
 ├── key identity → value
 └── insertion order
```

### Set

```text
collection
 ├── unique value
 ├── unique value
 └── insertion order
```

### WeakMap

```text
object identity
      ↓
metadata
      |
      +-- does not strongly retain key solely through entry
```

### WeakSet

```text
object identity
      ↓
membership
      |
      +-- does not strongly retain object solely through membership
```

### Core decision model

Ask:

```text
Is this a record?
Is this keyed dynamic state?
Is this uniqueness?
Is object lifetime relevant?
Should the collection keep the key alive?
```

---

## 6. Core Rules

### Rule 1 — Object property keys are String or Symbol

```js
const obj = {};

obj[1] = "x";
```

The key is represented through property-key conversion.

A Map key does not undergo the same Object property-key conversion.

---

### Rule 2 — Map keys preserve value identity/equality semantics

```js
const key = {};
const map = new Map();

map.set(key, 123);

map.get(key); // 123
```

---

### Rule 3 — Separate object literals are separate Map keys

```js
const map = new Map();

map.set({ id: 1 }, "A");

console.log(map.get({ id: 1 }));
```

Result:

```text
undefined
```

The second object is a different identity.

---

### Rule 4 — Map uses SameValueZero-style key equality

Therefore:

```js
const map = new Map();

map.set(NaN, "value");

map.get(NaN); // "value"
```

---

### Rule 5 — Set stores unique values according to its value equality semantics

```js
const set = new Set([NaN, NaN]);

set.size; // 1
```

---

### Rule 6 — `0` and `-0` are treated as the same Map/Set key

```js
const set = new Set([0, -0]);

set.size; // 1
```

---

### Rule 7 — Map preserves insertion order

```js
const map = new Map();

map.set("a", 1);
map.set("b", 2);
map.set("a", 3);
```

Iteration order is:

```text
a
b
```

Updating an existing key does not move it to the end.

---

### Rule 8 — Set preserves insertion order

```js
const set = new Set([3, 1, 2]);

[...set];
// [3, 1, 2]
```

---

### Rule 9 — Map and Set are iterable

```js
for (const entry of map) {}
for (const value of set) {}
```

---

### Rule 10 — Map's default iterator yields entries

```js
[...map]
```

produces:

```text
[[key1, value1], [key2, value2]]
```

---

### Rule 11 — Set's default iterator yields values

```js
[...set]
```

produces the unique values.

---

### Rule 12 — WeakMap keys must be objects or non-registered Symbols under the current language semantics

Do not memorize WeakMap as:

> "keys must be objects"

without checking the exact current ECMAScript version and supported key types.

The modern language allows appropriate Symbol keys while excluding registered Symbols.

For production compatibility, verify the target runtime.

---

### Rule 13 — WeakSet elements must satisfy WeakSet's weak-key requirements

They are restricted to values for which weak membership can be defined.

---

### Rule 14 — WeakMap and WeakSet are non-enumerable

There is no general:

```js
for (const entry of weakMap)
```

or:

```js
weakMap.size
```

This is intentional.

---

### Rule 15 — `Map.prototype.set()` returns the Map

```js
map.set("a", 1).set("b", 2);
```

This enables chaining.

---

### Rule 16 — `Set.prototype.add()` returns the Set

```js
set.add(1).add(2);
```

---

### Rule 17 — `set.keys()` and `set.values()` expose the same values

This supports API symmetry while preserving Set's single-value model.

---

### Rule 18 — `Object.create(null)` removes the normal object prototype

```js
const dict = Object.create(null);
```

This can be useful for dictionary-style data.

---

### Rule 19 — Objects do not have `Map`'s arbitrary-key semantics

```js
const obj = {};

obj[{}] = "value";
```

The object key goes through property-key conversion rather than retaining object identity.

---

### Rule 20 — Weak collections do not guarantee observable garbage-collection timing

Even when a key becomes unreachable elsewhere, collection behavior does not expose deterministic collection timing.

---

## 7. Syntax

### Object

```js
const object = {
  id: 1,
  name: "Asha",
};

object.name;
object["name"];
```

### Null-prototype dictionary

```js
const dict = Object.create(null);

dict.foo = 1;
```

### Map

```js
const map = new Map();

map.set(key, value);
map.get(key);
map.has(key);
map.delete(key);
map.clear();
map.size;
```

### Map iteration

```js
map.keys();
map.values();
map.entries();

for (const [key, value] of map) {
}
```

### Set

```js
const set = new Set();

set.add(value);
set.has(value);
set.delete(value);
set.clear();
set.size;
```

### Set iteration

```js
set.values();
set.keys();
set.entries();
```

### WeakMap

```js
const weakMap = new WeakMap();

weakMap.set(object, metadata);
weakMap.get(object);
weakMap.has(object);
weakMap.delete(object);
```

### WeakSet

```js
const weakSet = new WeakSet();

weakSet.add(object);
weakSet.has(object);
weakSet.delete(object);
```

---

## 8. Basic Examples

### Example 1 — Object record

```js
const user = {
  id: 1,
  name: "Milan",
};

console.log(user.name);
```

---

### Example 2 — Map with arbitrary keys

```js
const map = new Map();

const objectKey = {};

map.set(objectKey, "metadata");

console.log(map.get(objectKey));
```

---

### Example 3 — Set deduplication

```js
const values = [1, 2, 2, 3, 3, 3];

const unique = [...new Set(values)];

console.log(unique);
```

Result:

```text
[1, 2, 3]
```

---

### Example 4 — WeakMap metadata

```js
const metadata = new WeakMap();

const request = {};

metadata.set(request, {
  traceId: "abc",
});

console.log(metadata.get(request));
```

---

### Example 5 — Object versus Map key

```js
const object = {};
const map = new Map();

const key = {};

object[key] = "object";
map.set(key, "map");

console.log(object[key]);
console.log(map.get(key));
```

The Map preserves the object identity as the key.

The Object converts the key into a property key.

---

## 9. Execution Walkthrough

Consider:

```js
const key = {};
const map = new Map();

map.set(key, "value");
```

### Step 1 — Create key object

```text
key → Object(identity A)
```

### Step 2 — Create Map

A Map collection is created with no entries.

### Step 3 — `set(key, value)`

Map receives the actual key value:

```text
identity A → "value"
```

### Step 4 — Lookup

```js
map.get(key)
```

uses the same object identity.

### Step 5 — Different object

```js
map.get({})
```

creates a different object identity:

```text
identity B
```

There is no matching key.

Result:

```text
undefined
```

---

### Object comparison

```js
const key = {};
const object = {};

object[key] = "value";
```

Here the object key goes through property-key conversion.

The identity itself is not stored as the property key.

This is the central semantic difference between:

```text
Object dictionary
```

and:

```text
Map
```

---

## 10. Internal Mechanics

### 10.1 Map is an ECMAScript built-in collection

At the specification level, `Map` maintains collection state that is not exposed as ordinary Object properties.

The exact storage representation is engine-specific.

Do not model:

```js
map.set(k, v)
```

as:

```js
map[k] = v;
```

They are different semantic systems.

---

### 10.2 Map keys are values

A Map can use:

```js
map.set(object, value);
map.set(functionValue, value);
map.set(Symbol("x"), value);
map.set(42, value);
map.set(null, value);
```

subject to Map's key semantics.

---

### 10.3 SameValueZero

Map and Set lookup uses SameValueZero-style comparison.

This means:

```js
NaN
```

matches itself for collection-key purposes.

And:

```js
0
```

matches:

```js
-0
```

This differs from some historical language equality details.

---

### 10.4 Insertion order

A Map maintains an ordered collection of entries.

Setting an existing key updates its value while preserving its position.

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);

map.set("a", 3);
```

Iteration remains:

```text
a
b
```

---

### 10.5 Delete and reinsert

If:

```js
map.delete("a");
map.set("a", 4);
```

the key is inserted again.

The new entry appears at the end of insertion order.

This differs from merely updating an existing key.

---

### 10.6 Set is conceptually a unique-key collection

A Set can be understood as a collection where:

```text
value → membership
```

rather than:

```text
key → value
```

Its API still exposes iterator methods such as:

```js
keys()
values()
entries()
```

for interface compatibility.

---

### 10.7 Set entries

```js
[...new Set([1, 2])].length;
```

produces two values.

But:

```js
[...new Set([1, 2]).entries()]
```

produces:

```text
[[1, 1], [2, 2]]
```

The key/value pair contains the same value twice.

This supports generic collection-processing patterns.

---

### 10.8 WeakMap semantics

A WeakMap associates a weakly held key with a value.

The crucial property is:

```text
WeakMap entry does not by itself keep the key strongly reachable.
```

This is why:

```js
weakMap.size
```

cannot be exposed in a meaningful deterministic way.

---

### 10.9 Why WeakMap cannot be enumerated

Suppose a WeakMap allowed:

```js
[...weakMap]
```

Then iteration would reveal which keys are still alive.

Garbage collection could therefore change observable iteration results at arbitrary times.

This would create non-deterministic program-visible behavior.

The language deliberately does not expose such enumeration.

---

### 10.10 WeakMap value retention

A subtle point:

```js
weakMap.set(key, value);
```

weakens the key relationship, not necessarily the value.

If the value strongly references the key:

```text
key → value → key
```

the overall graph must still be analyzed.

Weak references do not magically break every cycle.

---

### 10.11 WeakMap metadata pattern

A common pattern:

```js
const metadata = new WeakMap();

function attachMetadata(object, data) {
  metadata.set(object, data);
}
```

The metadata lives as long as the key object remains reachable, subject to GC behavior.

This is useful when metadata should follow object lifetime automatically.

---

### 10.12 Object dictionary prototype hazards

Consider:

```js
const dict = {};

dict["toString"] = "value";
```

The dictionary begins with an inherited prototype.

This can cause:

- key collisions,
- prototype lookup surprises,
- prototype pollution hazards.

A null-prototype object can avoid inherited names:

```js
const dict = Object.create(null);
```

---

### 10.13 Map versus null-prototype object

Both can represent:

```text
key → value
```

But the semantics differ.

Object:

- property key domain is String/Symbol,
- prototype behavior exists unless removed,
- property descriptors exist,
- property order semantics apply.

Map:

- arbitrary key values,
- explicit collection API,
- direct size,
- explicit iteration,
- no prototype-key collision for stored entries.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Map data model

The specification defines Map objects with internal collection state rather than requiring a public property structure.

The exact internal slot representation is abstracted away.

---

### 11.2 `[[MapData]]`

At specification level, Map objects conceptually maintain collection entries through internal state often represented through a `[[MapData]]` internal slot.

This is not a normal user-visible property.

---

### 11.3 Map key matching

Map lookup uses a key equality mechanism based on:

```text
SameValueZero
```

This explains:

```js
NaN === NaN
```

being false while:

```js
new Set([NaN]).has(NaN)
```

is true.

---

### 11.4 Map insertion order

The specification models entries in a sequence that preserves insertion ordering.

Updates and deletions have defined effects on that sequence.

---

### 11.5 `Map.prototype.set`

The method:

```js
map.set(key, value)
```

finds an existing matching key or appends a new entry.

It returns the same Map object.

That explains chaining.

---

### 11.6 `Map.prototype.get`

Lookup returns the associated value or:

```js
undefined
```

if no matching key exists.

This means:

```js
map.get(key) === undefined
```

cannot alone distinguish:

```text
missing key
```

from:

```text
present key whose value is undefined
```

Use:

```js
map.has(key)
```

when that distinction matters.

---

### 11.7 `Map.prototype.has`

`has` directly answers the membership question.

```js
if (map.has(key)) {
  // key exists
}
```

---

### 11.8 `Map.prototype.delete`

Returns a Boolean indicating whether a matching entry was removed.

---

### 11.9 Set data model

Set is specified as a collection of unique values.

Conceptually it maintains:

```text
value entries
```

rather than key/value pairs.

---

### 11.10 Set uniqueness

When `add(value)` is called:

```text
if equivalent value already exists
    preserve membership
else
    append value
```

---

### 11.11 WeakMap key requirements

WeakMap semantics depend on weak-key eligibility.

The collection only works with values for which weak reachability can be defined according to the current specification.

The exact permitted key types should be checked against the targeted ECMAScript version and runtime.

---

### 11.12 Weak collections and garbage collection

ECMAScript deliberately abstracts over the exact garbage collector.

The specification does not expose:

```text
exact GC moment
```

as an ordinary synchronous program event.

Weak collections therefore cannot provide deterministic deletion notifications.

---

### 11.13 `ClearKeptObjects` / cleanup concepts

Weak-reference-related semantics interact with host/engine cleanup facilities.

Later chapters on:

- garbage collection,
- `WeakRef`,
- `FinalizationRegistry`,

will cover those concepts more deeply.

For this chapter, remember:

> Weak collections are designed around non-observable lifetime relationships.

---

### 11.14 Map/Set iterators

Map iterators can represent:

```text
keys
values
entries
```

Set iterators can represent:

```text
values
keys
entries
```

and are iterable themselves.

---

## 12. Advanced Behavior

### 12.1 Map with object keys

```js
const a = {};
const b = {};

const map = new Map();

map.set(a, "A");
map.set(b, "B");

console.log(map.size); // 2
```

Even if:

```text
JSON.stringify(a) === JSON.stringify(b)
```

they remain different keys.

---

### 12.2 Map with function keys

Functions are objects and can therefore be Map keys:

```js
const cache = new Map();

function compute() {}

cache.set(compute, "result");
```

This is useful for function-based memoization or metadata.

---

### 12.3 Set deduplication of objects

```js
const a = { id: 1 };
const b = { id: 1 };

const set = new Set([a, b]);

set.size; // 2
```

Structural equality is not used.

---

### 12.4 Set deduplication of primitives

```js
const set = new Set([
  1,
  1,
  "1",
  NaN,
  NaN,
]);

console.log(set.size);
```

`1` and `"1"` remain different because their values differ.

---

### 12.5 Object key coercion surprise

```js
const object = {};

object[1] = "number";
object["1"] = "string";

console.log(object["1"]);
```

Only one property exists because both property keys normalize to:

```text
"1"
```

A Map preserves the distinction:

```js
const map = new Map();

map.set(1, "number");
map.set("1", "string");
```

Now there are two keys.

---

### 12.6 Map and `undefined`

```js
const map = new Map();

map.set("x", undefined);

console.log(map.get("x"));
console.log(map.has("x"));
```

Output:

```text
undefined
true
```

This demonstrates why `has` matters.

---

### 12.7 Chaining

```js
const map = new Map()
  .set("a", 1)
  .set("b", 2);
```

and:

```js
const set = new Set()
  .add(1)
  .add(2);
```

work because the methods return the collection.

---

### 12.8 Map initialized from entries

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);
```

The constructor consumes an iterable of entry pairs.

This connects Map construction to the iterable protocol.

---

### 12.9 Set initialized from iterable

```js
const set = new Set([1, 2, 2, 3]);
```

The constructor consumes an iterable and enforces Set uniqueness.

---

### 12.10 Converting Map to Array

```js
const entries = [...map];
```

produces entry pairs.

```js
const keys = [...map.keys()];
const values = [...map.values()];
```

---

### 12.11 Converting Set to Array

```js
const values = [...set];
```

returns values in insertion order.

---

### 12.12 Map to Object

A possible conversion:

```js
const object = Object.fromEntries(map);
```

But this requires care.

Map keys are arbitrary values, while Object keys become Strings/Symbols.

For:

```js
new Map([[1, "a"], ["1", "b"]])
```

conversion to an Object creates a key collision because both become `"1"`.

---

### 12.13 Object to Map

```js
const map = new Map(Object.entries(object));
```

This captures enumerable own String-keyed properties.

It does not automatically include Symbol keys.

---

### 12.14 Symbol-keyed Object properties during conversion

If complete object-to-Map migration is required, explicitly decide how to handle:

```js
Object.getOwnPropertySymbols(object)
```

because:

```js
Object.entries(object)
```

does not include Symbol keys.

---

### 12.15 WeakMap as private-ish metadata

```js
const state = new WeakMap();

class User {
  constructor(name) {
    state.set(this, {
      name,
    });
  }

  getName() {
    return state.get(this).name;
  }
}
```

The metadata is not an ordinary enumerable property.

Unlike a Symbol property, callers cannot discover the WeakMap's keys through object reflection.

But the WeakMap itself must remain in module scope.

---

### 12.16 WeakMap versus private field

Private field:

```js
class User {
  #name;
}
```

WeakMap:

```js
const state = new WeakMap();
```

Both can support encapsulation, but their semantics and ergonomics differ.

Use private fields in modern class designs unless externalized state or specific architectural constraints justify WeakMap.

---

### 12.17 WeakSet as object marking

```js
const processed = new WeakSet();

function markProcessed(object) {
  processed.add(object);
}
```

No extra property is required on the object.

This is useful for identity-based markers whose lifecycle should follow the object.

---

### 12.18 WeakMap memoization

```js
const cache = new WeakMap();

function expensive(object) {
  if (cache.has(object)) {
    return cache.get(object);
  }

  const result = compute(object);

  cache.set(object, result);

  return result;
}
```

The cache entry does not by itself keep the object key strongly alive.

But the value may still hold significant references, so lifecycle analysis remains necessary.

---

### 12.19 WeakMap value cycles

```js
const cache = new WeakMap();

const object = {};

const value = {
  owner: object,
};

cache.set(object, value);
```

The graph contains:

```text
object → value → object
```

Weak-key semantics do not mean:

```text
object is guaranteed collectible regardless of value graph.
```

Reachability must be analyzed carefully.

---

### 12.20 Proxy keys

A Proxy is a distinct object identity from its target:

```js
const target = {};
const proxy = new Proxy(target, {});

const map = new Map();

map.set(target, "target");
map.set(proxy, "proxy");
```

These are distinct keys.

This is an important identity issue in meta-programmed systems.

---

## 13. Edge Cases

### Edge Case 1 — `NaN`

```js
const map = new Map([[NaN, "x"]]);

map.get(NaN); // "x"
```

---

### Edge Case 2 — `0` versus `-0`

```js
const map = new Map();

map.set(-0, "x");

console.log(map.get(0));
```

The keys match.

---

### Edge Case 3 — Object identity

```js
const a = {};
const b = {};

const map = new Map([[a, 1]]);

console.log(map.has(b)); // false
```

---

### Edge Case 4 — `undefined` value

```js
const map = new Map([["x", undefined]]);

console.log(map.get("x")); // undefined
console.log(map.has("x")); // true
```

---

### Edge Case 5 — Delete and re-add

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);

map.delete("a");
map.set("a", 3);
```

Iteration becomes:

```text
b
a
```

---

### Edge Case 6 — Set's keys and values

```js
const set = new Set([1, 2]);

console.log([...set.keys()]);
console.log([...set.values()]);
console.log([...set.entries()]);
```

The entries are:

```text
[1, 1]
[2, 2]
```

---

### Edge Case 7 — WeakMap primitive key

```js
const weakMap = new WeakMap();

weakMap.set(1, "x");
```

This is invalid because ordinary primitive numbers are not valid weak keys.

---

### Edge Case 8 — WeakMap enumeration

There is no supported:

```js
for (const entry of weakMap) {}
```

---

### Edge Case 9 — WeakMap size

There is no:

```js
weakMap.size
```

property.

---

### Edge Case 10 — Object prototype collision

```js
const object = {};

console.log("toString" in object); // true
```

A null-prototype dictionary avoids ordinary inherited properties:

```js
const dict = Object.create(null);

console.log("toString" in dict); // false
```

---

### Edge Case 11 — `Map` key mutation

```js
const key = {
  id: 1,
};

const map = new Map([[key, "value"]]);

key.id = 2;

console.log(map.get(key)); // "value"
```

Changing properties inside the object does not change object identity.

---

### Edge Case 12 — Immutable key identity

Using an immutable record-like representation does not make two separately created objects equal as Map keys.

Identity remains the key relation unless the application introduces structural hashing/equality itself.

---

### Edge Case 13 — Object key string collision

```js
const object = {};

object[1] = "number";
object["1"] = "string";
```

The second assignment overwrites the same property.

---

### Edge Case 14 — Map key `Symbol`

```js
const key = Symbol("x");
const map = new Map([[key, 1]]);
```

The Symbol identity is preserved exactly as a Map key.

This differs from using a Symbol property on an Object because Map's collection model is independent from object property descriptors.

---

### Edge Case 15 — `Object.fromEntries` Symbol keys

`Object.fromEntries` can create Symbol-keyed properties when the entry key is a Symbol:

```js
const symbol = Symbol("x");

const object = Object.fromEntries([
  [symbol, 123],
]);

console.log(object[symbol]);
```

This demonstrates that conversion APIs can preserve Symbol keys when explicitly given them, even though `Object.entries` does not enumerate Symbols.

---

## 14. Common Misconceptions

### Misconception 1 — "Objects are Maps."

False.

Objects use property-key semantics and prototypes.

Maps use arbitrary key values and collection semantics.

---

### Misconception 2 — "Map compares objects by structure."

False.

It compares keys by identity/equality semantics, not deep structure.

---

### Misconception 3 — "Set is just an Array without duplicates."

Not exactly.

Set provides different APIs, key equality semantics, and collection behavior.

---

### Misconception 4 — "WeakMap is just a Map with automatic deletion."

Incomplete.

The key relationship is intentionally weak and the collection is not enumerable.

Garbage collection is not directly observable through WeakMap APIs.

---

### Misconception 5 — "WeakMap can be iterated if you know the keys."

The API intentionally does not provide key enumeration.

---

### Misconception 6 — "WeakMap values are weak too."

False.

Weakness concerns the key reachability relationship.

---

### Misconception 7 — "If an object is a Map key, changing the object's properties breaks lookup."

False.

Lookup depends on object identity.

---

### Misconception 8 — "Object dictionaries are always unsafe."

Not universally.

Use:

```js
Object.create(null)
```

when an actual dictionary with String/Symbol property-key semantics is appropriate.

---

### Misconception 9 — "Map is always faster than Object."

False.

Performance depends on workload, engine, object shape, access pattern, key types, and system design.

---

### Misconception 10 — "WeakMap is a general-purpose memory optimization."

False.

Weak collections solve lifecycle/identity problems, not arbitrary performance problems.

---

### Misconception 11 — "Set guarantees sorted order."

False.

Set preserves insertion order, not sorted order.

---

### Misconception 12 — "Updating a Map key moves it to the end."

False.

Updating an existing key preserves its current position.

---

## 15. Common Mistakes

### Mistake 1 — Using Object for arbitrary object keys

```js
const cache = {};

cache[object] = result;
```

This usually loses object identity semantics.

Use:

```js
const cache = new Map();
```

when object identity is the intended key.

---

### Mistake 2 — Using `get()` alone to check membership

```js
if (map.get(key)) {
}
```

fails when the stored value is:

- `0`,
- `false`,
- `""`,
- `null`,
- `undefined`.

Use:

```js
map.has(key)
```

when checking presence.

---

### Mistake 3 — Using Set for sorted uniqueness

Set is not a sorted structure.

Sort after converting when appropriate:

```js
[...new Set(values)].sort(compare);
```

---

### Mistake 4 — Assuming WeakMap gives deterministic cleanup

It does not.

---

### Mistake 5 — Retaining a strong key elsewhere

A WeakMap cannot help if the application still holds:

```js
strongMap.set(object, metadata);
```

or another strong reference.

---

### Mistake 6 — Converting Map to Object without key-domain analysis

```js
Object.fromEntries(map);
```

can collapse distinct keys when they normalize to the same String property key.

---

### Mistake 7 — Treating WeakMap as enumerable metadata storage

You cannot later inspect all keys.

Design the ownership/debugging strategy accordingly.

---

### Mistake 8 — Using Object as a Set

Patterns like:

```js
seen[value] = true;
```

have key-coercion and prototype concerns.

A real Set is clearer:

```js
seen.add(value);
```

---

### Mistake 9 — Forgetting insertion order semantics

Map/Set order is part of their observable API behavior.

---

### Mistake 10 — Assuming memory behavior from syntax

The right collection is determined by reachability and lifecycle requirements, not by which API looks shorter.

---

## 16. Comparison With Related Concepts

| Structure | Key/value | Key domain | Ordered | Size | Iterable | Weak lifetime |
|---|---|---|---|---|---|---|
| Object | Yes | String/Symbol | Property order semantics | `Object.keys`-style | Not directly | No |
| Null-prototype Object | Yes | String/Symbol | Property order semantics | Explicit tracking needed | Not directly | No |
| Map | Yes | General values | Insertion | Yes | Yes | No |
| Set | No | Values | Insertion | Yes | Yes | No |
| WeakMap | Yes | Weak-key eligible values | Not exposed | No | No | Yes |
| WeakSet | No | Weak-key eligible values | Not exposed | No | No | Yes |

### Object versus Map

Choose Object for:

```text
record-like named data
```

Choose Map for:

```text
dynamic keyed collection
arbitrary key identity
explicit collection operations
```

---

### Map versus WeakMap

Use Map when the collection should retain the keys.

Use WeakMap when:

```text
key lifetime should not be extended by the association.
```

---

### Set versus Array

Use Set when:

```text
membership/uniqueness
```

is primary.

Use Array when:

```text
position/sequence/duplicates
```

are primary.

---

### WeakSet versus Set

Use WeakSet when object lifetime should not be extended by membership.

---

## 17. Performance Considerations

### 17.1 Do not use API identity as a performance guarantee

It is incorrect to claim:

```text
Map always faster than Object
```

or:

```text
Set always faster than Array
```

without a workload.

Modern engines optimize multiple collection forms aggressively.

---

### 17.2 Lookup complexity

Hash-table-like behavior is common in Map/Set implementations, but the ECMAScript specification does not promise a specific internal data structure.

Reason about expected operation cost, then benchmark critical workloads.

---

### 17.3 Object shapes

Objects with stable shapes can be highly optimized by JavaScript engines.

This is one reason record-style objects remain excellent for structured application data.

---

### 17.4 Map for dynamic keys

Map can be preferable when keys are:

- dynamic,
- numerous,
- heterogeneous,
- object identities.

Its explicit collection API can also make intent clearer.

---

### 17.5 Set for membership

Set avoids manual dictionary-style membership logic:

```js
seen[value] = true;
```

and expresses the semantic intent directly.

---

### 17.6 WeakMap overhead

WeakMap has semantics and implementation constraints around weak reachability.

Do not assume:

```text
WeakMap = faster Map
```

It is primarily a lifecycle tool.

---

### 17.7 Memory behavior

A Map strongly retains its keys and values through its entries.

A WeakMap does not strongly retain eligible keys solely through the weak association.

For caches and metadata, this distinction can be more important than raw lookup speed.

---

## 18. Memory Considerations

### Strong retention

```js
const map = new Map();

const object = {};
map.set(object, metadata);
```

The Map entry strongly participates in keeping the key and value reachable.

If the Map itself stays alive, the key does too.

---

### Weak retention

```js
const weakMap = new WeakMap();

weakMap.set(object, metadata);
```

The key does not remain strongly reachable solely because of the WeakMap entry.

---

### Values can retain keys

Always inspect the graph:

```text
WeakMap
  ↓
key → value
       ↓
      key
```

Do not interpret "weak key" as:

> all memory associated with the entry disappears automatically.

---

### Large Sets

Sets strongly retain their values.

A long-lived Set used as a "seen forever" structure can become a memory leak if its domain grows without bound.

Use lifecycle or boundedness.

---

### Map caches

An unbounded Map cache can become:

```text
correct but operationally unsafe
```

because old keys/values remain strongly retained.

Use eviction policies when needed.

---

## 19. Security Considerations

### Object dictionary prototype pollution

If untrusted keys enter an ordinary Object dictionary:

```js
const dict = {};

dict[userInput] = value;
```

prototype-related keys can create security and correctness issues.

Prefer:

```js
Object.create(null)
```

or:

```js
new Map()
```

when appropriate.

---

### Map does not magically validate keys

A Map prevents String coercion collisions, but application logic still needs validation.

---

### Weak collections and observability

Their non-enumerability can improve encapsulation, but also makes debugging and auditing harder.

Security-sensitive metadata should have explicit diagnostics where safe.

---

### Object identity from untrusted proxies

An attacker can supply distinct wrapper objects:

```js
target
proxy1
proxy2
```

which may all represent the same underlying conceptual resource but remain distinct identities.

Do not confuse object identity with domain identity.

---

### Set membership with attacker-controlled growth

An unbounded Set can be abused for memory exhaustion.

Membership structures need lifecycle and size limits when inputs are attacker-controlled.

---

## 20. Production Usage

### Use case 1 — API response records

Use an Object:

```js
const response = {
  id: 123,
  status: "ok",
  data: payload,
};
```

---

### Use case 2 — Dynamic in-memory index

```js
const usersById = new Map();

usersById.set(user.id, user);
```

Use Map when the collection's lifecycle and key semantics are dynamic.

---

### Use case 3 — Deduplication

```js
const uniqueEmails = [...new Set(emails)];
```

---

### Use case 4 — Object metadata

```js
const metadata = new WeakMap();

metadata.set(request, {
  traceId,
  startTime,
});
```

This is useful when metadata should follow object lifetime.

---

### Use case 5 — Cycle detection

```js
const visited = new WeakSet();

function walk(node) {
  if (visited.has(node)) {
    return;
  }

  visited.add(node);

  // traverse
}
```

This prevents object-graph cycles without permanently retaining every visited node after traversal state becomes unreachable.

---

### Use case 6 — Memoization by object identity

```js
const cache = new WeakMap();

function expensive(object) {
  const cached = cache.get(object);

  if (cached !== undefined) {
    return cached;
  }

  const result = compute(object);
  cache.set(object, result);

  return result;
}
```

Be careful when `undefined` is a legitimate cached value; use `has()` as needed.

---

### Use case 7 — Dictionary with untrusted String keys

```js
const dictionary = Object.create(null);

dictionary[userKey] = value;
```

or use:

```js
const dictionary = new Map();
```

depending on key and API requirements.

---

### Use case 8 — Reverse index

```js
const index = new Map();

for (const user of users) {
  index.set(user.email, user.id);
}
```

---

### Use case 9 — Grouping keys

A Map can represent:

```text
group key → array of members
```

```js
const groups = new Map();

for (const item of items) {
  const key = item.category;

  if (!groups.has(key)) {
    groups.set(key, []);
  }

  groups.get(key).push(item);
}
```

For modern environments, dedicated grouping APIs may offer more expressive alternatives, but understanding the underlying Map model remains essential.

---

### Production rule

Choose:

```text
Object
Map
Set
WeakMap
WeakSet
```

by answering:

1. What is the key/value domain?
2. Do I need insertion order?
3. Do I need explicit iteration?
4. Do I need arbitrary key identity?
5. Should the collection retain keys?
6. Do I need uniqueness?
7. Is this record data or collection data?
8. What are the security constraints?
9. What is the lifetime?
10. What happens as the collection grows?

---

## 21. Implementation From Scratch

### 21.1 MiniMap

Educational implementation:

```js
class MiniMap {
  #entries = [];

  set(key, value) {
    const index = this.#find(key);

    if (index === -1) {
      this.#entries.push([key, value]);
    } else {
      this.#entries[index][1] = value;
    }

    return this;
  }

  get(key) {
    const index = this.#find(key);

    return index === -1
      ? undefined
      : this.#entries[index][1];
  }

  has(key) {
    return this.#find(key) !== -1;
  }

  delete(key) {
    const index = this.#find(key);

    if (index === -1) {
      return false;
    }

    this.#entries.splice(index, 1);
    return true;
  }

  clear() {
    this.#entries.length = 0;
  }

  #find(key) {
    for (let i = 0; i < this.#entries.length; i++) {
      if (sameValueZero(this.#entries[i][0], key)) {
        return i;
      }
    }

    return -1;
  }

  *[Symbol.iterator]() {
    yield* this.#entries;
  }
}

function sameValueZero(a, b) {
  return a === b || (a !== a && b !== b);
}
```

This is intentionally O(n) and educational.

A real Map implementation uses engine-level data structures.

---

### 21.2 MiniSet

```js
class MiniSet {
  #values = [];

  add(value) {
    if (!this.has(value)) {
      this.#values.push(value);
    }

    return this;
  }

  has(value) {
    return this.#values.some(
      current => sameValueZero(current, value),
    );
  }

  delete(value) {
    const index = this.#values.findIndex(
      current => sameValueZero(current, value),
    );

    if (index === -1) {
      return false;
    }

    this.#values.splice(index, 1);
    return true;
  }

  get size() {
    return this.#values.length;
  }

  *values() {
    yield* this.#values;
  }

  *[Symbol.iterator]() {
    yield* this.values();
  }
}
```

---

### 21.3 WeakMap-like architecture

A true WeakMap cannot be implemented correctly in ordinary JavaScript using only:

```js
Map
```

because a Map would strongly retain its keys.

An educational approximation can demonstrate the API:

```js
class PretendWeakMap {
  #map = new Map();

  set(key, value) {
    this.#assertObject(key);
    this.#map.set(key, value);
    return this;
  }

  get(key) {
    return this.#map.get(key);
  }

  has(key) {
    return this.#map.has(key);
  }

  delete(key) {
    return this.#map.delete(key);
  }

  #assertObject(key) {
    if ((typeof key !== "object" || key === null) &&
        typeof key !== "function") {
      throw new TypeError("Invalid weak key");
    }
  }
}
```

This is **not weak**.

That limitation is the entire lesson.

---

### 21.4 Production metadata pattern

Prefer the actual platform WeakMap:

```js
const privateState = new WeakMap();

export class Service {
  constructor(config) {
    privateState.set(this, {
      config,
    });
  }

  run() {
    const { config } = privateState.get(this);
    return perform(config);
  }
}
```

Then compare with private fields before choosing.

---

### Implementation progression

**Guided**

- MiniMap
- MiniSet
- object/null-prototype dictionary

**Partially Guided**

- Map-backed memoization
- WeakMap metadata

**No Reference**

- Build a collection abstraction with explicit key/value policies.

**Edge-Case Hardened**

Support:

- `NaN`,
- `-0`,
- object identity,
- deletion/reinsertion order,
- undefined values.

**Production-Grade**

Add:

- benchmarks,
- memory tests,
- bounded caches,
- security validation,
- observability,
- API contracts.

---

## 22. Debugging Exercises

### Exercise 1 — Object key collision

```js
const cache = {};

const a = {};
const b = {};

cache[a] = "A";
cache[b] = "B";

console.log(cache);
```

Explain why object identity is not preserved as intended.

---

### Exercise 2 — Map identity

```js
const map = new Map();

map.set({}, "value");

console.log(map.get({}));
```

Explain the result.

---

### Exercise 3 — `undefined` membership bug

```js
const map = new Map();

map.set("x", undefined);

if (!map.get("x")) {
  console.log("missing");
}
```

Find the semantic mistake.

---

### Exercise 4 — Unbounded Set

```js
const seen = new Set();

for (const id of incomingIds) {
  seen.add(id);
}
```

Identify the potential memory-growth issue.

---

### Exercise 5 — WeakMap misconception

```js
const cache = new WeakMap();

const object = {};
cache.set(object, hugeResult);
```

Explain why this does not mean `hugeResult` is automatically weak.

---

### Exercise 6 — Prototype pollution

```js
const dict = {};

for (const [key, value] of entries) {
  dict[key] = value;
}
```

Identify the security/correctness concern.

---

### Exercise 7 — Conversion collision

```js
const map = new Map([
  [1, "number"],
  ["1", "string"],
]);

const object = Object.fromEntries(map);

console.log(object);
```

Explain why information can be lost.

---

### Exercise 8 — Proxy identity

```js
const target = {};
const proxy = new Proxy(target, {});

const set = new Set([target]);

console.log(set.has(proxy));
```

Explain the result.

---

## 23. Code Review Exercise

Review:

```js
class Cache {
  constructor() {
    this.data = {};
  }

  set(key, value) {
    this.data[key] = value;
  }

  get(key) {
    return this.data[key];
  }
}

const cache = new Cache();
```

### Questions

1. What types of keys are expected?
2. Is object identity required?
3. Could attacker-controlled keys interact with the prototype?
4. Can stored `undefined` be distinguished from absence?
5. Is `Map` a better fit?
6. Does the cache need eviction?
7. Should it be bounded?
8. What are the memory-retention semantics?
9. Does insertion order matter?
10. How should cache metrics be exposed?

Now review an alternative:

```js
class Cache {
  #data = new Map();

  set(key, value) {
    this.#data.set(key, value);
    return this;
  }

  get(key) {
    return this.#data.get(key);
  }

  has(key) {
    return this.#data.has(key);
  }
}
```

Explain which semantic requirements this design handles more directly.

---

## 24. Interview Questions

### Fundamentals

1. What is the difference between Object and Map?
2. What is Set?
3. What is WeakMap?
4. What is WeakSet?
5. Why are there multiple collection types?
6. What is the difference between a record and a collection?
7. Can an Object use an object as a true identity-based key?
8. What key types can Object properties use?
9. What key types can Map use?
10. Does Set preserve insertion order?

### Equality and identity

11. How are Map keys compared?
12. What is SameValueZero?
13. Why does a Map find `NaN`?
14. Why are `0` and `-0` treated as the same Map key?
15. Why are two identical-looking objects different Set values?
16. Why does mutating an object key not change its Map identity?
17. How do Proxy and target identities behave as Map keys?

### API behavior

18. Why does `Map.set()` return the Map?
19. Why does `Set.add()` return the Set?
20. What does `Map.get()` return for a missing key?
21. How do you distinguish a missing key from an `undefined` value?
22. What does `Map.delete()` return?
23. What do `Set.keys()`, `Set.values()`, and `Set.entries()` return?
24. What does the default Map iterator yield?

### Weak collections

25. Why are WeakMaps not iterable?
26. Why is `weakMap.size` unavailable?
27. What does weak reachability mean?
28. Does WeakMap make the value weak?
29. Can a WeakMap contain a value that references its key?
30. Why can't WeakMap provide deterministic cleanup notifications?
31. When should WeakMap be used for metadata?
32. When should private fields replace WeakMap?

### Object dictionaries

33. What problem does `Object.create(null)` solve?
34. How can prototype pollution affect object dictionaries?
35. What are the differences between null-prototype Objects and Map?

### Performance

36. Is Map always faster than Object?
37. Is Set always faster than Array?
38. What makes a benchmark of collections difficult?
39. How can unbounded Map/Set collections become memory problems?

### Principal-level

40. Choose Object, Map, or Set for a configuration/indexing problem and defend it.
41. Design an identity-based cache with correct lifecycle semantics.
42. Design metadata storage that does not retain ephemeral objects.
43. Decide whether a cache should use Map or WeakMap.
44. What observability would you add to a weakly keyed subsystem?
45. How would you handle attacker-controlled collection growth?
46. How would you migrate an Object dictionary to Map without breaking consumers?
47. What semantic information can be lost when converting Map to Object?
48. How would you test a collection abstraction for equality edge cases?
49. When would `Object.create(null)` be preferable to Map?
50. When would Set be the wrong choice for deduplication?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
const map = new Map();

map.set(NaN, "x");

console.log(map.get(NaN));
```

---

### Exercise B

```js
const set = new Set([0, -0]);

console.log(set.size);
```

---

### Exercise C

```js
const a = {};
const b = {};

const map = new Map([[a, "A"]]);

console.log(map.has(a));
console.log(map.has(b));
```

---

### Exercise D

```js
const map = new Map();

map.set("x", undefined);

console.log(map.get("x"));
console.log(map.has("x"));
```

---

### Exercise E

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);

map.set("a", 3);

console.log([...map.keys()]);
```

---

### Exercise F

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);

map.delete("a");
map.set("a", 3);

console.log([...map.keys()]);
```

---

### Exercise G

```js
const set = new Set([1, 2]);

console.log([...set.keys()]);
console.log([...set.values()]);
console.log([...set.entries()]);
```

---

### Exercise H

```js
const object = {};

const key = {};

object[key] = "value";

console.log(object[key]);
console.log(object[{}]);
```

Predict both carefully.

---

### Exercise I

```js
const map = new Map([
  [1, "number"],
  ["1", "string"],
]);

console.log(map.size);

const object = Object.fromEntries(map);

console.log(Object.keys(object));
```

---

### Exercise J

```js
const target = {};
const proxy = new Proxy(target, {});

const set = new Set([target]);

console.log(set.has(proxy));
```

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
Object
Map
Set
WeakMap
WeakSet
```

as five different semantic tools.

### Level 2 — Explain

Explain why:

```js
object[{}]
```

does not preserve object identity as a key.

### Level 3 — Predict

Predict Map/Set behavior for:

```text
NaN
0
-0
undefined
objects
functions
Proxies
Symbols
```

### Level 4 — Implement

Build:

```text
MiniMap
MiniSet
object dictionary
identity cache
```

### Level 5 — Debug

Fix:

- object-key collision,
- prototype pollution,
- missing-vs-undefined bug,
- unbounded Set growth,
- accidental strong retention.

### Level 6 — Compare

Compare:

```text
Object
Object.create(null)
Map
Set
WeakMap
WeakSet
```

using:

- key domain,
- identity,
- order,
- iteration,
- lifetime,
- memory,
- security,
- API ergonomics.

### Level 7 — Apply

Design a:

```text
request metadata system
```

where metadata must disappear naturally with the request object.

### Level 8 — Defend

Choose between:

```text
private fields
WeakMap
Symbol property
ordinary property
```

for internal state and defend the trade-offs.

### Level 9 — Principal Judgment

Design a production caching architecture that answers:

```text
What is the key?
Who owns the key?
How long should it live?
How many entries are allowed?
Can values retain keys?
How is eviction performed?
How is cache state observed?
What happens under attacker-controlled growth?
```

---

## 27. Key Takeaways

1. Object, Map, Set, WeakMap, and WeakSet solve different semantic problems.
2. Objects are primarily record/property abstractions.
3. Objects use String/Symbol property keys.
4. Map supports general key values and explicit collection semantics.
5. Map preserves insertion order.
6. Set preserves insertion order and uniqueness.
7. Map and Set use SameValueZero-style equality.
8. `NaN` can be a Map/Set key or member.
9. `0` and `-0` are treated as the same Map/Set key.
10. Object identity matters for object Map/Set keys.
11. Two structurally identical objects are still different identities.
12. `Map.get()` returning `undefined` does not prove absence.
13. `Map.has()` distinguishes missing keys from stored `undefined`.
14. Updating an existing Map key does not move it.
15. Delete + reinsert places a key at the end.
16. Set's `keys()` and `values()` expose the same values.
17. `Set.entries()` yields `[value, value]`.
18. WeakMap and WeakSet use weak lifetime semantics.
19. Weak collections are intentionally non-enumerable.
20. WeakMap has no deterministic cleanup API.
21. Weakness applies to key reachability, not automatically to values.
22. WeakMap is useful for object-associated metadata and identity-based memoization.
23. `Object.create(null)` is useful for dictionary-style objects without inherited prototype properties.
24. Map does not automatically make an application secure.
25. Object dictionaries require careful key/prototype handling.
26. Unbounded Map/Set growth can create memory problems.
27. Map is not automatically faster than Object.
28. WeakMap is not simply a faster Map.
29. Conversion between Object and Map can lose key-domain information.
30. Principal-level collection design is primarily about semantics, lifecycle, and operational constraints.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values / Types
- Chapter 07 — Equality / Coercion
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering
- Chapter 17 — Prototypes
- Chapter 19 — Proxy / Reflect
- Chapter 20 — Symbols
- Chapter 22 — Arrays
- Chapter 23 — Strings

### Builds Toward

- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 45 — Memory / Garbage Collection
- Chapter 46 — Weak References / Finalization
- Chapter 48 — V8 Internals / Optimization
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 78 — Production JS Architecture
- Chapter 81 — Database Integration
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios

### Related Concepts

- object identity
- SameValueZero
- property keys
- insertion order
- iterables
- weak reachability
- garbage collection
- memoization
- caching
- prototype pollution
- records versus collections
- lifecycle management

### Concepts Revisited

**Chapter 15:**  
Objects show the difference between property semantics and collection semantics.

**Chapter 16:**  
Property ordering and key enumeration explain why Object ordering is not identical to Map insertion ordering.

**Chapter 17:**  
Prototypes explain object dictionary hazards and null-prototype designs.

**Chapter 19:**  
Proxy identity demonstrates that a proxy and its target can be distinct collection keys.

**Chapter 20:**  
Symbols can be Object keys and Map keys, but those roles have different collection semantics.

**Chapter 22:**  
Arrays remain sequence abstractions, while Set/Map target different collection problems.

**Chapter 23:**  
String keys may be text data, but Object property-key conversion treats them as property identifiers, not user-language strings.

### Why This Chapter Matters Later

Production systems constantly need:

```text
records
indexes
membership
metadata
caches
deduplication
memoization
```

Choosing the wrong collection can create:

- correctness bugs,
- prototype pollution,
- accidental retention,
- memory leaks,
- hidden key collisions,
- poor API design.

The goal is not:

> "Know all Map methods."

The goal is:

> "Recognize the semantic shape of the problem and select the collection whose guarantees match it."

---

## Track A — Core Theory

### Level 1 — Intuition

```text
Object   → record/properties
Map      → key/value collection
Set      → unique membership
WeakMap  → weak object-associated state
WeakSet  → weak object membership
```

### Level 2 — Syntax

Know all core creation, mutation, lookup, deletion, and iteration APIs.

### Level 3 — Practical

Build:

- indexes,
- sets,
- memoization,
- metadata,
- dictionaries.

### Level 4 — Edge Cases

Understand:

- object identity,
- `NaN`,
- `0/-0`,
- undefined,
- deletion/reinsertion,
- prototype keys,
- proxies,
- conversion loss.

### Level 5 — Runtime/Internal

Understand:

- abstract collection state,
- strong versus weak reachability,
- iterator objects,
- engine-specific storage strategies.

### Level 6 — Specification Semantics

Be comfortable with:

- Map/Set collection data,
- SameValueZero,
- insertion order,
- weak-key rules,
- non-enumerability rationale.

### Level 7 — Performance/Security

Reason about:

- lookup behavior,
- engine specialization,
- unbounded growth,
- prototype pollution,
- retention,
- attacker-controlled collections.

### Level 8 — Production Engineering

Design:

- caches,
- indexes,
- metadata systems,
- bounded membership stores,
- migration strategies.

### Level 9 — Interview/Reasoning

Answer:

> Why is `Map` a better abstraction than Object for a dynamic identity-based index?

### Level 10 — Principal Judgment

Evaluate:

> Should this subsystem use Object, Map, Set, WeakMap, or WeakSet?

Defend the choice using:

```text
semantic key domain
lifecycle
order
memory
security
observability
scale
future changes
```

---

## Track B — Implementation

The implementation ladder is:

```text
1. Object record
        ↓
2. Null-prototype dictionary
        ↓
3. MiniMap
        ↓
4. MiniSet
        ↓
5. Identity cache
        ↓
6. WeakMap metadata
        ↓
7. Cycle detection
        ↓
8. Bounded cache
        ↓
9. Production collection abstraction
        ↓
10. Lifecycle-aware collection architecture
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain:

```text
Object key coercion
vs
Map key identity
```

### Drill 2

Explain:

```js
map.get(k) === undefined
```

is not enough to prove:

```text
key is absent
```

### Drill 3

Explain why WeakMap cannot expose deterministic enumeration.

### Drill 4

Choose between:

```text
Object.create(null)
Map
```

for a dictionary with untrusted String keys.

### Drill 5

Choose between:

```text
Map
WeakMap
```

for an object-keyed cache and defend the lifecycle semantics.

### Drill 6

Design a deduplication system and decide between:

```text
Set
Map
Array + indexOf
```

based on domain scale and identity requirements.

---

## 29. Completion Criteria

Mark Chapter 24 `[+] Completed` only when the learner can:

- [ ] Explain Object as a record/property abstraction.
- [ ] Explain Object dictionary use.
- [ ] Explain Map.
- [ ] Explain Set.
- [ ] Explain WeakMap.
- [ ] Explain WeakSet.
- [ ] Explain Object property-key conversion.
- [ ] Explain arbitrary Map key values.
- [ ] Explain object identity as a Map/Set key.
- [ ] Explain SameValueZero.
- [ ] Predict `NaN` behavior.
- [ ] Predict `0/-0` behavior.
- [ ] Explain insertion order.
- [ ] Explain update versus delete/reinsert ordering.
- [ ] Explain Map iteration.
- [ ] Explain Set iteration.
- [ ] Explain Set `entries()`.
- [ ] Explain `Map.has()` versus `Map.get()`.
- [ ] Explain weak reachability.
- [ ] Explain why WeakMap/WeakSet are non-enumerable.
- [ ] Explain why WeakMap has no deterministic cleanup mechanism.
- [ ] Explain weak-key versus value retention.
- [ ] Explain `Object.create(null)`.
- [ ] Explain prototype-pollution risks.
- [ ] Explain Map/Object conversion hazards.
- [ ] Explain Proxy/target identity.
- [ ] Implement MiniMap.
- [ ] Implement MiniSet.
- [ ] Design WeakMap metadata.
- [ ] Debug collection memory and identity bugs.
- [ ] Compare all five abstractions.
- [ ] Choose an appropriate production collection.

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

> Given a real production requirement, how do you choose among Object, `Object.create(null)`, Map, Set, WeakMap, and WeakSet while preserving correctness, lifecycle, security, observability, and operational safety?

---

## Chapter 24 Retrieval Set

### Retrieval 1

Why does:

```js
obj[{}]
```

not preserve object identity the way:

```js
map.set({}, value)
```

does?

### Retrieval 2

What equality model is used by Map and Set?

### Retrieval 3

Why does a Map find `NaN` when strict equality does not?

### Retrieval 4

Why does:

```js
map.set("x", undefined)
```

require `map.has("x")` to distinguish presence?

### Retrieval 5

What happens to Map insertion order after updating an existing key?

### Retrieval 6

What happens after deleting and re-adding a key?

### Retrieval 7

Why can a WeakMap not be safely enumerable?

### Retrieval 8

What does WeakMap's "weak" relationship actually mean?

### Retrieval 9

When would `Object.create(null)` be preferable to an ordinary Object?

### Retrieval 10

When would you choose Map over Object for a production index?

### Retrieval 11

What information can be lost when converting:

```js
Map → Object
```

### Retrieval 12

Why can a WeakMap still participate in a memory-retention cycle through its values?

---

## Chapter 24 Final Mental Model

Remember:

```text
                 COLLECTION CHOICE
                         |
        +----------------+----------------+
        |                |                |
      Object            Map              Set
        |                |                |
     record          key → value       unique values
        |
  String/Symbol keys
```

And:

```text
               WEAK COLLECTIONS
                    |
             +------+------+
             |             |
          WeakMap       WeakSet
             |             |
       object → data    object ∈ set
             |             |
       weak key life    weak member life
```

Equality:

```text
Map / Set
    ↓
SameValueZero-style matching
    ↓
NaN matches NaN
0 matches -0
object identity matters
```

Lifecycle:

```text
Map / Set
    ↓
strong retention

WeakMap / WeakSet
    ↓
weak association
    ↓
not enumerable
```

Finally:

> The correct collection is determined by the semantics of the problem, not by which data structure is fashionable or supposedly faster. Decide whether the problem is a record, key/value index, uniqueness set, or lifecycle-bound metadata relation—and then choose the structure whose guarantees make the intended behavior the default.
