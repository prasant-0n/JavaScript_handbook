
# Chapter 28 — JSON / Serialization / Structured Clone

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** Turning JavaScript values into transferable representations, JSON's data model and limitations, `JSON.stringify` / `JSON.parse`, structured cloning, transferables, cycles, prototypes, identity, serialization boundaries, and production-safe data interchange.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what serialization means.
2. Distinguish:
   - serialization,
   - encoding,
   - cloning,
   - parsing,
   - deserialization.
3. Explain what JSON is.
4. Explain the difference between:
   - the JSON data model,
   - JavaScript object values.
5. Explain why JSON is not "JavaScript objects written as text."
6. Explain the core JSON value types:
   - object,
   - array,
   - string,
   - number,
   - boolean,
   - null.
7. Explain which JavaScript values JSON cannot represent directly.
8. Explain the behavior of:
   - `undefined`
   - functions
   - Symbols
   - BigInt
   - `NaN`
   - `Infinity`
   - `-Infinity`
   - Dates
   - RegExp
   - Map
   - Set
   - typed arrays
   - ArrayBuffer.
9. Explain how `JSON.stringify()` handles:
   - object properties,
   - array elements,
   - unsupported values,
   - `toJSON()`,
   - replacers,
   - spacing.
10. Explain how `JSON.parse()` reconstructs JSON values.
11. Explain `reviver` semantics.
12. Explain why `JSON.stringify(JSON.parse(x))` is not necessarily identity-preserving.
13. Explain why JSON serialization loses:
    - prototypes,
    - methods,
    - object identity,
    - many special built-in types,
    - `undefined` in object properties,
    - Symbol properties.
14. Explain how cycles affect `JSON.stringify`.
15. Explain why:
    ```js
    JSON.stringify(circularObject)
    ```
    throws.
16. Explain the difference between:
    - omitted object property,
    - `null`,
    - omitted array element replacement by `null`.
17. Explain `toJSON()` hooks.
18. Explain deterministic/key-order implications of JSON serialization.
19. Explain canonical JSON conceptually.
20. Explain that ordinary JSON is not a cryptographic canonicalization standard.
21. Explain structured cloning.
22. Explain:
    ```js
    structuredClone(value)
    ```
23. Explain the difference between structured cloning and JSON round-tripping.
24. Explain which major JavaScript/host values structured clone can preserve more faithfully.
25. Explain what happens to:
    - prototypes,
    - Maps,
    - Sets,
    - Dates,
    - RegExps,
    - typed arrays,
    - ArrayBuffers,
    - errors,
    - cycles,
    - repeated references.
26. Explain what structured cloning does not clone, such as:
    - functions,
    - DOM-dependent objects in unsupported contexts,
    - certain host/resource objects.
27. Explain transferables.
28. Explain:
    - transferable ownership,
    - cloning,
    - zero-copy transfer.
29. Explain ArrayBuffer transfer in structured cloning.
30. Explain what happens to a transferred/detached ArrayBuffer.
31. Distinguish:
    - cloning,
    - transfer,
    - shared memory.
32. Explain why `SharedArrayBuffer` is not simply cloned like `ArrayBuffer`.
33. Explain structured clone support in browsers and Node.js as host/runtime capabilities.
34. Explain serialization boundaries in:
    - HTTP,
    - message passing,
    - workers,
    - caches,
    - databases,
    - storage,
    - logs.
35. Explain why JSON is often appropriate for API contracts but not for arbitrary object snapshots.
36. Explain security concerns:
    - prototype pollution,
    - unsafe revivers,
    - untrusted JSON,
    - resource amplification,
    - deeply nested inputs,
    - number precision,
    - Unicode concerns,
    - cyclic/object graph assumptions.
37. Explain why parsing JSON is not the same as validating its schema.
38. Explain schema validation as a separate concern.
39. Explain why deserializing data into domain objects requires an explicit construction policy.
40. Explain object identity loss during serialization.
41. Explain alias preservation under structured clone.
42. Implement a safe JSON serialization layer.
43. Implement replacer/reviver use cases.
44. Implement a cycle-aware serializer conceptually.
45. Implement a domain deserializer.
46. Debug JSON serialization mismatches.
47. Debug structured-clone failures.
48. Compare:
    - JSON,
    - structured clone,
    - custom binary serialization.
49. Choose the correct serialization format for a production requirement.
50. Reason about serialization at principal-engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 02 — Values / Types / Type System
- Chapter 04 — Strings / Unicode / Text Semantics
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 24 — Map / Set / WeakMap / WeakSet
- Chapter 27 — Typed Arrays / Binary Data

Strongly recommended:

- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 21 — Species / Subclassing
- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators

---

## 3. What Is It?

**Serialization** is the process of turning an in-memory value or object graph into a representation suitable for:

- storage,
- transmission,
- caching,
- logging,
- persistence,
- interoperability.

Examples:

```text
JavaScript object
      ↓
JSON text
```

or:

```text
JavaScript object
      ↓
binary encoding
```

**Deserialization** reconstructs a value or domain representation from that serialized form.

### JSON

JSON is a text data format with a limited data model.

A valid JSON document can represent:

```text
object
array
string
number
true
false
null
```

It cannot directly represent every JavaScript value.

---

### Structured clone

Structured cloning is a platform/language-integrated object-graph cloning mechanism used by:

```js
structuredClone(value)
```

and by APIs such as message passing in supported environments.

Unlike JSON, it is designed to preserve many JavaScript built-in value types and graph relationships.

---

### Core distinction

```text
JSON
  = interoperable data representation

Structured clone
  = structured object-graph cloning/transfer mechanism
```

Neither should automatically be treated as:

```text
"serialize anything"
```

---

## 4. Why Does It Exist?

### JSON solves interoperability

Different systems need a shared data format:

```text
browser
   ↓
HTTP
   ↓
Node
   ↓
database/service
```

JSON gives these systems a common textual representation.

### Structured clone solves object transfer/cloning

JavaScript environments frequently need to move complex values between execution contexts:

```text
main thread
    ↓
worker
```

or:

```text
runtime API
    ↓
message channel
```

Cloning through JSON is lossy:

```js
Map
Date
Set
TypedArray
cycles
shared references
```

may not survive correctly.

Structured clone is designed for a wider set of supported values.

---

## 5. Mental Model

Use five layers.

### Layer 1 — In-memory value

```text
JavaScript object graph
```

### Layer 2 — Data model

For JSON:

```text
object
array
string
number
boolean
null
```

### Layer 3 — Serialized representation

```text
UTF-8/Unicode JSON text
```

or:

```text
binary bytes
```

### Layer 4 — Reconstruction

```text
parse
clone
deserialize
```

### Layer 5 — Domain validation

```text
raw data
  ↓
shape validation
  ↓
semantic validation
  ↓
domain object construction
```

This last layer is critical.

Parsing valid JSON does not make the result valid application data.

---

## 6. Core Rules

### Rule 1 — JSON is a separate data model

Not every JavaScript value is a JSON value.

---

### Rule 2 — `JSON.stringify()` produces a String or can return `undefined`

For supported top-level values, serialization produces JSON text.

Some values can result in:

```js
undefined
```

rather than a JSON document.

---

### Rule 3 — `JSON.parse()` expects valid JSON text

It does not parse arbitrary JavaScript syntax.

For example:

```js
JSON.parse("{foo: 1}");
```

throws because property names are not quoted in valid JSON.

---

### Rule 4 — Object properties with `undefined` are omitted

```js
JSON.stringify({
  a: 1,
  b: undefined,
});
```

produces JSON containing only the representable property.

---

### Rule 5 — `undefined` in Arrays becomes `null`

```js
JSON.stringify([1, undefined, 3]);
```

produces:

```text
[1,null,3]
```

---

### Rule 6 — Functions are omitted from Objects and become `null` in Arrays

The exact JSON.stringify behavior depends on the containing position.

---

### Rule 7 — Symbol-keyed properties are not serialized by JSON.stringify

Even enumerable Symbol properties are outside the JSON object-key model.

---

### Rule 8 — BigInt cannot be serialized by JSON.stringify by default

A BigInt encountered during JSON serialization causes an error rather than silently becoming a Number.

---

