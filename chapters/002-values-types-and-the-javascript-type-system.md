# Chapter 02 --- Values, Types, and the JavaScript Type System

> **Status:** `[+] Completed`\
> **Role in curriculum:** Defines the fundamental things JavaScript
> programs operate on. Every later topic---variables, operators,
> objects, functions, coercion, equality, collections, memory,
> serialization, and APIs---depends on this chapter.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain what a JavaScript value is.
-   Distinguish values from variables, references, expressions, and
    storage locations.
-   Explain JavaScript's language-level type system.
-   List the eight ECMAScript language types.
-   Distinguish primitive values from objects.
-   Explain why `null` is a primitive value even though `typeof null` is
    `"object"`.
-   Explain `undefined`, `null`, `boolean`, `number`, `bigint`,
    `string`, `symbol`, and `object`.
-   Explain why functions are objects at the language level while also
    being callable.
-   Explain dynamic typing.
-   Explain why JavaScript is not statically typed even though every
    value has a language type.
-   Understand value semantics versus object identity.
-   Correctly explain the phrase "JavaScript passes objects by
    reference" as an imprecise shortcut.
-   Understand boxing and wrapper objects.
-   Distinguish primitive values from their wrapper objects.
-   Predict the result of `typeof`, equality, property access, and basic
    type-related operations.
-   Build a reliable mental model for later chapters involving coercion,
    object property access, prototypes, closures, memory, and APIs.

------------------------------------------------------------------------

## 2. Prerequisites

Chapter 01 established the language / engine / runtime / host
distinction.

This chapter now moves one layer inward:

``` text
JavaScript / ECMAScript
        ↓
What are the things programs manipulate?
        ↓
Values and Types
```

No advanced specification knowledge is required, but the learner should
remember that this chapter describes **language-level semantics**, not a
particular engine's physical memory layout.

------------------------------------------------------------------------

## 3. What Is a Value?

A **value** is a piece of data that can participate in JavaScript
program evaluation.

Examples:

``` js
42
"hello"
true
undefined
null
123n
Symbol("id")
{ name: "Prasanta" }
```

Each expression produces a value or a completion that includes a value
or an abrupt outcome such as an exception.

For example:

``` js
2 + 3
```

produces the numeric value:

``` text
5
```

And:

``` js
"hello"
```

produces a string value.

A critical distinction:

``` js
const age = 25;
```

The variable `age` is not the value `25`.

A useful model is:

``` text
binding
   ↓
value
```

The binding is associated with the value.

Later chapters will formalize this using lexical environments,
environment records, references, and `GetValue` / `PutValue`.

------------------------------------------------------------------------

# 4. Value vs Variable

Compare:

``` js
const age = 25;
```

There are several different concepts here.

### Identifier

``` text
age
```

The name used in source code.

### Binding

A language-level association between the identifier and its current
value.

### Value

``` text
25
```

The numeric value.

### Declaration

``` js
const age = 25;
```

The syntax that creates the binding.

This distinction becomes essential when learning:

-   reassignment,
-   mutation,
-   closures,
-   references,
-   destructuring,
-   function parameters,
-   modules.

------------------------------------------------------------------------

# 5. What Is a Type?

A **type** classifies values according to the operations and semantic
behavior associated with them.

At the ECMAScript language level, there are **eight language types**:

``` text
1. Undefined
2. Null
3. Boolean
4. String
5. Symbol
6. Number
7. BigInt
8. Object
```

A simplified diagram:

``` text
ECMAScript Language Types
│
├── Undefined
├── Null
├── Boolean
├── String
├── Symbol
├── Number
├── BigInt
└── Object
```

The first seven are **primitive types**.

`Object` is the object type.

Be careful with terminology here: "primitive" and "object" are
language-level classifications. They do not directly tell you how an
engine physically stores values.

------------------------------------------------------------------------

# 6. Primitive Values

Primitive values are values that are not objects.

The primitive types are:

``` text
undefined
null
boolean
number
bigint
string
symbol
```

Examples:

``` js
undefined
null
true
42
123n
"hello"
Symbol("id")
```

Primitives are immutable.

For example:

``` js
let x = "hello";
```

You cannot mutate the characters inside that string value.

You can replace the value associated with `x`:

``` js
x = "world";
```

But that is reassignment of a binding, not mutation of the original
string.

------------------------------------------------------------------------

# 7. Objects

Objects are values with object semantics.

Examples:

``` js
{}
[]
function () {}
new Date()
new Map()
new Set()
```

Objects can have properties and can participate in prototype-based
behavior.

For example:

``` js
const user = {
  name: "Prasanta",
  age: 25
};
```

The object has properties:

``` text
name → "Prasanta"
age  → 25
```

Objects also have identity.

Two separately created objects are distinct values:

``` js
const a = {};
const b = {};

console.log(a === b);
```

Result:

``` text
false
```

Even though they have the same shape and contents, they are different
object identities.

------------------------------------------------------------------------

# 8. Primitive vs Object --- First Fundamental Contrast

Compare:

``` js
const a = 10;
const b = 10;

a === b;
```

Result:

``` text
true
```

Now:

``` js
const a = {};
const b = {};

a === b;
```

Result:

``` text
false
```

Why?

Because primitive equality and object identity follow different
semantics.

For primitive values, the language compares values according to the
relevant equality algorithm.

For objects, equality is based on whether the operands refer to the same
object.

This distinction will become central in Chapter 07.

------------------------------------------------------------------------

# 9. Type Identity and `typeof`

