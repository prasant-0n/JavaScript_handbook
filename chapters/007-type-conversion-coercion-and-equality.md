
# Chapter 07 --- Type Conversion, Coercion, and Equality

> **Status:** `[+] Completed`\
> **Role in curriculum:** Explains how JavaScript moves between types
> and how equality is actually defined. This chapter connects the type
> system and operators to the specification's abstract
> conversion/equality algorithms and provides the foundation for
> understanding `==`, `===`, `Object.is`, truthiness, numeric
> conversion, string conversion, primitive conversion, BigInt
> interactions, property-key conversion, and the many "weird JavaScript"
> examples that are actually deterministic rules.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain the difference between explicit conversion and implicit
    coercion.
-   Explain why JavaScript needs conversion operations.
-   Distinguish:
    -   `ToPrimitive`
    -   `ToBoolean`
    -   `ToNumber`
    -   `ToString`
    -   `ToBigInt`
    -   `ToObject`
    -   property-key conversion concepts.
-   Explain how primitive conversion affects object operands.
-   Explain the role of `Symbol.toPrimitive`.
-   Explain value conversion in arithmetic and comparison operations.
-   Explain truthy and falsy values.
-   Explain `==` and `!=`.
-   Explain `===` and `!==`.
-   Explain `Object.is`.
-   Compare:
    -   Strict Equality
    -   Abstract Equality
    -   SameValue
    -   SameValueZero.
-   Explain why:
    -   `NaN !== NaN`
    -   `Object.is(NaN, NaN)` is true
    -   `0 === -0` is true
    -   `Object.is(0, -0)` is false.
-   Understand equality behavior involving `null` and `undefined`.
-   Understand Number/BigInt comparison semantics.
-   Predict common coercion expressions without relying on memorized
    "JavaScript weirdness" tables.
-   Understand why `+` is special.
-   Explain property-key coercion and why object keys often become
    strings.
-   Recognize dangerous coercion patterns in production code.
-   Design explicit conversion boundaries for APIs, databases,
    validation, security, and business logic.

------------------------------------------------------------------------

# 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.
-   Chapter 06 --- Operators and Expressions.

The dependency chain is:

``` text
Types
  ↓
Operators
  ↓
Conversion
  ↓
Equality
```

Without a conversion model, JavaScript equality looks arbitrary.

With a conversion model, it becomes deterministic.

------------------------------------------------------------------------

# 3. What Is Type Conversion?

Type conversion means changing a value from one type representation or
semantic form to another.

Example:

``` js
Number("42");
```

A String:

``` text
"42"
```

is converted to a Number:

``` text
42
```

This is an example of **explicit conversion**.

JavaScript also performs conversions automatically during many
operations.

Example:

``` js
"42" - 2
```

The subtraction operator needs numeric semantics, so the String is
converted as part of evaluating the operation.

This is often called:

``` text
implicit coercion
```

------------------------------------------------------------------------

# 4. Explicit Conversion vs Implicit Coercion

### Explicit

The programmer requests the conversion:

``` js
Number("42");
String(42);
Boolean(1);
BigInt("123");
```

### Implicit

The language semantics cause a conversion:

``` js
"42" - 2
1 == "1"
if (value) {
  ...
}
```

The distinction is useful for code review.

Explicit conversion communicates intent.

Implicit coercion can be convenient, but it can also obscure behavior.

------------------------------------------------------------------------

# 5. Why Conversion Exists

JavaScript is dynamically typed and operates across different value
types.

Therefore operations often need rules for mixed values.

Consider:

``` js
"10" + 20
```

Should this mean:

``` text
30
```

or:

``` text
"1020"
```

JavaScript defines an answer.

Likewise:

``` js
"10" - 2
```

requires numeric interpretation.

Without deterministic conversion rules, mixed-type expressions would be
undefined or inconsistent.

------------------------------------------------------------------------

# 6. The Central Conversion Pipeline

A useful mental model is:

``` text
value
  ↓
which semantic operation needs it?
  ↓
required conversion
  ↓
converted value
  ↓
operator / algorithm
```

Different operations use different conversions.

For example:

``` text
if (x)
   ↓
ToBoolean(x)
```

while:

``` text
x - y
   ↓
numeric conversion
```

and:

``` text
property access
   ↓
property-key semantics
```

Therefore:

> JavaScript does not have one universal "convert this value" algorithm.

------------------------------------------------------------------------

# 7. `ToBoolean`

`ToBoolean` converts a value to a Boolean according to JavaScript's
truthiness rules.

Falsy values include:

``` text
false
0
-0
0n
NaN
""
null
undefined
```

For ordinary values, including objects, the result is generally truthy.

Examples:

``` js
Boolean(false);      // false
Boolean(0);          // false
Boolean(-0);         // false
Boolean(0n);         // false
Boolean(NaN);        // false
Boolean("");         // false
Boolean(null);       // false
Boolean(undefined);  // false

Boolean("0");        // true
Boolean([]);         // true
Boolean({});         // true
```

The last two are especially important:

``` js
Boolean([]);
```

is:

``` text
true
```

because an object is truthy.

------------------------------------------------------------------------

# 8. Empty Array Is Truthy

This is a classic trap:

``` js
if ([]) {
  console.log("yes");
}
```

prints:

``` text
yes
```

Some developers reason:

> "An empty array means no elements, so it should be false."

That is a business-level interpretation, not JavaScript truthiness.

At the language level:

``` text
Array → Object
Object → truthy
```

So:

``` js
Boolean([]);
```

is:

``` text
true
```

------------------------------------------------------------------------

# 9. Empty Object Is Truthy

Likewise:

``` js
Boolean({});
```

returns:

``` text
true
```

An empty object is still an object value.

Truthiness is not:

``` text
empty vs non-empty
```

It is a language-defined conversion.

------------------------------------------------------------------------

# 10. `"0"` Is Truthy

Compare:

``` js
Boolean(0);    // false
Boolean("0");  // true
```

The first is Number zero.

The second is a non-empty String.

This illustrates an important rule:

> Truthiness depends on the value and its type, not merely on human
> interpretation of the text.

------------------------------------------------------------------------

# 11. `NaN` Is Falsy

Because:

``` js
Boolean(NaN)
```

is:

``` text
false
```

code such as:

``` js
if (value) {
  ...
}
```

will not enter the branch when `value` is NaN.

This can be useful, but it can also hide invalid numeric input.

For numeric validation, prefer explicit checks:

``` js
Number.isFinite(value)
Number.isNaN(value)
```

rather than relying on truthiness.

------------------------------------------------------------------------

# 12. BigInt Zero Is Falsy

``` js
Boolean(0n);
```

returns:

``` text
false
```

while:

``` js
Boolean(1n);
```

returns:

``` text
true
```

This is consistent with the falsy-value rules for numeric zero values.

------------------------------------------------------------------------

# 13. `ToNumber`

`ToNumber` converts values to the Number domain when required.

Examples:

``` js
Number("42");      // 42
Number(true);      // 1
Number(false);     // 0
Number(null);      // 0
Number(undefined); // NaN
Number("");        // 0
Number("   ");     // 0
```