### Rule 9 — `NaN` and infinities become `null`

JSON has no equivalent non-finite Number values.

---

### Rule 10 — Dates use `toJSON()`

Date serialization produces an ISO-like String representation through its defined `toJSON` behavior.

---

### Rule 11 — `toJSON()` can customize serialization

```js
const object = {
  toJSON() {
    return "custom";
  },
};
```

`JSON.stringify(object)` can serialize the value returned by `toJSON()`.

---

### Rule 12 — JSON serialization does not preserve prototypes

A class instance serialized to JSON normally reconstructs as an ordinary Object when parsed.

---

### Rule 13 — JSON serialization does not preserve methods

Functions are not represented in JSON.

---

### Rule 14 — JSON serialization does not preserve object identity

If two properties reference the same object:

```js
const shared = {};
const value = {
  a: shared,
  b: shared,
};
```

JSON does not encode that alias relationship by default.

---

### Rule 15 — Cyclic object graphs cannot be represented directly by normal JSON.stringify

A circular reference causes an error.

---

### Rule 16 — `JSON.parse()` returns ordinary JavaScript data structures

It does not automatically instantiate domain classes.

---

### Rule 17 — `JSON.parse()` is not schema validation

Valid JSON can still be invalid application data.

---

### Rule 18 — Structured clone preserves more built-in semantics than JSON

For supported types, it can preserve:

- Maps,
- Sets,
- Dates,
- RegExps,
- typed arrays,
- cycles,
- repeated references.

---

### Rule 19 — Structured clone does not clone every JavaScript value

Functions and certain host/resource objects are not generally cloneable.

---

### Rule 20 — Structured clone generally does not preserve custom prototypes as executable class instances

Cloned ordinary/class-created objects are reconstructed according to structured-clone semantics rather than preserving arbitrary prototype identity.

---

### Rule 21 — Transfer is different from cloning

Transfer moves ownership of supported transferable resources rather than copying their underlying storage.

---

### Rule 22 — Transferred ArrayBuffers become detached from the original owner

After transfer, operations that depend on the original backing data no longer behave as if the original buffer still owns those bytes.

---

### Rule 23 — SharedArrayBuffer uses shared-memory semantics

It is not simply copied like an ordinary ArrayBuffer.

---

### Rule 24 — JSON is text, not inherently a byte encoding contract

JSON text still needs an encoding when sent/stored as bytes.

UTF-8 is common, but the application boundary should define the encoding.

---

### Rule 25 — Serialization must be treated as an API contract

Do not assume that serializing an object graph and reconstructing it later recreates the same runtime semantics.

---

## 7. Syntax

### Stringify

```js
JSON.stringify(value)
JSON.stringify(value, replacer)
JSON.stringify(value, replacer, space)
```

### Parse

```js
JSON.parse(text)
JSON.parse(text, reviver)
```

### Clone

```js
structuredClone(value)
```

### Clone with transfer

```js
structuredClone(value, {
  transfer: [arrayBuffer],
});
```

### Date serialization

```js
date.toJSON()
```

### Replacer

```js
JSON.stringify(value, (key, value) => {
  return value;
});
```

### Reviver

```js
JSON.parse(text, (key, value) => {
  return value;
});
```

---

## 8. Basic Examples

### Example 1 — Basic JSON

```js
const value = {
  id: 1,
  name: "Asha",
  active: true,
};

const json = JSON.stringify(value);

console.log(json);
```

Conceptually:

```text
{"id":1,"name":"Asha","active":true}
```

---

### Example 2 — Parse

```js
const value = JSON.parse(
  '{"id":1,"name":"Asha"}'
);

console.log(value.name);
```

---

### Example 3 — Undefined omission

```js
const value = {
  present: 1,
  missing: undefined,
};

console.log(JSON.stringify(value));
```

The `missing` property is omitted.

---

### Example 4 — Undefined in Array

```js
console.log(
  JSON.stringify([1, undefined, 3]),
);
```

Result:

```text
[1,null,3]
```

---

### Example 5 — NaN and Infinity

```js
console.log(
  JSON.stringify({
    a: NaN,
    b: Infinity,
    c: -Infinity,
  }),
);
```

All three are represented as:

```text
null
```

inside the JSON result.

---

### Example 6 — Date

```js
const date = new Date("2026-01-01T00:00:00.000Z");

console.log(JSON.stringify({ date }));
```

The Date contributes its serialized String representation.

---

### Example 7 — BigInt failure

```js
JSON.stringify({ value: 1n });
```

This throws a TypeError under standard JSON serialization semantics.

---

### Example 8 — Structured clone

```js
const original = {
  date: new Date(),
  values: new Set([1, 2, 3]),
};

const clone = structuredClone(original);

console.log(clone.date instanceof Date);
console.log(clone.values instanceof Set);
```

---

## 9. Execution Walkthrough

Consider:

```js
const value = {
  name: "Milan",
  score: NaN,
  missing: undefined,
  nested: {
    x: 1,
  },
};

const json = JSON.stringify(value);
```

### Step 1 — Start JSON serialization

`JSON.stringify` determines the value to serialize.

### Step 2 — Visit Object properties

The String-keyed enumerable properties participating in serialization are examined according to JSON serialization semantics.

### Step 3 — Serialize `name`

```text
"Milan"
```

is a JSON String.

### Step 4 — Serialize `score`

```js
NaN
```

has no JSON Number representation.

JSON serialization converts it to:

```text
null
```

### Step 5 — Serialize `missing`

```js
undefined
```

cannot become a JSON property value in an Object.

The property is omitted.

### Step 6 — Serialize `nested`

The nested Object is recursively serialized.

### Step 7 — Produce JSON text

The final result is a String containing the JSON representation.

---

### Structured clone walkthrough

Consider:

```js
const shared = { x: 1 };

const value = {
  a: shared,
  b: shared,
};
```

With:

```js
const clone = structuredClone(value);
```

the cloned graph can preserve the alias relationship:

```js
clone.a === clone.b
```

while:

```js
clone.a !== shared
```

This is fundamentally different from JSON round-tripping.

---

## 10. Internal Mechanics

### 10.1 JSON data model

JSON represents:

```text
Object:
    String key → JSON value

Array:
    ordered JSON values

String
Number
Boolean
Null
```

It has no direct data model for:

```text
undefined
function
symbol
BigInt
Map
Set
prototype
class instance
reference identity
cycle
```

---

### 10.2 JSON.stringify recursive traversal

Conceptually:

```text
value
 ↓
determine serializable form
 ↓
walk object/array structure
 ↓
convert supported values
 ↓
omit/rewrite unsupported values
 ↓
produce JSON text
```

---

### 10.3 Property ordering

JSON.stringify's output uses property traversal rules based on ECMAScript property ordering/serialization algorithms.

Do not interpret this as a universal JSON canonicalization guarantee.

For ordinary objects, predictable property order can make output repeatable for the same in-memory structure, but reproducibility still depends on:

- property creation order,
- replacer behavior,
- custom `toJSON`,
- runtime/application inputs.

---

### 10.4 `toJSON`

Before ordinary serialization of an eligible Object value, JSON serialization can invoke a `toJSON` method when present.

This is why:

```js
Date
```

has custom JSON behavior.

The returned value is then serialized.

---

### 10.5 Replacer function

A replacer function can transform values during traversal.

```js
JSON.stringify(
  value,
  (key, currentValue) => {
    if (key === "password") {
      return undefined;
    }

    return currentValue;
  },
);
```

Returning `undefined` can omit an Object property or produce `null` in Array position according to JSON.stringify semantics.

---

### 10.6 Replacer array

A replacer Array can act as a property allowlist for object serialization.

This is useful for projection:

```js
JSON.stringify(user, [
  "id",
  "name",
]);
```

---

### 10.7 Space argument

The third argument controls indentation for human-readable output.

It is for formatting, not semantic canonicalization.

---

### 10.8 JSON.parse

`JSON.parse`:

1. reads valid JSON text,
2. constructs the corresponding JavaScript values,
3. optionally walks the resulting structure with a reviver.

---

### 10.9 Reviver

A reviver transforms parsed values after the basic structure is constructed.

Example:

```js
const value = JSON.parse(
  '{"createdAt":"2026-01-01T00:00:00.000Z"}',
  (key, currentValue) => {
    if (key === "createdAt") {
      return new Date(currentValue);
    }

    return currentValue;
  },
);
```

This provides an application-level reconstruction policy.

---

### 10.10 Reviver is not a validator

A reviver can transform data, but it should not be treated as a full security/schema-validation mechanism.

Validate independently.

---

### 10.11 JSON and Symbols

Symbol-keyed properties are outside the JSON property-key model.

```js
const key = Symbol("secret");

JSON.stringify({
  [key]: 123,
});
```

does not include the Symbol-keyed property.

---

### 10.12 JSON and BigInt

BigInt is intentionally not silently converted into Number.

This avoids silent precision loss.

---

### 10.13 JSON and Date

Date serialization typically becomes an ISO-format String.

Parsing it back gives:

```js
typeof parsed.date === "string"
```

unless application code reconstructs a Date.

---

### 10.14 JSON and RegExp

A RegExp object does not have native JSON syntax.

Without custom serialization, it serializes as an ordinary object with relevant enumerable properties, often resulting in:

```text
{}
```

for a normal RegExp.

Do not rely on accidental serialization.

---

### 10.15 JSON and Map

```js
JSON.stringify(new Map([["a", 1]]));
```

does not produce a JSON representation of Map entries by default.

A custom conversion is required.

---

### 10.16 JSON and Set

Similarly:

```js
JSON.stringify(new Set([1, 2, 3]));
```

does not automatically produce:

```text
[1,2,3]
```

A custom representation is needed.

---

### 10.17 JSON and TypedArrays

Typed arrays do not become a portable binary serialization simply because JSON.stringify can inspect their enumerable/index-like properties.

A JSON representation of typed-array state is not equivalent to the original binary backing store.

Use explicit binary encoding when byte-exact representation is required.

---

### 10.18 JSON and cycles

The JSON format itself has no native reference mechanism.

Therefore:

```js
const a = {};
a.self = a;
```

cannot be serialized by ordinary JSON.stringify.

---

### 10.19 Shared references

Even without a cycle:

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};
```

JSON encodes:

```text
two object structures
```

rather than an explicit shared-reference edge.

The graph structure is therefore not preserved.

---

### 10.20 Structured clone graph traversal

Structured cloning works on the object graph rather than reducing everything to JSON values.

It can preserve supported repeated references:

```text
source.a === source.b
        ↓ clone
clone.a === clone.b
```

while making the clone graph distinct from the source graph.

---

### 10.21 Structured clone and cycles

A cyclic graph can be cloned where all involved values are supported:

```js
const object = {};

object.self = object;

const clone = structuredClone(object);

console.log(clone.self === clone);
```

The cycle can remain a cycle in the cloned graph.

---

### 10.22 Structured clone and prototypes

Structured cloning does not generally preserve arbitrary custom prototype chains as live class semantics.

A class instance should not be assumed to remain:

```js
clone instanceof MyClass
```

with the same user-defined prototype behavior.

Domain reconstruction should be explicit.

---

### 10.23 Structured clone and private state

Private fields and arbitrary closures are not a general structured-clone mechanism.

An object with behavior dependent on inaccessible execution state should not be assumed to preserve that behavior through cloning.

---

### 10.24 Structured clone and functions

Functions are not generally cloneable by structured clone.

This prevents copying executable closure state as if it were ordinary data.

---

## 11. ECMAScript / Specification Semantics

### 11.1 JSON is standardized separately from ECMAScript object identity

The JSON data interchange model is deliberately smaller than the full ECMAScript value system.

This difference is a major source of serialization surprises.

---

### 11.2 `JSON.stringify` algorithm

ECMAScript defines detailed abstract operations for JSON serialization, including conceptual operations around:

```text
SerializeJSONProperty
SerializeJSONObject
SerializeJSONArray
```

These algorithms determine how values become JSON text.

---

### 11.3 `Str` and serialization state

The JSON serialization algorithm maintains traversal state and a stack for detecting cyclic structures.

This is why recursive object graphs can fail rather than recursively serialize forever.

---

### 11.4 `JSON.parse` algorithm

Parsing transforms JSON lexical structure into ECMAScript values.

The grammar is intentionally JSON-specific and rejects many forms accepted by JavaScript source syntax.

---

### 11.5 JSON numbers

JSON's Number model does not preserve the full distinction between all ECMAScript numeric values.

In particular:

```text
NaN
Infinity
-Infinity
```

are not JSON number literals.

---

### 11.6 Structured clone is a standard serialization-oriented algorithm family

Structured cloning is defined as a structured serialization/deserialization process used across language/platform APIs.

The exact set of cloneable types can evolve with the standard and host APIs.

Always distinguish:

```text
ECMAScript built-in support
```

from:

```text
host-defined structured-clone support
```

---

### 11.7 Structured serialize / deserialize

At specification level, structured cloning is described through serialization/deserialization algorithms that:

1. inspect the input graph,
2. assign internal references,
3. encode supported values,
4. reconstruct equivalent graph relationships.

This is fundamentally different from reducing data to plain JSON.

---

### 11.8 Transfer lists

Structured clone operations can accept a transfer list for transferable values.

Conceptually:

```text
clone graph
+
ownership-transfer set
```

determines whether particular resources are copied or transferred.

---

### 11.9 Transfer data

For an ArrayBuffer included in a transfer list:

```text
source backing store
       ↓
transfer ownership
       ↓
destination object
```

The source buffer becomes detached according to the transfer semantics.

---

### 11.10 Clone versus transfer

Clone:

```text
source storage remains valid
destination gets equivalent copied data
```

Transfer:

```text
ownership moves
source becomes detached/unusable for the transferred backing data
destination owns the data
```

---

### 11.11 SharedArrayBuffer and structured clone

Shared memory has separate semantics because multiple agents may observe the same backing storage.

The implementation and specification model therefore differs from ordinary ArrayBuffer transfer.

---

## 12. Advanced Behavior

### 12.1 Object property omission

```js
const value = {
  a: undefined,
  b: null,
};

console.log(JSON.stringify(value));
```

Result conceptually:

```text
{"b":null}
```

This difference matters in APIs where:

```text
missing
```

and:

```text
explicit null
```

have distinct meanings.

---

### 12.2 Array undefined conversion

```js
const value = [
  undefined,
  function () {},
  Symbol("x"),
];

console.log(JSON.stringify(value));
```

The unsupported positions become JSON `null` entries rather than disappearing from the Array structure.

---

### 12.3 Top-level unsupported values

```js
JSON.stringify(undefined);
JSON.stringify(function () {});
JSON.stringify(Symbol("x"));
```

can produce:

```text
undefined
```

rather than a JSON text representation.

This is distinct from an Array position.

---

### 12.4 `toJSON` context

A `toJSON` method receives a key argument indicating the context in which serialization occurs.

```js
const object = {
  toJSON(key) {
    return {
      serializedUnder: key,
    };
  },
};
```

This can lead to context-sensitive output.

Use carefully.

---

### 12.5 Custom BigInt serialization

Applications can define their own representation for BigInt, but the format must be explicit.

Example:

```js
const value = {
  id: 123n,
};

const json = JSON.stringify(value, (key, currentValue) => {
  if (typeof currentValue === "bigint") {
    return `${currentValue}n`;
  }

  return currentValue;
});
```

A matching reviver can reconstruct BigInt.

Never assume:

```text
"123n"
```

has a standard JSON BigInt meaning.

It is application-defined.

---

### 12.6 Precision loss

JSON numbers are commonly consumed as JavaScript Numbers.

Large integers beyond the safe integer range can lose precision when represented as ordinary Number values.

For identifiers, prefer explicit String or BigInt-aware serialization policies.

---

### 12.7 Prototype reconstruction

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return `Hello ${this.name}`;
  }
}

const user = new User("Asha");

const clone = JSON.parse(JSON.stringify(user));
```

`clone` is not a `User` instance with the original prototype methods.

The serialized representation contained data, not class behavior.

---

### 12.8 Structured clone of Map

```js
const original = new Map([
  ["a", { x: 1 }],
]);

const clone = structuredClone(original);
```

The result remains a Map in supported environments.