JavaScript provides the `typeof` operator for runtime type
classification.

Examples:

``` js
typeof undefined;   // "undefined"
typeof null;        // "object"
typeof true;        // "boolean"
typeof 42;          // "number"
typeof 123n;        // "bigint"
typeof "hello";     // "string"
typeof Symbol();    // "symbol"
typeof {};          // "object"
typeof function(){};// "function"
```

The last line is important.

`function` is not one of the eight ECMAScript language types.

The ECMAScript language has an **Object** type, and functions are
callable objects.

`typeof` nevertheless has a special result:

``` js
typeof function () {};
```

returns:

``` text
"function"
```

This is a language-level behavior of the `typeof` operator, not evidence
that there is a ninth "Function" language type.

------------------------------------------------------------------------

# 10. The `typeof null` Historical Anomaly

One of the most famous JavaScript behaviors is:

``` js
typeof null;
```

Result:

``` text
"object"
```

Yet:

``` text
Null
```

is a distinct ECMAScript language type.

Therefore:

``` js
typeof null === "object"
```

does not mean:

``` text
null is an Object
```

It means the `typeof` operator has a historical language behavior that
reports `"object"` for `null`.

This is a classic example of why:

``` text
operator output
```

and:

``` text
language type identity
```

are not always interchangeable concepts.

Do not write:

> "`null` is an object."

That is false at the ECMAScript type level.

Write:

> "`null` has the Null language type, while `typeof null` returns
> `"object"` for historical compatibility."

------------------------------------------------------------------------

# 11. Undefined

`undefined` is a primitive value of the Undefined type.

Example:

``` js
let value;
console.log(value);
```

Result:

``` text
undefined
```

A variable declared without an initializer has the value `undefined`
after its declaration is instantiated and initialized appropriately.

You also commonly encounter `undefined` when:

-   an object property does not exist,
-   a function does not explicitly return a value,
-   an array element is a hole-aware or missing result depending on the
    operation,
-   APIs intentionally use it as a sentinel.

Example:

``` js
const user = {};

console.log(user.name);
```

Result:

``` text
undefined
```

But:

``` js
user.name
```

being `undefined` does not tell you whether the property exists.

These can differ:

``` js
const a = {};
const b = { name: undefined };

console.log(a.name); // undefined
console.log(b.name); // undefined

console.log("name" in a); // false
console.log("name" in b); // true
```

This distinction becomes important in property semantics and API design.

------------------------------------------------------------------------

# 12. Null

`null` is a primitive value of the Null type.

It is commonly used to represent:

-   intentional absence,
-   empty object-like state,
-   "no value" chosen by an API or programmer.

Example:

``` js
let selectedUser = null;
```

This can communicate:

``` text
The variable is intentionally empty.
```

That is different from:

``` js
let selectedUser;
```

which produces:

``` text
undefined
```

These meanings are not universally enforced by the language. They are
semantic conventions used by programs and APIs.

------------------------------------------------------------------------

# 13. Boolean

The Boolean type has exactly two values:

``` js
true
false
```

Example:

``` js
const isAuthenticated = true;
```

Boolean values are frequently produced by:

``` js
===
>
<
in
instanceof
```

and by logical operations after the relevant coercion rules are applied.

Do not confuse:

``` js
false
```

with:

``` js
0
""
null
undefined
NaN
```

Those are different values of different types, even though several are
**falsy** in Boolean contexts.

Truthiness is a coercion topic and will be handled in depth in Chapter
07.

------------------------------------------------------------------------

# 14. Number

JavaScript's `Number` type represents numeric values using the IEEE 754
binary64 floating-point format.

That means it can represent:

-   ordinary integers,
-   fractions,
-   `NaN`,
-   positive infinity,
-   negative infinity,
-   positive zero,
-   negative zero.

Examples:

``` js
42
3.14
NaN
Infinity
-Infinity
0
-0
```

Because of floating-point representation, not every mathematical real
number can be represented exactly.

For example:

``` js
0.1 + 0.2
```

produces a result that is not exactly the mathematical decimal `0.3`.

This is not an accidental implementation bug. It follows from the
representation used by the Number type.

Chapter 03 will examine this deeply.

------------------------------------------------------------------------

# 15. BigInt

The `BigInt` type represents arbitrary-precision integers.

Example:

``` js
const big = 1234567890123456789012345678901234567890n;
```

The trailing `n` indicates a BigInt literal.

BigInt exists because Number cannot exactly represent every integer
beyond its safe-integer range.

Example:

``` js
9007199254740991
```

is `Number.MAX_SAFE_INTEGER`.

For integers outside the safe range, using Number arithmetic can lose
integer precision.

BigInt solves a different problem:

``` text
Number
  → floating-point numeric representation

BigInt
  → arbitrary-precision integer representation
```

They are deliberately distinct types.

For example:

``` js
1n + 2n;
```

works.

But:

``` js
1n + 2;
```

throws a `TypeError`.

JavaScript does not silently mix Number and BigInt arithmetic.

Chapter 03 will explore the reason and the exact rules.

------------------------------------------------------------------------

# 16. String

A String is a primitive value representing textual data.

Examples:

``` js
"hello"
'hello'
`hello`
```

Strings are immutable values.

Example:

``` js
let text = "hello";

text[0] = "H";

console.log(text);
```

The attempt does not mutate the primitive string into `"Hello"`.

Instead, string operations produce new string values.

String semantics become significantly more complex with:

-   UTF-16 code units,
-   Unicode code points,
-   surrogate pairs,
-   grapheme clusters,
-   normalization,
-   locale-sensitive operations.