Some conversions may throw.

For example, converting a BigInt through Number is permitted as an
explicit conversion but may lose precision.

Conversion behavior must therefore be treated as a semantic operation,
not guessed from intuition.

------------------------------------------------------------------------

# 14. Numeric String Parsing

A String may represent a valid numeric syntax or may not.

Examples:

``` js
Number("42");      // 42
Number("3.14");    // 3.14
Number("1e3");     // 1000
Number("0x10");    // 16
Number("hello");   // NaN
```

The exact accepted grammar is specified by the conversion operation and
differs in details from functions such as `parseInt`.

------------------------------------------------------------------------

# 15. `Number()` vs `parseInt()`

Compare:

``` js
Number("42px");
```

and:

``` js
parseInt("42px", 10);
```

The former yields:

``` text
NaN
```

while the latter can produce:

``` text
42
```

because `parseInt` parses an integer prefix under its own grammar.

Therefore:

``` text
Number()
```

and:

``` text
parseInt()
```

are not interchangeable.

Use the function matching the input contract.

------------------------------------------------------------------------

# 16. `parseInt` and Radix

Always specify the radix when using `parseInt` for ordinary integer
parsing:

``` js
parseInt(value, 10);
```

This communicates that the input is expected to be decimal.

However, `parseInt` still has prefix-parsing semantics.

For strict validation, convert and validate according to the expected
grammar rather than relying on partial parsing.

------------------------------------------------------------------------

# 17. `ToString`

`ToString` converts values to Strings according to language rules.

Examples:

``` js
String(42);          // "42"
String(true);        // "true"
String(false);       // "false"
String(null);        // "null"
String(undefined);   // "undefined"
String(10n);         // "10"
```

Objects require additional conversion semantics because they may define
primitive conversion behavior.

------------------------------------------------------------------------

# 18. Object-to-Primitive Conversion

Suppose:

``` js
const obj = {
  valueOf() {
    return 10;
  }
};
```

When an operation needs a primitive representation, JavaScript can
attempt to convert the object.

Conceptually:

``` text
Object
  ↓
ToPrimitive
  ↓
Primitive
```

The resulting primitive is then used by the larger operation.

This is why object operands can produce surprising arithmetic/string
results.

------------------------------------------------------------------------

# 19. `ToPrimitive`

`ToPrimitive` is one of the most important abstract operations in
JavaScript.

Its purpose is to obtain a primitive value from an object.

Conceptually:

``` text
Object
   ↓
check custom primitive conversion hooks
   ↓
try conversion methods
   ↓
obtain primitive
```

The actual specification algorithm is more precise.

The important idea:

> Many operators do not directly perform numeric or string conversion on
> an object. They may first convert the object to a primitive.

------------------------------------------------------------------------

# 20. `Symbol.toPrimitive`

An object can explicitly define its primitive conversion behavior using:

``` js
Symbol.toPrimitive
```

Example:

``` js
const value = {
  [Symbol.toPrimitive](hint) {
    if (hint === "number") return 10;
    if (hint === "string") return "ten";
    return 20;
  }
};
```

This lets the object participate in language operations with custom
semantics.

For example, operations may request different hints.

This is powerful metaprogramming and must be used carefully.

------------------------------------------------------------------------

# 21. Conversion Hints

`ToPrimitive` can operate with hints such as:

``` text
"string"
"number"
"default"
```

The operator determines the context.

This allows an object to respond differently depending on how it is
being used.

For ordinary objects, the algorithm can involve methods such as:

``` text
toString
valueOf
```

The exact ordering depends on the hint and language semantics.

------------------------------------------------------------------------

# 22. `valueOf` and `toString`

These methods can participate in primitive conversion.

Example:

``` js
const obj = {
  valueOf() {
    return 10;
  },
  toString() {
    return "hello";
  }
};
```

Depending on the conversion context, one method may be tried before the
other.

This is why these methods should not be treated merely as "display
formatting."

They can participate in actual operator semantics.

------------------------------------------------------------------------

# 23. A Classic Object Conversion Example

Consider:

``` js
const obj = {
  valueOf() {
    return 10;
  }
};

console.log(obj + 5);
```

The operation may conceptually follow:

``` text
obj
 ↓
ToPrimitive
 ↓
10
 ↓
10 + 5
 ↓
15
```

The exact conversion context and final operation are defined by the
language specification.

This example demonstrates the layered model:

``` text
Object
→ primitive
→ operator
→ result
```

------------------------------------------------------------------------

# 24. The Special Nature of `+`

The binary `+` operator has both:

``` text
numeric addition
```

and:

``` text
string concatenation
```

behavior.

Very roughly:

``` text
evaluate operands
   ↓
primitive conversion
   ↓
if string-concatenation path applies
   → concatenate
otherwise
   → numeric addition
```

The exact specification algorithm is more detailed.

This is why:

``` js
1 + 2
```

and:

``` js
"1" + 2
```

produce fundamentally different results.

------------------------------------------------------------------------

# 25. `ToObject`

`ToObject` wraps primitive values in object-like semantics when an
operation requires an object.

Conceptually:

``` text
primitive
   ↓
ToObject
   ↓
wrapper/object representation for the operation
```

This is related to the boxing behavior discussed in Chapter 02.

For example:

``` js
"hello".length
```

can be understood through property-access semantics that allow primitive
string values to participate in object-style property lookup.

------------------------------------------------------------------------

# 26. Property-Key Conversion

Object property keys are generally either:

``` text
String
```

or:

``` text
Symbol
```

When using bracket notation:

``` js
obj[key]
```

the key expression is evaluated and converted according to property-key
semantics.

For example:

``` js
const obj = {
  "42": "answer"
};

console.log(obj[42]);
```

This can access the `"42"` property because the numeric property key is
converted into the corresponding property-key representation.

This becomes crucial when working with objects as dictionaries.

------------------------------------------------------------------------

# 27. `ToPropertyKey`

At specification level, property access often involves a conversion
conceptually represented by:

``` text
ToPropertyKey
```

A simplified mental model is:

``` text
input
 ↓
ToPrimitive
 ↓
if Symbol → keep Symbol
otherwise → String
```

Thus:

``` js
obj[42]
```

can target:

``` text
"42"
```

while:

``` js
const s = Symbol("x");
obj[s]
```

targets a Symbol property directly.

------------------------------------------------------------------------

# 28. Why Object Keys Become Strings

Consider:

``` js
const obj = {};

obj[1] = "one";
obj["1"] = "one-again";
```

These target the same string-named property:

``` text
"1"
```

So:

``` js
obj[1]
```

and:

``` js
obj["1"]
```

refer to the same property in an ordinary object property-key context.

This is one reason `Map` is often preferable when keys should preserve
their original value identity/type semantics.

------------------------------------------------------------------------

# 29. `Map` vs Object Key Coercion

With:

``` js
const map = new Map();

map.set(1, "number");
map.set("1", "string");
```

the keys:

``` text
1
"1"
```

are distinct.

With an ordinary object:

``` js
const obj = {};

obj[1] = "number";
obj["1"] = "string";
```

they target the same property key.

Therefore:

``` text
Object property keys
  → String / Symbol

Map keys
  → preserve key value semantics
```

Chapter 24 will cover this deeply.

------------------------------------------------------------------------

# 30. `ToBigInt`

Some operations require conversion into the BigInt domain.

Example:

``` js
BigInt("123");
```

produces:

``` text
123n
```

But conversion rules are stricter than Number conversion in important
cases.

For example, fractional Number values cannot simply become arbitrary
BigInts:

``` js
BigInt(1.5);
```

throws.

This protects against silently inventing an integer from a fractional
value.

------------------------------------------------------------------------

# 31. Number → BigInt

An integer Number can be converted explicitly:

``` js
BigInt(42);
```

Result:

``` text
42n
```

But the Number must represent an appropriate integer value.

More importantly:

> If precision has already been lost in the Number representation,
> converting it to BigInt does not recover the original mathematical
> integer.

Example:

``` js
const n = 9007199254740993;
const big = BigInt(n);
```

`big` represents the Number value that actually exists, not necessarily
the mathematical integer originally intended by the source text.

------------------------------------------------------------------------

# 32. BigInt → Number

Explicit conversion:

``` js
Number(42n);
```

works.

But for large values, precision may be lost.

Example:

``` js
const big = 9007199254740993n;
const n = Number(big);
```

The Number cannot represent that exact integer.

Therefore:

``` text
BigInt → Number
```

can be a lossy conversion.

Production code must treat such conversions as deliberate boundaries.

------------------------------------------------------------------------

# 33. Abstract Equality `==`

The loose equality operator:

``` js
==
```

uses the specification's **Abstract Equality Comparison** algorithm.

It can compare different types by performing defined conversions.

Example:

``` js
1 == "1"
```

is:

``` text
true
```

This is not because JavaScript "doesn't care about types."

It is because the abstract equality algorithm defines cross-type
comparison behavior.

------------------------------------------------------------------------

# 34. Strict Equality `===`

The strict equality operator uses the **Strict Equality Comparison**
algorithm.

Example:

``` js
1 === "1"
```

is:

``` text
false
```

The different language types prevent equality in this case.

Strict equality does not perform the same broad cross-type coercion
performed by `==`.

------------------------------------------------------------------------

# 35. `==` Is Not Random

One reason developers distrust `==` is the huge collection of examples
online.

But the algorithm is deterministic.

For example:

``` js
null == undefined
```

is:

``` text
true
```

because the Abstract Equality Comparison algorithm explicitly defines
this relationship.

But:

``` js
null == 0
```

is:

``` text
false
```

There is no blanket rule:

``` text
"null acts like zero"
```

That would be incorrect.

------------------------------------------------------------------------

# 36. The Special `null` / `undefined` Equality Rule

One of the most important abstract equality rules is:

``` js
null == undefined
```

returns:

``` text
true
```

But:

``` js
null === undefined
```

returns:

``` text
false
```

And:

``` js
null == false
```

is false.

Likewise:

``` js
undefined == 0
```

is false.

This is why memorizing "falsy values are loosely equal" is wrong.

They are not.

------------------------------------------------------------------------

# 37. Equality Is Not Truthiness

These are different mechanisms:

``` text
ToBoolean
```

and:

``` text
Equality comparison
```

For example:

``` js
Boolean(0);      // false
0 == false;      // true
0 === false;     // false
```

Three different operations are involved.

Do not infer equality from truthiness.

------------------------------------------------------------------------

# 38. `false == "0"`

Interesting examples can arise from chained coercion rules.

For example:

``` js
false == "0"
```

can evaluate to:

``` text
true
```

because abstract equality follows conversion steps.

But:

``` js
false === "0"
```

is false.

The important skill is not memorizing this pair.

The skill is knowing how to trace:

``` text
types
→ equality algorithm
→ conversions
→ final comparison
```

------------------------------------------------------------------------

# 39. Equality Algorithm: High-Level Strategy

When analyzing:

``` js
x == y
```

ask:

``` text
1. What are the operand types?
2. Are the types already comparable without conversion?
3. Does the algorithm have a special pair of rules?
4. Is one operand converted?
5. What target type is used?
6. Is the resulting comparison recursive?
7. What final equality rule applies?
```

This procedure is much more robust than a list of memorized examples.

------------------------------------------------------------------------

# 40. Strict Equality and Objects

For ordinary objects:

``` js
{} === {}
```

is:

``` text
false
```

because the two expressions create distinct object identities.

But:

``` js
const a = {};
const b = a;

a === b;
```

is:

``` text
true
```

because both references identify the same object.

Strict equality of objects therefore behaves as identity comparison.

------------------------------------------------------------------------

# 41. Loose Equality and Object/Primitive Pairs

Loose equality can convert an object in order to compare it with a
primitive.

Example:

``` js
const obj = {
  valueOf() {
    return 1;
  }
};

obj == 1;
```

This can be true because the object can be converted to a primitive and
compared according to the Abstract Equality Comparison algorithm.

This is another reason `==` can be surprising when custom objects are
involved.

------------------------------------------------------------------------

# 42. Strict Equality Does Not Convert Ordinary Objects to Primitives

Consider:

``` js
const obj = {
  valueOf() {
    return 1;
  }
};

obj === 1;
```

Result:

``` text
false
```

Strict equality sees:

``` text
Object
vs
Number
```

and does not make the same object-to-number coercion that loose equality
can perform.

------------------------------------------------------------------------

# 43. `Object.is`

`Object.is` uses a different comparison algorithm called **SameValue**.

Examples:

``` js
Object.is(NaN, NaN); // true
Object.is(0, -0);    // false
```

This differs from `===`.

Compare:

``` js
NaN === NaN;         // false
Object.is(NaN, NaN); // true

0 === -0;            // true
Object.is(0, -0);    // false
```

Therefore `Object.is` should not be thought of as merely "stricter ===."

It represents a different equality definition.

------------------------------------------------------------------------

# 44. SameValue

The SameValue algorithm treats:

``` text
NaN
```

as equal to itself and distinguishes:

``` text
+0
-0
```

This is useful when the exact semantic identity of numeric values
matters.

It underlies some language/platform behaviors where those distinctions
are relevant.

------------------------------------------------------------------------

# 45. SameValueZero

SameValueZero is another equality algorithm.

It is similar to SameValue except:

``` text
+0
-0
```

are considered equal.

Like SameValue:

``` text
NaN
```

is equal to itself.

This algorithm is used by important data structures/APIs.

For example:

``` text
Set
Map key equality
Array.prototype.includes
```

use SameValueZero-like semantics in relevant contexts.

------------------------------------------------------------------------

# 46. Equality Algorithms Comparison

  Comparison                  `NaN` vs `NaN`   `+0` vs `-0`
  ------------------------- ---------------- --------------
  `===` / Strict Equality              false           true
  `Object.is` / SameValue               true          false
  SameValueZero                         true           true
  `==`                                 false           true

The `==` row also depends on operand types and the Abstract Equality
algorithm.

The table is a summary, not a replacement for the algorithms.

------------------------------------------------------------------------

# 47. `includes` vs `indexOf`

A classic example:

``` js
[NaN].indexOf(NaN);
```