The Map structure is preserved, but the object values are separately cloned.

---

### 12.9 Structured clone of Set

```js
const original = new Set([
  { x: 1 },
]);

const clone = structuredClone(original);
```

The Set remains a Set and its object elements are cloned.

---

### 12.10 Structured clone repeated references

```js
const shared = { x: 1 };

const original = {
  first: shared,
  second: shared,
};

const clone = structuredClone(original);

console.log(clone.first === clone.second);
```

Expected:

```text
true
```

The alias edge is preserved inside the cloned graph.

---

### 12.11 Structured clone cycles

```js
const original = {};

original.self = original;

const clone = structuredClone(original);

console.log(clone.self === clone);
```

Expected:

```text
true
```

---

### 12.12 Structured clone of Date

A Date can be cloned as a Date in supported structured-clone environments:

```js
const original = new Date();

const clone = structuredClone(original);

console.log(clone instanceof Date);
```

---

### 12.13 Structured clone of RegExp

A supported RegExp can be cloned while preserving relevant RegExp state such as:

- source,
- flags,
- lastIndex according to clone semantics.

Do not confuse this with JSON serialization.

---

### 12.14 Typed-array cloning

Structured cloning can preserve supported typed-array types and their data.

The clone does not generally share the ordinary ArrayBuffer backing store unless transfer/shared-memory semantics apply.

---

### 12.15 ArrayBuffer transfer

```js
const buffer = new ArrayBuffer(8);

const result = structuredClone(buffer, {
  transfer: [buffer],
});
```

The resulting buffer receives the transferred storage.

The original becomes detached.

---

### 12.16 Transfer views versus transfer buffer

If a typed-array view references an ArrayBuffer, the transferable resource is the backing ArrayBuffer rather than the view object itself.

Therefore the transfer design should account for all views that reference the same buffer.

---

### 12.17 Transfer and aliases

If several views share one ArrayBuffer and the buffer is transferred, all source-side views are affected by detachment of the shared backing storage.

This is a major ownership consideration.

---

### 12.18 Structured cloning and accessors

Cloning does not simply execute every custom getter/setter as though performing a deep copy with ordinary property assignment.

The structured-clone semantics define which data is copied and how supported properties are represented.

Do not infer clone behavior from:

```js
for...in
```

or:

```js
Object.assign
```

---

### 12.19 Structured clone and non-enumerable properties

Do not assume arbitrary property descriptors are preserved.

Structured clone is a semantic reconstruction mechanism, not a general descriptor-preserving object copier.

---

### 12.20 Structured clone and custom classes

Treat class-instance transfer as:

```text
data transfer
```

not:

```text
preserve the original executable class identity
```

If the domain requires methods/invariants, reconstruct explicitly.

---

## 13. Edge Cases

### Edge Case 1 — Circular object

```js
const value = {};
value.self = value;

JSON.stringify(value);
```

Throws.

---

### Edge Case 2 — Circular array

```js
const value = [];
value.push(value);

JSON.stringify(value);
```

Throws.

---

### Edge Case 3 — Repeated shared object

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};

JSON.stringify(value);
```

Does not preserve the alias relationship.

---

### Edge Case 4 — `undefined` top level

```js
JSON.stringify(undefined);
```

returns:

```text
undefined
```

not:

```text
"undefined"
```

and not JSON `null`.

---

### Edge Case 5 — `null`

```js
JSON.stringify(null);
```

returns:

```text
"null"
```

This is a JSON document.

---

### Edge Case 6 — Symbol property

```js
const key = Symbol("x");

JSON.stringify({
  [key]: 1,
});
```

The Symbol-keyed property is not serialized.

---

### Edge Case 7 — Symbol value

```js
JSON.stringify({
  x: Symbol("x"),
});
```

The property is omitted.

---

### Edge Case 8 — BigInt array

```js
JSON.stringify([1n]);
```

throws.

Do not assume Array position converts BigInt to `null`.

BigInt is a hard serialization failure in ordinary JSON.stringify.

---

### Edge Case 9 — Date round-trip

```js
const original = new Date();

const parsed = JSON.parse(JSON.stringify(original));
```

The parsed result is a String, not a Date.

---

### Edge Case 10 — Precision

```js
const value = {
  id: 9007199254740993,
};