Those details are intentionally covered in Chapter 04.

------------------------------------------------------------------------

# 17. Symbol

A Symbol is a primitive value intended primarily for unique property
keys and certain language-level customization mechanisms.

Example:

``` js
const id = Symbol("id");
```

Two separately created Symbols are different:

``` js
Symbol("id") === Symbol("id");
```

Result:

``` text
false
```

The descriptions can be identical while the symbols remain distinct
values.

Symbols also participate in the language's meta-level protocols through
well-known symbols such as:

``` js
Symbol.iterator
Symbol.toPrimitive
Symbol.toStringTag
Symbol.hasInstance
```

Those will be covered later.

------------------------------------------------------------------------

# 18. Object

The Object type covers object values.

Objects can be ordinary objects:

``` js
{}
```

or specialized built-in objects:

``` js
[]
new Map()
new Set()
new Date()
/abc/
new Promise(...)
```

Functions are also objects.

This is why you can write:

``` js
function add(a, b) {
  return a + b;
}

add.description = "addition";
```

The function is callable, but it can also have properties.

This is an important conceptual pattern:

``` text
Object
├── properties
├── identity
├── prototype relationship
└── may have [[Call]]
```

Not every object is callable.

Callable objects are a special subset of objects.

------------------------------------------------------------------------

# 19. Functions Are Callable Objects

Consider:

``` js
function greet() {
  return "hello";
}
```

You can:

``` js
greet();
```

because the object is callable.

You can also:

``` js
greet.description = "Greeting function";
```

because functions can have properties.

Therefore:

``` text
Function
=
Object
+
callable behavior
```

This is a much better mental model than treating `function` as a
completely separate fundamental value category.

Some function objects can also be constructable:

``` js
function Person(name) {
  this.name = name;
}

new Person("A");
```

Others are callable but not constructable:

``` js
const arrow = () => 42;

new arrow();
```

throws a `TypeError`.

Callable and constructable are distinct concepts.

Chapter 14 and the object-model chapters will explore this in detail.

------------------------------------------------------------------------

# 20. Dynamic Typing

JavaScript is dynamically typed.

This means the type is a property of the **value**, not a permanent
static property of the variable name.

Example:

``` js
let value = 10;
```

The current value is a Number.

Later:

``` js
value = "hello";
```

The current value is a String.

Later:

``` js
value = { active: true };
```

The current value is an Object.

The identifier `value` did not acquire one immutable language type.

The value associated with the binding changed.

A better statement is:

``` text
JavaScript values have types.
Bindings can be associated with values of different types over time.
```

------------------------------------------------------------------------

# 21. Dynamic Typing Does Not Mean "No Types"

A common misconception is:

> "JavaScript has no types."

False.

JavaScript has a well-defined language type system.

For example:

``` js
42
```

has Number type.

``` js
"42"
```

has String type.

``` js
42n
```

has BigInt type.

``` js
{}
```

has Object type.

The difference is that JavaScript does not require variables to be
declared with a static type like:

``` text
int
string
User
```

as a requirement of the ECMAScript language.

------------------------------------------------------------------------

# 22. Static vs Dynamic Typing

A useful conceptual comparison:

### Statically typed language

Type relationships are checked primarily before execution as part of the
language/toolchain model.

Example:

``` text
int x = 10;
x = "hello";  // type error in a typical statically typed system
```

### Dynamically typed JavaScript

``` js
let x = 10;
x = "hello";
```

This is valid.

The binding can now refer to a String value.

Important:

> Static vs dynamic typing and strong vs weak typing are not identical
> classification axes.

Those concepts should not be collapsed into one simplistic label.

JavaScript's type coercion rules are complex and will be treated
separately.

------------------------------------------------------------------------

# 23. Value Semantics

Primitive values behave like values.

Example:

``` js
let a = 10;
let b = a;

b = 20;

console.log(a);
console.log(b);
```

Result:

``` text
10
20
```

Conceptually:

``` text
a ──→ 10

b ──→ 10

then b changes:

a ──→ 10
b ──→ 20
```

The assignment copied the value relationship.

The two bindings do not become one shared mutable numeric object.

------------------------------------------------------------------------

# 24. Object Identity

Now compare:

``` js
const a = { count: 10 };
const b = a;

b.count = 20;

console.log(a.count);
```

Result:

``` text
20
```

Why?

Because `a` and `b` are associated with the same object.

A useful conceptual model:

``` text
a ─────┐
       ├──→ Object #1 { count: 10 }
b ─────┘
```

After:

``` js
b.count = 20;
```

both bindings still identify Object #1.

So:

``` js
a === b
```

is:

``` text
true
```

------------------------------------------------------------------------

# 25. "Passed by Reference" Is an Imprecise Shortcut

You will often hear:

> "Objects are passed by reference in JavaScript."

This is an unreliable explanation.

A better model is:

> JavaScript passes values. When the value is an object, that value
> provides access to the same object identity.

Example:

``` js
function update(user) {
  user.name = "Changed";
}

const person = { name: "Original" };

update(person);

console.log(person.name);
```

Result:

``` text
Changed
```

The function receives a value representing the object.

The parameter binding and the original binding both allow access to the
same object.

But the parameter itself is a separate binding.

------------------------------------------------------------------------

# 26. The Parameter Binding Experiment

Consider:

``` js
function replace(user) {
  user = { name: "New object" };
}

const person = { name: "Original" };

replace(person);

console.log(person.name);
```

Result:

``` text
Original
```

Why?

