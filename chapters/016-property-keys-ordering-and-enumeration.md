
# Chapter 16 — Property Keys, Ordering, and Enumeration

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 02, 06, 07, 15 — Values/Types, Expressions, Coercion, and Objects/Property Semantics
>
> **Next:** Chapter 17 — Prototypes and Prototype Chains

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- explain what a JavaScript property key is;
- distinguish string property keys from symbol property keys;
- explain how numeric-looking property access becomes string-keyed property access on ordinary objects;
- distinguish integer-index-like keys from arbitrary string keys;
- explain the standardized ordering rules relevant to own-property key enumeration;
- compare `Object.keys()`, `Object.values()`, `Object.entries()`, `Object.getOwnPropertyNames()`, `Object.getOwnPropertySymbols()`, and `Reflect.ownKeys()`;
- explain how `for...in` differs from own-key enumeration;
- explain why symbol keys do not appear in `Object.keys()` or ordinary `for...in`;
- distinguish enumerable from non-enumerable properties;
- understand why property ordering became standardized and why deterministic ordering matters;
- explain why JSON serialization order should not be treated as a universal synonym for every object-enumeration API;
- reason about numeric index ordering such as `"2"`, `"10"`, and `"1"`;
- explain how insertion order interacts with string keys;
- explain the special handling of symbols in key enumeration;
- understand how inherited enumerable properties affect `for...in`;
- explain how deletion and re-addition can affect observable key order;
- understand how property order affects object spread, destructuring, copying, serialization, testing, and deterministic output;
- distinguish language-level ordering guarantees from engine implementation folklore;
- reason about proxy `ownKeys` behavior and invariants at a conceptual level;
- identify security and correctness risks caused by relying on inherited enumeration;
- implement a simplified property-key ordering model;
- debug unexpected enumeration behavior systematically;
- choose the appropriate enumeration API for a production use case;
- reason at principal level about deterministic data representations, canonicalization, and API contracts.

---

# 2. Prerequisites

You should understand:

```text
property
property descriptor
own property
inherited property
enumerability
object identity
prototype at a high level
property-key conversion
```

Chapter 15 established:

```text
object
  ↓
property
  ↓
property descriptor
  ↓
own/inherited
```

Chapter 16 adds:

```text
property
  ↓
property key
  ↓
key classification
  ↓
ordering
  ↓
enumeration
```

---

# 3. What Is a Property Key?

A property key is the identifier used to address a property of an object.

For ordinary ECMAScript property access, keys are:

```text
string
or
symbol
```

Examples:

```js
obj.name
obj["name"]
obj[42]
obj[someSymbol]
```

Even when the source code uses:

```js
42
```

as the key expression, ordinary object property-key conversion can produce the string:

```text
"42"
```

Symbols remain symbols.

---

# 4. Why Does Key Semantics Matter?

Because these are not equivalent questions:

```text
What properties exist?
Which properties are own?
Which are enumerable?
What order should they appear in?
Are symbols included?
Should inherited properties appear?
```

Different APIs answer different subsets of those questions.

For example:

```js
Object.keys(obj)
```

means approximately:

```text
own
+
string-keyed
+
enumerable
```

while:

```js
Reflect.ownKeys(obj)
```

is broader:

```text
own
+
string keys
+
symbol keys
+
non-enumerable keys
```

Choosing an enumeration API is therefore a semantic decision.

---

# 5. Property Key Categories

A useful high-level classification is:

```text
String keys
│
├── integer-index-like keys
└── other string keys

Symbol keys
```

For ordinary own-key ordering, these categories participate differently.

A common conceptual order is:

```text
1. integer-index-like string keys, ascending numerically
2. other string keys, insertion order
3. symbol keys, insertion order
```

This is the rule to internalize before memorizing examples.

---

# 6. Numeric-Looking Keys Are Still Strings

Consider:

```js
const obj = {};

obj[1] = "one";
obj["1"] = "ONE";

console.log(obj);
```

These address the same ordinary object property key:

```text
"1"
```

The second write replaces the first property value.

Therefore:

```js
obj[1]
```

and:

```js
obj["1"]
```

can refer to the same property.

---

# 7. Integer-Index-Like Ordering

Consider:

```js
const obj = {};

obj["10"] = "ten";
obj["2"] = "two";
obj["1"] = "one";
```

Then:

```js
Object.keys(obj);
```

is expected to reflect numeric ordering for these index-like keys:

```text
["1", "2", "10"]
```

not:

```text
["10", "2", "1"]
```

The source insertion order is not the deciding factor for this category.

This is one of the most commonly surprising ordering rules.

---

# 8. String Insertion Order

Now consider:

```js
const obj = {};

obj.a = 1;
obj.c = 2;
obj.b = 3;
```

The non-index string keys preserve insertion order:

```text
["a", "c", "b"]
```

This gives a useful mental model:

```text
index-like strings
→ numeric ordering

other strings
→ insertion ordering
```

---

# 9. Symbols

Consider:

```js
const first = Symbol("first");
const second = Symbol("second");

const obj = {
  [first]: 1,
  [second]: 2
};
```

Symbol-keyed properties are distinct from string keys.

They are generally not included by:

```js
Object.keys(obj)
```

or:

```js
Object.getOwnPropertyNames(obj)
```

To retrieve symbols:

```js
Object.getOwnPropertySymbols(obj);
```

To retrieve both string and symbol own keys:

```js
Reflect.ownKeys(obj);
```

---

# 10. Key Ordering Mental Model

Suppose:

```js
const s1 = Symbol("s1");
const s2 = Symbol("s2");

const obj = {
  "10": "ten",
  "2": "two",
  "a": "A",
  "b": "B",
  [s1]: 1,
  [s2]: 2
};
```

A useful conceptual `Reflect.ownKeys(obj)` ordering is:

```text
"2"
"10"
"a"
"b"
s1
s2
```

The precise ordering algorithm is specification-defined.

The essential mental model is:

```text
numeric-index-like strings
→ numeric ascending

other strings
→ insertion order

symbols
→ insertion order
```

---

# 11. Why `Object.keys()` Is Different

`Object.keys()` returns:

```text
own
string-keyed
enumerable
```

So if:

```js
const hidden = Symbol("hidden");

const obj = {
  a: 1,
  [hidden]: 2
};

Object.defineProperty(obj, "nonEnum", {
  value: 3,
  enumerable: false
});
```

then:

```js
Object.keys(obj);
```

returns only:

```text
["a"]
```

because:

```text
hidden → symbol
nonEnum → non-enumerable
```

---

# 12. `Object.values()`

`Object.values(obj)` follows the enumerable own string-key ordering corresponding to `Object.keys()`.

