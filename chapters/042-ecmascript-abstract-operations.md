# Chapter 42 — ECMAScript Abstract Operations

## Chapter Metadata

```text
Chapter: 42
Title: ECMAScript Abstract Operations
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Define an ECMAScript abstract operation.
- Explain why the specification uses abstract operations.
- Distinguish abstract operations from:
  - JavaScript functions;
  - methods;
  - internal methods;
  - internal slots;
  - specification records.
- Read an abstract-operation algorithm systematically.
- Understand how abstract operations compose into larger semantic algorithms.
- Explain specification notation such as:
  - `Let`;
  - `If`;
  - `Return`;
  - `Assert`;
  - `?`;
  - `!`.
- Understand important abstract operations and operation families, including:
  - `Type`;
  - `ToPrimitive`;
  - `ToBoolean`;
  - `ToNumber`;
  - `ToNumeric`;
  - `ToBigInt`;
  - `ToString`;
  - `ToObject`;
  - `ToPropertyKey`;
  - `SameValue`;
  - `SameValueZero`;
  - `IsCallable`;
  - `IsConstructor`;
  - `Call`;
  - `Construct`;
  - `Get`;
  - `Set`;
  - `HasProperty`;
  - `DeletePropertyOrThrow`;
  - `CreateDataProperty`;
  - `CreateDataPropertyOrThrow`;
  - `GetMethod`;
  - `GetIterator`;
  - `GetAsyncIterator`;
  - completion-related operations.
- Explain how conversion chains work.
- Explain why `ToPrimitive` is central to coercion.
- Distinguish `ToNumber` from `ToNumeric`.
- Explain why BigInt requires distinct numeric semantics.
- Explain the role of `ToPropertyKey`.
- Understand equality-related abstract operations.
- Understand callability and constructability checks.
- Understand abstract `Get` and `Set` behavior versus direct property access intuition.
- Understand method retrieval and `this` preservation.
- Understand iterator acquisition as an abstract semantic process.
- Understand abrupt completion propagation through abstract operations.
- Trace compound language behavior by following abstract-operation chains.
- Predict surprising coercion and equality results.
- Implement simplified teaching versions of important abstract operations.
- Build an abstract-operation dependency graph.
- Use abstract operations to debug language semantics.
- Distinguish normative abstract-operation semantics from implementation details.
- Explain how abstract operations allow different language features to share consistent rules.
- Apply principal-level specification reasoning to difficult JavaScript behavior.

### Mastery Gate

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

Required:

- Chapter 02 — Values, Types, Type System
- Chapter 03 — Numbers, Floating Point, BigInt
- Chapter 04 — Strings, Unicode, Text Semantics
- Chapter 06 — Operators, Expressions
- Chapter 07 — Type Conversion, Coercion, Equality
- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 14 — `this` / Invocation / Binding
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 25 — Iterables / Iterators
- Chapter 29 — Errors / Error Handling
- Chapter 35 — Promises
- Chapter 41 — ECMAScript Specification Architecture

Builds directly toward:

- Chapter 43 — Ordinary Object Internal Methods
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 64 — ES Modules
- Chapter 90 — Modern ECMAScript Features
- Chapter 91 — TC39 Proposal Tracking
- Chapter 92 — Temporal
- Chapter 93 — Decorators
- Chapter 94 — Compatibility Engineering

---

## 3. What Is It?

An **abstract operation** is a specification-defined semantic algorithm used to express reusable language behavior.

It is called “abstract” because it is part of the specification's abstract machine rather than an ordinary JavaScript function exposed to program code.

A conceptual example:

```text
ToPropertyKey(argument)
```

can be invoked by many different language constructs.

Instead of separately defining property-key conversion in every feature, the specification defines one reusable operation.

The architecture is:

```text
Language feature
      ↓
Runtime semantics
      ↓
Abstract operation
      ↓
More abstract operations / internal methods
      ↓
Completion / result
```

Abstract operations are therefore the specification's reusable semantic vocabulary.

---

## 4. Why Does It Exist?

Without reusable abstract operations, the specification would repeatedly duplicate the same rules.

Imagine every feature separately explaining:

```text
How do I convert this value to a property key?
How do I determine whether this is callable?
How do I convert this value to a primitive?
How do I obtain an iterator?
How do I invoke this function?
```

That would create:

```text
duplication
+
inconsistency risk
+
harder maintenance
```

Abstract operations solve this by centralizing common semantic behavior.

They provide:

- reuse;
- precision;
- composability;
- consistency;
- easier specification evolution.

The key idea is:

> An abstract operation is a named semantic building block.

---

## 5. Mental Model

Think of abstract operations as the “standard library” of the ECMAScript specification.

Application code has:

```js
Math.max(...)
JSON.stringify(...)
Array.from(...)
```

The specification has semantic building blocks such as:

```text
ToPrimitive(...)
ToNumber(...)
ToString(...)
ToPropertyKey(...)
IsCallable(...)
Get(...)
Set(...)
Call(...)
Construct(...)
GetIterator(...)
```

The difference is important:

```text
JavaScript API
→ callable from application code

Abstract operation
→ callable by specification algorithms
```

A semantic graph can look like:

```text
          Property Access
                │
                ▼
           ToPropertyKey
                │
                ▼
               Get
                │
                ▼
             [[Get]]
                │
                ▼
            Prototype
             Lookup
```

Another:

```text
       Binary + Operation
              │
              ▼
        ToPrimitive
         /        \
        /          \
     left          right
       │             │
       ▼             ▼
    ToNumeric     ToNumeric
         \          /
          \        /
           \      /
            Add