Inside `replace`:

``` text
parameter user
     ↓
new object
```

The reassignment changes the local parameter binding.

It does not change the caller's `person` binding.

Conceptually:

``` text
Before call:

person ─────→ Object #1


Inside function initially:

person ─────→ Object #1
user   ─────→ Object #1


After user = { ... }:

person ─────→ Object #1
user   ─────→ Object #2
```

This is the cleanest way to understand JavaScript's parameter passing.

------------------------------------------------------------------------

# 27. Mutation vs Reassignment

This distinction is essential.

### Mutation

Changing the existing object:

``` js
user.name = "New";
```

The same object is modified.

### Reassignment

Changing which value a binding refers to:

``` js
user = { name: "New" };
```

The binding now points to a different value.

These are not the same operation.

Many confusing interview questions become trivial once you ask:

> "Did we mutate the object, or did we reassign the binding?"

------------------------------------------------------------------------

# 28. Immutability of Primitive Values

Primitive values are immutable.

Example:

``` js
let x = "abc";
```

There is no operation that mutates the existing string value itself.

Operations create another value:

``` js
x.toUpperCase();
```

produces:

``` text
"ABC"
```

and leaves the original string value unchanged.

Likewise:

``` js
const n = 10;
```

The Number value `10` cannot be mutated into `11`.

You can create a new numeric value and reassign a mutable binding:

``` js
let n = 10;
n = 11;
```

Again:

``` text
reassignment ≠ mutation
```

------------------------------------------------------------------------

# 29. `const` Does Not Make Objects Immutable

A critical consequence:

``` js
const user = {
  name: "A"
};

user.name = "B";
```

This is valid.

Why?

`const` protects the binding from reassignment.

It does not recursively freeze the object.

This is legal:

``` js
user.name = "B";
```

This is not:

``` js
user = {};
```

The distinction:

``` text
const
  → binding reassignment restriction

Object.freeze
  → object-level property mutation restrictions
```

Even `Object.freeze` itself is shallow and does not automatically
recursively freeze nested objects.

These topics return in the object and immutability discussions.

------------------------------------------------------------------------

# 30. Boxing

JavaScript allows primitive values to participate in property access and
method calls.

For example:

``` js
const text = "hello";

console.log(text.toUpperCase());
```

Strings are primitive values.

Yet property/method syntax works:

``` js
text.toUpperCase()
```

Conceptually, JavaScript can perform an operation equivalent to
temporarily treating the primitive as an appropriate object wrapper so
the property can be resolved.

For strings, the conceptual wrapper is related to `String`.

For numbers:

``` js
(42).toString();
```

For booleans:

``` js
true.toString();
```

This process is commonly called **boxing** or is described through
primitive-to-object conversion semantics.

Do not confuse this with the primitive becoming permanently converted
into a wrapper object.

------------------------------------------------------------------------

# 31. Wrapper Objects

JavaScript also provides explicit wrapper constructors:

``` js
new String("hello");
new Number(42);
new Boolean(true);
```

These produce objects, not primitives.

Compare:

``` js
typeof "hello";
```

with:

``` js
typeof new String("hello");
```

Results:

``` text
"string"
"object"
```

And:

``` js
"hello" === new String("hello");
```

Result:

``` text
false
```

Why?

Different types and different categories:

``` text
"hello"
   → primitive String

new String("hello")
   → Object wrapping a string
```

Wrapper objects are usually not what you want in ordinary application
code.

Use primitives:

``` js
const name = "hello";
const count = 42;
const enabled = true;
```

rather than wrapper objects.

------------------------------------------------------------------------

# 32. Why Boxing Exists

Without primitive property behavior, code such as:

``` js
"hello".length
```

would be awkward.

The language provides semantics allowing primitives to interact
naturally with object-provided methods and properties.

This gives JavaScript a useful illusion:

``` text
primitive string
   ↓
can access String-related behavior
```

while preserving the primitive's underlying value semantics.

The exact specification algorithms involved include operations such as
`ToObject`, property access, and internal method dispatch.

These will be formally studied later.

------------------------------------------------------------------------

# 33. `null` and `undefined` Are Not the Same

Although both commonly represent absence, they are distinct values.

``` js
null === undefined
```

Result:

``` text
false
```

And:

``` js
typeof null      // "object"
typeof undefined // "undefined"
```

Their semantic conventions often differ:

``` text
undefined
  → missing / not initialized / not returned / absent result

null
  → deliberate explicit empty value
```

But these are conventions, not a universal language rule.

Libraries and APIs may choose one or the other.

A good API designer should document which one is used.

------------------------------------------------------------------------

# 34. Object Identity Is Not Structural Equality

Compare:

``` js
const a = { x: 1 };
const b = { x: 1 };

a === b;
```

Result:

``` text
false
```

Even though:

``` text
a.x === b.x
```

is:

``` text
true
```

JavaScript does not define `===` for ordinary objects as "compare all
fields recursively."

Objects have identity.

This matters for:

-   caching,
-   `Map` keys,
-   `Set`,
-   memoization,
-   React state comparisons,
-   database object representations,
-   deduplication,
-   event listeners.

------------------------------------------------------------------------

# 35. Same Primitive, Same Value

Compare:

``` js
const a = 100;
const b = 100;

a === b;
```

Result:

``` text
true
```

The values are the same according to strict equality.

The language does not need two distinct "100 objects" in the semantic
model.

Again, do not turn this into a physical-memory claim.

Whether the engine internally stores small integers using a particular
representation is an implementation detail.

The language-level fact is about values and equality.