console.log(
  JSON.stringify(value),
);
```

The number has already passed through JavaScript Number precision limits before serialization.

Serialization cannot recover information that was already lost.

---

### Edge Case 11 — Reviver deletion

A reviver can return `undefined` for a parsed Object property, causing that property to be deleted during the reviver walk.

This can create surprising transformations.

---

### Edge Case 12 — Replacer side effects

A replacer function executes during traversal.

It can:

- mutate objects,
- allocate,
- throw,
- access getters,
- trigger proxies.

Serialization is executable code when customization hooks are involved.

---

### Edge Case 13 — `toJSON` side effects

```js
object.toJSON()
```

can execute arbitrary code.

Do not assume serialization is pure.

---

### Edge Case 14 — Deep nesting

Extremely deeply nested JSON can cause:

- parser cost,
- memory pressure,
- stack/resource issues,
- validation complexity.

Apply input limits where untrusted data is involved.

---

### Edge Case 15 — Huge arrays

A valid JSON document can be operationally dangerous if it contains:

```text
millions of elements
```

The parser may be correct while the application still exhausts memory.

---

### Edge Case 16 — Transfer after view creation

Transferring an ArrayBuffer affects all source views that depend on that buffer.

Always model ownership at the backing-storage level.

---

### Edge Case 17 — Detached buffer access

After transfer, operations on a source typed-array/DataView may throw or otherwise reflect detached-buffer semantics.

Do not continue using them as though they still own valid storage.

---

### Edge Case 18 — Unsupported structured-clone host object

Browser/Node APIs can include host objects that are not structured-cloneable.

A clone operation can fail with a `DataCloneError`-class failure.

---

### Edge Case 19 — SharedArrayBuffer

Shared backing memory has separate semantics and should not be treated as an ordinary transferable ownership object.

---

### Edge Case 20 — JSON text encoding

A String result from:

```js
JSON.stringify(...)
```

still needs an encoding when converted to bytes for transport.

Do not conflate JSON syntax with UTF-8 byte storage.

---

## 14. Common Misconceptions

### Misconception 1 — "JSON is a snapshot of any JavaScript object."

False.

It is a restricted data-interchange representation.

---

### Misconception 2 — "`JSON.parse(JSON.stringify(x))` deep-clones everything."

False.

It is lossy and fails for cycles/BigInt.

---

### Misconception 3 — "JSON preserves class instances."

False.

Prototype behavior is not preserved.

---

### Misconception 4 — "JSON preserves undefined."

False.

Its behavior depends on context, but it does not represent `undefined` as a JSON value.

---

### Misconception 5 — "JSON preserves Symbols."

False.

Symbol keys/values are outside the JSON data model.

---

### Misconception 6 — "JSON preserves Map and Set."

False.

They require explicit conversion.

---

### Misconception 7 — "Structured clone preserves everything about objects."

False.

It supports a defined set of values and does not preserve arbitrary executable/prototype semantics.

---

### Misconception 8 — "Structured clone always copies bytes."

False.

Supported transferables can be transferred instead.

---

### Misconception 9 — "Transfer and shared memory are the same."

False.

Transfer changes ownership; shared memory allows multiple agents to access shared storage.

---

### Misconception 10 — "Valid JSON is safe application input."

False.

It can still contain malicious or resource-exhausting data.

---

### Misconception 11 — "Parsing JSON validates business rules."

False.

Syntax validity and domain validity are different.

---

### Misconception 12 — "JSON.stringify is deterministic enough for hashing."

Not automatically.

Canonicalization requirements must be explicit.

---

### Misconception 13 — "Whitespace changes JSON meaning."

Whitespace can change the textual representation without changing parsed data.

Formatting matters for:

- bytes,
- hashes,
- signatures,
- diffs,

even when parsed semantics are equivalent.

---

### Misconception 14 — "A serialized object can automatically regain its methods."

False.

Deserialization must explicitly reconstruct domain behavior.

---

### Misconception 15 — "JSON is a binary format because HTTP sends bytes."

False.

JSON is a text syntax that can be encoded as bytes for transmission.

---

## 15. Common Mistakes

### Mistake 1 — Blind JSON deep clone

Avoid:

```js
const clone = JSON.parse(JSON.stringify(value));
```

when the object graph contains:

- Dates,
- Maps,
- Sets,
- BigInts,
- cycles,
- undefined-sensitive state,
- typed arrays.

---

### Mistake 2 — Losing undefined versus null

APIs should define whether:

```text
missing
undefined
null
```

mean different things.

---

### Mistake 3 — Serializing class instances without a reconstruction strategy

Store explicit domain data.

---

### Mistake 4 — Treating IDs as arbitrary JSON Numbers

Large integer identifiers can lose precision.

Use strings or an explicit BigInt-aware format.

---

### Mistake 5 — Parsing without schema validation

Valid syntax does not guarantee valid input.

---

### Mistake 6 — Accepting huge JSON bodies without limits

Apply:

- byte limits,
- depth limits where appropriate,
- array/object size limits,
- request timeouts.

---

### Mistake 7 — Assuming `toJSON()` is harmless

Custom serialization is user code.

---

### Mistake 8 — Assuming structured clone preserves prototypes

It does not serve as a general class-instance transporter.

---

### Mistake 9 — Transferring a buffer without ownership analysis

All views over the transferred buffer can be affected.

---

### Mistake 10 — Reusing detached views

After transfer, source-side code must treat the original backing storage as detached.

---

### Mistake 11 — Using JSON for cryptographic byte data

Serialize bytes explicitly, such as:

```text
Base64
hex
binary format
```

according to protocol requirements.

---

### Mistake 12 — Using JSON as cache serialization without versioning

Serialized shape changes can invalidate stored data.

---

## 16. Comparison With Related Concepts

| Technique | Text | Cycles | Maps/Sets | Prototypes | Identity/aliases | Transfer | Typical use |
|---|---:|---:|---:|---:|---:|---:|---|
| JSON | Yes | No | No, by default | No | No | No | APIs/storage |
| Structured clone | No required text form | Yes, supported graph | Yes, supported | Limited/defined | Yes, supported | Yes, supported transferables | workers/message passing |
| Custom binary | Usually no | Depends | Depends | Depends | Depends | Depends | high-performance protocols |
| Manual DTO mapping | Optional | Domain-defined | Domain-defined | Explicit | Explicit | Explicit | business boundaries |

### JSON versus structured clone

JSON:

```text
interoperability
human-readable text
small common data model
```

Structured clone:

```text
richer JS object graph
built-in type preservation
cycles
transfer
```

---

### JSON versus custom binary

JSON:

- easy to inspect,
- broadly interoperable,
- more verbose,
- limited types.

Binary:

- efficient,
- precise,
- compact,
- more schema/protocol complexity.

---

### Structured clone versus manual serialization

Structured clone is convenient for supported values.

Manual serialization is preferable when:

- wire format must be stable,
- domain schema matters,
- security boundaries require explicit fields,
- versioning is critical,
- cross-language interoperability is required.

---

## 17. Performance Considerations

### 17.1 JSON stringify cost

Serialization is generally proportional to the amount of traversed data, but actual cost depends on:

- object graph shape,
- strings,
- nested structures,
- replacers,
- `toJSON`,
- getters/proxies,
- allocations.

---

### 17.2 JSON parse cost

Parsing cost depends on:

- input size,
- nesting,
- number of properties,
- numbers/strings,
- validation after parsing.

Large JSON documents can create temporary and retained allocations.

---

### 17.3 JSON versus binary

JSON often carries more textual overhead than compact binary encodings.

But the correct choice depends on:

- network bandwidth,
- CPU budget,
- development complexity,
- interoperability,
- observability,
- compression.

Compressed JSON can still be highly effective.

---

### 17.4 Structured clone cost

Structured clone traverses and reconstructs the supported object graph.

Large graphs can require:

- traversal CPU,
- allocation,
- copying.

Transferables can avoid copying certain backing stores.

---

### 17.5 Transfer advantage

For large ArrayBuffers:

```text
copy
    → O(n) data movement

transfer
    → ownership change
```

The exact implementation cost is environment-dependent.

---

### 17.6 Schema validation cost

Validation adds CPU work, but skipping validation on untrusted boundaries can create much larger correctness/security costs.

---

### 17.7 Replacer/reviver overhead

Customization functions execute repeatedly.

For large payloads, they can become a dominant cost.

---

### 17.8 Canonicalization cost

Canonical output often requires:

- stable ordering,
- normalization,
- deterministic numeric/string encoding.

This adds processing overhead and should be justified by the protocol requirement.

---

## 18. Memory Considerations

### JSON stringify

The serialized String itself consumes memory in addition to the original object graph.

```text
object graph
+
JSON text
```

may coexist.

---

### JSON parse

During parsing, the input text and resulting object graph can overlap in lifetime.

Large payloads can therefore create significant peak memory.

---

### Structured clone

Cloning generally creates a second object graph:

```text
source graph
+
clone graph
```

until the source becomes unreachable.

---

### Transfer

Transfer can reduce duplicate backing storage for supported resources:

```text
source
    ↓ ownership transfer
destination
```

instead of:

```text
source bytes
+
destination copy
```

---

### Shared references

Structured clone can preserve aliases inside the destination graph without duplicating every repeated reference as an independent object.

---

### Unbounded serialization

An application that repeatedly serializes growing caches/logs can cause:

- memory spikes,
- GC pressure,
- latency.

Measure payload sizes.

---

## 19. Security Considerations

### Untrusted JSON

JSON.parse does not execute JavaScript code merely because the JSON contains strings such as:

```text
"alert(1)"
```

But the resulting data becomes dangerous if application code later evaluates/interprets it unsafely.

---

### Prototype pollution

Modern JSON.parse produces ordinary own properties according to its parsing semantics, but application code that merges parsed data into Objects can reintroduce prototype-pollution risks.

For example:

```js
Object.assign(target, parsed);
```

should be reviewed carefully when input is untrusted.

---

### Reviver attacks

A reviver executes application code during parsing.

Do not treat it as a safe declarative parser.

---

### Resource exhaustion

Attackers can send:

```text
huge object
deep nesting
huge arrays
very long strings
```

Apply limits before/while parsing where the host/API permits.

---

### Number precision

Using JSON Numbers for security-critical identifiers can create:

```text
collision
misrouting
authorization mismatch
```

if the receiving system rounds large integers.

---

### Canonicalization attacks

Different JSON text representations can parse to semantically equivalent data:

```text
whitespace changes
ordering changes
escaping differences
```

If signatures/hashes depend on bytes, define canonical serialization.

---

### Unicode

JSON Strings carry Unicode text.

Security-sensitive systems still need to consider:

- normalization,
- confusables,
- controls,
- bidi characters.

See Chapter 23 and Chapter 57 later.

---

### Deserialization attacks

Structured clone itself is not arbitrary code execution, but application-level reconstruction can be dangerous if it converts untrusted data into privileged objects or operations without validation.

---

### Transfer ownership bugs

A transferred buffer can invalidate assumptions on the sending side.

Incorrect ownership handling can become a data-corruption bug or availability problem.

---

## 20. Production Usage

### Use case 1 — REST/HTTP DTOs

Use JSON for stable API data:

```js
{
  id: "user_123",
  name: "Asha",
  active: true
}
```

Define:

- required fields,
- optional fields,
- nullability,
- numeric/string semantics,
- versioning.

---

### Use case 2 — Worker messaging

Use structured clone when supported and appropriate:

```js
worker.postMessage({
  values: new Set([1, 2, 3]),
});
```

This avoids manually JSON-encoding every supported value.

---

### Use case 3 — Large binary worker transfer

For a large binary payload:

```js
worker.postMessage(
  { buffer },
  [buffer],
);
```

when the API supports transferable ownership.

This can avoid copying the backing data.

---

### Use case 4 — Domain DTO boundary

Prefer:

```text
wire data
  ↓
validate
  ↓
construct domain object
```

over:

```text
JSON.parse
  ↓