```

This decomposition is the central skill of this chapter.

---

## 6. Core Rules

### Rule 1 — Abstract operations are specification mechanisms

They are not automatically exposed as JavaScript functions.

### Rule 2 — Abstract operations are reusable

Many language constructs call the same operation.

### Rule 3 — Abstract-operation results can be abrupt

An abstract operation may produce a completion that represents a failure.

### Rule 4 — Conversion is semantic behavior

Do not describe coercion as “JavaScript magically converts it”.

Identify the actual operation chain.

### Rule 5 — `ToPrimitive` often precedes further conversion

For object operands, primitive conversion can invoke user-defined behavior.

### Rule 6 — `ToNumeric` is broader than `ToNumber`

It can preserve BigInt as BigInt.

### Rule 7 — Property keys are Strings or Symbols

`ToPropertyKey` is central to computed property access.

### Rule 8 — `IsCallable` and `IsConstructor` are distinct

A value can be callable without being constructable, or constructable without the relevant call behavior assumptions.

### Rule 9 — Abstract `Call` is not ordinary method invocation syntax

It is a specification-level operation that defines how callable values are invoked.

### Rule 10 — Abstract `Get` and object `[[Get]]` are related but distinct concepts

An abstract operation can delegate to an internal method.

### Rule 11 — `GetMethod` can retrieve a method and preserve the receiver semantics required by subsequent invocation

### Rule 12 — Iterator acquisition is itself a semantic protocol

It involves obtaining the appropriate iterator method and invoking it correctly.

### Rule 13 — Abrupt completions propagate through `?`

This is central to reading algorithms.

### Rule 14 — `!` asserts normal completion

It is not error suppression.

### Rule 15 — Specification operations describe required semantics, not engine implementation

---

## 7. Syntax

Abstract operations are written conceptually like:

```text
OperationName(argument)
```

Examples:

```text
ToString(value)
ToNumber(value)
ToPrimitive(value)
ToPropertyKey(value)
IsCallable(value)
IsConstructor(value)
Get(object, propertyKey)
Set(object, propertyKey, value, receiver)
Call(func, thisValue, argumentsList)
Construct(constructor, argumentsList, newTarget)
```

A specification algorithm may use:

```text
Let primitive be ? ToPrimitive(value).
```

or:

```text
Let key be ? ToPropertyKey(argument).
```

The operation name does not imply:

```js
ToPropertyKey(...)
```

exists as a global JavaScript function.

---

## 8. Basic Examples

### Example 1 — ToBoolean

```js
Boolean(0);      // false
Boolean("x");    // true
Boolean(null);   // false
```

The semantic explanation is based on the language's Boolean-conversion rules.

### Example 2 — ToNumber

```js
Number("42");    // 42
Number(null);    // 0
Number(true);    // 1
```

### Example 3 — ToString

```js
String(42);      // "42"
String(false);   // "false"
```

### Example 4 — ToPropertyKey

```js
const key = {
  toString() {
    return "name";
  }
};

const obj = {
  name: 123
};

obj[key]; // 123
```

Conceptually:

```text
key
→ ToPropertyKey
→ ToPrimitive
→ String conversion
→ "name"
```

### Example 5 — IsCallable

```js
typeof function () {};
```

This is not itself the specification operation `IsCallable`, but callable status is represented by specification mechanisms rather than only by source syntax.

### Example 6 — Get

```js
obj.name
```

conceptually involves:

```text
Get(obj, "name")
```

which connects to the object's internal property semantics.

### Example 7 — Call

```js
fn(1, 2);
```

conceptually reaches:

```text
Call(fn, thisValue, [1, 2])
```

through the relevant runtime semantics.

---

## 9. Execution Walkthrough

Consider:

```js
const value = obj[key];
```

A simplified semantic chain:

```text
Evaluate base obj
      ↓
Evaluate property expression key
      ↓
ToPropertyKey(key)
      ↓
Get(obj, propertyKey)
      ↓
obj.[[Get]](propertyKey, receiver)
      ↓
Prototype-aware lookup
      ↓
Completion
      ↓
value
```

Now consider:

```js
const result = a + b;
```

For non-BigInt/non-string cases, a simplified conceptual chain can involve:

```text
Evaluate a
Evaluate b
      ↓
ToPrimitive(a)
ToPrimitive(b)
      ↓
depending on resulting primitive types
ToNumeric(...)
or string-concatenation path
      ↓
numeric/string operation
      ↓
result completion
```

The exact specification path depends on the operator and operand types.

---

## 10. Internal Mechanics

### 10.1 `Type`

The specification uses a `Type` operation to classify a specification value.

Conceptual categories include:

```text
Undefined
Null
Boolean
String
Symbol
Number
BigInt
Object
```

This is a specification operation, not equivalent to JavaScript's `typeof`.

Important distinction:

```text
specification Type(value)
≠
JavaScript typeof value
```

### 10.2 `ToPrimitive`

`ToPrimitive(input, preferredType)` converts an object into a primitive value.

Conceptually:

```text
Object
  ↓
existing primitive hint/strategy
  ↓
user-observable conversion hooks
  ↓
Primitive
```

Object conversion can involve:

```text
@@toPrimitive
valueOf
toString
```

depending on the relevant rules and ordering.

### 10.3 `ToBoolean`

Boolean conversion defines truthiness.

Examples include:

```text
false
0
-0
NaN
""
null
undefined
```

being falsy, while ordinary objects are truthy.

Do not describe this as arbitrary engine behavior.

### 10.4 `ToNumber`

Converts supported input values into Number semantics.

Examples:

```text
undefined → NaN
null      → +0
true      → 1
false     → +0
```

String conversion follows the specified numeric-string grammar/conversion rules.

### 10.5 `ToNumeric`

`ToNumeric` first handles primitive conversion as required, then produces:

```text
Number
or
BigInt
```

This is critical for arithmetic semantics.

### 10.6 `ToBigInt`

Converts permitted values into BigInt semantics and can throw for unsupported inputs.

Do not assume every Number can safely become BigInt.

### 10.7 `ToString`

Converts language values to String according to standardized semantics.

Object conversion can invoke primitive conversion first.

### 10.8 `ToObject`

Converts values into object form where the operation permits it.

For example:

```text
primitive
→ wrapper-like object semantics
```

But:

```text
null
undefined
```

can produce an abrupt completion in contexts that require an object.

### 10.9 `ToPropertyKey`

This is a foundational operation for property access.

Conceptually:

```text
input
 ↓
ToPrimitive(input, string)
 ↓
if Symbol → Symbol
else → ToString
 ↓