does not find the value because `indexOf` uses strict equality
semantics.

But:

``` js
[NaN].includes(NaN);
```

returns:

``` text
true
```

because `includes` uses SameValueZero-style equality.

This is an excellent practical example of why equality algorithms
matter.

------------------------------------------------------------------------

# 48. `Set` and `NaN`

Consider:

``` js
const set = new Set();

set.add(NaN);

console.log(set.has(NaN));
```

Result:

``` text
true
```

The relevant equality semantics treat NaN as equal to itself.

This is one reason understanding SameValueZero matters for real
application code.

------------------------------------------------------------------------

# 49. Object Key Equality in `Map`

Map keys use SameValueZero-style comparison.

Therefore:

``` js
const map = new Map();

map.set(NaN, "value");

map.get(NaN);
```

retrieves the same entry.

Likewise:

``` js
const map = new Map();

map.set(0, "zero");

map.get(-0);
```

can retrieve the same entry because positive and negative zero are
treated as equal in this key comparison model.

------------------------------------------------------------------------

# 50. Strict Equality and `BigInt`

Strict equality distinguishes:

``` js
1n
```

from:

``` js
1
```

Thus:

``` js
1n === 1
```

is:

``` text
false
```

But:

``` js
1n == 1
```

can be:

``` text
true
```

because abstract equality defines a cross-numeric-type comparison path.

------------------------------------------------------------------------

# 51. Relational Comparison vs Equality

Do not assume equality and relational comparison use the same
algorithms.

For example:

``` js
1n < 2
```

can be valid and true.

But:

``` js
1n + 2
```

throws.

The language defines each operator independently.

This reinforces a general rule:

> Never generalize one operator's coercion behavior to another operator
> without checking the actual semantics.

------------------------------------------------------------------------

# 52. Coercion and `Symbol.toPrimitive`

Consider:

``` js
const value = {
  [Symbol.toPrimitive](hint) {
    console.log(hint);
    return hint === "string" ? "x" : 10;
  }
};
```

Different operations can request different conversion behavior.

This demonstrates why:

``` text
object + object
object == primitive
String(object)
Number(object)
```

cannot always be predicted from `valueOf()` alone.

Custom primitive conversion can alter observable behavior.

------------------------------------------------------------------------

# 53. Coercion Is Observable

Conversion is not necessarily an invisible implementation detail.

Objects can define:

``` js
[Symbol.toPrimitive]
valueOf
toString
```

and those methods can:

-   log,
-   mutate state,
-   throw,
-   perform I/O in pathological code,
-   return different values,
-   create surprising side effects.

Therefore a seemingly simple expression such as:

``` js
obj + 1
```

can run user code.

This is one reason explicit conversion can improve predictability.

------------------------------------------------------------------------

# 54. Side Effects in Equality

Even equality can invoke user-defined conversion when loose equality
compares objects with primitives.

Therefore:

``` js
obj == 10
```

can have side effects if `obj` customizes primitive conversion.

By contrast:

``` js
obj === 10
```

does not perform the same object-to-primitive conversion.

This is a concrete reason why strict equality is often easier to reason
about.

------------------------------------------------------------------------

# 55. Explicit Conversion as Documentation

Compare:

``` js
if (value == 1) {
  ...
}
```

with:

``` js
if (Number(value) === 1) {
  ...
}
```

The second makes a conversion boundary explicit.

But explicit conversion is not automatically safer.

If:

``` js
value = "hello"
```

then:

``` js
Number(value)
```

is:

``` text
NaN
```

You still need validation.

A good pattern is:

``` js
const numericValue = Number(value);

if (!Number.isFinite(numericValue)) {
  throw new TypeError("Invalid numeric value");
}
```

------------------------------------------------------------------------

# 56. Strict Equality Is Usually the Default

For ordinary application logic:

``` js
===
!==
```

are generally easier to reason about than:

``` js
==
!=
```

because they avoid many implicit cross-type conversions.

This is not a statement that `==` is "broken."

It is a design/readability recommendation.

There are specific places where `==` can express useful semantics, but
those cases should be deliberate.

------------------------------------------------------------------------

# 57. A Legitimate Use of `==`

One commonly cited deliberate use is:

``` js
value == null
```

which intentionally matches:

``` text
null
undefined
```

and excludes other common falsy values such as:

``` text
0
false
""
```

This can be concise when the exact intended contract is:

> "treat null and undefined as equivalent absence."

However, team coding standards may prefer:

``` js
value === null || value === undefined
```

or:

``` js
value == null // deliberate and documented
```

The important part is intent.

------------------------------------------------------------------------

# 58. Truthiness vs Nullishness

Compare:

``` js
if (value) {
  ...
}
```

with:

``` js
if (value != null) {
  ...
}
```

The first checks:

``` text
truthiness
```

The second deliberately treats:

``` text
null + undefined
```

as absence under loose equality semantics.

These are different business rules.

Do not substitute one for the other without understanding the domain.

------------------------------------------------------------------------

# 59. Coercion Tables --- Core Cases

Useful examples:

  Expression            Result
  --------------------- ---------------
  `Number(null)`        `0`
  `Number(undefined)`   `NaN`
  `Number("")`          `0`
  `Number("42")`        `42`
  `Number("42x")`       `NaN`
  `String(null)`        `"null"`
  `String(undefined)`   `"undefined"`
  `Boolean(0)`          `false`
  `Boolean("0")`        `true`
  `Boolean([])`         `true`
  `Boolean({})`         `true`

These are best learned as consequences of conversion algorithms.

------------------------------------------------------------------------

# 60. The "Weird JavaScript" Family

Examples frequently shown online include:

``` js
[] == false
```

``` js
"" == 0
```

``` js
"0" == false
```

``` js
null == undefined
```

These are not arbitrary special cases.

They emerge from the Abstract Equality Comparison algorithm plus
conversion operations.

The goal of this chapter is to stop thinking:

``` text
JavaScript is weird.
```

and start thinking:

``` text
Which equality algorithm ran?
Which conversion occurred?
In what order?
```

------------------------------------------------------------------------

# 61. Example Trace --- `[] == false`

Consider:

``` js
[] == false
```

A conceptual high-level trace is:

``` text
Object
vs
Boolean
```

Abstract equality does not simply ask whether the object is truthy.

It enters its defined conversion path.

The Boolean is converted to a Number:

``` text
false → 0
```

The object is converted to a primitive:

``` text
[] → ""
```

Then the String is converted numerically:

``` text
"" → 0
```

The final comparison is effectively:

``` text
0 == 0
```

which is true.

The key lesson is not the final result.

The key lesson is the conversion chain:

``` text
[]
 ↓
""
 ↓
0

false
 ↓
0
```

------------------------------------------------------------------------

# 62. Example Trace --- `[] === false`

Now:

``` js
[] === false
```

The operands have different types:

``` text
Object
Boolean
```

Strict equality does not perform the abstract cross-type conversion
path.

Therefore the result is immediately:

``` text
false
```

This contrast is one of the best demonstrations of strict vs abstract
equality.

------------------------------------------------------------------------

# 63. Example Trace --- `"0" == false`