pretend this is already a trusted domain entity
```

---

### Use case 5 — Cache storage

For cache entries:

```text
version
+
payload
+
metadata
```

Use explicit versioning:

```js
{
  version: 2,
  data: ...
}
```

---

### Use case 6 — Audit logs

Serialize a deliberate projection rather than blindly stringifying sensitive domain objects.

Use explicit redaction.

---

### Use case 7 — Binary APIs

For cryptographic, image, compressed, or protocol-level bytes:

```text
Uint8Array/DataView/Buffer
```

with explicit binary encoding.

Do not force byte-exact data into JSON Numbers.

---

### Use case 8 — Reconstructing domain types

Example:

```js
function deserializeUser(input) {
  if (
    !input ||
    typeof input !== "object" ||
    typeof input.id !== "string" ||
    typeof input.name !== "string"
  ) {
    throw new TypeError("Invalid user data");
  }

  return new User(input.id, input.name);
}
```

This makes reconstruction explicit.

---

### Production rule

Separate the layers:

```text
transport format
      ↓
parse
      ↓
validate
      ↓
normalize
      ↓
deserialize
      ↓
domain object
```

This prevents format semantics from silently becoming domain semantics.

---

## 21. Implementation From Scratch

### 21.1 Safe JSON DTO serializer

```js
function serializeUser(user) {
  return JSON.stringify({
    id: user.id,
    name: user.name,
    active: user.active,
  });
}
```

Explicit projection prevents accidental leakage of internal fields.

---

### 21.2 BigInt-aware DTO

```js
function serializeAccount(account) {
  return JSON.stringify({
    id: account.id.toString(),
    balance: account.balance.toString(),
  });
}
```

The wire schema must document these fields as Strings.

---

### 21.3 Reviver for ISO dates

```js
function parseWithDates(text) {
  return JSON.parse(text, (key, value) => {
    if (
      typeof value === "string" &&
      /^\d{4}-\d{2}-\d{2}T/.test(value)
    ) {
      return new Date(value);
    }

    return value;
  });
}
```

This is intentionally simplistic and should not be used as a generic "all ISO strings are dates" rule.

---

### 21.4 Custom Map DTO

```js
function serializeMap(map) {
  return JSON.stringify(
    [...map.entries()],
  );
}

function deserializeMap(text) {
  const entries = JSON.parse(text);

  if (!Array.isArray(entries)) {
    throw new TypeError("Expected entry array");
  }

  return new Map(entries);
}
```

This is domain-specific and requires validation.

---

### 21.5 Cycle-aware conceptual serializer

A custom graph serializer can assign IDs:

```js
function createReferenceTable(root) {
  const ids = new Map();
  const queue = [root];

  let nextId = 0;

  while (queue.length) {
    const value = queue.shift();

    if (
      value === null ||
      typeof value !== "object" ||
      ids.has(value)
    ) {
      continue;
    }

    ids.set(value, nextId++);

    for (const child of Object.values(value)) {
      queue.push(child);
    }
  }

  return ids;
}
```

This demonstrates the extra machinery required to preserve object identity.

---

### 21.6 Domain deserializer

```js
function deserializeOrder(input) {
  if (!input || typeof input !== "object") {
    throw new TypeError("Invalid order");
  }

  if (typeof input.id !== "string") {
    throw new TypeError("Invalid order id");
  }

  if (!Array.isArray(input.items)) {
    throw new TypeError("Invalid items");
  }

  return new Order(
    input.id,
    input.items.map(deserializeItem),
  );
}
```

---

### 21.7 Binary alternative

For a binary protocol:

```text
version
type
length
payload
```

reuse the BinaryReader/BinaryWriter pattern from Chapter 27.

---

### 21.8 Structured clone graph test

Build tests for:

```text
Date
Map
Set
RegExp
TypedArray
ArrayBuffer
cycle
shared references
nested objects
unsupported function
custom class
```

Then compare with JSON round-tripping.

---

### Implementation progression

**Guided**

- DTO projection,
- JSON parse/stringify,
- date reviver.

**Partially Guided**

- Map serialization,
- BigInt serialization,
- explicit domain reconstruction.

**No Reference**

- Build a versioned serialization layer.

**Edge-Case Hardened**

Support:

- cycles,
- repeated references,
- precision-sensitive integers,
- invalid schema,
- huge inputs,
- unsupported types.

**Production-Grade**

Add:

- schema versioning,
- validation,
- fuzz tests,
- size limits,
- redaction,
- compatibility tests,
- migration logic,
- observability.

---

## 22. Debugging Exercises

### Exercise 1 — Missing field

```js
const value = {
  name: "Asha",
  role: undefined,
};

const json = JSON.stringify(value);
```

Explain why `role` disappears.

---

### Exercise 2 — Array null surprise

```js
const value = [undefined, () => 1];

console.log(JSON.stringify(value));
```

Explain why both positions remain present but become JSON `null`.

---

### Exercise 3 — Date type loss

```js
const date = new Date();

const parsed = JSON.parse(
  JSON.stringify({ date }),
);

console.log(parsed.date instanceof Date);
```

Explain.

---

### Exercise 4 — BigInt failure

```js
JSON.stringify({
  id: 123n,
});
```

Diagnose and design a wire representation.

---

### Exercise 5 — Shared identity lost

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};

const clone = JSON.parse(
  JSON.stringify(value),
);

console.log(clone.a === clone.b);
```

Explain why the relationship is lost.

---

### Exercise 6 — Structured clone identity

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};

const clone = structuredClone(value);

console.log(clone.a === clone.b);
```

Explain the difference.

---

### Exercise 7 — Circular JSON

```js
const value = {};

value.self = value;

JSON.stringify(value);
```

Diagnose.

---

### Exercise 8 — Large identifier

```js
const id = 9007199254740993;

console.log(id);
```

Explain why converting this to/from JSON as a Number cannot recover the intended integer if precision was already lost.

---

### Exercise 9 — Buffer offset

Serialize a typed-array view and verify that your binary path respects:

```js
byteOffset
byteLength
```

rather than assuming the backing buffer equals the logical payload.

---

### Exercise 10 — Transfer

Transfer an ArrayBuffer and then inspect the source-side view.

Explain the detached-buffer state.

---

## 23. Code Review Exercise

Review:

```js
function saveUser(user) {
  localStorage.setItem(
    "user",
    JSON.stringify(user),
  );
}

function loadUser() {
  return JSON.parse(
    localStorage.getItem("user"),
  );
}
```

### Questions

1. What happens if `user` is a class instance?
2. What happens to Dates?
3. What happens to undefined fields?
4. What happens to methods?
5. What happens to private state?
6. Is the stored data versioned?
7. Is the input trusted?
8. Is schema validation performed?
9. What happens if old stored data has a different schema?
10. What happens if parsing fails?
11. Can storage contain attacker-controlled content in the application context?
12. Should a domain object be reconstructed explicitly?
13. How should migration work?
14. Should sensitive fields be persisted?
15. Is JSON even the right persistence format?

---

## 24. Interview Questions

### Fundamentals

1. What is serialization?
2. What is deserialization?
3. What is JSON?
4. How is JSON different from JavaScript object syntax?
5. What data types can JSON represent?
6. What is the difference between JSON data and JavaScript values?
7. What does `JSON.stringify` do?
8. What does `JSON.parse` do?
9. When does `JSON.stringify` return `undefined`?
10. Why does JSON not represent `undefined`?

### JSON behavior

11. What happens to `undefined` in objects?
12. What happens to `undefined` in arrays?
13. What happens to functions?
14. What happens to Symbols?
15. What happens to BigInt?
16. What happens to NaN and Infinity?
17. How are Dates serialized?
18. What is `toJSON()`?
19. What is a replacer?
20. What is a reviver?
21. What happens with circular references?
22. Does JSON preserve object identity?
23. Does JSON preserve prototypes?
24. Does JSON preserve class instances?
25. Does JSON preserve Map/Set?
26. Does JSON preserve typed-array binary semantics?

### Structured clone

27. What is structured cloning?
28. What does `structuredClone()` do?
29. How is structured clone different from JSON round-tripping?
30. Can structured clone preserve cycles?
31. Can it preserve repeated references?
32. Can it clone Maps and Sets?
33. Can it clone Dates?
34. Can it clone typed arrays?
35. Can it clone functions?
36. Does it preserve custom prototypes?
37. What happens to unsupported host objects?
38. What is a `DataCloneError`-style failure?

### Transfer

39. What is a transferable?
40. What is the difference between cloning and transfer?
41. What happens to an ArrayBuffer after transfer?
42. What happens to typed-array views over a transferred buffer?
43. How is SharedArrayBuffer different?
44. When is transfer better than copying?
45. When is copying safer?

### Architecture/security

46. Why is JSON.parse not schema validation?
47. How can large JSON inputs cause resource exhaustion?
48. How can large integers create correctness/security issues?
49. How can prototype pollution appear after parsing?
50. Why should domain objects be explicitly reconstructed?
51. When should a system use binary serialization instead of JSON?
52. How would you version serialized data?
53. What does canonical JSON mean?
54. Why is ordinary JSON output not automatically suitable for cryptographic signing?

### Principal-level

55. Design a public API serialization contract.
56. Design a persistent cache format with migrations.
57. Design worker messaging for large binary payloads.
58. Decide between JSON, structured clone, and a binary protocol.
59. Design a safe domain deserializer.
60. How would you preserve object identity across serialization?
61. How would you sign serialized data safely?
62. How would you bound untrusted serialization/deserialization workloads?
63. How would you handle forward/backward compatibility?
64. How would you audit serialized data for secrets?
65. How would you test serialization compatibility over multiple versions?
66. How would you guarantee a binary payload is not accidentally transformed into text?
67. What should be logged when deserialization fails in production?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
console.log(
  JSON.stringify({
    a: 1,
    b: undefined,
    c: null,
  }),
);
```