property key
```

Therefore:

```text
PropertyKey = String | Symbol
```

### 10.10 `SameValue`

Defines a sameness relation used in particular language/library semantics.

Important distinctions exist between:

```text
SameValue
SameValueZero
strict equality
abstract equality
```

### 10.11 `SameValueZero`

This relation treats:

```text
NaN as the same as NaN
```

and treats:

```text
+0 and -0
```

as the same.

This differs from some other equality relations.

### 10.12 `IsCallable`

Determines whether a value has callable behavior according to specification semantics.

This is more precise than “is it a function according to my intuition?”

### 10.13 `IsConstructor`

Determines whether a value supports construction semantics.

This is separate from callability.

### 10.14 `Call`

Conceptually:

```text
Call(F, thisArgument, argumentsList)
```

invokes a callable value according to the specified function-call semantics.

### 10.15 `Construct`

Conceptually:

```text
Construct(F, argumentsList, newTarget)
```

performs construction semantics.

Callability and constructability are therefore distinct capabilities.

### 10.16 `Get`

A high-level abstract property retrieval operation.

Conceptually:

```text
Get(O, P)
→ O.[[Get]](P, O)
```

with the exact specification context determining details.

### 10.17 `Set`

A high-level abstract property assignment operation.

Conceptually:

```text
Set(O, P, V, Throw)
```

can delegate to:

```text
O.[[Set]](...)
```

The receiver argument matters in some property-assignment/inheritance cases.

### 10.18 `HasProperty`

Determines whether a property exists according to object/prototype semantics.

This differs from simply checking own properties.

### 10.19 `GetMethod`

Conceptually:

```text
GetMethod(V, P)
```

obtains a property and checks whether the resulting value is callable when one is required.

This appears frequently in protocol-driven behavior.

### 10.20 Iterator Operations

Important operations include conceptual mechanisms for:

```text
GetIterator
GetAsyncIterator
IteratorNext
IteratorComplete
IteratorValue
IteratorClose
```

These compose the iterable protocol.

### 10.21 Completion Operations

Abstract operations frequently interact with completion machinery.

A useful pattern:

```text
Operation
→ Completion
→ ? propagates abrupt
→ ! assumes normal
```

---

## 11. ECMAScript / Specification Semantics

### 11.1 Why `Type` matters

Specification algorithms often branch by semantic type.

Conceptually:

```text
if Type(value) is String
if Type(value) is BigInt
if Type(value) is Object
```

This is more precise than source-level `typeof`.

### 11.2 Conversion stack

Many coercions form chains:

```text
Object
 ↓
ToPrimitive
 ↓
Primitive
 ↓
ToNumber / ToString / ToNumeric
 ↓
Target semantic domain
```

Understanding the chain is more important than memorizing isolated outcomes.

### 11.3 Preferred type hint

Object-to-primitive conversion can involve a preferred type hint.

Conceptually:

```text
string hint
number/default hint
```

This affects which conversion methods are considered first, subject to the exact specification algorithm.

### 11.4 `@@toPrimitive`

An object can provide:

```js
[Symbol.toPrimitive](hint) {
  ...
}
```

This gives user code explicit participation in primitive conversion.

### 11.5 Fallback conversion

If the specialized primitive-conversion mechanism does not provide a result, ordinary conversion methods can participate according to the specification.

### 11.6 Property-key conversion

A computed access:

```js
obj[key]
```

requires a property key.

An object key can therefore trigger user code during conversion.

### 11.7 Equality relations

JavaScript contains multiple comparison semantics.

Conceptually:

```text
==       → abstract equality semantics
===      → strict equality semantics
Object.is → SameValue-like semantics
Map/Set    → uses SameValueZero-style key equality in relevant semantics
```

Use exact specification relations when precision matters.

### 11.8 Numeric domain selection

For arithmetic, the semantic domain may be:

```text
Number
or
BigInt
```

Mixing domains can produce an exception rather than implicit cross-domain arithmetic.

### 11.9 Callable versus constructable

A value can satisfy:

```text
IsCallable
```

without satisfying:

```text
IsConstructor
```

This distinction explains why:

```js
fn()
```

and:

```js
new fn()
```

can have different validity.

### 11.10 Get/Set and prototypes

High-level property operations can delegate to internal methods and therefore interact with:

```text
own properties
prototype chain
descriptors
accessors
Proxy behavior
```

### 11.11 Iterator protocol

A construct such as:

```js
for (const x of iterable) {
}
```

depends on iterator acquisition and iteration abstract operations.

### 11.12 Async iterator protocol

Similarly:

```js
for await (const x of iterable) {
}
```

uses asynchronous iterator semantics.

### 11.13 Iterator closing

Abrupt exits can trigger iterator-closing behavior where required.

This connects abstract operations with cleanup semantics.

---

## 12. Advanced Behavior

### 12.1 Coercion can invoke user code

Consider:

```js
const value = {
  valueOf() {
    console.log("valueOf");
    return 10;
  }
};

value + 1;
```

The conversion is not a passive bit-level transformation.

It can execute code.

### 12.2 `Symbol.toPrimitive` can override fallback paths

```js
const value = {
  [Symbol.toPrimitive](hint) {
    console.log(hint);
    return 10;
  }
};
```

Now primitive conversion can follow that protocol.

### 12.3 ToNumeric and BigInt

Consider:

```js
1n + 2n;
```

The numeric domain remains BigInt.

But:

```js
1n + 2;
```

does not perform automatic mixed-domain arithmetic.

### 12.4 ToPropertyKey can execute code

```js
const key = {
  toString() {
    console.log("toString");
    return "x";
  }
};