Conceptually:

``` text
String
vs
Boolean
```

The Boolean is converted to Number:

``` text
false → 0
```

Then the String is converted to Number:

``` text
"0" → 0
```

Then:

``` text
0 == 0
```

So the result is:

``` text
true
```

Again, deterministic conversion explains the result.

------------------------------------------------------------------------

# 64. Example Trace --- `null == false`

Compare:

``` js
null == false
```

The result is:

``` text
false
```

Why does it not follow the same route as other falsy values?

Because Abstract Equality has an explicit special case for:

``` text
null / undefined
```

It does not generally convert `null` into every other falsy
representation.

This is exactly why memorizing "all falsy values are equal with ==" is
wrong.

------------------------------------------------------------------------

# 65. Example Trace --- `0 == "0"`

Conceptually:

``` text
Number
vs
String
```

Abstract equality converts the String to Number:

``` text
"0" → 0
```

Then:

``` text
0 == 0
```

is true.

Strict equality:

``` js
0 === "0"
```

is false.

------------------------------------------------------------------------

# 66. Example Trace --- Object With `valueOf`

Consider:

``` js
const x = {
  valueOf() {
    return 7;
  }
};

console.log(x == 7);
```

Conceptually:

``` text
Object
 ↓
ToPrimitive
 ↓
7
 ↓
compare with Number 7
```

The result can be:

``` text
true
```

Now:

``` js
x === 7
```

is false because strict equality does not perform the same
object-to-primitive conversion.

------------------------------------------------------------------------

# 67. Equality and Reference Identity

For objects:

``` js
const a = {};
const b = {};

a == b;
```

is false.

Why?

Both operands are objects and they are distinct identities.

Loose equality does not automatically perform deep structural comparison
of two objects.

Similarly:

``` js
a === b
```

is also false.

Thus:

``` text
== does not mean deep equality.
```

------------------------------------------------------------------------

# 68. Deep Equality Is an Application Problem

JavaScript does not have a single built-in operator that means:

``` text
recursively compare arbitrary object graphs
```

Applications use:

-   custom comparison,
-   library utilities,
-   serialization-based comparison in limited cases,
-   structural algorithms.

But deep equality has difficult questions:

``` text
cycles
Dates
Maps
Sets
typed arrays
Symbols
property descriptors
prototypes
functions
NaN
-0
```

Do not mistake `==` for structural equality.

------------------------------------------------------------------------

# 69. Object.is and Deep Equality

`Object.is` still does not perform deep equality.

Example:

``` js
Object.is({}, {});
```

is:

``` text
false
```

It compares two object identities under SameValue semantics.

Therefore:

``` text
strict equality
SameValue
SameValueZero
```

are identity/value comparison algorithms, not recursive object
comparison.

------------------------------------------------------------------------

# 70. Coercion and Arrays

Arrays can participate in primitive conversion.

For example:

``` js
String([1, 2, 3]);
```

produces a string representation based on the array's object conversion
behavior.

This contributes to surprising expressions such as:

``` js
[1, 2] + [3, 4]
```

The `+` operation may convert both arrays to primitives and then
concatenate the resulting strings.

Do not use such coercion tricks in production code.

------------------------------------------------------------------------

# 71. Coercion and Dates

Date objects have special primitive-conversion behavior compared with
ordinary objects.

This historically contributes to differences between Date and general
objects when converted to primitives.

The broader lesson is:

``` text
not all objects have identical conversion behavior
```

Built-ins can define specialized semantics.

------------------------------------------------------------------------

# 72. Coercion and Symbols

Symbols have deliberate conversion restrictions.

For example:

``` js
String(Symbol("x"));
```

works.

But some implicit string/number coercions involving Symbols throw.

This prevents Symbols from silently collapsing into ambiguous primitive
representations.

Whenever Symbols are involved, use explicit semantics.

------------------------------------------------------------------------

# 73. BigInt and Unary / Numeric Conversion

BigInt interacts with conversion and operators under stricter rules.

For example:

``` js
Number(1n)
```

works explicitly.

But:

``` js
+1n
```

throws.

This prevents an operation that historically means numeric Number
conversion from silently moving a BigInt into a potentially lossy Number
domain.

This is a useful example of language design prioritizing explicitness at
a precision boundary.

------------------------------------------------------------------------

# 74. `Object()` and Boxing

The function:

``` js
Object(value)
```

can produce object-wrapped forms for primitives.

Examples:

``` js
Object(42);
Object("hello");
Object(true);
```

This is different from:

``` js
new Number(42);
```

in API semantics, though both produce object values in these primitive
cases.

In ordinary code, avoid unnecessary wrapper objects.

------------------------------------------------------------------------

# 75. Equality and `Object.create(null)`

An object created using:

``` js
Object.create(null)
```

has no ordinary `Object.prototype` in its prototype chain.

But equality semantics still follow the object identity/type rules.

This is useful to remember because:

``` text
property lookup
```

and:

``` text
equality
```

are different concerns.

Prototype configuration changes lookup behavior, not the fundamental
identity semantics of objects.

------------------------------------------------------------------------

# 76. Coercion and Security

Implicit coercion can become dangerous around:

``` text
authorization
input validation
rate limits
numeric constraints
property keys
logging
serialization
```

Example:

``` js
if (user.role == 1) {
  grantAccess();
}
```

This potentially accepts more representations than intended.

A stronger contract might validate:

``` js
if (user.role === 1) {
  ...
}
```

and ensure the input has already been validated as a Number.

The important principle:

> Normalize and validate at trust boundaries before making security
> decisions.

------------------------------------------------------------------------

# 77. Numeric Input Security

Consider:

``` js
const amount = Number(input);
```

This is only conversion.

It does not answer:

``` text
Was the input syntactically acceptable?
Is it finite?
Is it safe?
Is it within range?
Is it the correct unit?
Is it a decimal or integer domain?
```

A production numeric boundary should usually perform multiple checks.

------------------------------------------------------------------------

# 78. Property-Key Security

Dynamic access:

``` js
obj[userInput]
```

causes property-key semantics to be applied.

If the application assumes:

``` text
only ordinary own properties are accessed
```

but user input controls the key, prototype-related behavior can become
security-relevant.

This is part of the broader prototype-pollution/security topic.

The conversion itself is deterministic.

The security problem is what the resulting key is allowed to access.

------------------------------------------------------------------------

# 79. Performance Considerations

Implicit coercion can have costs such as:

-   object-to-primitive conversion,
-   method calls,
-   allocation,
-   string construction,
-   numeric parsing,
-   repeated conversions.

In most normal application code, these costs are minor.

In hot loops or performance-sensitive systems, unnecessary conversion
can matter.

Do not optimize by replacing readable code with obscure coercion tricks.

Measure first.

------------------------------------------------------------------------

# 80. Memory Considerations

Conversions can create temporary values.

Examples:

``` js
Number(string)
String(object)
```

may create intermediate representations.

Object-to-primitive conversion may call user code and generate new
primitive values.

Repeated string concatenation/conversion can also generate temporary
strings.

The actual physical allocation pattern is engine-specific.

Measure memory behavior with runtime profiling when it matters.