---

### Exercise B

```js
console.log(
  JSON.stringify([
    1,
    undefined,
    () => 1,
    Symbol("x"),
  ]),
);
```

---

### Exercise C

```js
console.log(JSON.stringify(NaN));
console.log(JSON.stringify(Infinity));
```

---

### Exercise D

```js
console.log(JSON.stringify(undefined));
```

---

### Exercise E

```js
const date = new Date(
  "2026-01-01T00:00:00.000Z",
);

const parsed = JSON.parse(
  JSON.stringify({ date }),
);

console.log(typeof parsed.date);
console.log(parsed.date instanceof Date);
```

---

### Exercise F

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};

const clone = JSON.parse(
  JSON.stringify(value),
);

console.log(clone.a === clone.b);
```

---

### Exercise G

```js
const shared = {};

const value = {
  a: shared,
  b: shared,
};

const clone = structuredClone(value);

console.log(clone.a === clone.b);
console.log(clone.a === shared);
```

---

### Exercise H

```js
const value = {};

value.self = value;

JSON.stringify(value);
```

Predict whether it returns or throws.

---

### Exercise I

```js
const value = {
  map: new Map([["a", 1]]),
  set: new Set([1, 2]),
};

console.log(JSON.stringify(value));
```

---

### Exercise J

```js
const value = {
  toJSON(key) {
    return `serialized:${key}`;
  },
};

console.log(JSON.stringify(value));
```

Predict the output and explain the key context.

---

### Exercise K

```js
const buffer = new ArrayBuffer(8);

const clone = structuredClone(buffer);

console.log(buffer.byteLength);
console.log(clone.byteLength);
```

---

### Exercise L

Conceptual transfer:

```js
const buffer = new ArrayBuffer(8);

const clone = structuredClone(buffer, {
  transfer: [buffer],
});
```

Predict the relationship between:

```text
source buffer
destination buffer
```

after the operation.

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
serialization
parsing
cloning
transfer
deserialization
```

as separate concepts.

### Level 2 — Explain

Explain why:

```js
JSON.parse(JSON.stringify(value))
```

is not a universal deep-clone algorithm.

### Level 3 — Predict

Predict JSON behavior for:

```text
undefined
function
Symbol
BigInt
NaN
Infinity
Date
Map
Set
cycle
shared reference
```

### Level 4 — Implement

Build:

```text
DTO serializer
DTO validator
DTO deserializer
Map serializer
BigInt-safe identifier serializer
```

### Level 5 — Debug

Fix:

- Date type loss,
- precision loss,
- missing-vs-null bugs,
- cycle failures,
- identity loss,
- detached-buffer misuse.

### Level 6 — Compare

Compare:

```text
JSON
structured clone
binary serialization
manual DTO mapping
```

using:

- interoperability,
- fidelity,
- speed,
- memory,
- security,
- versioning.

### Level 7 — Apply

Design a worker messaging protocol that supports:

```text
JSON metadata
+
large ArrayBuffer payload
```

without unnecessary copying.

### Level 8 — Defend

Choose whether:

```text
JSON
structured clone
binary
```

should be the transport for a production feature.

### Level 9 — Principal Judgment

Design a complete serialization architecture with:

```text
wire format
schema
validation
versioning
migration
security
size limits
observability
compatibility
```

and explicitly define what information is intentionally lost.

---

## 27. Key Takeaways

1. Serialization creates a representation suitable for storage or transport.
2. Deserialization reconstructs an application representation.
3. JSON is a restricted data-interchange format.
4. JSON is not the full JavaScript value model.
5. `JSON.stringify` serializes according to JSON-specific semantics.
6. `JSON.parse` parses JSON syntax, not arbitrary JavaScript.
7. Object `undefined` properties are omitted.
8. Undefined/function/Symbol Array positions become JSON `null`.
9. Top-level unsupported values can produce `undefined`.
10. BigInt causes JSON serialization failure by default.
11. NaN and infinities become `null`.
12. Dates use `toJSON`.
13. `toJSON`, replacers, and revivers execute application code.
14. JSON does not preserve prototypes or class behavior.
15. JSON does not preserve object identity or aliases.
16. JSON cannot directly represent cycles.
17. JSON does not natively represent Map/Set semantics.
18. JSON does not preserve typed-array binary semantics.
19. JSON Numbers can be unsafe for large integer identifiers.
20. Structured clone handles a richer set of supported object types.
21. Structured clone can preserve supported cycles.
22. Structured clone can preserve repeated references within the cloned graph.
23. Structured clone does not preserve arbitrary executable class behavior.
24. Functions are not generally cloneable.
25. Transfer can move ownership of supported resources without copying their backing storage.
26. Transfer can detach the source ArrayBuffer.
27. SharedArrayBuffer uses shared-memory semantics rather than ordinary ownership transfer.
28. Parsing valid JSON is not validation.
29. Domain reconstruction should be explicit.
30. Production serialization is an API/versioning/security contract.
31. Binary data should remain byte-oriented across binary boundaries.
32. Principal-level serialization design starts by defining exactly what information must survive the boundary.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values / Types
- Chapter 04 — Strings / Unicode / Text Semantics
- Chapter 07 — Coercion / Equality
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering
- Chapter 17 — Prototypes
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 24 — Map / Set / Weak Collections
- Chapter 27 — Typed Arrays / Binary Data

### Builds Toward

- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Async Fundamentals
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 45 — Memory / GC
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 57 — JavaScript Security Engineering
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 66 — package.json / Resolution
- Chapter 67 — Dependency / Supply Chain
- Chapter 78 — Production JS Architecture
- Chapter 79 — API Design
- Chapter 80 — Library Authoring
- Chapter 81 — Database Integration
- Chapter 82 — API Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 96 — WebAssembly / Native Interoperability

### Related Concepts

- DTO
- wire format
- JSON
- structured clone
- transferables
- ArrayBuffer
- SharedArrayBuffer
- object graphs
- identity
- aliases
- cycles
- schema validation
- domain reconstruction
- canonicalization
- compatibility/versioning
- binary encoding

### Concepts Revisited

**Chapter 15:**  
Serialization demonstrates why object properties and object graphs are not equivalent to wire representations.

**Chapter 16:**  
Property ordering affects JSON text generation, but JSON output should not be casually treated as a canonical representation.

**Chapter 17:**  
Prototype relationships are not automatically preserved by JSON or structured clone.

**Chapter 22:**  
Arrays have special JSON behavior for unsupported values and sparse positions.

**Chapter 23:**  
JSON is textual, but text still has encoding and Unicode semantics at the transport boundary.