obj[key];
```

Property access can therefore have side effects before the actual property lookup.

### 12.5 `-0`

Different equality relations treat:

```text
+0
-0
```

differently.

This matters for:

```js
Object.is(-0, 0); // false
```

while strict equality treats them as equal.

### 12.6 `NaN`

`NaN` has unusual comparison semantics:

```js
NaN === NaN; // false
Object.is(NaN, NaN); // true
```

The distinction comes from different semantic relations.

### 12.7 Method retrieval and receiver

Consider:

```js
obj.method();
```

The method value and the reference/receiver relationship matter.

Extracting it changes semantics:

```js
const method = obj.method;
method();
```

The semantic path is not identical.

### 12.8 Getters

`Get` can result in getter execution.

Therefore property access can run arbitrary user code.

### 12.9 Setters

`Set` can invoke setters.

### 12.10 Proxy interaction

Abstract property operations may eventually invoke Proxy-specific internal methods.

This is why:

```js
proxy.x
```

cannot always be reasoned about as ordinary data-property retrieval.

### 12.11 Property existence

`HasProperty` may succeed through the prototype chain.

Compare conceptually:

```text
HasOwnProperty
vs
HasProperty
```

### 12.12 `GetMethod`

Protocol-driven language behavior often needs:

```text
get property
→ if undefined/null, treat as absent
→ otherwise require callable value
```

This is why method-retrieval abstractions are important.

### 12.13 Iterator acquisition can fail

An iterable can have a malformed or throwing iterator method.

### 12.14 Iterator result validation

The iterator protocol expects a result object satisfying required semantics.

### 12.15 Iterator closing

A loop exited through:

```text
break
throw
return
```

may need to close the iterator.

### 12.16 Async iterator bridging

Some constructs can adapt synchronous iterables into asynchronous iteration semantics.

### 12.17 Completion propagation through layers

Suppose:

```text
ToPrimitive
→ calls user method
→ method throws
```

Then:

```text
ToPrimitive → abrupt completion
→ outer operation sees abrupt
→ ? propagates
→ caller observes throw
```

### 12.18 Abstract operations are not always “small”

Some can contain extensive algorithms and invoke many other operations.

### 12.19 Abstract operation composition

A high-level operation can be a semantic coordinator:

```text
GetMethod
→ Get
→ IsCallable
```

This allows language features to reuse guarantees.

### 12.20 Observable side effects

An abstract operation chain can invoke:

```text
getters
setters
proxy traps
custom primitive conversion
iterators
user functions
```

Therefore semantic tracing can reveal hidden side effects.

### 12.21 Error timing

The exact point at which an abstract operation is invoked can determine:

```text
when
and
where
```

an exception occurs.

### 12.22 Receiver semantics

Property setting/getting can distinguish:

```text
target object
receiver object
```

This matters in inherited accessors and Proxy behavior.

### 12.23 Throw-or-return policy

Some abstract operations use a flag or contextual policy deciding whether failure:

```text
throws
or
returns failure indication
```

Understanding that context is crucial.

### 12.24 Completion values

An abstract operation may return:

```text
normal completion carrying value
```

rather than directly returning the language value in specification machinery.

### 12.25 Type errors from capability checks

Operations such as:

```text
IsCallable
IsConstructor
```

can determine whether a later operation is valid.

---

## 13. Edge Cases

- `typeof null` does not mean specification `Type(null)` is “Object”.
- `Type(value)` is not the same operation as JavaScript `typeof`.
- Object-to-primitive conversion can execute user code.
- `Symbol.toPrimitive` can control primitive conversion.
- `valueOf()` and `toString()` can have side effects and can throw.
- `ToNumber` and `ToNumeric` are not interchangeable.
- BigInt and Number arithmetic are distinct numeric domains.
- Property-key conversion can invoke user code.
- Symbols can remain Symbols through `ToPropertyKey`.
- Strings become property keys through string conversion.
- `SameValue`, `SameValueZero`, `===`, and `==` are different relations.
- `+0` and `-0` need special attention.
- `NaN` needs special attention.
- A callable value need not be constructable.
- A constructable value need not be treated as an ordinary callable function in every context.
- Property retrieval can invoke getters.
- Property assignment can invoke setters.
- Proxy traps can alter property behavior.
- `HasProperty` can find inherited properties.
- `GetMethod` may reject a non-callable property value.
- Iterator acquisition can throw.
- Iterator results can be malformed.
- Iterator closing can itself throw.
- Async iteration introduces additional asynchronous completion behavior.
- Abrupt completion can escape through many nested abstract operations.
- Specification operations are not necessarily one-to-one with engine functions.
- A specification algorithm may use mathematical/specification records absent from source code.
- An operation's surrounding context determines whether an abrupt completion is propagated, handled, or transformed.

---

## 14. Common Misconceptions

### Misconception 1 — “Abstract operations are hidden global JavaScript functions.”

No.

They are specification algorithms.

### Misconception 2 — “`ToNumber()` is the same as `Number()`.”

Related conceptually, but not identical as specification and API constructs.

### Misconception 3 — “`ToNumeric()` always produces Number.”

No.

It can preserve BigInt semantics.

### Misconception 4 — “All objects convert to strings immediately.”

No.

Primitive conversion has its own algorithm.

### Misconception 5 — “Property keys can be any JavaScript value.”

No. After property-key conversion, the property key is a String or Symbol.

### Misconception 6 — “`===` is the only equality relation.”

No.

The language has multiple semantic comparison relations.

### Misconception 7 — “If something is callable, `new` will work.”

Not necessarily.

### Misconception 8 — “`Get()` directly reads a field.”

It represents a semantic operation that may invoke accessors, prototypes, or Proxy behavior.

### Misconception 9 — “Iterator acquisition is just a loop optimization.”

No. It is a language protocol.

### Misconception 10 — “`?` catches errors.”

No. It propagates abrupt completion.

### Misconception 11 — “`!` makes errors disappear.”

No. It assumes normal completion in the current semantic context.

### Misconception 12 — “Abstract operations are implementation details.”

No. They are normative specification machinery.

### Misconception 13 — “Every abstract operation has a direct JavaScript API equivalent.”

No.

### Misconception 14 — “Specification operations execute literally inside V8.”

No.

### Misconception 15 — “Knowing the final output is enough.”

Principal-level understanding requires explaining the semantic path that produced it.

---

## 15. Common Mistakes

1. Memorizing conversion tables without learning operation chains.
2. Treating `typeof` as the specification `Type` operation.
3. Treating `Number()` as a literal implementation of `ToNumber`.
4. Ignoring `ToPrimitive`.
5. Ignoring `ToPropertyKey`.
6. Ignoring BigInt/Number domain distinctions.
7. Conflating `SameValue`, `SameValueZero`, strict equality, and abstract equality.
8. Treating `Get` as a simple field lookup.
9. Forgetting getters/setters.
10. Forgetting Proxy behavior.
11. Confusing `IsCallable` with `IsConstructor`.
12. Ignoring iterator acquisition.
13. Ignoring iterator closing.
14. Ignoring abrupt completion propagation.
15. Reading `?` and `!` incorrectly.
16. Assuming every abstract operation maps to one runtime function.
17. Treating specification records as application objects.
18. Reasoning from engine implementation before determining normative semantics.

---

## 16. Comparison With Related Concepts

| Concept | Layer | Purpose |
|---|---|---|
| Abstract operation | Specification | Reusable semantic algorithm |
| Internal method | Specification | Object semantic operation |
| Internal slot | Specification | Hidden object state |
| JavaScript function | Language/application | Callable source-level value |
| Built-in API | Language/host | Program-facing capability |
| Specification Record | Specification | Semantic bookkeeping |
| Completion Record | Specification | Control-flow/result representation |

### Abstract operation vs JavaScript function

```text
Abstract operation:
ToString(x)