------------------------------------------------------------------------

# 81. Common Misconceptions

## Misconception 1 --- "`==` is broken."

False.

`==` has a complicated but deterministic specification.

## Misconception 2 --- "All falsy values are equal with `==`."

False.

For example:

``` js
null == false
```

is false.

## Misconception 3 --- "`===` always means deeply equal."

False.

For objects it checks identity.

## Misconception 4 --- "`Object.is` is just stricter `===`."

False.

It uses SameValue semantics.

## Misconception 5 --- "Every conversion goes through `ToNumber`."

False.

Different operations invoke different abstract conversion algorithms.

## Misconception 6 --- "Objects automatically become strings."

Not always.

`ToPrimitive` and contextual hints determine the conversion path.

## Misconception 7 --- "Explicit conversion cannot lose data."

False.

For example:

``` js
Number(largeBigInt)
```

can lose precision.

## Misconception 8 --- "Truthiness tells you whether data is valid."

False.

A valid value can be falsy, and an invalid value can be truthy.

## Misconception 9 --- "`Map` keys behave like object keys."

False.

Map preserves key value/identity semantics rather than converting
everything to string property keys.

------------------------------------------------------------------------

# 82. Common Mistakes

### Mistake: loose equality in authorization

``` js
role == 1
```

can accept representations beyond the intended domain.

### Mistake: truthiness validation

``` js
if (!amount) {
  ...
}
```

incorrectly groups:

``` text
0
""
false
null
undefined
NaN
```

and more depending on the condition.

### Mistake: converting BigInt to Number casually

Potential precision loss.

### Mistake: assuming `String(value)` is harmless for every object

Custom conversion hooks can execute user code.

### Mistake: using `parseInt` where strict numeric validation is required

It may accept prefixes rather than the entire input.

### Mistake: treating `Object.is` as deep equality

It is not.

------------------------------------------------------------------------

# 83. Comparison Table --- Conversion APIs

  API / Operation     Main behavior
  ------------------- ------------------------------------------------
  `Number(x)`         explicit Number conversion
  `String(x)`         explicit String conversion
  `Boolean(x)`        explicit Boolean conversion
  `BigInt(x)`         explicit BigInt conversion under BigInt rules
  `Object(x)`         converts/wraps into an Object where applicable
  `x + y`             special primitive/string/numeric semantics
  `x - y`             numeric-oriented conversion
  `if (x)`            Boolean conversion
  `obj[key]`          property-key conversion
  `x == y`            Abstract Equality Comparison
  `x === y`           Strict Equality Comparison
  `Object.is(x, y)`   SameValue

------------------------------------------------------------------------

# 84. Comparison Table --- Equality Algorithms

  Algorithm                  Type coercion                 `NaN` self-equal   `+0` / `-0`
  -------------------------- ----------------------------- ------------------ -------------
  Abstract Equality (`==`)   Yes, according to algorithm   No                 Equal
  Strict Equality (`===`)    No cross-type coercion        No                 Equal
  SameValue (`Object.is`)    No                            Yes                Distinct
  SameValueZero              No                            Yes                Equal

------------------------------------------------------------------------

# 85. Execution Walkthrough --- `1 + "2"`

Consider:

``` js
1 + "2"
```

Conceptually:

``` text
1
↓
Number

"2"
↓
String

+ operator
↓
primitive/string-concatenation path
↓
"1" + "2"
↓
"12"
```

The exact specification sequence should be learned in the detailed `+`
semantics later, but the high-level reason is:

``` text
+ has special String behavior
```

------------------------------------------------------------------------

# 86. Execution Walkthrough --- `"5" - 2`

``` js
"5" - 2
```

Conceptually:

``` text
"5"
↓
numeric conversion
↓
5

2
↓
2

5 - 2
↓
3
```

The result is:

``` text
3
```

Unlike `+`, subtraction has numeric semantics.

------------------------------------------------------------------------

# 87. Execution Walkthrough --- `null == undefined`

``` js
null == undefined
```

Conceptually:

``` text
Null
vs
Undefined
```

The Abstract Equality Comparison algorithm contains an explicit rule for
this pair.

Result:

``` text
true
```

No general "null converts to undefined" rule is implied.

------------------------------------------------------------------------

# 88. Execution Walkthrough --- `0 === false`

``` js
0 === false
```

Types:

``` text
Number
Boolean
```

Strict equality sees different types.

Result:

``` text
false
```

There is no abstract equality conversion path.

------------------------------------------------------------------------

# 89. Execution Walkthrough --- `0 == false`

``` js
0 == false
```

Conceptual path:

``` text
Number
vs
Boolean

Boolean
↓
Number
↓
0

compare:
0 vs 0
↓
true
```

This example demonstrates how Abstract Equality can bridge different
types.

------------------------------------------------------------------------

# 90. Execution Walkthrough --- `Object.is(NaN, NaN)`

``` js
Object.is(NaN, NaN)
```

Uses SameValue semantics.

Under SameValue:

``` text
NaN is equal to NaN
```

Result:

``` text
true
```

This is different from:

``` js
NaN === NaN
```

which is false.

------------------------------------------------------------------------

# 91. Execution Walkthrough --- `Object.is(0, -0)`

``` js
Object.is(0, -0)
```

SameValue distinguishes the signs of zero.

Result:

``` text
false
```

But:

``` js
0 === -0
```

is true.

This distinction matters in numerical code where signed zero is
semantically meaningful.

------------------------------------------------------------------------

# 92. Implementation From Scratch --- `toBoolean`

Create:

``` js
function toBooleanModel(value) {
  // intentionally model ToBoolean semantics
}
```

Do not call:

``` js
Boolean(value)
```

inside it.

Implement the semantic cases directly.

Test:

``` text
false
true
0
-0
1
NaN
0n
1n
""
"x"
null
undefined
[]
{}
```

The goal is to reconstruct a specification concept from behavior.

------------------------------------------------------------------------

# 93. Implementation --- Equality Tracer

Build a debugging utility:

``` js
traceEquality(a, b)
```

that reports:

``` text
left value/type
right value/type
strict equality result
Object.is result
loose equality result
```

For selected cases, also explain the likely conversion path.

This should not pretend to be a full ECMAScript implementation.

The exercise is about reasoning.

------------------------------------------------------------------------

# 94. Implementation Progression

### Guided

Build:

``` text
type reporter
truthiness reporter
equality reporter
```

### Partially Guided

Add:

``` text
primitive conversion tracing
numeric conversion checks
property-key conversion examples
```

### No Reference

Build a "JavaScript coercion laboratory" CLI.

Input:

``` text
JSON-ish value specification
```

Output:

``` text
type
truthiness
String conversion
Number conversion where valid
strict equality comparisons
SameValue comparison
```

### Edge-Case Hardened

Test:

``` text
NaN
-0
BigInt
Symbol
arrays
objects
custom Symbol.toPrimitive
custom valueOf
custom toString
null
undefined
```

### Production-Grade

Add:

-   deterministic test suite,
-   explicit unsupported cases,
-   documentation,
-   security-safe output,
-   no execution of untrusted code.

------------------------------------------------------------------------