**Chapter 24:**  
Map/Set demonstrate the difference between JavaScript collections and JSON's smaller data model.

**Chapter 25/26:**  
Iterables and generators can produce serialized values lazily for streaming systems.

**Chapter 27:**  
Typed arrays and ArrayBuffers explain why binary serialization should remain byte-oriented when exact representation matters.

### Why This Chapter Matters Later

Serialization is the boundary between:

```text
runtime semantics
```

and:

```text
external representation
```

Almost every production system crosses this boundary:

```text
API
database
cache
worker
queue
file
log
browser storage
network
```

The central engineering lesson is:

> A serialized representation is a contract, not a transparent mirror of runtime state.

Once this is understood, JSON surprises, structured-clone behavior, version migrations, precision bugs, and binary protocol decisions become much easier to reason about.

---

## Track A — Core Theory

### Level 1 — Intuition

> Serialization turns runtime data into a representation; deserialization reconstructs data according to an explicit contract.

### Level 2 — Syntax

Know:

```js
JSON.stringify()
JSON.parse()
structuredClone()
```

and replacer/reviver/transfer options.

### Level 3 — Practical

Build:

- DTOs,
- JSON APIs,
- domain deserializers,
- worker message payloads.

### Level 4 — Edge Cases

Understand:

- undefined,
- functions,
- Symbols,
- BigInt,
- NaN,
- Infinity,
- Dates,
- cycles,
- aliases,
- precision,
- transfers.

### Level 5 — Runtime/Internal

Understand:

- JSON traversal,
- graph traversal,
- cycle detection,
- structured serialization records,
- cloning,
- transfer,
- detachment.

### Level 6 — Specification Semantics

Be comfortable with:

- `SerializeJSONProperty`
- `SerializeJSONObject`
- `SerializeJSONArray`
- JSON parse/stringify semantics
- structured serialization/deserialization
- transfer lists
- detached ArrayBuffers.

### Level 7 — Performance/Security

Reason about:

- payload size,
- allocation,
- memory peaks,
- copy versus transfer,
- input amplification,
- precision,
- prototype pollution,
- canonicalization.

### Level 8 — Production Engineering

Design:

- versioned wire schemas,
- validation,
- DTO mapping,
- worker transport,
- cache formats,
- migration systems.

### Level 9 — Interview/Reasoning

Answer:

> Why is `JSON.parse(JSON.stringify(x))` fundamentally incapable of being a universal deep clone?

### Level 10 — Principal Judgment

Evaluate:

> What information must survive this system boundary, and what information should intentionally be discarded?

Use:

```text
data
identity
prototype
behavior
binary representation
version
security metadata
```

as separate questions.

---

## Track B — Implementation

The implementation ladder is:

```text
1. JSON DTO projection
        ↓
2. JSON parser/reconstructor
        ↓
3. replacer/reviver
        ↓
4. Map/Set serialization
        ↓
5. BigInt-safe identifiers
        ↓
6. schema validation
        ↓
7. versioned DTO migration
        ↓
8. worker structured cloning
        ↓
9. transferable binary payloads
        ↓
10. production serialization architecture
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain:

```text
serialization
vs
cloning
vs
transfer
```

### Drill 2

Explain why:

```js
JSON.stringify(new Set([1, 2, 3]))
```

does not automatically produce a JSON Array.

### Drill 3

Explain why:

```text
JSON
```

loses:

```text
identity
prototype
methods
cycles
```

### Drill 4

Explain why structured clone can preserve:

```js
clone.a === clone.b
```

for repeated references.

### Drill 5

Choose between:

```text
JSON
structured clone
binary
```

for:

```text
large worker payload
REST API
cryptographic packet
persistent cache
```

### Drill 6

Design a deserializer that separates:

```text
parse
validate
normalize
construct
```

### Drill 7

Explain why canonical JSON is a separate engineering requirement from simply calling `JSON.stringify`.

### Drill 8

Design a versioned serialization contract that supports both old and new consumers.

---

## 29. Completion Criteria

Mark Chapter 28 `[+] Completed` only when the learner can:

- [ ] Define serialization.
- [ ] Define deserialization.
- [ ] Define JSON.
- [ ] Explain JSON's value model.
- [ ] Distinguish JSON from JavaScript values.
- [ ] Explain `JSON.stringify`.
- [ ] Explain `JSON.parse`.
- [ ] Explain undefined behavior.
- [ ] Explain function behavior.
- [ ] Explain Symbol behavior.
- [ ] Explain BigInt behavior.
- [ ] Explain NaN/Infinity behavior.
- [ ] Explain Date serialization.
- [ ] Explain `toJSON`.
- [ ] Explain replacer.
- [ ] Explain reviver.
- [ ] Explain circular-reference failure.
- [ ] Explain alias/identity loss.
- [ ] Explain prototype loss.
- [ ] Explain Map/Set behavior.
- [ ] Explain typed-array limitations.
- [ ] Explain number precision limits.
- [ ] Explain structured cloning.
- [ ] Explain supported graph cloning.
- [ ] Explain cycle preservation.
- [ ] Explain repeated-reference preservation.
- [ ] Explain custom prototype limitations.
- [ ] Explain non-cloneable values.
- [ ] Explain transferables.
- [ ] Explain ArrayBuffer transfer.
- [ ] Explain detachment.
- [ ] Explain SharedArrayBuffer distinction.
- [ ] Explain schema validation as a separate step.
- [ ] Implement safe DTO serialization.
- [ ] Implement explicit domain deserialization.
- [ ] Implement replacer/reviver patterns.
- [ ] Design versioned serialization.
- [ ] Compare JSON, structured clone, and binary formats.
- [ ] Debug serialization failures.
- [ ] Debug precision and identity problems.
- [ ] Design size and security limits.
- [ ] Defend a production serialization architecture.

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

> Given an external boundary, exactly what runtime information must survive, what information may be discarded, and which representation—JSON, structured clone, custom binary, or explicit DTO mapping—best enforces that contract?

---

## Chapter 28 Retrieval Set

### Retrieval 1

Why is:

```js
JSON.stringify(undefined)
```

different from:

```js
JSON.stringify([undefined])
```

### Retrieval 2

Why does JSON serialization fail on BigInt by default?

### Retrieval 3

Why are:

```text
NaN
Infinity
-Infinity
```

serialized as `null`?

### Retrieval 4

Why does a Date become a String after JSON round-tripping?

### Retrieval 5

Why does JSON lose:

```text
object identity
prototype
methods
```

### Retrieval 6

Why does JSON fail on cycles?

### Retrieval 7

Why can structured clone preserve repeated references?

### Retrieval 8

Why can structured clone clone a Map but JSON cannot preserve Map semantics automatically?

### Retrieval 9

What is the difference between cloning and transfer?

### Retrieval 10

What happens to the original ArrayBuffer after transfer?

### Retrieval 11

Why is JSON.parse not schema validation?

### Retrieval 12

Why can large numeric IDs become corrupted through JSON Number representation?

### Retrieval 13

When should binary data bypass JSON entirely?

### Retrieval 14

Why should domain-object reconstruction be explicit?

---

## Chapter 28 Final Mental Model

Remember:

```text
                    RUNTIME DATA
                         |
              +----------+----------+
              |                     |
             JSON              STRUCTURED CLONE
              |                     |
        restricted model      richer object graph
              |                     |
        text representation     clone / transfer
              |                     |
        interoperability       runtime messaging
```

JSON:

```text
JS value
   ↓
restricted data model
   ↓
text
   ↓
parse
   ↓
plain data
```

Structured clone:

```text
source object graph
        ↓
graph serialization
        ↓
clone graph
        |
        +---- cycles preserved
        +---- aliases preserved
        +---- supported built-ins preserved
        +---- transferables may transfer ownership
```

Production boundary:

```text
raw input
   ↓
parse
   ↓
validate
   ↓
normalize
   ↓
deserialize
   ↓
domain object
```

Finally:

> Serialization is not a mirror of runtime state. It is a deliberate contract about which information crosses a boundary. The strongest engineers make that contract explicit—data shape, identity, precision, prototypes, binary representation, versioning, security, and lifecycle—rather than assuming a generic "deep clone" or `JSON.stringify()` will preserve semantics automatically.