Example:

```js
const obj = {
  b: 2,
  a: 1
};

Object.values(obj);
```

returns values in the same key order represented by:

```js
Object.keys(obj);
```

Conceptual sequence:

```text
keys
→ ordered
→ values projected from those keys
```

---

# 13. `Object.entries()`

`Object.entries(obj)` similarly provides:

```text
[key, value]
```

pairs for enumerable own string-keyed properties.

Example:

```js
const obj = {
  a: 1,
  b: 2
};

Object.entries(obj);
```

produces:

```js
[
  ["a", 1],
  ["b", 2]
]
```

This makes it convenient for deterministic transformations.

---

# 14. `Object.getOwnPropertyNames()`

This returns:

```text
own string-keyed property names
```

including:

```text
non-enumerable string properties
```

Example:

```js
const obj = {};

Object.defineProperty(obj, "hidden", {
  value: 1,
  enumerable: false
});

Object.getOwnPropertyNames(obj);
```

includes:

```text
"hidden"
```

This differs from:

```js
Object.keys(obj)
```

---

# 15. `Object.getOwnPropertySymbols()`

This returns:

```text
own symbol keys
```

regardless of enumerability.

Example:

```js
const s = Symbol("s");

const obj = {
  [s]: 1
};

Object.getOwnPropertySymbols(obj);
```

returns an array containing:

```text
s
```

---

# 16. `Reflect.ownKeys()`

`Reflect.ownKeys(obj)` returns all own property keys:

```text
string keys
+
symbol keys
```

including non-enumerable properties.

It is often the broadest standard own-key enumeration primitive.

Conceptual result:

```text
all own keys in defined property-key order
```

---

# 17. Enumeration API Comparison

| API | Own | Inherited | String | Symbol | Non-enumerable |
|---|---:|---:|---:|---:|---:|
| `Object.keys` | Yes | No | Yes | No | No |
| `Object.values` | Yes | No | Yes | No | No |
| `Object.entries` | Yes | No | Yes | No | No |
| `Object.getOwnPropertyNames` | Yes | No | Yes | No | Yes |
| `Object.getOwnPropertySymbols` | Yes | No | No | Yes | Yes |
| `Reflect.ownKeys` | Yes | No | Yes | Yes | Yes |
| `for...in` | Own + inherited | Yes | Yes | No | No |

The table is a selection guide, not a substitute for understanding each operation's exact semantics.

---

# 18. `for...in`

`for...in` behaves differently.

Example:

```js
const parent = {
  inherited: 1
};

const child = Object.create(parent);
child.own = 2;

for (const key in child) {
  console.log(key);
}
```

The loop may observe:

```text
own
inherited
```

subject to enumerability and the iteration algorithm.

This is why:

> `for...in` is not the same as `Object.keys()`.

---

# 19. `for...in` and Prototypes

Because `for...in` traverses enumerable properties across the prototype chain, it can observe properties added to prototypes.

That can create surprising behavior in:

```js
Object.prototype
```

pollution scenarios.

Example risk pattern:

```js
for (const key in untrustedObject) {
  ...
}
```

The code may process inherited properties that were never intended to be part of the input record.

Use:

```js
Object.keys(untrustedObject)
```

when the contract is:

```text
own enumerable string keys only
```

---

# 20. `for...in` Ordering

Modern ECMAScript specifies important ordering behavior for `for...in`, but its semantics are not simply:

```text
Reflect.ownKeys(obj).filter(...)
```

The algorithm involves:

```text
enumerability
prototype traversal
duplicate suppression
visited keys
```

Therefore, do not reduce:

```js
for (const key in obj)
```

to an exact alias of:

```js
for (const key of Object.keys(obj))
```

when prototypes are involved.

---

# 21. Duplicate Suppression in `for...in`

Consider:

```js
const parent = {
  x: 1
};

const child = Object.create(parent);
child.x = 2;
```

The key:

```text
x
```

exists on both objects.

`for...in` should not simply produce duplicate `x` entries.

The enumeration process tracks keys already encountered.

Conceptually:

```text
child.x
→ x seen

prototype.x
→ x already processed
→ skip
```

This is another reason `for...in` is its own semantic operation.

---

# 22. Enumerable vs Non-Enumerable

Example:

```js
const obj = {
  visible: 1
};

Object.defineProperty(obj, "hidden", {
  value: 2,
  enumerable: false
});
```

Then:

```js
Object.keys(obj);
```

returns:

```text
["visible"]
```

while:

```js
Object.getOwnPropertyNames(obj);
```

returns both:

```text
["visible", "hidden"]
```

The property exists.

Only its enumeration visibility differs.

---

# 23. Enumerability Is a Descriptor Attribute

For data properties and accessor properties, enumeration visibility is represented by:

```text
[[Enumerable]]
```

This is not a separate object flag external to property semantics.

For example:

```js
Object.getOwnPropertyDescriptor(obj, "hidden");
```

can reveal:

```text
enumerable: false
```

---

# 24. Changing Enumerability

You can change it when property configurability permits:

```js
Object.defineProperty(obj, "x", {
  enumerable: false
});
```

The property remains present.

But operations such as:

```js
Object.keys
for...in
object spread
```

can stop observing it as an enumerable own string key.

This demonstrates how descriptors influence higher-level language features.

---

# 25. Object Spread and Enumeration

Consider:

```js
const source = {
  a: 1,
  b: 2
};

const copy = {
  ...source
};
```

Object spread copies selected own enumerable properties.

It does not:

```text
copy every own property
copy inherited properties
copy non-enumerable properties
```

Symbol-keyed enumerable own properties are also relevant to object spread.

Therefore spread should not be simplified to:

```text
Object.keys(source)
```

because that would omit enumerable symbols.

A more accurate high-level model is:

```text
own enumerable string + symbol keys
→ copy using property-definition semantics
```

---

# 26. Object Destructuring

Property selection in:

```js
const { a, b } = obj;
```

is not generic enumeration.

It directly requests specific property keys.

Thus:

```text
destructuring
```

and:

```text
Object.keys
```

solve different problems.

This distinction matters when analyzing proxies and getters.

---

# 27. Object Spread vs `Object.assign`

Both can copy enumerable own properties, but their exact semantics differ in important ways.

For example:

```js
const copy = {
  ...source
};
```

and:

```js
Object.assign(target, source);
```

both involve property reads and writes, but object spread uses object-initialization semantics while `Object.assign` writes to an existing target.

Neither is a descriptor-preserving clone.

---

# 28. Insertion Order and Deletion

Consider:

```js
const obj = {};

obj.a = 1;
obj.b = 2;
obj.c = 3;

delete obj.b;

obj.b = 4;
```