# 95. Debugging Exercises

## Exercise 1

Explain:

``` js
[] == false
```

without calling it "JavaScript weirdness."

## Exercise 2

Explain:

``` js
[] === false
```

## Exercise 3

Explain:

``` js
null == undefined
```

## Exercise 4

Explain:

``` js
null == 0
```

## Exercise 5

Explain:

``` js
0 == false
```

and:

``` js
0 === false
```

## Exercise 6

Explain:

``` js
NaN === NaN
```

and:

``` js
Object.is(NaN, NaN)
```

## Exercise 7

Explain:

``` js
0 === -0
```

and:

``` js
Object.is(0, -0)
```

## Exercise 8

Explain:

``` js
1n == 1
```

and:

``` js
1n === 1
```

## Exercise 9

Create an object with:

``` js
Symbol.toPrimitive
```

and explain how it changes an expression.

------------------------------------------------------------------------

# 96. Code Review Exercise

Review:

``` js
function authorize(user) {
  return user.isAdmin == 1;
}
```

Identify the risk.

Potential issue:

``` text
loose equality permits type coercion
```

A stronger design validates the domain type first and then compares
explicitly:

``` js
return user.isAdmin === 1;
```

But even that may not be the best contract.

If the property is conceptually Boolean, a stronger model is:

``` js
return user.isAdmin === true;
```

provided the application contract defines it as a Boolean.

The deeper review question is:

> What exact type does the domain model promise?

------------------------------------------------------------------------

# 97. Code Review Exercise --- Truthiness Validation

Review:

``` js
function setLimit(limit) {
  if (!limit) {
    throw new Error("Limit required");
  }
}
```

Potential bug:

``` js
limit = 0
```

may be valid depending on the domain.

Better:

``` js
if (limit === undefined || limit === null) {
  ...
}
```

or, better still, validate:

``` text
type
integer-ness
range
domain constraints
```

explicitly.

------------------------------------------------------------------------

# 98. Code Review Exercise --- Numeric Conversion

Review:

``` js
function getAmount(input) {
  return Number(input);
}
```

A production review should ask:

``` text
Is empty string allowed?
Is whitespace allowed?
Is Infinity allowed?
Is NaN allowed?
Are fractions allowed?
What range is valid?
Is exact decimal precision required?
Is this money?
Should the original string be preserved?
```

Conversion is not validation.

------------------------------------------------------------------------

# 99. Interview Questions

## Junior

1.  What is type coercion?
2.  What is the difference between explicit and implicit conversion?
3.  What are falsy values?
4.  What is the difference between `==` and `===`?
5.  What does `Object.is` do?

## Mid-Level

6.  Why is `[]` truthy?
7.  Why can `[] == false` be true?
8.  Why is `null == undefined` true?
9.  Why is `null == 0` false?
10. What is `ToPrimitive`?
11. What does `Symbol.toPrimitive` do?
12. Why is `Number("42x")` NaN while `parseInt("42x", 10)` can return
    42?

## Senior

13. Explain Abstract Equality Comparison.
14. Explain Strict Equality Comparison.
15. Explain SameValue and SameValueZero.
16. Why do `Map` and `Set` handle NaN differently from `indexOf`?
17. Why does `Object.is(0, -0)` differ from `0 === -0`?
18. How does object-to-primitive conversion work?
19. Why is `+` special?

## Staff / Principal

20. Should a production codebase ban `==`?
21. How would you define explicit coercion policy for an API boundary?
22. How can implicit coercion create authorization vulnerabilities?
23. How should numeric coercion be handled for financial values?
24. How would you design a JavaScript coercion utility without hiding
    domain semantics?
25. How would you explain equality algorithms to a team that frequently
    writes `==` and `===` interchangeably?
26. How do conversion semantics affect performance and security?

------------------------------------------------------------------------

# 100. Predict-the-Output Exercises

Predict before executing.

### 1

``` js
console.log(Boolean(0));
```

### 2

``` js
console.log(Boolean("0"));
```

### 3

``` js
console.log(Boolean([]));
```

### 4

``` js
console.log(Boolean({}));
```

### 5

``` js
console.log(Number(""));
```

### 6

``` js
console.log(Number(" "));
```

### 7

``` js
console.log(Number("42"));
```

### 8

``` js
console.log(Number("42x"));
```

### 9

``` js
console.log(String(null));
```

### 10

``` js
console.log(String(undefined));
```

### 11

``` js
console.log(1 + "2");
```

### 12

``` js
console.log("5" - 2);
```

### 13

``` js
console.log(0 == false);
```

### 14

``` js
console.log(0 === false);
```

### 15

``` js
console.log(null == undefined);
```

### 16

``` js
console.log(null === undefined);
```

### 17

``` js
console.log(null == 0);
```

### 18

``` js
console.log("0" == false);
```

### 19

``` js
console.log([] == false);
```

### 20

``` js
console.log([] === false);
```

### 21

``` js
console.log(NaN === NaN);
```

### 22

``` js
console.log(Object.is(NaN, NaN));
```

### 23

``` js
console.log(0 === -0);
```

### 24

``` js
console.log(Object.is(0, -0));
```

### 25

``` js
console.log(1n == 1);
```

### 26

``` js
console.log(1n === 1);
```

### 27

``` js
console.log([1] == 1);
```

### 28

``` js
console.log([1, 2] == "1,2");
```

### 29

``` js
const x = {
  valueOf() {
    return 5;
  }
};

console.log(x == 5);
```

### 30

``` js
const x = {
  valueOf() {
    return 5;
  }
};

console.log(x === 5);
```

------------------------------------------------------------------------

# 101. Mastery Exercises

## Exercise A --- Explain the Conversion Family

Without notes, explain:

``` text
ToPrimitive
ToBoolean
ToNumber
ToString
ToBigInt
ToObject
ToPropertyKey
```

For each, state:

``` text
purpose
typical trigger
possible failure
representative example
```

## Exercise B --- Equality Algorithm Trace

Manually trace:

``` js
[] == false
```

using:

``` text
operand types
conversion
intermediate values
final comparison
```

Do the same for:

``` js
"0" == false
null == undefined
0 == "0"
```

## Exercise C --- Equality Matrix

Build a matrix for:

``` text
0
"0"
false
null
undefined
NaN
0n
[]
{}
```

Compare them using:

``` text
==
===
Object.is
```

Do not merely record results; explain the representative patterns.

## Exercise D --- Conversion Security Review

Given:

``` js
const role = Number(input);
```

define every validation rule required before the value is used for
authorization.

## Exercise E --- Custom Primitive Conversion

Build an object with:

``` js
Symbol.toPrimitive
```

that behaves differently for:

``` text
string context
number context
default context
```

Then document every expression that invokes it.

------------------------------------------------------------------------

# 102. Production Decision Framework

When a value crosses a system boundary, ask:

``` text
1. What type is expected?
2. Should conversion happen?
3. Should conversion be explicit?
4. What input grammar is accepted?
5. What values are invalid?
6. What range is valid?
7. Is precision important?
8. Is null equivalent to undefined?
9. Is empty string meaningful?
10. Should truthiness be used at all?
11. Does equality need identity, exact type, or domain equivalence?
12. Could custom object conversion run?
13. Could conversion produce side effects?
14. How is the normalized value stored and serialized?
```