------------------------------------------------------------------------

# 36. Language Types vs Physical Storage

This is another critical boundary.

The ECMAScript specification may say:

``` text
The value has Number type.
```

That does **not** mean:

``` text
the engine stores it as a particular fixed memory structure
```

An engine may use:

-   tagged representations,
-   immediate values,
-   pointer compression,
-   specialized numeric representations,
-   object layouts,
-   optimized machine values.

Those implementation choices can change.

Therefore:

``` text
Language type
      ≠
physical memory representation
```

Chapter 47 and Chapter 48 will explore engine internals.

------------------------------------------------------------------------

# 37. Object Type Is Broader Than Plain `{}`

When developers hear "object," they often imagine:

``` js
{}
```

But ECMAScript objects include many specialized objects:

``` js
[]
new Date()
new RegExp("x")
new Map()
new Set()
new WeakMap()
new Promise(...)
function () {}
new Uint8Array()
```

All are object values at the language level.

Yet their behavior can differ significantly because they expose
different internal semantics, methods, slots, and protocols.

Thus:

``` text
Object
```

is a language category, not a synonym for:

``` text
plain object literal
```

------------------------------------------------------------------------

# 38. Arrays Are Objects

This is a foundational fact:

``` js
typeof [];
```

Result:

``` text
"object"
```

Arrays are objects with specialized array behavior.

Therefore:

``` js
Array.isArray([]);
```

returns:

``` text
true
```

But:

``` js
typeof []
```

does not return:

``` text
"array"
```

There is no `array` result in `typeof`.

Chapter 22 will examine array semantics in depth.

------------------------------------------------------------------------

# 39. Dates Are Objects

Likewise:

``` js
typeof new Date();
```

returns:

``` text
"object"
```

The Date object is not a primitive date value.

Its semantics are implemented through object behavior.

This distinction becomes important when learning serialization and the
newer Temporal model.

------------------------------------------------------------------------

# 40. Regular Expressions Are Objects

Consider:

``` js
const re = /hello/;
```

The value is an object.

It can have methods such as:

``` js
re.test("hello");
```

and special behavior through well-known symbol protocols.

Again, "object" is the broad category.

------------------------------------------------------------------------

# 41. BigInt and Number Are Different Types

Consider:

``` js
typeof 10;
```

Result:

``` text
"number"
```

while:

``` js
typeof 10n;
```

Result:

``` text
"bigint"
```

This matters because arithmetic semantics differ.

``` js
10 + 20
```

produces:

``` text
30
```

while:

``` js
10n + 20n
```

produces:

``` text
30n
```

But:

``` js
10n + 20;
```

throws.

JavaScript intentionally prevents implicit arithmetic mixing between
these numeric types.

------------------------------------------------------------------------

# 42. Symbols and Property Keys

Symbols are especially important because property keys in JavaScript are
not limited to strings.

An object property key can be:

``` text
String
Symbol
```

Example:

``` js
const secret = Symbol("secret");

const user = {
  name: "A",
  [secret]: 123
};
```

The symbol property is a real property but behaves differently from
ordinary string-keyed enumeration APIs.

Later chapters will cover:

-   property keys,
-   symbol keys,
-   enumeration,
-   `Reflect.ownKeys`,
-   well-known symbols.

------------------------------------------------------------------------

# 43. The Type System Is Connected to Every Later Chapter

The type system is not a chapter you memorize and forget.

It controls how later concepts behave.

### Operators

``` js
1 + 2
```

depends on numeric semantics.

``` js
"1" + 2
```

depends on string conversion.

### Equality

``` js
1 === "1"
```

depends on type identity.

### Properties

``` js
"hello".length
```

depends on primitive/object interaction.

### Functions

Functions are objects with callable behavior.

### Collections

`Map` distinguishes keys based on value/identity semantics.

### Promises

Promises are objects with standardized internal state and behavior.

### Serialization

Different types serialize differently.

### Memory

Object identity affects reachability and garbage collection.

The type system is therefore a dependency hub.

------------------------------------------------------------------------

# 44. Common Misconceptions

## Misconception 1 --- "JavaScript has seven types."

Incorrect.

At the ECMAScript language level there are eight language types:

``` text
Undefined
Null
Boolean
String
Symbol
Number
BigInt
Object
```

The confusion usually comes from listing only the seven primitive types.

------------------------------------------------------------------------

## Misconception 2 --- "Functions are a separate ECMAScript language type."

Not as one of the eight language types.

Functions are objects with callable behavior.

------------------------------------------------------------------------

## Misconception 3 --- "`typeof null === "object"` means null is an object."

False.

`null` has Null type.

`typeof` has a historical result of `"object"` for it.

------------------------------------------------------------------------

## Misconception 4 --- "Objects are passed by reference."

This is an imprecise shortcut.

JavaScript passes values. Object values identify objects, so copied
parameter values can provide access to the same object identity.

------------------------------------------------------------------------

## Misconception 5 --- "`const` makes the object immutable."

False.

`const` restricts reassignment of the binding.

------------------------------------------------------------------------

## Misconception 6 --- "Primitive and object storage can be inferred from `typeof`."

False.

`typeof` reports language-level classification.

It does not reveal the physical representation inside an engine.

------------------------------------------------------------------------

## Misconception 7 --- "An object means a plain object literal."

False.

Arrays, functions, maps, sets, dates, regular expressions, promises,
typed arrays, and many other values are objects.

------------------------------------------------------------------------

# 45. Common Mistakes

### Mistake: comparing objects structurally with `===`