JavaScript:
String(x)
```

The first defines semantics inside the specification.

The second is a callable language API.

### Abstract operation vs internal method

```text
Get(O, P)
```

is an abstract operation.

```text
O.[[Get]](P, Receiver)
```

is an internal-method operation.

The abstract operation can coordinate higher-level behavior and delegate to the internal method.

### `Type` vs `typeof`

```text
Type(value)
```

is specification machinery.

```js
typeof value
```

is a source-level operator with its own specified semantics.

### `SameValue` vs `===`

These represent different equality relations.

### `SameValueZero` vs `SameValue`

Important distinction:

```text
SameValueZero(+0, -0) → true
SameValue(+0, -0)     → false
```

### `ToNumber` vs `ToNumeric`

```text
ToNumber → Number domain
ToNumeric → Number or BigInt domain
```

### `Get` vs `HasProperty`

```text
Get → retrieve a value
HasProperty → determine whether property exists
```

---

## 17. Performance Considerations

Abstract operations describe semantics, not literal runtime cost.

### 17.1 Conversion overhead

Conversions may be optimized, but user-defined behavior can prevent simplification.

### 17.2 Property access

`Get`/`Set` semantics may map to optimized inline caches or specialized paths in an engine.

### 17.3 Proxies

Proxy observability can limit certain optimizations.

### 17.4 Getters/setters

Property access that can invoke user code has different optimization constraints from simple data-property reads.

### 17.5 Iterator protocol overhead

Generic iteration can involve:

```text
method lookup
call
result object handling
```

Engines may optimize common cases.

### 17.6 Coercion

Repeated conversion can be expensive, especially when it invokes user code.

### 17.7 BigInt

BigInt arithmetic has a different cost model from Number arithmetic.

### 17.8 Specification size versus runtime cost

A long abstract-operation chain does not necessarily mean many literal runtime function calls.

### 17.9 Measurement discipline

For performance questions:

```text
specification
→ engine behavior
→ benchmark/profile
```

must remain separate.

---

## 18. Memory Considerations

Abstract operations do not directly dictate heap layout.

However, their semantics can imply observable allocation or retention behavior.

Potentially relevant areas:

```text
ToObject
temporary objects
iterator result objects
boxed primitives
closures invoked during conversion
Proxy state
```

Do not infer exact allocation counts from specification pseudocode alone.

For physical allocation behavior, consult engine-level knowledge and profiling.

---

## 19. Security Considerations

Abstract operations reveal hidden execution boundaries.

### 19.1 Coercion can run code

Never assume:

```js
obj + value
```

is side-effect free.

### 19.2 Property access can run code

Getters and Proxy traps can execute code.

### 19.3 Property-key conversion can run code

Computed properties can trigger conversion hooks.

### 19.4 Iterator protocols can execute user code

Iteration invokes protocol methods.

### 19.5 Capability checks

Understanding `IsCallable` and `IsConstructor` helps reason about dynamic invocation.

### 19.6 Prototype behavior

`Get` and `HasProperty` can traverse prototypes.

### 19.7 Denial of service

Adversarial coercion/iterator/property operations can create unexpected work.

### 19.8 Error propagation

Unhandled abrupt completion can terminate critical operations.

### 19.9 Host boundaries

Security properties involving browser or OS isolation require host/runtime standards, not ECMAScript abstract operations alone.

---

## 20. Production Usage

### 20.1 Specification-driven bug analysis

For surprising behavior:

```text
source expression
→ identify semantic construct
→ identify abstract operation
→ follow nested operations
→ find first divergence from intuition
→ reproduce
```

### 20.2 Polyfills

Abstract-operation understanding helps implement standard behavior rather than approximating surface syntax.

### 20.3 Library design

Library authors should understand what seemingly harmless operations can trigger:

```text
coercion
getters
setters
iterators
proxies
```

### 20.4 Serialization and validation

Implicit conversions can cause side effects if untrusted objects are processed.

### 20.5 Defensive coding

Where semantics are security-sensitive, make conversions explicit.

### 20.6 Code review

Reviewers should ask:

```text
Can this conversion execute user code?
Can this property access invoke a getter?
Can this iterator be hostile?
Can this value be callable/constructable?
```

### 20.7 Cross-engine behavior

When standardized semantics are clear, use them as the baseline before investigating implementation differences.

---

## 21. Implementation From Scratch

The implementation exercises are deliberately educational.

### Stage 1 — `Type`

Implement a teaching function:

```js
function specType(value) {
  // return a semantic type label
}
```

Do not make it identical to `typeof`.

### Stage 2 — `ToBoolean`

Implement:

```js
function toBoolean(value) {
  // specification-inspired truthiness conversion
}
```

Test all important falsy values.

### Stage 3 — `ToPrimitive`

Implement a simplified model supporting:

```text
Symbol.toPrimitive
valueOf
toString
```

with explicit hint handling.

### Stage 4 — `ToNumber`

Implement:

```js
function toNumber(value) {
  // teaching model only
}
```

Handle:

```text
undefined
null
boolean
number
string
```

and explicitly reject unsupported object cases unless conversion is provided.

### Stage 5 — `ToNumeric`

Implement:

```js
function toNumeric(value) {
  // return Number or BigInt semantics
}
```

### Stage 6 — `ToPropertyKey`

Implement:

```js
function toPropertyKey(value) {
  // primitive conversion
  // preserve Symbol
  // otherwise String conversion
}
```

### Stage 7 — `SameValue`

Implement a teaching version that distinguishes:

```text
NaN
+0
-0
```

### Stage 8 — `Get`

Build:

```js
function specGet(object, propertyKey, receiver = object) {
  // teaching model
}
```

Connect it to prototype lookup and accessors.

### Stage 9 — Iterator Acquisition

Implement a simplified:

```text
GetIterator
IteratorNext
IteratorClose
```

model.

### Stage 10 — Semantic Trace Engine

Given:

```js
obj[key]
```

produce:

```text
Evaluate base
→ Evaluate key
→ ToPropertyKey
→ Get
→ [[Get]]
→ prototype path
→ result/completion
```

---

## 22. Debugging Exercises

### Exercise 1 — `ToPrimitive`

Explain why:

```js
const x = {
  valueOf() {
    console.log("valueOf");
    return 10;
  }
};