This turns coercion from a language curiosity into an engineering
contract.

------------------------------------------------------------------------

# 103. A Recommended Equality Policy

For most ordinary application code:

``` text
Use === / !== by default.
```

Use:

``` text
Object.is
```

when the exact SameValue distinction matters, particularly:

``` text
NaN
signed zero
```

Use:

``` text
==
```

only when the specific Abstract Equality semantics are intentionally
part of the design.

Use domain-specific equality when:

``` text
case-insensitive strings
normalized Unicode
decimal amounts
timestamps
database identifiers
semantic object equality
```

are involved.

Language equality is not always domain equality.

------------------------------------------------------------------------

# 104. Domain Equality vs JavaScript Equality

Suppose two usernames differ only by canonical Unicode representation.

JavaScript:

``` js
a === b
```

may be false.

The product may still define them as the same account identity after
normalization.

Similarly, two database timestamps may represent the same instant
despite having different source formatting.

Therefore:

``` text
JavaScript equality
≠
business equality
```

A mature system explicitly defines domain equivalence.

------------------------------------------------------------------------

# 105. Debugging Method --- "What Conversion Happened?"

Whenever an expression surprises you, write:

``` text
Left value:
Left type:

Right value:
Right type:

Operator:

Equality / conversion algorithm:

Conversions:

Intermediate values:

Final comparison/operation:

Result:
```

This format is especially powerful for:

``` text
==
+
-
<
>
logical defaulting
property-key access
```

It transforms guessing into deterministic analysis.

------------------------------------------------------------------------

# 106. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.
-   Chapter 06 --- Operators and Expressions.

## Builds Toward

-   Chapter 08 --- Control Flow and Iteration.
-   Chapter 14 --- `this`, Invocation, and Binding.
-   Chapter 15 --- Objects and Property Semantics.
-   Chapter 16 --- Property Keys and Enumeration.
-   Chapter 17 --- Prototypes and Prototype Chains.
-   Chapter 19 --- Proxy and Metaprogramming.
-   Chapter 20 --- Symbols and Well-Known Symbols.
-   Chapter 24 --- Map, Set, WeakMap, WeakSet.
-   Chapter 28 --- Serialization.
-   Chapter 42 --- Abstract Operations.
-   Chapter 43 --- Ordinary Object Internal Methods.
-   Chapter 56 --- Browser Security.
-   Chapter 57 --- JavaScript Security Engineering.
-   Chapter 79 --- API Design.
-   Chapter 81 --- Database Integration.

## Related Concepts

``` text
ToPrimitive
ToBoolean
ToNumber
ToString
ToBigInt
ToObject
ToPropertyKey
Abstract Equality
Strict Equality
SameValue
SameValueZero
truthiness
nullishness
identity
domain equality
```

## Concepts Revisited Later

The specification chapters will revisit these algorithms formally.

The object model will explain:

``` text
valueOf
toString
Symbol.toPrimitive
property-key conversion
```

The collection chapter will explain:

``` text
Map / Set key equality
```

The security chapters will revisit:

``` text
coercion at trust boundaries
```

The API/production chapters will revisit:

``` text
explicit normalization and validation
```

## Why This Chapter Matters Later

JavaScript often appears unpredictable only when the conversion
algorithm is hidden.

Once the learner can ask:

``` text
Which algorithm?
Which conversion?
Which intermediate value?
Which equality relation?
```

many advanced behaviors stop being mysterious.

------------------------------------------------------------------------

# 107. Key Takeaways

1.  JavaScript has both explicit conversion and implicit coercion.
2.  Different operations use different abstract conversion algorithms.
3.  `ToBoolean` defines truthiness.
4.  Objects such as `[]` and `{}` are truthy even when empty.
5.  `ToNumber` explains many numeric coercions.
6.  `Number()` and `parseInt()` have different parsing semantics.
7.  `ToString` defines string conversion.
8.  Objects can be converted to primitives through `ToPrimitive`.
9.  `Symbol.toPrimitive`, `valueOf`, and `toString` can influence object
    conversion.
10. `ToObject` explains primitive/object interaction for APIs such as
    property access.
11. Property keys are based on String or Symbol semantics.
12. Ordinary object numeric keys often become string keys.
13. `Map` keys do not behave like ordinary object property keys.
14. BigInt and Number conversions must be explicit when crossing numeric
    domains.
15. Number conversion can lose BigInt precision.
16. Abstract Equality (`==`) performs defined cross-type conversion.
17. Strict Equality (`===`) does not perform the same cross-type
    conversion.
18. `null == undefined` is true due to a specific Abstract Equality
    rule.
19. `null == 0` is false.
20. Truthiness and equality are separate mechanisms.
21. `Object.is` uses SameValue semantics.
22. SameValue distinguishes `+0` and `-0` and treats NaN as equal to
    itself.
23. SameValueZero treats NaN as equal to itself and treats `+0` and `-0`
    as equal.
24. `includes`, `Set`, and `Map` demonstrate why equality algorithm
    choice matters.
25. `+` is special because it can perform concatenation or numeric
    addition.
26. Implicit coercion can invoke user-defined code and therefore can
    have side effects.
27. Conversion is not the same thing as validation.
28. JavaScript equality is not necessarily business/domain equality.
29. Strict equality is a practical default for most application logic.
30. Production systems should make conversion, normalization, and
    equality contracts explicit at important boundaries.

------------------------------------------------------------------------

# 108. Completion Criteria

### Understand

-   Explain explicit conversion vs implicit coercion.
-   Explain the major abstract conversion operations.
-   Explain the four major equality algorithms.

### Explain

-   Explain truthiness.
-   Explain object-to-primitive conversion.
-   Explain `Symbol.toPrimitive`.
-   Explain `==` vs `===`.
-   Explain `Object.is`.
-   Explain SameValue vs SameValueZero.
-   Explain `null` / `undefined` equality.

### Predict

-   Predict common coercion expressions.
-   Trace mixed-type equality.
-   Predict `NaN`, signed-zero, Number/BigInt, object, and array
    comparison behavior.

### Implement

-   Build conversion/equality diagnostic tools.
-   Build a coercion laboratory.
-   Implement a model of ToBoolean semantics.

### Debug

-   Diagnose unexpected loose equality.
-   Diagnose numeric/string coercion.
-   Diagnose truthiness bugs.
-   Diagnose property-key coercion.
-   Diagnose Number/BigInt conversion issues.

### Apply

-   Define explicit conversion/validation rules at APIs, storage
    boundaries, and security checks.

### Compare

-   `==` vs `===`
-   `===` vs `Object.is`
-   SameValue vs SameValueZero
-   `Number()` vs `parseInt()`
-   truthiness vs nullishness
-   object keys vs Map keys
-   language equality vs domain equality

### Defend

-   Defend an equality policy.
-   Defend an explicit conversion strategy.
-   Explain exactly why a coercion result occurs.
-   Identify where implicit coercion creates security or maintainability
    risk.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 08 --- Control Flow and Iteration