The key:

```text
b
```

is deleted and later re-added.

For ordinary non-index string keys, re-adding it can place it at the later insertion position.

Conceptual result:

```text
a
c
b
```

This matters for:

- deterministic output;
- snapshot tests;
- caches represented by objects;
- serialization expectations.

---

# 29. Updating Does Not Reinsert

Compare:

```js
const obj = {
  a: 1,
  b: 2
};

obj.a = 3;
```

Updating:

```text
a
```

does not normally move it to the end of string-key insertion order.

Result remains conceptually:

```text
a
b
```

Changing the value is different from deleting and re-adding the property.

---

# 30. Numeric Keys and Reassignment

For index-like keys:

```js
const obj = {};

obj["3"] = 3;
obj["1"] = 1;
obj["2"] = 2;
```

enumeration still reflects numeric ordering:

```text
1
2
3
```

Deleting/reinserting does not turn an index-like key into ordinary insertion-ordered behavior.

The key category matters first.

---

# 31. What Counts as an Integer Index?

Do not say:

```text
"every numeric string"
```

is an array index.

The specification has precise notions around:

```text
array index
integer index
canonical numeric string
```

These concepts are related but not identical.

For everyday object ordering, use the safe mental approximation:

```text
non-negative integer-like string property keys
→ numeric ordering
```

Then consult the formal specification for edge cases.

---

# 32. Edge Cases in Numeric-Looking Keys

Consider:

```text
"1"
"01"
"-1"
"1.0"
"4294967294"
"4294967295"
```

Do not assume they all belong to the same index-order category.

Some are treated as numeric-index-like for particular algorithms, while others are ordinary string keys.

This is one of the areas where specification-level precision matters.

---

# 33. Leading Zeros

Consider:

```js
const obj = {};

obj["01"] = "leading";
obj["1"] = "one";
obj["2"] = "two";
```

Do not assume:

```text
"01"
```

is identical to:

```text
"1"
```

as a property key.

They are different strings.

And their ordering category can differ.

---

# 34. Negative-Looking Keys

Consider:

```js
obj["-1"] = "minus";
obj["1"] = "one";
```

The key:

```text
"-1"
```

is still a string property key and is not simply treated as the non-negative index `"1"`.

This is why numeric-looking property ordering requires precise key classification.

---

# 35. Large Numeric-Looking Keys

Very large numeric-looking strings have special boundaries.

For example:

```js
obj["4294967295"] = "large";
```

should not be casually assumed to follow exactly the same ordering rule as:

```js
obj["1"]
```

The formal index definitions matter.

This is a specification-level edge case worth recognizing even if most application code never relies on it.

---

# 36. Array Indices vs Ordinary Object Keys

Arrays provide specialized indexed-element behavior.

Example:

```js
const arr = [];
arr[2] = "c";
arr[0] = "a";
arr[1] = "b";
```

Array indexing is tightly related to integer-index-like property keys.

But an array is still an object with properties.

For example:

```js
arr.extra = true;
```

adds an ordinary string-keyed property.

You therefore have:

```text
indexed elements
+
ordinary named properties
```

with distinct ordering/iteration implications.

Chapter 22 will explore arrays deeply.

---

# 37. Object Key Order Is Observable

Consider:

```js
const obj = {
  b: 2,
  a: 1,
  c: 3
};

console.log(Object.keys(obj));
```

The order is observable.

That means object property order can affect:

```text
tests
logs
serialization
hashing strategies
canonicalization
UI rendering
cache keys
diffs
snapshot comparisons
```

Therefore deterministic ordering can become an application-level concern.

---

# 38. Deterministic Output

Suppose:

```js
function serializeRecord(record) {
  return Object.entries(record)
    .map(([key, value]) => `${key}=${value}`)
    .join("&");
}
```

Property order affects the output.

If two logically equivalent records are created in different insertion orders:

```js
{ a: 1, b: 2 }
```

versus:

```js
{ b: 2, a: 1 }
```

the resulting strings may differ.

If the output is intended as a cache key or signature input, this may matter.

Do not assume:

```text
logical object equality
=
identical serialized byte sequence
```

without a canonicalization contract.

---

# 39. Object Order Is Not Deep Canonicalization

Even when own-key ordering is standardized, this does not automatically provide a canonical serialization scheme for arbitrary data.

You still need decisions about:

```text
nested objects
arrays
numbers
undefined
symbols
special values
dates
custom classes
whitespace
encoding
```

Therefore:

```text
standardized key order
```

is not equivalent to:

```text
cryptographic canonical JSON
```

---

# 40. `JSON.stringify` and Ordering

`JSON.stringify` uses its own serialization algorithm.

For ordinary object properties, its behavior is closely connected to the object's enumerable string-key ordering.

But do not state:

> JSON and every object enumeration API have identical semantics.

They do not.

Important differences include:

```text
symbols
non-enumerable properties
undefined
functions
accessors
replacers
arrays
toJSON
```

JSON serialization is its own semantic layer.

Chapter 28 will cover it deeply.

---

# 41. Symbols and JSON

Symbol-keyed properties are not emitted as normal JSON object properties.

Example:

```js
const key = Symbol("secret");

const obj = {
  visible: 1,
  [key]: 2
};

JSON.stringify(obj);
```

The symbol-keyed property is not serialized as an ordinary JSON member.

This reinforces:

```text
enumeration API
≠
serialization API
```

---

# 42. `Object.keys()` and Symbol Absence

This is intentional:

```js
Object.keys({
  [Symbol()]: 1
});
```

returns:

```text
[]
```

because `Object.keys` is specifically defined for:

```text
own enumerable string-keyed properties
```

Use:

```js
Reflect.ownKeys(obj)
```

or:

```js
Object.getOwnPropertySymbols(obj)
```

when symbols matter.

---

# 43. `Reflect.ownKeys()` as Full Own-Key View

For a diagnostic tool, the broadest own-key view is often useful:

```js
Reflect.ownKeys(obj);
```

It can reveal:

```text
enumerable strings
non-enumerable strings
symbols
```

This is especially useful when debugging:

```text
hidden properties
framework internals
library metadata
symbol-based protocols
```

---

# 44. Property Enumeration and Security

Enumeration becomes security-sensitive when dealing with untrusted objects.

Potential problems include:

```text
inherited properties
prototype pollution
unexpected getters
proxy traps
symbol metadata
non-enumerable hidden state
```

A security-sensitive input-processing API should specify:

```text
own vs inherited
string vs symbol
enumerable vs all
```

rather than simply saying:

```text
"loop through the object"
```

---

# 45. Accessors During Enumeration