x + 1;
```

can execute user code.

### Exercise 2 — Symbol conversion

Explain:

```js
const key = Symbol("x");
obj[key];
```

Why does property-key conversion preserve the Symbol?

### Exercise 3 — Object key

Explain:

```js
const key = {
  toString() {
    console.log("key");
    return "x";
  }
};

obj[key];
```

### Exercise 4 — BigInt

Explain:

```js
1n + 2n;
```

versus:

```js
1n + 2;
```

### Exercise 5 — Equality

Explain:

```js
Object.is(NaN, NaN);
Object.is(-0, 0);
NaN === NaN;
```

using the distinct equality relations.

### Exercise 6 — Callable vs constructor

Create values for which:

```text
IsCallable = true
IsConstructor = false
```

and explain the resulting source-level consequences.

### Exercise 7 — Getter

Explain:

```js
const obj = {
  get x() {
    console.log("get");
    return 1;
  }
};

obj.x;
```

### Exercise 8 — Prototype property

Explain:

```js
const proto = { x: 1 };
const obj = Object.create(proto);

obj.x;
```

using `Get`/`[[Get]]` reasoning.

### Exercise 9 — Iterator

Create an iterable whose iterator method throws.

Trace the abstract-operation path.

### Exercise 10 — Abrupt completion

Create a `Symbol.toPrimitive` implementation that throws.

Trace how the abrupt completion propagates to the outer expression.

---

## 23. Code Review Exercise

Review:

```js
function normalizeKey(key) {
  return String(key);
}

function getValue(object, key) {
  return object[normalizeKey(key)];
}
```

Determine whether this is semantically equivalent to generic property-key conversion.

Analyze cases involving:

```text
Symbol
object with Symbol.toPrimitive
object with throwing toString
object with custom valueOf
```

Then redesign the function if exact property-key semantics are required.

---

## 24. Interview Questions

### Foundational

1. What is an abstract operation?
2. Why does ECMAScript use abstract operations?
3. Are abstract operations JavaScript functions?
4. What is `Type`?
5. What is `ToPrimitive`?
6. What is `ToBoolean`?
7. What is `ToNumber`?
8. What is `ToNumeric`?
9. What is `ToString`?
10. What is `ToPropertyKey`?

### Intermediate

11. What is the difference between `ToNumber` and `ToNumeric`?
12. Why does `ToPrimitive` matter?
13. How can `Symbol.toPrimitive` affect language behavior?
14. What is `SameValue`?
15. What is `SameValueZero`?
16. How do they differ from `===`?
17. What is `IsCallable`?
18. What is `IsConstructor`?
19. What is `Get`?
20. What is `Set`?

### Advanced

21. Explain `GetMethod`.
22. Explain iterator acquisition using abstract operations.
23. Explain iterator closing.
24. Explain `?` and abrupt-completion propagation.
25. Explain `!`.
26. Explain receiver semantics in `Get`/`Set`.
27. Explain how Proxy behavior participates in abstract property operations.
28. Explain how coercion can execute user code.
29. Explain why specification operations are not one-to-one with engine functions.
30. Explain the relationship between abstract operations and internal methods.

### Principal-Level

31. Trace `obj[key]` from source syntax through abstract operations.
32. Trace `a + b` through conversion decisions.
33. Explain exactly where user-defined coercion can execute.
34. Design a semantic dependency graph for a difficult expression.
35. Explain a surprising equality result using the correct semantic relation.
36. Design a conformance test for a conversion operation.
37. Determine whether a library's “simplified conversion” is standards-equivalent.
38. Separate specification semantics from V8 implementation behavior.
39. Explain why abstract operations are critical to specification maintainability.
40. Teach `ToPrimitive` to another principal engineer without relying on memorized conversion tables.

---

## 25. Predict-the-Output Exercises

For every exercise:

```text
Predict
→ Run
→ Compare
→ Trace abstract operations
→ Explain
```

### Exercise A

```js
console.log(Number(null));
console.log(Number(undefined));
console.log(Number(true));
```

Identify the conversion paths.

### Exercise B

```js
const value = {
  valueOf() {
    console.log("valueOf");
    return 10;
  }
};

console.log(value + 1);
```

Predict the side-effect ordering.

### Exercise C

```js
const value = {
  [Symbol.toPrimitive](hint) {
    console.log(hint);
    return 10;
  }
};

console.log(value + 1);
```

Predict the hint.

### Exercise D

```js
const key = {
  toString() {
    console.log("key");
    return "x";
  }
};

const obj = { x: 10 };

console.log(obj[key]);
```

Trace `ToPropertyKey`.

### Exercise E

```js
console.log(Object.is(NaN, NaN));
console.log(Object.is(-0, 0));
console.log(NaN === NaN);
console.log(-0 === 0);
```

Identify which semantic equality relations explain each result.

### Exercise F

```js
const obj = {
  get x() {
    console.log("get");
    return 1;
  }
};

console.log(obj.x);
```

Trace property retrieval.

### Exercise G

```js
const proto = { x: 10 };
const obj = Object.create(proto);

console.log(obj.x);
```

Trace prototype-aware `Get`.

### Exercise H

```js
const iterable = {
  [Symbol.iterator]() {
    throw new Error("boom");
  }
};