``` js
{} === {}
```

is `false`.

### Mistake: using wrapper objects

Avoid:

``` js
new Number(5)
new String("x")
new Boolean(false)
```

in normal application code.

Prefer primitives.

### Mistake: treating missing and explicit `undefined` as identical API states

``` js
"key" in obj
```

may distinguish them.

### Mistake: saying a variable "has a permanent type"

The current value has a type; a mutable binding can later refer to a
different type of value.

### Mistake: confusing mutation with reassignment

Always ask which binding and which object are being changed.

------------------------------------------------------------------------

# 46. Comparison Table

  -----------------------------------------------------------------------
  Concept      Example          Mutable?     Identity?    Typical
                                                          `typeof`
  ------------ ---------------- ------------ ------------ ---------------
  Undefined    `undefined`      No           No object    `"undefined"`
                                             identity     

  Null         `null`           No           No object    `"object"`
                                             identity     

  Boolean      `true`           No           No object    `"boolean"`
                                             identity     

  Number       `42`             No           No object    `"number"`
                                             identity     

  BigInt       `42n`            No           No object    `"bigint"`
                                             identity     

  String       `"hello"`        No           No object    `"string"`
                                             identity     

  Symbol       `Symbol()`       No           Symbol       `"symbol"`
                                             identity     
                                             matters      

  Object       `{}`             Yes,         Yes          `"object"`
                                depending on              
                                properties                

  Function     `function(){}`   Object can   Yes          `"function"`
  object                        have mutable              
                                properties                
  -----------------------------------------------------------------------

The `typeof` column is an operator result, not a direct list of all
language types.

------------------------------------------------------------------------

# 47. Specification-Level Mental Model

At specification level, avoid imagining variables as boxes containing
raw bytes.

Instead think in terms of:

``` text
Identifier
   ↓
Binding / environment record
   ↓
Value
```

When the value is an object:

``` text
Binding
   ↓
Object identity
   ↓
Object properties / internal state
```

This abstraction lets the language define behavior independently of
physical machine layout.

Later:

``` text
Execution Context
   ↓
Lexical Environment
   ↓
Environment Record
   ↓
Binding
   ↓
Value
```

This is the bridge from this chapter to the scope and execution
chapters.

------------------------------------------------------------------------

# 48. Execution Walkthrough

Consider:

``` js
let value = 10;

value = "hello";
```

At a conceptual level:

### Step 1

A mutable binding named `value` is created.

### Step 2

The initializer expression:

``` js
10
```

produces a Number value.

### Step 3

The binding becomes associated with that value.

Conceptually:

``` text
value → 10
```

### Step 4

The second assignment evaluates:

``` js
"hello"
```

which produces a String value.

### Step 5

The mutable binding is updated:

``` text
value → "hello"
```

No "variable type conversion" occurred.

The binding changed which value it contains/identifies.

------------------------------------------------------------------------

# 49. Execution Walkthrough --- Object Mutation

Consider:

``` js
const user = {
  name: "A"
};

user.name = "B";
```

Conceptually:

### Step 1

Create an object.

``` text
Object #1
name → "A"
```

### Step 2

Create an immutable binding:

``` text
user → Object #1
```

### Step 3

Property assignment finds Object #1.

### Step 4

Its `name` property is updated.

Final state:

``` text
user → Object #1
             name → "B"
```

The `user` binding itself was never reassigned.

------------------------------------------------------------------------

# 50. Execution Walkthrough --- Parameter Passing

``` js
function change(user) {
  user.name = "B";
}

const person = { name: "A" };
change(person);
```

Conceptually:

``` text
person ─────→ Object #1

Call:
user   ─────→ Object #1
```

Then:

``` js
user.name = "B";
```

mutates Object #1.

After the function returns:

``` text
person ─────→ Object #1
             name → "B"
```

The important fact is shared object identity.

------------------------------------------------------------------------

# 51. Performance Considerations

Do not infer performance from language type names alone.

For example:

``` js
const x = 10;
```

and:

``` js
const obj = { value: 10 };
```

have very different language-level semantics.

An engine may internally optimize either representation aggressively.

Performance depends on:

-   engine implementation,
-   object shapes/layouts,
-   type stability,
-   allocation behavior,
-   access patterns,
-   JIT optimizations,
-   deoptimization,
-   garbage collection,
-   workload.

Therefore:

``` text
language type
```

is not itself a complete performance model.

Chapter 48 will examine engine-specific type/shape optimizations.

------------------------------------------------------------------------

# 52. Memory Considerations

From the language perspective:

-   Primitive values do not have mutable object identity.
-   Objects have identity and can contain references to other values.
-   Object graphs can therefore become large and interconnected.
-   Garbage collection depends on object reachability, not simply
    whether a variable name still appears in source code.

Example:

``` js
let a = {
  child: {}
};

let b = a;

a = null;
```

The object may still be reachable through `b`.

Therefore assigning:

``` js
a = null;
```

does not necessarily make the object collectible.

The memory chapters will formalize reachability and garbage collection.

------------------------------------------------------------------------

# 53. Security Considerations

Type confusion at application boundaries can create security problems.

Examples include:

``` js
if (typeof input === "string") {
  // treat as trusted
}
```

A type check alone does not establish that data is safe.

Likewise:

``` js
typeof value === "object"
```

does not tell you:

-   whether it is `null`,
-   whether it came from an untrusted source,
-   whether it has malicious properties,
-   whether it is an instance you expect,
-   whether it has unexpected prototype behavior.

Security requires both:

``` text
type correctness
+
trust-boundary validation
```

The security chapters will build this into a broader input-validation
model.

------------------------------------------------------------------------

# 54. Production Usage

A production engineer should establish explicit conventions around
absence and optional data.

For example, APIs may distinguish:

``` text
field omitted
```

from:

``` text
field present with null
```

from:

``` text
field present with empty string
```

These may represent different business meanings.

Consider a PATCH-style API:

``` json
{}
```

might mean:

``` text
do not change name
```

while:

``` json
{ "name": null }
```

might mean:

``` text
clear name
```

That distinction is not merely syntax.

It is a data-model and API-contract decision.

------------------------------------------------------------------------

# 55. Production Example --- Database Data

Suppose a database record contains:

``` js
{
  middleName: null
}
```

and your JavaScript application receives:

``` js
const middleName = record.middleName;
```

You must know whether:

``` text
null
```

means "known to be empty"

versus:

``` text
undefined
```

meaning "field was not included / is unavailable."

Do not normalize these values casually without understanding the
contract.

------------------------------------------------------------------------

# 56. Production Example --- Configuration

Consider:

``` js
const timeout = config.timeout;
```

These states may mean different things:

``` text
undefined → use system default
null      → explicitly disable / invalid depending on contract
0         → no timeout, or invalid, depending on contract
```

A robust configuration system defines these semantics explicitly.

Types are therefore part of system design, not merely language trivia.

------------------------------------------------------------------------

# 57. Implementation From Scratch --- Type Classifier

Build a function without using libraries:

``` js
function classify(value) {
  // return a richer classification than typeof
}
```

Required outcomes:

``` text
undefined → Undefined
null      → Null
true      → Boolean
42        → Number
42n       → BigInt
"hello"   → String
Symbol()  → Symbol
{}        → Object
[]        → Array/Object
function  → Function/Object
```

The goal is to discover why `typeof` alone is insufficient.

A production-quality classifier must carefully define whether it is
reporting:

-   ECMAScript language type,
-   built-in category,
-   array-ness,
-   callable status,
-   or application-specific type.

Do not mix those concepts.

------------------------------------------------------------------------

# 58. Implementation Progression

### Guided

Implement:

``` js
function isPrimitive(value) {
  // ...
}
```

### Partially Guided

Add helpers:

``` js
isNull
isUndefined
isObject
isCallable
```

### No Reference

Create:

``` js
describeValue(value)
```

that reports:

``` text
language-ish category
primitive/object
callable
array
nullish
```

### Edge-Case Hardened

Test:

``` js
null
undefined
[]
{}
function(){}
class Person {}
new Date()
new Map()
Object.create(null)
```

### Production-Grade

Document exactly what your function guarantees and does not guarantee.

This reinforces a principal-level lesson:

> A classifier is only correct relative to the classification contract
> it promises.

------------------------------------------------------------------------

# 59. Debugging Exercises

## Exercise 1

Explain:

``` js
console.log(typeof null);
```

without saying:

> "null is an object."

## Exercise 2

Explain:

``` js
const a = {};
const b = a;

console.log(a === b);
```

## Exercise 3

Explain:

``` js
const a = {};
const b = {};

console.log(a === b);
```

## Exercise 4

Explain:

``` js
function test(x) {
  x = {};
}

const original = {};
test(original);
```

Does `original` change?

## Exercise 5

Explain:

``` js
function test(x) {
  x.value = 2;
}

const original = { value: 1 };
test(original);
```

Does `original` change?

The important skill is identifying:

``` text
reassignment
vs
mutation
```

------------------------------------------------------------------------

# 60. Predict-the-Output Exercises

Predict before running.

### 1

``` js
console.log(typeof undefined);
```

### 2

``` js
console.log(typeof null);
```

### 3

``` js
console.log(typeof 10n);
```

### 4

``` js
console.log(typeof Symbol("x"));
```

### 5

``` js
console.log(typeof []);
```

### 6

``` js
console.log(typeof function () {});
```

### 7

``` js
const a = {};
const b = {};
console.log(a === b);
```

### 8

``` js
const a = {};
const b = a;
console.log(a === b);
```

### 9

``` js
let x = 10;
let y = x;
y = 20;

console.log(x, y);
```

### 10

``` js
const x = { n: 1 };
const y = x;
y.n = 2;

console.log(x.n, y.n);
```

### 11

``` js
function change(x) {
  x = { n: 2 };
}

const obj = { n: 1 };
change(obj);

console.log(obj.n);
```

### 12

``` js
function change(x) {
  x.n = 2;
}

const obj = { n: 1 };
change(obj);

console.log(obj.n);
```

------------------------------------------------------------------------

# 61. Code Review Exercise

Review this explanation:

> "`const user = {}` makes `user` immutable because const means the
> value can't change."

Identify the error.

Correct explanation:

``` text
const prevents reassignment of the binding.
It does not make the object immutable.
```

Then review:

``` js
const user = Object.freeze({
  name: "A"
});
```

Explain why this changes the situation, and why `Object.freeze` still
does not imply deep recursive immutability.

------------------------------------------------------------------------

# 62. Interview Questions

## Junior

1.  What are the JavaScript primitive types?
2.  What is the difference between a primitive and an object?
3.  What does `typeof` do?
4.  Why is `typeof null` `"object"`?
5.  Is an array an object?

## Mid-Level

6.  Why does `typeof function () {}` return `"function"`?
7.  What is dynamic typing?
8.  Does JavaScript have types if it is dynamically typed?
9.  Explain mutation vs reassignment.
10. Why does assigning one object variable to another make mutations
    visible through both?