Enumeration itself usually discovers keys.

But consuming the keys often reads their values:

```js
for (const key of Object.keys(obj)) {
  const value = obj[key];
}
```

If `obj[key]` is an accessor, the getter executes.

Thus:

```text
key enumeration
```

and:

```text
value retrieval
```

are separate steps.

Do not assume enumerating keys is equivalent to reading every property.

---

# 46. Proxies and `ownKeys`

A Proxy can intercept own-key queries through:

```js
ownKeys(target) { ... }
```

Example:

```js
const proxy = new Proxy(
  { a: 1 },
  {
    ownKeys(target) {
      return ["a"];
    }
  }
);
```

Now:

```js
Reflect.ownKeys(proxy)
```

consults proxy semantics.

However, proxy traps are constrained by invariants.

The proxy cannot arbitrarily report an impossible property configuration in every circumstance.

---

# 47. Proxy Key-Ordering Responsibilities

A proxy `ownKeys` trap can return keys in a chosen order.

But the surrounding operation still applies its own requirements and invariants.

Therefore:

```text
object key order
```

can become dynamically programmable through proxies.

This is another reason proxy-heavy code must be distinguished from ordinary-object assumptions.

---

# 48. Proxy Invariants

At a high level, proxy own-key results must be consistent with important target invariants involving:

```text
non-configurable properties
non-extensible targets
```

For example, a proxy cannot simply pretend a non-configurable own property does not exist.

This protects language invariants even when behavior is intercepted.

Chapter 19 will study the complete proxy machinery.

---

# 49. `Object.getOwnPropertyNames()` Ordering

This API follows the standardized own-property-key ordering for its returned string keys, while including non-enumerable string properties.

Therefore:

```js
Reflect.ownKeys(obj)
```

and:

```js
Object.getOwnPropertyNames(obj)
```

can agree on the string-key ordering while differing in symbol inclusion.

---

# 50. Symbol Ordering

If:

```js
const s1 = Symbol("s1");
const s2 = Symbol("s2");

const obj = {};

obj[s1] = 1;
obj[s2] = 2;
```

then symbol enumeration preserves their relative creation/insertion ordering in the relevant own-key ordering algorithms.

This gives:

```text
strings
→ symbol group
→ symbol insertion order
```

---

# 51. Enumerability and Spread

Consider:

```js
const hidden = Symbol("hidden");

const source = {};

Object.defineProperty(source, "secret", {
  value: 1,
  enumerable: false
});

source[hidden] = 2;

const copy = { ...source };
```

Conceptually:

```text
secret → omitted because non-enumerable
hidden → copied because enumerable symbol property
```

This is a useful demonstration that:

```text
Object.keys(source)
```

is not sufficient to model object spread.

---

# 52. Enumerability and Descriptors

For every enumeration problem, check:

```text
property exists?
own?
string/symbol?
enumerable?
```

Then choose:

```text
Object.keys
Object.getOwnPropertyNames
Object.getOwnPropertySymbols
Reflect.ownKeys
for...in
```

This simple classification solves most API-choice questions.

---

# 53. Common Misconceptions

## Misconception 1

> Object keys are always returned in insertion order.

Not exactly.

Index-like string keys have special ordering.

---

## Misconception 2

> `Object.keys()` returns all keys.

It excludes:

```text
symbols
non-enumerable properties
inherited properties
```

---

## Misconception 3

> `for...in` is just `Object.keys()`.

False when prototypes matter.

---

## Misconception 4

> Numeric keys are numbers.

Ordinary object property keys are strings or symbols.

---

## Misconception 5

> `JSON.stringify()` defines the canonical order of every object operation.

False.

---

## Misconception 6

> Spread copies every property.

False.

---

## Misconception 7

> Updating a property moves it to the end.

Generally false for ordinary string-key insertion ordering.

---

## Misconception 8

> Deleting and re-adding a property is equivalent to updating it.

False; re-addition can affect insertion position.

---

## Misconception 9

> Symbol properties are private.

False.

They are simply a different key type.

---

# 54. Common Mistakes

Avoid:

```js
for (const key in input) {
  process(input[key]);
}
```

when the requirement is:

```text
own enumerable keys only
```

Prefer:

```js
for (const key of Object.keys(input)) {
  process(input[key]);
}
```

when that contract matches the use case.

For all own keys:

```js
for (const key of Reflect.ownKeys(input)) {
  ...
}
```

For symbols only:

```js
Object.getOwnPropertySymbols(input);
```

The right choice depends on the intended property set.

---

# 55. Performance Considerations

Enumeration cost depends on:

```text
number of properties
prototype depth
proxy involvement
accessor behavior
allocation of result arrays
consumer processing
```

For large objects:

```js
Object.keys(obj)
```

allocates an array of keys.

Repeatedly enumerating large objects in hot paths can matter.

Potential strategies:

```text
reuse data structures
avoid repeated full scans
use Map for collection-like workloads
maintain indexes where appropriate
```

Measure before optimizing.

---

# 56. Memory Considerations

Potential allocations include:

```text
Object.keys → key array
Object.entries → nested pair arrays
Object.values → value array
```

For:

```js
const entries = Object.entries(hugeObject);
```

the temporary representation may be substantial.

If the workload is inherently stream-like or highly dynamic, a different data structure may be more appropriate.

---

# 57. Security Considerations

For untrusted data, prefer explicit enumeration semantics.

Potentially risky:

```js
for (const key in payload) {
  ...
}
```

Safer when only own enumerable string properties are allowed:

```js
for (const key of Object.keys(payload)) {
  ...
}
```

Also consider:

```text
getter execution
proxy traps
prototype pollution
dynamic key validation
symbol exclusion/inclusion
```

Remember that:

```js
Object.keys(payload)
```

does not make property-value access safe; `payload[key]` may still invoke accessor/proxy behavior.

---

# 58. Production Usage

Choose enumeration based on the data contract.

### Record processing

Use:

```js
Object.entries(record)
```

when you need:

```text
own enumerable string key/value pairs
```

### Full own metadata inspection

Use:

```js
Reflect.ownKeys(obj)
```

when symbol and non-enumerable properties matter.

### Prototype-aware traversal

Use:

```js
for...in
```

only when inherited enumerable properties are intentionally part of the contract.

### Dynamic dictionary

Consider:

```js
Map
```

when the data is truly a key/value collection rather than a record.

---

# 59. Deterministic Serialization Design

Suppose you need a stable cache key.

Do not immediately write:

```js
JSON.stringify(obj)
```

unless your contract explicitly permits the resulting semantics.

Instead define:

```text
key selection
ordering
value normalization
nested ordering
special values
encoding
```