try {
  for (const value of iterable) {
    console.log(value);
  }
} catch {
  console.log("caught");
}
```

Trace iterator acquisition and abrupt completion.

---

## 26. Mastery Exercises

### Exercise 1 — Conversion Library

Implement teaching versions of:

```text
Type
ToBoolean
ToPrimitive
ToNumber
ToNumeric
ToBigInt
ToString
ToObject
ToPropertyKey
```

### Exercise 2 — Equality Library

Implement:

```text
SameValue
SameValueZero
```

and compare them with:

```text
===
==
```

through test cases.

### Exercise 3 — Property Operations

Implement:

```text
Get
Set
HasProperty
```

with prototype and accessor support.

### Exercise 4 — Callability

Implement teaching checks for:

```text
IsCallable
IsConstructor
```

and create a table of different JavaScript values.

### Exercise 5 — Iterator Model

Implement:

```text
GetIterator
IteratorNext
IteratorComplete
IteratorValue
IteratorClose
```

### Exercise 6 — Abrupt Completion

Connect every implemented abstract operation to a teaching Completion Record model.

### Exercise 7 — Semantic Trace

Build a tracer for:

```js
obj[key]
a + b
fn(x)
new Fn(x)
for (const x of iterable) {}
```

### Exercise 8 — Conformance Test Suite

Create tests for:

```text
NaN
+0
-0
BigInt
Symbol
getters
setters
proxies
custom primitive conversion
iterators
throwing conversions
```

### Exercise 9 — Specification Cross-Reference

For ten difficult JavaScript behaviors, identify:

```text
runtime syntax
→ abstract operation
→ internal method
→ completion path
→ observable result
```

### Exercise 10 — Principal Challenge

Take a surprising expression and produce a full semantic proof:

```text
source
→ syntax
→ evaluation
→ abstract operations
→ internal methods
→ user-code callbacks
→ completion propagation
→ final result
```

---

## 27. Key Takeaways

1. Abstract operations are reusable semantic algorithms in the ECMAScript specification.
2. They are not automatically JavaScript functions.
3. They provide a common semantic vocabulary.
4. `Type` is specification machinery and is not the same as `typeof`.
5. `ToPrimitive` is central to object coercion.
6. Object-to-primitive conversion can execute user code.
7. `ToBoolean` defines language truthiness.
8. `ToNumber` targets the Number semantic domain.
9. `ToNumeric` can produce Number or BigInt.
10. `ToPropertyKey` converts values into String-or-Symbol property keys.
11. `SameValue`, `SameValueZero`, `===`, and `==` represent distinct semantic relations.
12. `IsCallable` and `IsConstructor` answer different capability questions.
13. `Call` and `Construct` represent different invocation paths.
14. `Get`, `Set`, and `HasProperty` are high-level semantic property operations.
15. These operations can reach internal methods and therefore prototypes, accessors, and Proxy behavior.
16. Iterator acquisition is defined through reusable abstract operations.
17. Iterator closing is part of correctness and cleanup.
18. `?` propagates abrupt completion.
19. `!` assumes normal completion.
20. Abstract operations are about semantics, not physical engine implementation.
21. The central principle is:

> When JavaScript behavior seems magical, look for the abstract-operation chain. The language is often not performing one mysterious action; it is composing small, precisely defined semantic operations.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values, Types, Type System
- Chapter 03 — Numbers, Floating Point, BigInt
- Chapter 04 — Strings, Unicode, Text Semantics
- Chapter 06 — Operators, Expressions
- Chapter 07 — Type Conversion, Coercion, Equality
- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 14 — `this` / Invocation / Binding
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 25 — Iterables / Iterators
- Chapter 29 — Errors / Error Handling
- Chapter 35 — Promises
- Chapter 41 — ECMAScript Specification Architecture

### Builds Toward

- Chapter 43 — Ordinary Object Internal Methods
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 64 — ES Modules
- Chapter 90 — Modern ECMAScript Features
- Chapter 91 — TC39 Proposal Tracking
- Chapter 92 — Temporal
- Chapter 93 — Decorators
- Chapter 94 — Compatibility Engineering

### Related Concepts

- Type
- ToPrimitive
- ToBoolean
- ToNumber
- ToNumeric
- ToBigInt
- ToString
- ToObject
- ToPropertyKey
- SameValue
- SameValueZero
- IsCallable
- IsConstructor
- Call
- Construct
- Get
- Set
- HasProperty
- GetMethod
- GetIterator
- GetAsyncIterator
- IteratorClose
- Completion
- Internal Methods
- Internal Slots
- Specification Records

### Concepts Revisited

This chapter revisits:

- coercion;
- equality;
- property access;
- prototypes;
- functions;
- symbols;
- iteration;
- errors;
- completion propagation.

### Why This Chapter Matters Later

Chapter 42 is the semantic connective tissue between the specification architecture of Chapter 41 and the object internals of Chapter 43.

Chapter 41 taught:

```text
how the specification is structured
```

Chapter 42 now teaches:

```text
how the specification actually composes behavior
```

Once the learner can follow:

```text
source
→ runtime semantics
→ abstract operation
→ internal method
→ completion
```

many of JavaScript's “weird” behaviors stop being memorized exceptions and become derivable consequences.

---

## 29. Completion Criteria

### Conceptual Understanding

- [ ] Define abstract operation.
- [ ] Explain why abstract operations exist.
- [ ] Explain `Type`.
- [ ] Explain `ToPrimitive`.
- [ ] Explain `ToBoolean`.
- [ ] Explain `ToNumber`.
- [ ] Explain `ToNumeric`.
- [ ] Explain `ToBigInt`.
- [ ] Explain `ToString`.
- [ ] Explain `ToObject`.
- [ ] Explain `ToPropertyKey`.
- [ ] Explain `SameValue`.
- [ ] Explain `SameValueZero`.
- [ ] Explain `IsCallable`.
- [ ] Explain `IsConstructor`.
- [ ] Explain `Call`.
- [ ] Explain `Construct`.
- [ ] Explain `Get`.
- [ ] Explain `Set`.
- [ ] Explain `HasProperty`.
- [ ] Explain `GetMethod`.
- [ ] Explain iterator acquisition.
- [ ] Explain abrupt completion propagation.

### Predictive Mastery

- [ ] Predict primitive conversion.
- [ ] Predict custom `Symbol.toPrimitive` behavior.
- [ ] Predict property-key conversion.
- [ ] Predict `NaN` equality behavior.
- [ ] Predict `+0` / `-0` equality behavior.
- [ ] Predict Number vs BigInt arithmetic.
- [ ] Predict getter/setter execution.
- [ ] Predict prototype-aware property access.
- [ ] Predict callable vs constructable behavior.
- [ ] Predict iterator-acquisition failures.

### Implementation

- [ ] Implement `Type`.
- [ ] Implement `ToBoolean`.
- [ ] Implement `ToPrimitive`.
- [ ] Implement `ToNumber`.
- [ ] Implement `ToNumeric`.
- [ ] Implement `ToString`.
- [ ] Implement `ToPropertyKey`.
- [ ] Implement `SameValue`.
- [ ] Implement `SameValueZero`.
- [ ] Implement simplified `Get`/`Set`.
- [ ] Implement simplified iterator operations.
- [ ] Build semantic tracing.

### Debugging

- [ ] Trace coercion.
- [ ] Trace property-key conversion.
- [ ] Trace property access.
- [ ] Trace equality.
- [ ] Trace getter/setter execution.
- [ ] Trace Proxy interactions.
- [ ] Trace iterator acquisition.
- [ ] Trace abrupt completion.
- [ ] Identify the first surprising abstract operation.
- [ ] Separate semantic rules from engine implementation.

### Production Engineering

- [ ] Identify hidden user-code execution during coercion.
- [ ] Identify getter/setter side effects.
- [ ] Identify iterator side effects.
- [ ] Review implicit conversion in security-sensitive code.
- [ ] Use specification reasoning for compatibility/polyfill work.

### Interview Readiness

- [ ] Explain abstract operations.
- [ ] Explain `ToPrimitive`.
- [ ] Explain `ToNumeric`.
- [ ] Explain `ToPropertyKey`.
- [ ] Explain equality relations.
- [ ] Explain `IsCallable` vs `IsConstructor`.
- [ ] Explain `Get`/`Set`.
- [ ] Explain iterator operations.
- [ ] Trace a complex semantic chain.
- [ ] Defend the semantic explanation at principal level.

### Track A — Core Theory

- [ ] Abstract operation model understood.
- [ ] Conversion family understood.
- [ ] Property-operation family understood.
- [ ] Invocation family understood.
- [ ] Iterator family understood.
- [ ] Completion propagation understood.

### Track B — Implementation

- [ ] Guided implementations completed.
- [ ] Partially guided implementations completed.
- [ ] No-reference implementations completed.
- [ ] Edge-case hardened implementations completed.
- [ ] Semantic trace tool reviewed.

### Track C — Interview / Reasoning

- [ ] Prediction exercises completed.
- [ ] Conversion debugging completed.
- [ ] Property-semantics debugging completed.
- [ ] Equality reasoning completed.
- [ ] Iterator reasoning completed.
- [ ] Principal-level semantic proof completed.

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

# Chapter 42 — Revision / Retrieval Record

## Retrieval Prompts

1. What is an abstract operation?
2. Why does ECMAScript use abstract operations?
3. Are abstract operations JavaScript functions?
4. What is the specification `Type` operation?
5. How is `Type` different from `typeof`?
6. What does `ToPrimitive` do?
7. How can object conversion execute user code?
8. What is `Symbol.toPrimitive`?
9. What does `ToBoolean` do?
10. What does `ToNumber` do?
11. What does `ToNumeric` do?
12. Why does `ToNumeric` matter for BigInt?
13. What does `ToPropertyKey` do?
14. Why are property keys Strings or Symbols?
15. What is `SameValue`?
16. What is `SameValueZero`?
17. How do they differ from `===`?
18. What is `IsCallable`?
19. What is `IsConstructor`?
20. Why are callability and constructability separate?
21. What is `Call`?
22. What is `Construct`?
23. What is `Get`?
24. What is `Set`?
25. What is `HasProperty`?
26. What is `GetMethod`?
27. How does iterator acquisition work?
28. What is iterator closing?
29. How does `?` propagate abrupt completion?
30. How does `!` differ?
31. How can getters and Proxy traps become visible through `Get`?
32. How can property-key conversion execute user code?
33. How would you trace `obj[key]`?
34. How would you trace `a + b`?
35. How would you investigate a surprising coercion result?

## Weak Areas

```text
-
-
-
```

## Revision Queue

```text
- [ ] Revisit specification Type
- [ ] Revisit ToPrimitive
- [ ] Revisit ToBoolean
- [ ] Revisit ToNumber
- [ ] Revisit ToNumeric
- [ ] Revisit ToBigInt
- [ ] Revisit ToString
- [ ] Revisit ToObject
- [ ] Revisit ToPropertyKey
- [ ] Revisit SameValue
- [ ] Revisit SameValueZero
- [ ] Revisit IsCallable / IsConstructor
- [ ] Revisit Call / Construct
- [ ] Revisit Get / Set / HasProperty
- [ ] Revisit GetMethod
- [ ] Revisit Iterator operations
- [ ] Revisit abrupt completion propagation
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

# Chapter 42 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — primary source for abstract operations and normative language semantics.
2. TC39 materials — feature evolution and specification design context.
3. Browser/WHATWG standards — host behavior that interacts with ECMAScript.
4. Node.js official documentation — runtime behavior beyond the language.
5. Engine documentation/source — implementation mechanisms.
6. Developer documentation such as MDN — practical explanations.

Always distinguish:

```text
abstract operation
vs
JavaScript API
vs
internal method
vs
engine implementation
```

For difficult behavior, use:

```text
source construct
→ runtime semantic rule
→ abstract-operation chain
→ internal method where applicable
→ completion path
→ observable result
```

Do not claim that an abstract operation is literally implemented as one JavaScript or C++ function in every engine.

Do not replace normative specification semantics with one engine's implementation behavior.

---

# Chapter 42 — Completion Snapshot

```text
Chapter: 42
Title: ECMAScript Abstract Operations
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```