## Senior

11. Explain precisely why "objects are passed by reference" is
    misleading.
12. How does object identity differ from structural equality?
13. What are wrapper objects?
14. What is boxing?
15. What is the relationship between functions and objects?

## Staff / Principal

16. How would you define a type abstraction for an API boundary?
17. When is `null` preferable to `undefined`, and who should decide?
18. How can incorrect assumptions about object identity create cache
    bugs?
19. Why should language-level type classifications not be treated as
    engine memory layouts?
20. How would you design a runtime type-validation layer for untrusted
    JSON?

------------------------------------------------------------------------

# 63. Mastery Exercises

## Exercise A --- Explain the Eight Types

Without notes, explain all eight ECMAScript language types and give one
representative value for each.

## Exercise B --- Binding vs Value vs Object

Explain the difference between:

``` text
identifier
binding
value
object identity
property
mutation
reassignment
```

using one example.

## Exercise C --- Pass-by-Value Defense

Defend the statement:

> "JavaScript is pass-by-value."

Then explain why developers still experience shared object mutations.

## Exercise D --- Design an API Contract

Design an API where these states are meaningful:

``` text
omitted
undefined
null
empty string
false
0
```

Document exactly what each means.

## Exercise E --- Build a Type Matrix

Create a table covering:

``` text
value
language type
typeof result
primitive/object
identity
mutable/immutable
callable
constructable
```

for:

``` js
undefined
null
true
0
0n
""
Symbol()
{}
[]
function(){}
async function(){}
class A {}
new Map()
```

------------------------------------------------------------------------

# 64. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.

## Builds Toward

-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.
-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.
-   Chapter 09 --- Functions.
-   Chapter 10 --- Scope and Lexical Environments.
-   Chapter 12 --- Execution Contexts.
-   Chapter 13 --- Closures.
-   Chapter 15 --- Object Property Semantics.
-   Chapter 17 --- Prototypes.
-   Chapter 22 --- Arrays.
-   Chapter 24 --- Map / Set / WeakMap / WeakSet.
-   Chapter 28 --- Serialization.
-   Chapter 45 --- Memory and Garbage Collection.
-   Chapter 48 --- V8 Optimization.
-   Chapter 57 --- Security Engineering.

## Related Concepts

``` text
values
types
bindings
identity
mutation
reassignment
coercion
equality
boxing
objects
functions
```

## Concepts Revisited Later

The following ideas will return at greater depth:

-   `ToPrimitive`
-   `ToObject`
-   `GetValue`
-   `PutValue`
-   equality algorithms,
-   property access,
-   object identity,
-   callable objects,
-   constructable objects,
-   garbage collection,
-   internal slots,
-   engine representations.

## Why This Chapter Matters Later

Every JavaScript feature ultimately operates on values.

If the learner cannot answer:

``` text
What value is this?
What language type does it have?
Is it an object?
Does it have identity?
Was it mutated or reassigned?
Is the behavior language-level or engine-specific?
```

then later topics become memorization rather than understanding.

------------------------------------------------------------------------

# 65. Key Takeaways

1.  JavaScript values have language-defined types.
2.  ECMAScript has eight language types.
3.  Seven are primitive types; one is Object.
4.  Primitive values are immutable.
5.  Objects have identity and properties.
6.  Functions are objects with callable behavior.
7.  `typeof` is useful but does not directly enumerate the eight
    language types.
8.  `typeof null === "object"` is a historical behavior; `null` itself
    has Null type.
9.  JavaScript is dynamically typed, not typeless.
10. Bindings can be reassigned to values of different types.
11. Objects are not best understood as being "passed by reference";
    JavaScript passes values, and object values can identify shared
    object identity.
12. Mutation and reassignment are fundamentally different operations.
13. `const` protects a binding from reassignment; it does not make an
    object immutable.
14. Primitive property/method access can involve boxing-like object
    conversion semantics.
15. Wrapper objects such as `new Number()` are objects and are generally
    not preferred in normal application code.
16. Language types are semantic categories, not physical memory-layout
    descriptions.
17. Type semantics form the foundation for coercion, equality, objects,
    functions, memory, serialization, security, and performance.

------------------------------------------------------------------------

# 66. Completion Criteria

### Understand

-   Identify all eight ECMAScript language types.
-   Distinguish primitive values from objects.

### Explain

-   Explain dynamic typing.
-   Explain object identity.
-   Explain mutation vs reassignment.
-   Explain why `typeof null` is `"object"`.

### Predict

-   Correctly predict `typeof` results.
-   Correctly predict object identity comparisons.
-   Correctly predict mutation vs parameter reassignment behavior.

### Implement

-   Implement a type/value classifier with an explicit contract.
-   Build tests for primitive/object/callable/array/null cases.

### Debug

-   Diagnose bugs caused by incorrect assumptions about shared object
    identity.
-   Diagnose bugs caused by confusing `undefined`, `null`, and omitted
    properties.

### Apply

-   Use explicit type and absence conventions in API/configuration
    design.

### Compare

-   Primitive vs object.
-   Mutation vs reassignment.
-   `null` vs `undefined`.
-   Number vs BigInt.
-   Primitive string vs `String` wrapper object.
-   Callable object vs ordinary object.

### Defend

-   Defend the statement that JavaScript is dynamically typed.
-   Defend the statement that JavaScript passes values.
-   Explain why language type and engine representation must not be
    conflated.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 03 --- Numbers, Floating Point,
and BigInt