Then implement canonicalization accordingly.

Object key-order rules can be one building block, but not the entire design.

---

# 60. Canonicalization Example

A simple record canonicalizer could be:

```js
function canonicalRecord(record) {
  return Object.keys(record)
    .sort()
    .map(key => [
      key,
      record[key]
    ]);
}
```

This intentionally imposes:

```text
lexicographic key ordering
```

rather than relying on source insertion order.

Why might that matter?

Because:

```text
semantic record equality
```

can then map to:

```text
deterministic key order
```

for the chosen record domain.

But:

```text
sort()
```

is an application policy, not an ECMAScript default.

---

# 61. When Not to Sort

Sorting all keys adds cost and can destroy meaningful order.

If the domain defines:

```text
insertion sequence = semantic sequence
```

then sorting is incorrect.

Examples might include:

```text
ordered configuration directives
UI ordering data
priority declarations
migration steps
```

Do not canonicalize away meaningful domain order.

---

# 62. Object vs Map Ordering

`Map` has explicitly defined iteration behavior around insertion order.

Objects have:

```text
index-like string ordering
+
other-string insertion ordering
+
symbol ordering
```

If your data model needs:

```text
arbitrary keys
and
pure insertion-order iteration
```

a `Map` can communicate that intent more directly.

Again, choose based on semantics.

---

# 63. Enumeration and Testing

Snapshot tests often expose key ordering.

Example:

```js
expect(Object.keys(obj)).toEqual([
  "a",
  "b",
  "c"
]);
```

A test that depends on ordering should document why order matters.

Otherwise, the test may accidentally couple to implementation details.

For order-insensitive data, normalize explicitly:

```js
Object.keys(obj).sort()
```

before comparing.

---

# 64. Enumeration and API Compatibility

Changing property creation order can change:

```text
Object.keys
Object.entries
spread
serialization-related output
logs
snapshots
```

Even if:

```text
the set of keys
```

remains identical.

Therefore API changes that alter insertion order can be observable.

---

# 65. Enumeration and Refactoring

Refactor:

```js
const obj = {};
obj.a = 1;
obj.b = 2;
```

into:

```js
const obj = {
  b: 2,
  a: 1
};
```

and you may change observable key order.

The data values are the same.

The enumeration order can differ.

This is a useful reminder:

> Source-level refactoring can change behavior when object property order is observable.

---

# 66. Enumeration and Object Literals

Object literal source order contributes to insertion order for ordinary non-index string keys.

Example:

```js
const obj = {
  first: 1,
  second: 2,
  third: 3
};
```

The ordinary string-key enumeration order reflects:

```text
first
second
third
```

unless index-like keys participate in their specialized ordering category.

---

# 67. Computed Properties

Example:

```js
const first = "a";
const second = "b";

const obj = {
  [first]: 1,
  [second]: 2
};
```

The computed key expressions are evaluated during object creation.

The resulting property keys participate in normal property-key ordering based on their actual keys and categories.

---

# 68. Duplicate Property Definitions

Example:

```js
const obj = {
  a: 1,
  a: 2
};
```

The final value of `a` is:

```text
2
```

The property is not simply treated as:

```text
two independent keys
```

There is one property key:

```text
"a"
```

with the final applicable property state.

Likewise, updating an existing key does not ordinarily create a new insertion position.

---

# 69. Numeric Duplicate Keys

Example:

```js
const obj = {
  1: "one",
  "1": "ONE"
};
```

Both refer to:

```text
"1"
```

The later definition determines the resulting property value.

This demonstrates again:

```text
source syntax type
≠
final property-key type
```

---

# 70. Symbols Are Unique

Even:

```js
Symbol("a") !== Symbol("a")
```

Each symbol instance is a distinct key.

Compare:

```js
const a = Symbol("x");
const b = Symbol("x");
```

Then:

```text
a !== b
```

Therefore:

```js
obj[a] = 1;
obj[b] = 2;
```

creates two distinct symbol-keyed properties.

---

# 71. Global Symbol Registry

Symbols from:

```js
Symbol.for("shared")
```

participate in the global symbol registry.

For example:

```js
const a = Symbol.for("x");
const b = Symbol.for("x");

a === b;
```

is:

```text
true
```

This matters when symbol identity must be shared across independently created code that uses the same registry key.

But registry symbols remain symbols; they do not become string property keys.

---

# 72. Property Order and Reflection

Reflective APIs expose object structure.

Important APIs:

```js
Reflect.ownKeys(obj);
Object.getOwnPropertyNames(obj);
Object.getOwnPropertySymbols(obj);
Object.getOwnPropertyDescriptor(obj, key);
```

These are valuable for:

```text
debugging
framework code
serialization tooling
metaprogramming
testing
```

Reflective code should document whether ordering itself is semantically important.

---

# 73. Specification-Oriented Vocabulary

Important terms:

```text
Property Key
String Key
Symbol Key
Integer-Index-like Key
Own Property
Enumerable Property
OwnPropertyKeys
EnumerableOwnProperties
For-In enumeration
Property Descriptor
[[OwnPropertyKeys]]
[[Enumerable]]
```

Relevant abstract operations include concepts such as:

```text
ToPropertyKey
OrdinaryOwnPropertyKeys
EnumerableOwnProperties
```

Do not memorize names without understanding the algorithms they represent.

---

# 74. `OrdinaryOwnPropertyKeys`

A particularly important specification-level operation is the conceptual source of ordinary own-key ordering.

Its broad result categories are:

```text
integer-index-like string keys
→ ascending numeric order

other string keys
→ property creation order

symbol keys
→ symbol property creation order
```

This is one of the highest-value specification concepts in the chapter.

---

# 75. `EnumerableOwnProperties`

Higher-level APIs can build on the ordered own-key model while filtering for:

```text
enumerability
string keys
values
key/value pairs
```

This helps explain why:

```js
Object.keys
Object.values
Object.entries
```

have predictable relationships.

---

# 76. `for...in` Specification Mental Model

A simplified mental model:

```text
start object
   ↓
enumerate own enumerable string keys
   ↓
track visited keys
   ↓
walk prototype
   ↓
enumerate inherited enumerable string keys
   ↓
skip duplicates
   ↓
continue until prototype chain ends
```

The actual specification algorithm has more nuance.

Use this model for reasoning, then consult the formal algorithm when implementing language tooling.

---

# 77. Proxy vs Ordinary Own Keys

For an ordinary object:

```text
OrdinaryOwnPropertyKeys
```

provides the default own-key order.

For a Proxy:

```text
ownKeys trap
```

can participate.

But proxy invariants constrain the result.

This is an important abstraction boundary:

```text
ordinary object
vs
meta-programmable object
```

---

# 78. Debugging Workflow

When enumeration order surprises you:

```text
1. What API is being used?
2. Own or inherited?
3. String or symbol?
4. Enumerable or all?
5. Are any keys integer-index-like?
6. When was each property created?
7. Was any key deleted and re-added?
8. Is a proxy involved?
9. Is the object an array or exotic object?
10. Is the consumer actually relying on order?
```

This workflow prevents many incorrect assumptions.

---

# 79. Debugging Exercise 1

Predict:

```js
const obj = {};

obj["10"] = "ten";
obj["2"] = "two";
obj["1"] = "one";
obj["a"] = "A";
obj["b"] = "B";

console.log(Object.keys(obj));
```

Explain both:

```text
result
+
ordering rule
```

---

# 80. Debugging Exercise 2

```js
const obj = {
  a: 1,
  b: 2
};

delete obj.a;
obj.a = 3;

console.log(Object.keys(obj));
```

Determine the new position of `a`.

---

# 81. Debugging Exercise 3

```js
const s = Symbol("s");

const obj = {
  a: 1,
  [s]: 2
};

console.log(Object.keys(obj));
console.log(Object.getOwnPropertySymbols(obj));
console.log(Reflect.ownKeys(obj));
```

Explain why each API returns a different view.

---

# 82. Debugging Exercise 4

```js
const parent = {
  inherited: 1
};

const child = Object.create(parent);
child.own = 2;

console.log(Object.keys(child));

for (const key in child) {
  console.log(key);
}
```

Explain the difference.

---

# 83. Debugging Exercise 5

```js
const obj = {};

Object.defineProperty(obj, "hidden", {
  value: 1,
  enumerable: false
});

console.log(Object.keys(obj));
console.log(Object.getOwnPropertyNames(obj));
```

---

# 84. Debugging Exercise 6

```js
const obj = {
  "01": "leading",
  "1": "one",
  "2": "two"
};

console.log(Object.keys(obj));
```

Do not assume:

```text
"01" = numeric index 1
```

Explain the classification.

---

# 85. Code Review Exercise

Review:

```js
function copy(input) {
  const output = {};

  for (const key in input) {
    output[key] = input[key];
  }

  return output;
}
```

Potential issues:

```text
inherits enumerable properties
does not include symbols
may trigger getters
may trigger setters on output
prototype-related risks
not descriptor-preserving
```

A better contract-specific implementation might use:

```js
Object.keys(input)
```

if only own enumerable string data is desired.

But if symbols matter, object spread or another explicit mechanism may be more appropriate.

---

# 86. Code Review Exercise — Full Reflection

Review:

```js
function inspect(obj) {
  return Object.keys(obj);
}
```

Question:

```text
Does this inspect everything?
```

No.

For deep reflection:

```js
Reflect.ownKeys(obj)
```

may be more appropriate.

The API name should match the intended inspection scope.

---

# 87. Code Review Exercise — Order Dependency

Review:

```js
function makeCacheKey(record) {
  return JSON.stringify(record);
}
```

Question:

```text
Does this produce the canonical identifier for logically equivalent records?
```

Not necessarily.

The design must decide:

```text
property ordering
nested ordering
special values
normalization
```

A canonicalization function should encode those requirements explicitly.

---

# 88. Implementation From Scratch

Build a simplified key store.

Represent properties as:

```js
{
  key,
  kind: "string" | "symbol",
  enumerable,
  creationOrder
}
```

Then implement:

```text
add property
update property
delete property
re-add property
getOwnKeys
getEnumerableKeys
```

---

# 89. Key Classifier

Implement:

```js
function classifyKey(key) {
  if (typeof key === "symbol") {
    return "symbol";
  }

  if (isIntegerIndexLike(key)) {
    return "index";
  }

  return "string";
}
```

The important lesson is the classification step.

Do not implement a weak test such as:

```js
!Number.isNaN(Number(key))
```

because that accepts many strings that do not belong to the relevant ordering category.

---

# 90. Simplified Ordering Algorithm

Use:

```js
function orderKeys(entries) {
  const indexKeys = [];
  const stringKeys = [];
  const symbolKeys = [];

  for (const entry of entries) {
    if (entry.kind === "symbol") {
      symbolKeys.push(entry);
    } else if (entry.category === "index") {
      indexKeys.push(entry);
    } else {
      stringKeys.push(entry);
    }
  }

  indexKeys.sort((a, b) => Number(a.key) - Number(b.key));

  return [
    ...indexKeys,
    ...stringKeys,
    ...symbolKeys
  ];
}
```

This is a teaching implementation.

It intentionally separates:

```text
classification
+
ordering
```

---

# 91. Simplified `Object.keys`

Implement:

```js
function simpleObjectKeys(entries) {
  return orderKeys(
    entries.filter(entry =>
      entry.kind === "string" &&
      entry.enumerable
    )
  ).map(entry => entry.key);
}
```

This captures the conceptual shape:

```text
own
+
string
+
enumerable
+
ordered
```

---

# 92. Simplified `Reflect.ownKeys`

```js
function simpleOwnKeys(entries) {
  return orderKeys(entries)
    .map(entry => entry.key);
}
```

This models:

```text
all own keys
+
string and symbol
+
ordered
```

---

# 93. Guided Implementation

Implement:

```text
isIntegerIndexLike
classifyKey
addProperty
deleteProperty
reAddProperty
orderKeys
objectKeys
ownPropertyNames
ownPropertySymbols
reflectOwnKeys
```

Tests must include:

```text
"1"
"10"
"01"
"-1"
"a"
symbols
deleted/re-added keys
non-enumerable keys
```

---

# 94. Partially Guided Implementation

Add support for:

```text
own vs inherited
enumerability
duplicate keys
symbol keys
prototype traversal
```

Then implement:

```text
simpleForIn
```

with:

```text
visited-key tracking
```

---

# 95. No-Reference Implementation

Build your own property enumeration library from memory.

Required APIs:

```js
keys(obj)
values(obj)
entries(obj)
ownPropertyNames(obj)
ownPropertySymbols(obj)
ownKeys(obj)
forIn(obj, callback)
```

Then compare each against native behavior.

---

# 96. Edge-Case Hardening

Add support for:

```text
proxy-like ownKeys interception
non-extensible objects
non-configurable properties
array-like objects
very large numeric-looking keys
symbols
getters
duplicate inherited keys
```

The goal is not to recreate the entire spec.

The goal is to learn where simplified assumptions break.

---

# 97. Production-Grade Exercise

Build a deterministic record serializer with explicit policy:

```text
allowed keys
sorting rule
symbol policy
undefined policy
nested object policy
number normalization
cycle handling
encoding
```

Then prove:

```text
same logical record
→ same canonical representation
```

within the defined domain.

---

# 98. Performance Test

Create an object with:

```text
1,000
10,000
100,000
```

own properties.

Benchmark:

```js
Object.keys
Object.entries
Object.values
Reflect.ownKeys
```

Then measure:

```text
runtime
allocation
GC pressure
```

Do not infer scalability from tiny-object benchmarks.

---

# 99. Principal-Level Enumeration Design

For a data model, decide:

```text
Is this a record?
Is order semantic?
Are symbols meaningful?
Should inherited fields count?
Should hidden fields count?
Should keys be canonicalized?
Is deterministic output required?
Would Map be clearer?
```

A principal engineer defines the contract before choosing the iteration API.

---

# 100. Interview Questions

## Beginner

1. What is a JavaScript property key?
2. What types can property keys have?
3. Why does `obj[1]` usually address the same property as `obj["1"]`?
4. What does `Object.keys()` return?
5. What does `for...in` return?

## Intermediate

6. Why do numeric-looking keys appear in numeric order?
7. What is the difference between `Object.keys` and `Object.getOwnPropertyNames`?
8. Why are symbols absent from `Object.keys()`?
9. What does `Reflect.ownKeys()` return?
10. Why can `for...in` observe inherited properties?
11. What happens when a property is deleted and re-added?

## Advanced

12. Explain the three broad categories in ordinary own-key ordering.
13. Explain why `"01"` is not simply `"1"`.
14. Explain why `Object.keys()` is not enough to model object spread.
15. Explain how `for...in` suppresses duplicate keys across the prototype chain.
16. How can proxies alter own-key enumeration?
17. Why is JSON serialization not the same thing as object-key reflection?
18. How can enumeration order affect cache keys and snapshots?

## Principal

19. How would you design a canonical representation for a record?
20. When would you choose `Map` over an object?
21. How would you audit `for...in` over untrusted input?
22. How would you design a library API that promises deterministic key order?
23. Which ordering guarantees are language semantics and which are engine folklore?
24. How would you benchmark enumeration at scale?
25. How would you decide whether property order is part of a public API contract?

---

# 101. Predict-the-Output Exercises

Predict before running.

## Exercise A

```js
const obj = {};

obj["10"] = "ten";
obj["2"] = "two";
obj["1"] = "one";

console.log(Object.keys(obj));
```

## Exercise B

```js
const obj = {};

obj.b = 2;
obj.a = 1;
obj.c = 3;

console.log(Object.keys(obj));
```

## Exercise C

```js
const obj = {};

obj.a = 1;
obj.b = 2;

delete obj.a;
obj.a = 3;

console.log(Object.keys(obj));
```

## Exercise D

```js
const s = Symbol("s");

const obj = {
  "2": "two",
  a: "A",
  [s]: "symbol"
};

console.log(Object.keys(obj));
console.log(Object.getOwnPropertySymbols(obj));
console.log(Reflect.ownKeys(obj));
```

## Exercise E

```js
const parent = {
  inherited: 1
};

const child = Object.create(parent);

child.own = 2;

console.log(Object.keys(child));

for (const key in child) {
  console.log(key);
}
```

## Exercise F

```js
const obj = {};

Object.defineProperty(obj, "hidden", {
  value: 1,
  enumerable: false
});

console.log(Object.keys(obj));
console.log(Object.getOwnPropertyNames(obj));
```

## Exercise G

```js
const obj = {
  "01": "leading",
  "1": "one",
  "2": "two"
};

console.log(Object.keys(obj));
```

## Exercise H

```js
const a = Symbol("x");
const b = Symbol("x");

const obj = {
  [a]: 1,
  [b]: 2
};

console.log(Object.getOwnPropertySymbols(obj).length);
```

---

# 102. Mastery Exercises

## Level 1 — Understand

Define:

```text
property key
string key
symbol key
enumerable
own property
index-like key
insertion order
```

---

## Level 2 — Explain

Explain why:

```js
Object.keys(obj)
```

and:

```js
Reflect.ownKeys(obj)
```

can return different sets of keys.

---

## Level 3 — Predict

Predict ordering for mixed:

```text
index-like strings
ordinary strings
symbols
```

---

## Level 4 — Implement

Build the simplified key-ordering engine.

---

## Level 5 — Debug

Diagnose:

```text
unexpected inherited property
unexpected symbol omission
unexpected numeric ordering
re-addition order change
proxy ownKeys behavior
```

---

## Level 6 — Defend

Defend:

> Object property order is standardized and observable, but the ordering model is not simply “whatever insertion order the engine happens to use.”

---

# 103. Principal-Level Reasoning Problems

## Problem 1 — Canonical cache keys

You need a cache key for:

```js
{
  userId: 42,
  role: "admin",
  enabled: true
}
```

Two clients can construct the same logical record in different property insertion orders.

Should you:

```text
trust object order
sort keys
use schema order
use a Map
hash canonical bytes
```

Defend your choice.

---

## Problem 2 — Security-sensitive iteration

An API processes:

```js
for (const key in payload) {
  process(payload[key]);
}
```

Input is attacker-controlled.

Identify:

```text
prototype risks
getter risks
proxy risks
inherited keys
unexpected symbols
```

Then specify the exact safe property set required by the API.

---

## Problem 3 — Public API ordering

A library returns:

```js
{
  first: ...,
  second: ...,
  third: ...
}
```

Consumers begin relying on:

```js
Object.keys(result)
```

being in a particular order.

Should the library now treat order as a compatibility guarantee?

Discuss:

```text
documentation
tests
semantic intent
refactoring freedom
backward compatibility
```

---

# 104. Production Design Framework

Before writing enumeration code, state:

```text
Property universe:
  own / inherited

Key types:
  string / symbol / both

Visibility:
  enumerable / all

Ordering:
  semantic / incidental / canonical

Data structure:
  object / Map

Security:
  trusted / untrusted

Runtime features:
  ordinary / proxy / exotic
```

Then select the API.

---

# 105. Enumeration API Decision Table

| Requirement | API |
|---|---|
| Own enumerable string keys | `Object.keys` |
| Own enumerable string values | `Object.values` |
| Own enumerable string entries | `Object.entries` |
| All own string keys | `Object.getOwnPropertyNames` |
| All own symbol keys | `Object.getOwnPropertySymbols` |
| All own keys | `Reflect.ownKeys` |
| Own + inherited enumerable string traversal | `for...in` |
| Arbitrary key/value collection | Consider `Map` |

---

# 106. Completion Criteria

### Understand

You can define:

- property key;
- string/symbol key;
- enumerable;
- index-like key;
- insertion order;
- own vs inherited.

### Explain

You can explain:

- numeric key ordering;
- string insertion order;
- symbol ordering;
- `Object.keys`;
- `Object.values`;
- `Object.entries`;
- `Object.getOwnPropertyNames`;
- `Object.getOwnPropertySymbols`;
- `Reflect.ownKeys`;
- `for...in`;
- object spread.

### Predict

You can correctly predict:

- mixed numeric/string key order;
- deletion/re-addition;
- hidden properties;
- symbol inclusion;
- prototype enumeration;
- duplicate inherited keys;
- proxy-aware behavior at a conceptual level.

### Implement

You can build:

```text
key classifier
ordering engine
enumerability filter
own-key reflection
prototype-aware for-in model
```

### Debug

You can identify:

```text
wrong API
wrong key category assumption
unexpected prototype key
unexpected symbol omission
wrong canonicalization assumption
```

### Principal Judgment

You can define an enumeration contract before implementation and choose:

```text
Object
Map
canonicalized representation
```

based on correctness, security, determinism, performance, and maintainability.

**Evidence of mastery:**

- 90%+ output-prediction accuracy;
- working key-ordering simulator;
- correct API selection in scenario exercises;
- successful analysis of prototype/inherited enumeration;
- explicit canonicalization policy for deterministic systems.

---

# 107. Key Takeaways

1. **Ordinary ECMAScript property keys are strings or symbols.**
2. **Numeric source syntax can become a string property key on ordinary objects.**
3. **Object key ordering is observable and standardized.**
4. **Index-like string keys use special numeric ordering.**
5. **Other string keys preserve property creation/insertion ordering.**
6. **Symbol keys form a separate ordering group and preserve their relative insertion order.**
7. **`Object.keys()` means own, enumerable, string-keyed properties.**
8. **`Object.values()` and `Object.entries()` follow the corresponding enumerable own string-key view.**
9. **`Object.getOwnPropertyNames()` includes non-enumerable own string keys.**
10. **`Object.getOwnPropertySymbols()` returns own symbol keys.**
11. **`Reflect.ownKeys()` exposes all own string and symbol keys.**
12. **`for...in` can traverse inherited enumerable string properties.**
13. **`for...in` is not merely an alias for `Object.keys()`.**
14. **Missing, non-enumerable, inherited, and symbol properties require different enumeration APIs.**
15. **Updating an existing property normally does not move it in ordinary string-key insertion order.**
16. **Deleting and re-adding a property can change its insertion position.**
17. **Index-like keys remain governed by their numeric-order category.**
18. **Property spread copies own enumerable string and symbol properties, not every property.**
19. **Enumeration and value retrieval are separate operations; accessors can execute when values are read.**
20. **Proxies can customize own-key reporting subject to language invariants.**
21. **JSON serialization is a separate semantic layer from general object reflection.**
22. **Deterministic object order is not the same thing as complete canonical serialization.**
23. **`for...in` over untrusted input can expose inherited-property and prototype-related risks.**
24. **`Map` is often a clearer choice for collection semantics requiring arbitrary keys and insertion-order iteration.**
25. **Principal-level enumeration design starts with an explicit property universe, key type, visibility, ordering, and security contract.**

---

# 108. Final Mastery Drill

For every case, answer:

```text
1. What property keys exist?
2. Which are strings?
3. Which are symbols?
4. Which are index-like?
5. Which are enumerable?
6. Which are own?
7. Which are inherited?
8. Which enumeration API is being used?
9. What ordering rule applies?
10. Can getters/proxies execute?
11. Does deletion/re-addition matter?
12. Is the consumer incorrectly relying on order?
```

### Drill 1

```js
const obj = {};

obj["10"] = 10;
obj["2"] = 2;
obj["a"] = "a";
obj["1"] = 1;
```

### Drill 2

```js
const obj = {};

obj.first = 1;
obj.second = 2;
delete obj.first;
obj.first = 3;
```

### Drill 3

```js
const symbol = Symbol("internal");

const obj = {
  visible: 1,
  [symbol]: 2
};

Object.defineProperty(obj, "hidden", {
  value: 3,
  enumerable: false
});
```

### Drill 4

```js
const parent = {
  inherited: 1
};

const child = Object.create(parent);

child.own = 2;
```

### Drill 5

```js
const obj = {
  get value() {
    return 42;
  }
};
```

Explain the difference between:

```text
enumerating "value"
```

and:

```text
reading obj.value
```

### Drill 6

```js
const proxy = new Proxy(
  { a: 1 },
  {
    ownKeys() {
      return ["a"];
    }
  }
);
```

Explain which layer controls:

```text
own-key reporting
```

and which layer still enforces:

```text
proxy invariants
```

---

# 109. Transition to Chapter 17

Chapter 15 established:

```text
What an object is.
How properties behave.
How descriptors control properties.
```

Chapter 16 established:

```text
What a property key is.
How keys are classified.
How keys are ordered.
How different APIs enumerate them.
```

Chapter 17 will connect this to:

```text
Where does a missing property come from?
How does the prototype chain work?
Why does an object inherit behavior?
How does property lookup walk prototypes?
What is Object.create() doing?
What is [[Prototype]]?
How are inherited properties different from copied properties?
Why does `instanceof` depend on prototypes?
How do class methods actually participate in prototype lookup?
```

The next conceptual progression is:

```text
object
   ↓
own properties
   ↓
missing own property
   ↓
prototype
   ↓
prototype chain
   ↓
inherited behavior
```

That is the bridge from object/property semantics to the actual inheritance model of JavaScript.

---

# 110. Chapter Completion Record

**Chapter:** 16 — Property Keys, Ordering, and Enumeration

**Status:** `[+] Completed`

**Strong Areas Expected:**

- property-key types;
- index-like ordering;
- string insertion order;
- symbol ordering;
- enumerability;
- own vs inherited;
- enumeration API selection;
- `for...in`;
- reflection;
- deterministic ordering;
- canonicalization.

**Revision Triggers:**

- saying all object keys are numbers or strings;
- saying all keys preserve insertion order;
- treating `Object.keys()` as “all properties”;
- treating `for...in` as `Object.keys()`;
- forgetting symbols;
- confusing key enumeration with value retrieval;
- assuming JSON provides a universal canonical representation;
- relying on incidental ordering without a contract.

**Evidence of Mastery:**

- 90%+ prediction accuracy;
- working key-ordering simulator;
- correct enumeration API selection;
- correct handling of symbols and non-enumerable keys;
- correct prototype-aware enumeration reasoning;
- explicit deterministic serialization policy where required.

---