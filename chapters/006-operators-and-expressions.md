
# Chapter 06 --- Operators and Expressions

> **Status:** `[+] Completed`\
> **Role in curriculum:** Establishes how JavaScript combines values
> into computations. Operators are not isolated symbols: they are part
> of the expression grammar and evaluation model, with defined
> precedence, associativity, conversions, short-circuit behavior, side
> effects, references, and result values.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain what an expression is.
-   Explain what an operator is.
-   Distinguish expressions from statements.
-   Explain unary, binary, and conditional operators.
-   Understand operator precedence and associativity.
-   Predict evaluation order correctly.
-   Understand that precedence determines grouping, while evaluation
    order determines when subexpressions are evaluated.
-   Explain arithmetic, comparison, logical, assignment, bitwise,
    relational, unary, and conditional operators.
-   Explain `typeof`, `instanceof`, `in`, `delete`, `void`, and `new`.
-   Explain short-circuiting for `&&`, `||`, and `??`.
-   Understand the difference between `||` and `??`.
-   Explain optional chaining `?.`.
-   Understand spread syntax and rest syntax and why they are related
    but not identical.
-   Explain exponentiation and its associativity.
-   Understand prefix vs postfix increment/decrement.
-   Understand logical assignment operators:
    -   `&&=`
    -   `||=`
    -   `??=`
-   Explain operator behavior involving objects and implicit coercion.
-   Understand why `+` is special because it can perform string
    concatenation.
-   Understand how property access, function calls, and assignment
    involve references.
-   Identify side effects and understand their relationship with
    evaluation order.
-   Predict complex expressions before execution.
-   Avoid relying on deeply nested or overly clever expressions in
    production code.

------------------------------------------------------------------------

# 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.

This chapter connects values and bindings to actual computation:

``` text
values
  ↓
expressions
  ↓
operators
  ↓
evaluation
  ↓
result
```

Chapter 07 will then explain the coercion/equality algorithms that many
operators use internally.

------------------------------------------------------------------------

# 3. What Is an Expression?

An expression is syntax that can be evaluated to produce a value or, in
some contexts, an abrupt completion such as an exception.

Examples:

``` js
42
```

``` js
2 + 3
```

``` js
user.name
```

``` js
getUser()
```

``` js
condition ? "yes" : "no"
```

Expressions can be nested.

For example:

``` js
(2 + 3) * 4
```

contains smaller expressions:

``` text
2
3
2 + 3
4
(2 + 3) * 4
```

This recursive structure is fundamental to JavaScript grammar.

------------------------------------------------------------------------

# 4. Expression vs Statement

A statement performs an action in program structure.

An expression computes a value.

Examples of expressions:

``` js
1 + 2
user.name
foo()
a = 10
condition ? a : b
```

Examples of statements:

``` js
if (condition) {
  ...
}
```

``` js
for (...) {
  ...
}
```

``` js
return value;
```

The distinction is useful, but JavaScript intentionally allows
expression statements:

``` js
foo();
```

The call is an expression used as a statement.

Similarly:

``` js
x = 10;
```

is an assignment expression used as a statement.

------------------------------------------------------------------------

# 5. What Is an Operator?

An operator is syntax that performs a defined operation on one or more
operands.

Example:

``` js
a + b
```

Here:

``` text
+  → operator
a  → left operand
b  → right operand
```

Operators can be:

``` text
unary
binary
conditional / ternary
```

Examples:

### Unary

``` js
!value
typeof value
-value
++value
```

### Binary

``` js
a + b
a * b
a === b
a && b
a ?? b
```

### Conditional

``` js
condition ? a : b
```

The number of syntactic operands is not the same thing as the complexity
of the underlying semantic operation.

------------------------------------------------------------------------

# 6. Operator Categories

JavaScript provides several broad operator groups:

``` text
Arithmetic
Assignment
Comparison
Equality
Relational
Logical
Bitwise
Unary
Property access
Call / construct
Conditional
Nullish
Optional chaining
```

Some syntax belongs to multiple conceptual categories.

For example:

``` js
x += 1
```

is assignment syntax with an embedded arithmetic operation.

------------------------------------------------------------------------

# 7. Arithmetic Operators

Common arithmetic operators:

``` js
+
-
*
/
%
**
```

Examples:

``` js
10 + 5; // 15
10 - 5; // 5
10 * 5; // 50
10 / 5; // 2
10 % 3; // 1
2 ** 3; // 8
```

But arithmetic operators do not always operate only on simple numeric
primitives.

They may invoke conversions.

The `+` operator is especially important because it can perform either:

``` text
numeric addition
```

or:

``` text
string concatenation
```

depending on operand conversion.

------------------------------------------------------------------------

# 8. Division and Special Numbers

JavaScript Number division follows Number semantics.

For example:

``` js
10 / 2;
```

produces:

``` text
5
```

But:

``` js
1 / 0;
```

produces:

``` text
Infinity
```

and:

``` js
0 / 0;
```

produces:

``` text
NaN
```

This connects directly to Chapter 03.

------------------------------------------------------------------------

# 9. Remainder `%`

The remainder operator:

``` js
a % b
```

does not mean mathematical modulo in every programming-language sense.

Its semantics are based on the Number/BigInt remainder operation.

For example:

``` js
5 % 2; // 1
```

For negative operands, the sign behavior follows JavaScript's remainder
semantics rather than a universally positive modulo definition.

This distinction matters in cyclic calculations.

------------------------------------------------------------------------

# 10. Exponentiation `**`

Example:

``` js
2 ** 3
```

produces:

``` text
8
```

Exponentiation is right-associative.

Therefore:

``` js
2 ** 3 ** 2
```

groups as:

``` js
2 ** (3 ** 2)
```

which yields:

``` text
512
```

not:

``` js
(2 ** 3) ** 2
```

which would be:

``` text
64
```

This is a classic precedence/associativity test.

------------------------------------------------------------------------

# 11. Unary Operators

Unary operators operate on one operand.

Examples:

``` js
+value
-value
!value
~value
typeof value
void value
delete target
++value
--value
```

Their semantics vary considerably.

For example:

``` js
+"42"
```

performs numeric conversion.

While:

``` js
-"42"
```

performs numeric conversion and negation.

And:

``` js
!value
```

performs Boolean-oriented conversion.

These are not merely syntactic variations.

------------------------------------------------------------------------

# 12. Unary Plus `+value`

Unary plus attempts numeric conversion.

Example:

``` js
+"42"
```

Result:

``` text
42
```

While:

``` js
+"hello"
```

produces:

``` text
NaN
```

Unary plus is therefore a concise numeric-conversion operation.

Be careful:

``` js
+1n
```

is invalid because BigInt does not support unary plus in the same way.

This is another example of type-specific operator rules.

------------------------------------------------------------------------

# 13. Unary Minus `-value`

Unary minus performs numeric conversion and negation.

Examples:

``` js
-10;      // -10
-"10";    // -10
```

It also interacts with signed zero:

``` js
-0
```

is negative zero.

This connects to Chapter 03's floating-point model.

------------------------------------------------------------------------

# 14. Logical NOT `!`

The logical NOT operator converts its operand to Boolean and reverses
the logical value.

Example:

``` js
!true;  // false
!false; // true
```

For arbitrary values, ToBoolean semantics are applied.

Examples:

``` js
!0;        // true
!"hello";  // false
!null;     // true
!undefined;// true
```

Double negation is often used to force Boolean conversion:

``` js
!!value
```

However, explicit Boolean conversion:

``` js
Boolean(value)
```

can sometimes be clearer.

------------------------------------------------------------------------

# 15. Bitwise NOT `~`

The bitwise NOT operator performs numeric conversion into the bitwise
integer model and then flips bits.

Example:

``` js
~0
```

produces:

``` text
-1
```

Bitwise operators use JavaScript's defined 32-bit integer conversion
behavior for Number operands.

Do not confuse:

``` text
Number
```

with:

``` text
arbitrary-width integer
```

BigInt bitwise behavior has separate semantics.

------------------------------------------------------------------------

# 16. Increment and Decrement

JavaScript provides:

``` js
++x
x++
--x
x--
```

There are two dimensions to understand:

``` text
prefix vs postfix
```

and:

``` text
value update
```

Example:

``` js
let x = 1;

const a = ++x;
```

After execution:

``` text
x = 2
a = 2
```

Postfix:

``` js
let x = 1;

const a = x++;
```

After execution:

``` text
x = 2
a = 1
```

The operator changes the binding and also produces a result value.

------------------------------------------------------------------------

# 17. Avoid Clever Increment Expressions

This:

``` js
let x = 1;
const y = x++ + ++x;
```

is technically analyzable, but it creates unnecessary cognitive load.

Production code should prefer explicit statements when side effects and
value production are difficult to reason about.

Correctness and maintainability usually matter more than saving one
line.

------------------------------------------------------------------------

# 18. Comparison Operators

Relational operators include:

``` js
<
>
<=
>=
```

Examples:

``` js
5 > 3;   // true
5 < 3;   // false
5 >= 5;  // true
5 <= 5;  // true
```

But comparing values of different types can involve conversion rules.

For example:

``` js
"10" < 20
```

does not simply compare two strings.

Understanding the exact result requires coercion semantics.

Chapter 07 will derive these rules.

------------------------------------------------------------------------

# 19. Equality Operators

JavaScript has:

``` js
==
!=
===
!==
```

The most important distinction is:

``` text
loose equality
vs
strict equality
```

Example:

``` js
1 == "1";   // true
1 === "1";  // false
```

The loose operator can perform coercion.

The strict operator does not perform the same cross-type coercion.

Equality algorithms are sufficiently important to receive their own
chapter.

------------------------------------------------------------------------

# 20. Logical AND `&&`

`&&` is both:

``` text
logical operator
```

and:

``` text
short-circuit evaluation operator
```

Example:

``` js
true && "hello"
```

produces:

``` text
"hello"
```

Notice:

``` text
"hello"
```

not:

``` text
true
```

The operator returns one of its operand values.

This is essential.

`&&` is not simply a Boolean-only operator.

------------------------------------------------------------------------

# 21. `&&` Short-Circuiting

Consider:

``` js
false && expensiveOperation();
```

The right-hand operand is not evaluated because the left side already
determines the result.

Conceptually:

``` text
evaluate left
   ↓
falsy?
   ↓ yes
return left
   ↓
do not evaluate right
```

This provides both:

``` text
logical selection
```

and:

``` text
conditional evaluation
```

------------------------------------------------------------------------

# 22. `||`

The logical OR operator returns one of its operands.

Example:

``` js
"hello" || "default"
```

returns:

``` text
"hello"
```

while:

``` js
"" || "default"
```

returns:

``` text
"default"
```

It uses truthiness, not nullishness.

This distinction is critical.

------------------------------------------------------------------------

# 23. Why `||` Can Be Dangerous for Defaults

Consider:

``` js
const retries = config.retries || 3;
```

If:

``` js
config.retries = 0;
```

the expression returns:

``` text
3
```

because `0` is falsy.

If zero is a valid value, this is a bug.

Use:

``` js
const retries = config.retries ?? 3;
```

when the intended fallback condition is specifically:

``` text
null or undefined
```

------------------------------------------------------------------------

# 24. Nullish Coalescing `??`

The nullish coalescing operator returns the right operand only when the
left operand is:

``` text
null
```

or:

``` text
undefined
```

Example:

``` js
null ?? "default";
```

returns:

``` text
"default"
```

But:

``` js
0 ?? "default";
```

returns:

``` text
0
```

and:

``` js
"" ?? "default";
```

returns:

``` text
""
```

Therefore:

``` text
|| → falsy fallback

?? → nullish fallback
```

These operators solve different problems.

------------------------------------------------------------------------

# 25. `&&`, `||`, and `??` Return Values

This is a major conceptual point.

Consider:

``` js
const result = a && b;
```

The result is not necessarily Boolean.

Similarly:

``` js
a || b
```

returns an operand value.

And:

``` js
a ?? b
```

returns an operand value.

Therefore they can be used as:

``` text
conditional value selectors
```

not merely truth-test operators.

------------------------------------------------------------------------

# 26. Short-Circuit Evaluation and Side Effects

Consider:

``` js
condition && doSomething();
```

The function call occurs only when the condition is truthy.

Similarly:

``` js
condition || doSomething();
```

calls `doSomething` only when the left side is falsy.

This can be concise, but side-effect-heavy expressions can become
difficult to review.

For important business logic, explicit `if` statements may be clearer.

------------------------------------------------------------------------

# 27. Optional Chaining `?.`

Optional chaining safely stops property/call evaluation when the base
value is `null` or `undefined`.

Example:

``` js
user?.profile?.name
```

If:

``` js
user === null
```

the chain produces:

``` text
undefined
```

instead of throwing a property-access error.

This is useful for optional data.

------------------------------------------------------------------------

# 28. Optional Chaining Is Nullish, Not Falsy

Consider:

``` js
const user = {
  name: ""
};

user?.name
```

returns:

``` text
""
```

because the object is not nullish.

Optional chaining does not mean:

``` text
if truthy
```

It means:

``` text
if null or undefined, stop this chain
```

This makes it conceptually closer to `??` than `||`.

------------------------------------------------------------------------

# 29. Optional Method Calls

Optional chaining can be used for method calls:

``` js
logger?.info?.("message");
```

This can avoid failure when either:

``` text
logger
```

or:

``` text
logger.info
```

is nullish.

But note that optional chaining is still subject to ordinary call
semantics when the target exists.

A non-callable existing property can still produce an error when called.

------------------------------------------------------------------------

# 30. Optional Element Access

You can also use:

``` js
arr?.[index]
```

or:

``` js
obj?.[key]
```

This provides nullish-safe element access.

Example:

``` js
const value = data?.items?.[0];
```

------------------------------------------------------------------------

# 31. Optional Chaining Is Not a Universal Error Suppression Mechanism

This:

``` js
user?.profile?.name
```

can prevent nullish property-access failures.

It does not mean:

``` text
all errors disappear
```

If evaluating a getter, function, proxy trap, or unrelated subexpression
throws, optional chaining does not magically suppress every exception.

Use it specifically for nullish-safe traversal.

------------------------------------------------------------------------

# 32. Conditional Operator `?:`

The conditional operator is JavaScript's ternary operator:

``` js
condition ? ifTrue : ifFalse
```

Example:

``` js
const label = isAdmin ? "Admin" : "User";
```

Only one branch expression is evaluated.

This makes it a conditional expression rather than a statement-level
`if`.

Nested ternaries are valid but can become difficult to read.

------------------------------------------------------------------------

# 33. Precedence

Operator precedence determines how an expression is grouped when
parentheses are absent.

Example:

``` js
2 + 3 * 4
```

is interpreted as:

``` js
2 + (3 * 4)
```

not:

``` js
(2 + 3) * 4
```

because multiplication has higher precedence than addition.

------------------------------------------------------------------------

# 34. Precedence Is Not Evaluation Order

This distinction is extremely important.

Consider:

``` js
a() + b() * c()
```

Precedence determines grouping:

``` js
a() + (b() * c())
```

But evaluation order concerns when the function calls occur.

JavaScript evaluates subexpressions according to its defined evaluation
order, which generally processes relevant operands from left to right
while following the grammar and operator semantics.

Do not use a precedence table as a substitute for understanding
evaluation order.

------------------------------------------------------------------------

# 35. Parentheses as Documentation

Use parentheses when they improve clarity.

Example:

``` js
const allowed = (isAdmin || isOwner) && isActive;
```

is easier to review than relying entirely on precedence.

Parentheses can serve two purposes:

``` text
change grouping
```

and:

``` text
communicate intent
```

Production code should optimize for human correctness, not parser
cleverness.

------------------------------------------------------------------------

# 36. Associativity

Associativity determines grouping when operators of the same precedence
appear repeatedly.

For example, subtraction is left-associative:

``` js
10 - 5 - 2
```

means:

``` js
(10 - 5) - 2
```

while exponentiation is right-associative:

``` js
2 ** 3 ** 2
```

means:

``` js
2 ** (3 ** 2)
```

Associativity is a grammar concept.

It is not identical to execution order.

------------------------------------------------------------------------

# 37. Assignment Operators

Basic assignment:

``` js
x = 10;
```

Compound assignment:

``` js
x += 10;
```

Logical assignment:

``` js
x ||= 10;
x &&= 10;
x ??= 10;
```

All update a target according to distinct rules.

Assignment is an expression and produces a value.

Example:

``` js
const y = (x = 10);
```

Afterward:

``` text
x = 10
y = 10
```

------------------------------------------------------------------------

# 38. Logical Assignment `||=`

Example:

``` js
config.timeout ||= 5000;
```

Conceptually:

``` text
if config.timeout is falsy
    assign 5000
```

Importantly, it is not simply textual:

``` js
config.timeout = config.timeout || 5000;
```

because real property assignment can involve:

-   getter evaluation,
-   setter behavior,
-   reference identity,
-   side-effect differences.

The language defines logical assignment with its own semantics.

------------------------------------------------------------------------

# 39. Nullish Assignment `??=`

Example:

``` js
config.timeout ??= 5000;
```

This assigns only if the current value is:

``` text
null
```

or:

``` text
undefined
```

Therefore:

``` js
let timeout = 0;
timeout ??= 5000;
```

leaves:

``` text
0
```

This is usually the appropriate defaulting operator when zero is
meaningful.

------------------------------------------------------------------------

# 40. Logical AND Assignment `&&=`

Example:

``` js
enabled &&= validate();
```

This evaluates the right-hand side only when the current left-hand value
is truthy.

Again, because property access can have side effects, the operator's
semantics are more nuanced than naive textual rewriting.

------------------------------------------------------------------------

# 41. Bitwise Operators

JavaScript provides:

``` js
&
|
^
~
<<
>>
>>>
```

For Number operands, they use the specified 32-bit integer conversion
behavior.

Examples:

``` js
5 & 1;
5 | 1;
5 ^ 1;
1 << 3;
```

These can be useful for:

-   bit masks,
-   compact flags,
-   low-level protocols,
-   certain algorithms.

But they are not general-purpose integer arithmetic.

------------------------------------------------------------------------

# 42. BigInt Bitwise Operators

BigInt supports appropriate bitwise operators with BigInt operands.

For example:

``` js
5n & 1n
```

works.

But mixing:

``` js
5 & 1n
```

is invalid.

Again, numeric type boundaries matter.

------------------------------------------------------------------------

# 43. `typeof`

The `typeof` operator returns a string classification.

Examples:

``` js
typeof 42;          // "number"
typeof "x";         // "string"
typeof undefined;   // "undefined"
typeof null;        // "object"
typeof function(){};// "function"
```

It is especially useful when checking whether a binding currently has a
value of an expected broad category.

However, it is not a complete type system and cannot distinguish:

``` text
array
date
map
set
plain object
```

all by itself.

------------------------------------------------------------------------

# 44. `instanceof`

The `instanceof` operator checks an object against a constructor's
prototype-based relationship.

Example:

``` js
const user = new User();

user instanceof User;
```

returns:

``` text
true
```

But `instanceof` has important limitations:

-   it depends on prototype relationships,
-   cross-realm objects can behave unexpectedly,
-   custom `Symbol.hasInstance` can change behavior,
-   it is not a universal "is this class exactly?" test.

Chapter 20 will explain `Symbol.hasInstance`.

------------------------------------------------------------------------

# 45. `in`

The `in` operator checks whether a property key exists in an object or
its prototype chain.

Example:

``` js
const user = {
  name: "A"
};

"name" in user;
```

returns:

``` text
true
```

But:

``` js
"toString" in user;
```

can also be true because inherited properties count.

If you specifically want an own property, use an appropriate
own-property check such as:

``` js
Object.hasOwn(user, "name");
```

This distinction becomes crucial in object/prototype security.

------------------------------------------------------------------------

# 46. `delete`

The `delete` operator removes a property when the relevant
object-property semantics permit it.

Example:

``` js
const user = {
  name: "A",
  age: 20
};

delete user.age;
```

Now the property may no longer exist:

``` js
Object.hasOwn(user, "age");
```

returns:

``` text
false
```

`delete` does not mean:

``` text
delete variable from memory
```

It operates on property/reference semantics.

------------------------------------------------------------------------

# 47. `delete` and Arrays

Consider:

``` js
const arr = [10, 20, 30];

delete arr[1];
```

The result is not necessarily:

``` text
[10, 30]
```

Instead, the array can become sparse:

``` text
[10, <empty>, 30]
```

The length remains:

``` text
3
```

This is a major example of why deletion and removal are not identical.

Use array-specific methods such as `splice()` when you mean to remove an
element and shift later indexes.

------------------------------------------------------------------------

# 48. `void`

The `void` operator evaluates an expression and produces:

``` text
undefined
```

Example:

``` js
void 123;
```

returns:

``` text
undefined
```

It is uncommon in ordinary application code.

Historically, it has been used in patterns where:

``` text
evaluate expression
but intentionally discard its result
```

Production code should prefer clearer constructs unless `void` is
solving a specific problem.

------------------------------------------------------------------------

# 49. `new`

The `new` operator performs construction.

Example:

``` js
const user = new User("A");
```

Conceptually, constructing an object can involve:

``` text
constructor selection
prototype setup
new object creation
constructor invocation
return-value rules
```

The exact semantics are important enough that later chapters cover
`[[Construct]]`, constructors, classes, and prototypes.

Do not think:

> "`new` just calls a function."

It does substantially more.

------------------------------------------------------------------------

# 50. `new` With Non-Constructable Functions

Not every callable value is constructable.

For example:

``` js
const fn = () => {};

new fn();
```

throws a `TypeError`.

This reinforces the distinction:

``` text
callable
≠
constructable
```

------------------------------------------------------------------------

# 51. Property Access

Property access can use:

``` js
obj.property
```

or:

``` js
obj["property"]
```

and:

``` js
obj?.property
obj?.["property"]
```

Dot notation requires identifier-like property syntax.

Bracket notation evaluates an expression to obtain a property key.

Example:

``` js
const key = "name";

user[key];
```

The value of `key` determines the actual property access.

------------------------------------------------------------------------

# 52. Function Call Operator

Calling:

``` js
fn(arg);
```

is an expression.

It evaluates:

1.  the function/reference,
2.  argument expressions,
3.  the call according to function/call semantics.

The result can be any JavaScript value or an abrupt completion.

This becomes especially important when evaluating:

``` js
obj.method();
```

because the reference can preserve the base object used for `this`
binding.

Chapter 14 will explain that distinction deeply.

------------------------------------------------------------------------

# 53. Spread Syntax

Spread expands an iterable or, in object spread contexts, an object-like
source into another structure.

Array example:

``` js
const a = [1, 2];
const b = [...a, 3];
```

Result:

``` text
[1, 2, 3]
```

Function call:

``` js
fn(...args);
```

Object spread:

``` js
const copy = { ...user };
```

Spread is syntax, not one universal runtime operation.

Its exact behavior depends on the context.

------------------------------------------------------------------------

# 54. Spread Is Not Deep Copy

This is a common mistake:

``` js
const copy = { ...user };
```

This performs a shallow property copy.

Nested object references can remain shared.

Example:

``` js
const user = {
  profile: {
    name: "A"
  }
};

const copy = { ...user };

copy.profile.name = "B";

console.log(user.profile.name);
```

The nested object can also show:

``` text
B
```

because both objects can refer to the same nested object.

------------------------------------------------------------------------

# 55. Rest Syntax

Rest collects remaining values into an array/object structure depending
on context.

Function example:

``` js
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
```

Here:

``` text
...numbers
```

collects remaining arguments into an Array.

Rest and spread use the same `...` token but solve opposite directions:

``` text
spread
  → expand

rest
  → collect
```

They are related syntax forms, not one operator with one behavior.

------------------------------------------------------------------------

# 56. Rest vs Spread

Compare:

``` js
const values = [1, 2, 3];

function log(a, b, c) {}
log(...values);
```

Spread expands:

``` text
[1, 2, 3]
```

into:

``` text
a, b, c
```

Now:

``` js
function log(...values) {}
```

rest collects:

``` text
a, b, c
```

into:

``` text
[ a, b, c ]
```

The conceptual direction is:

``` text
spread:  collection → individual positions
rest:    positions → collection
```

------------------------------------------------------------------------

# 57. Assignment Destructuring

Destructuring assignment is an expression-level binding/value extraction
mechanism.

Example:

``` js
const user = {
  name: "A",
  age: 30
};

const { name, age } = user;
```

For arrays:

``` js
const [first, second] = [10, 20];
```

Destructuring uses pattern syntax rather than simple left/right
assignment syntax.

The details matter when handling:

-   defaults,
-   rest elements,
-   nested structures,
-   iterables,
-   property access,
-   errors.

------------------------------------------------------------------------

# 58. Evaluation Order

One of the most important concepts in operator reasoning is:

> JavaScript does not evaluate an expression by "following precedence
> from highest to lowest."

Instead:

``` text
grammar
+
operator semantics
+
left-to-right evaluation rules
+
short-circuit rules
```

determine what happens.

Example:

``` js
const result = f() + g() * h();
```

Grouping:

``` js
f() + (g() * h())
```

Evaluation of the relevant operand expressions proceeds according to the
specified evaluation rules.

You must reason about both grouping and timing.

------------------------------------------------------------------------

# 59. Side Effects Reveal Evaluation Order

Consider:

``` js
let x = 0;

function a() {
  x += 1;
  return x;
}

function b() {
  x += 10;
  return x;
}

const result = a() + b();
```

To predict the result you must consider:

``` text
which function executes first?
what side effect happens?
what values are returned?
```

This is why evaluation order matters much more than operator precedence
alone.

------------------------------------------------------------------------

# 60. Short-Circuiting Changes What Executes

Example:

``` js
false && a();
```

`a()` is not evaluated.

Similarly:

``` js
true || b();
```

does not evaluate `b()`.

And:

``` js
null ?? c();
```

does evaluate `c()`.

But:

``` js
0 ?? c();
```

does not evaluate `c()` because zero is not nullish.

Short-circuiting is both a value-selection rule and a control-flow
mechanism inside expressions.

------------------------------------------------------------------------

# 61. Mixed `??` With `||` and `&&`

JavaScript deliberately restricts certain combinations without explicit
grouping.

For example:

``` js
a ?? b || c
```

is a syntax error unless parentheses clarify the intended grouping.

Use:

``` js
(a ?? b) || c
```

or:

``` js
a ?? (b || c)
```

depending on the intended semantics.

This design prevents ambiguous or error-prone mixing of nullish and
logical defaulting logic.

------------------------------------------------------------------------

# 62. Operator Precedence --- Practical Core

You do not need to memorize every precedence level to write good code.

The most practically important groups include:

``` text
member access / calls
unary operators
exponentiation
multiplication / division / remainder
addition / subtraction
relational
equality
logical AND
logical OR
nullish coalescing
conditional
assignment
comma
```

When an expression becomes nontrivial, use parentheses.

The goal is not to test memory of a precedence table.

The goal is to produce code whose meaning is obvious.

------------------------------------------------------------------------

# 63. Comma Operator

The comma operator evaluates expressions from left to right and returns
the final expression's value.

Example:

``` js
const x = (1, 2, 3);
```

Result:

``` text
3
```

The earlier expressions are still evaluated.

The comma operator is relatively uncommon in normal production
application code.

Do not confuse it with commas used as punctuation in declarations,
arrays, arguments, or object literals.

------------------------------------------------------------------------

# 64. Ternary vs `if`

Ternary:

``` js
const label = active ? "Active" : "Inactive";
```

is an expression.

`if`:

``` js
let label;

if (active) {
  label = "Active";
} else {
  label = "Inactive";
}
```

is a statement structure.

Use the ternary when the result can be expressed clearly.

Use `if` when the logic has multiple conditions, side effects, or
significant branching.

------------------------------------------------------------------------

# 65. Object Coercion and Operators

Operators can interact with objects through conversion.

Example:

``` js
const obj = {
  valueOf() {
    return 10;
  }
};

obj + 5;
```

The operator may request a primitive representation of the object.

This is why Chapter 07 will cover:

``` text
ToPrimitive
ToNumber
ToString
```

in detail.

Never assume operators only operate on already-primitive values.

------------------------------------------------------------------------

# 66. Why `+` Is Special

Consider:

``` js
1 + 2;
```

Numeric addition.

Now:

``` js
"1" + 2;
```

String concatenation.

And:

``` js
1 + "2";
```

also produces a String result.

The `+` operator can therefore involve:

``` text
ToPrimitive
+
string preference in relevant cases
+
numeric addition otherwise
```

This is one of JavaScript's most important coercion behaviors.

------------------------------------------------------------------------

# 67. `-`, `*`, `/`, and `%` Are More Numeric-Oriented

Compare:

``` js
"10" - 2
```

This performs numeric conversion and produces:

``` text
8
```

while:

``` js
"10" + 2
```

can produce:

``` text
"102"
```

This asymmetry is a major source of beginner confusion.

It is not arbitrary.

It follows from the distinct semantic algorithms for the operators.

------------------------------------------------------------------------

# 68. Assignment and Reference Targets

Consider:

``` js
x = 10;
```

The left side is not first evaluated as an ordinary value.

It is evaluated as a location/reference-like target that can receive the
assignment.

Similarly:

``` js
obj.name = "A";
```

must identify:

``` text
object
+
property key
+
assignment target
```

This is why assignment semantics are closely connected to references,
`GetValue`, and `PutValue`.

Chapter 42 will formalize this.

------------------------------------------------------------------------

# 69. Property Assignment Can Trigger Getters/Setters

Consider:

``` js
const obj = {
  set value(v) {
    console.log("setting", v);
  }
};

obj.value = 10;
```

The assignment does not simply write raw storage.

The object's property semantics can invoke a setter.

Therefore:

``` text
assignment operator
+
property semantics
```

can have observable side effects.

This is another reason naive textual reasoning can fail.

------------------------------------------------------------------------

# 70. Optional Chaining and Side Effects

Consider:

``` js
obj?.method();
```

If `obj` is nullish, the access/call path short-circuits.

But if `obj` exists, property access and call semantics continue
normally.

Optional chaining therefore changes control flow around a specific
nullish boundary.

It does not turn the entire expression into a non-throwing expression.

------------------------------------------------------------------------

# 71. `delete` Is Not Garbage Collection

This misconception is common.

``` js
delete obj.value;
```

removes a property when permitted.

It does not directly command:

``` text
garbage collector → free this memory now
```

Garbage collection is a separate reachability-based mechanism.

Removing a property can make an object graph less connected, but it does
not mean immediate memory release.

Chapter 45 covers this distinction.

------------------------------------------------------------------------

# 72. `typeof` Does Not Prove Safe Property Access

Example:

``` js
if (typeof value === "object") {
  console.log(value.name);
}
```

This still includes:

``` js
value === null
```

because:

``` js
typeof null === "object"
```

A robust condition may require:

``` js
value !== null && typeof value === "object"
```

depending on the contract.

This is a direct example of why operator output must not be interpreted
without understanding semantics.

------------------------------------------------------------------------

# 73. `in` vs `hasOwn`

Compare:

``` js
"name" in user
```

with:

``` js
Object.hasOwn(user, "name")
```

The first includes inherited properties.

The second checks own properties.

These are semantically different.

This distinction becomes especially important with:

-   prototype pollution,
-   dictionaries,
-   untrusted keys,
-   serialization,
-   object-shape validation.

------------------------------------------------------------------------

# 74. Performance Considerations

Operators themselves are rarely the dominant performance problem.

The real cost can come from:

-   conversions,
-   allocations,
-   function calls,
-   property lookups,
-   object proxies,
-   string creation,
-   repeated coercion,
-   algorithmic complexity.

For example:

``` js
value + ""
```

may require conversion.

A `Proxy` can add significant semantic machinery around property access.

A regex or string operation may dominate arithmetic.

Therefore:

``` text
operator syntax
```

is not a reliable performance metric by itself.

------------------------------------------------------------------------

# 75. Memory Considerations

Expressions can create temporary values.

For example:

``` js
const result = a + b + c + d;
```

may involve intermediate results conceptually.

An engine can optimize representation and allocation, but programmers
should still avoid needless data transformations in hot paths when
measurement shows they matter.

More importantly, references created by expressions can keep objects
reachable.

For example:

``` js
const callback = () => largeObject.value;
```

can retain access to `largeObject` through closure relationships.

That is a later memory topic.

------------------------------------------------------------------------

# 76. Security Considerations

Operators become security-relevant when they are used for:

-   authorization decisions,
-   validation,
-   property access,
-   dynamic keys,
-   prototype checks,
-   defaulting,
-   URL building,
-   security policy evaluation.

Examples:

``` js
if (user.permissions?.admin) {
  ...
}
```

and:

``` js
if ("isAdmin" in user) {
  ...
}
```

do not mean the same thing.

Likewise:

``` js
config.timeout || defaultTimeout
```

can accidentally overwrite a valid zero.

Security-sensitive code must define semantics explicitly.

------------------------------------------------------------------------

# 77. Production Engineering --- Avoid Clever Expressions

This is valid:

``` js
const x = a && b ? c : d ?? e;
```

But it is difficult to audit.

Prefer:

``` js
let x;

if (a && b) {
  x = c;
} else {
  x = d ?? e;
}
```

when the logic carries business meaning or security implications.

Readable code is easier to:

``` text
review
test
debug
secure
maintain
```

------------------------------------------------------------------------

# 78. Production Engineering --- Defaulting Rules

Use:

``` js
|| 
```

when:

``` text
any falsy value should trigger fallback
```

Use:

``` js
??
```

when:

``` text
only null/undefined should trigger fallback
```

Example:

``` js
const label = input.label ?? "Untitled";
```

preserves:

``` text
""
0
false
```

when those are meaningful values.

This is a small syntax choice with large practical consequences.

------------------------------------------------------------------------

# 79. Production Engineering --- Parentheses as Contracts

Consider:

``` js
const allowed =
  isAuthenticated &&
  (isAdmin || isOwner) &&
  !isSuspended;
```

The parentheses communicate the business rule:

``` text
authenticated
AND
(admin OR owner)
AND
not suspended
```

This is preferable to forcing reviewers to reconstruct precedence
mentally.

------------------------------------------------------------------------

# 80. Production Engineering --- Avoid Side Effects in Conditions

Instead of:

``` js
if (items.shift() && validate()) {
  ...
}
```

prefer:

``` js
const item = items.shift();

if (item && validate()) {
  ...
}
```

The second version exposes state changes more clearly.

Side-effect-free conditions are generally easier to reason about and
test.

------------------------------------------------------------------------

# 81. Common Misconceptions

## Misconception 1 --- Precedence determines evaluation order.

False.

Precedence determines grouping.

Evaluation order is governed by the language's evaluation semantics.

------------------------------------------------------------------------

## Misconception 2 --- `&&` and `||` always return booleans.

False.

They return operand values.

------------------------------------------------------------------------

## Misconception 3 --- `??` is just another spelling of `||`.

False.

`||` uses truthiness.

`??` checks only nullish values.

------------------------------------------------------------------------

## Misconception 4 --- Optional chaining catches errors.

False.

It short-circuits on nullish bases; it does not suppress arbitrary
exceptions.

------------------------------------------------------------------------

## Misconception 5 --- `delete` frees memory.

False.

It removes a property when permitted. Garbage collection is separate.

------------------------------------------------------------------------

## Misconception 6 --- Spread creates a deep copy.

False.

Object and array spread are generally shallow copying/expansion
mechanisms.

------------------------------------------------------------------------

## Misconception 7 --- Rest and spread are the same operation.

False.

Spread expands; rest collects.

------------------------------------------------------------------------

## Misconception 8 --- `instanceof` means "this object was created by exactly this class."

Not always.

Prototype relationships and custom `Symbol.hasInstance` semantics
matter.

------------------------------------------------------------------------

## Misconception 9 --- `in` means own property.

False.

It includes inherited properties.

------------------------------------------------------------------------

## Misconception 10 --- All arithmetic operators behave like `+`.

False.

`+` has special string-concatenation semantics.

------------------------------------------------------------------------

# 82. Common Mistakes

### Mistake: `||` for numeric defaults

``` js
const limit = userLimit || 10;
```

can turn a valid `0` into `10`.

### Mistake: relying on precedence in security checks

Use parentheses.

### Mistake: nested ternary expressions for complex business rules

Use explicit branching.

### Mistake: deleting array elements with `delete`

It can create sparse arrays.

### Mistake: assuming `typeof value === "object"` excludes null

It does not.

### Mistake: confusing shallow spread with deep copying

Nested references can remain shared.

### Mistake: assuming optional chaining means "ignore all errors"

It does not.

### Mistake: mixing Number and BigInt arithmetic

Explicitly choose the numeric domain.

------------------------------------------------------------------------

# 83. Comparison Table --- Logical Operators

  -----------------------------------------------------------------------
  Operator          Short-circuits?   Main test         Result
  ----------------- ----------------- ----------------- -----------------
  `a && b`          Yes               `a` truthiness    `a` if falsy,
                                                        otherwise `b`

  `a || b`          Yes               `a` truthiness    `a` if truthy,
                                                        otherwise `b`

  `a ?? b`          Yes               `a` nullishness   `a` unless
                                                        null/undefined,
                                                        otherwise `b`
  -----------------------------------------------------------------------

This table is more useful than memorizing vague phrases like "AND means
true."

------------------------------------------------------------------------

# 84. Comparison Table --- Equality and Relational

  Operator group         Main idea
  ---------------------- --------------------------------------
  `===`, `!==`           strict equality / inequality
  `==`, `!=`             equality with coercion rules
  `<`, `>`, `<=`, `>=`   relational comparison
  `Object.is()`          separate identity/equality algorithm

Detailed equality semantics are reserved for Chapter 07.

------------------------------------------------------------------------

# 85. Comparison --- Spread, Rest, Destructuring

  Syntax                         Direction   Typical purpose
  ------------------------------ ----------- -------------------------------------
  `[...items]`                   expand      copy/compose iterable contents
  `fn(...args)`                  expand      pass iterable elements as arguments
  `{...obj}`                     expand      shallow object property copy
  `function fn(...args)`         collect     collect remaining arguments
  `const [a, ...rest] = items`   collect     split sequence
  `const {a, ...rest} = obj`     collect     split object properties

------------------------------------------------------------------------

# 86. Execution Walkthrough --- `||` Default

Consider:

``` js
const timeout = config.timeout || 5000;
```

Suppose:

``` js
config.timeout = 0;
```

### Step 1

Read:

``` js
config.timeout
```

Result:

``` text
0
```

### Step 2

Apply truthiness.

`0` is falsy.

### Step 3

Because the left side is falsy, evaluate the right side:

``` js
5000
```

### Step 4

Return:

``` text
5000
```

This may violate the application's intended contract if `0` is valid.

------------------------------------------------------------------------

# 87. Execution Walkthrough --- `??`

Now:

``` js
const timeout = config.timeout ?? 5000;
```

with:

``` js
config.timeout = 0;
```

### Step 1

Read the left value:

``` text
0
```

### Step 2

Check whether it is:

``` text
null
```

or:

``` text
undefined
```

It is neither.

### Step 3

Return:

``` text
0
```

The fallback is not evaluated.

This is the key semantic difference.

------------------------------------------------------------------------

# 88. Execution Walkthrough --- `&&`

``` js
const result = user && user.profile;
```

If:

``` js
user = null;
```

then:

### Step 1

Evaluate `user`.

### Step 2

`null` is falsy.

### Step 3

Return `null`.

### Step 4

Do not evaluate:

``` js
user.profile
```

No property access is attempted.

This is why short-circuiting can prevent errors.

------------------------------------------------------------------------

# 89. Execution Walkthrough --- Optional Chaining

``` js
const result = user?.profile?.name;
```

If:

``` js
user = null;
```

then the chain short-circuits and produces:

``` text
undefined
```

The result differs from:

``` js
user && user.profile && user.profile.name
```

which can preserve a falsy intermediate value.

Optional chaining explicitly targets nullish traversal.

------------------------------------------------------------------------

# 90. Execution Walkthrough --- Spread

``` js
const a = [1, 2];
const b = [...a, 3];
```

Conceptually:

``` text
evaluate a
   ↓
obtain iterable
   ↓
iterate values
   ↓
insert 1
insert 2
then insert 3
```

The result is a new Array.

It is not a deep clone of all recursively referenced objects.

------------------------------------------------------------------------

# 91. Execution Walkthrough --- Rest

``` js
function log(first, ...others) {
  return others;
}
```

Call:

``` js
log(10, 20, 30);
```

Conceptually:

``` text
first → 10
others → [20, 30]
```

The remaining arguments are collected into a new Array-like data
structure according to the function parameter semantics.

------------------------------------------------------------------------

# 92. Implementation From Scratch --- Mini Expression Evaluator

Build a tiny evaluator for a deliberately limited expression language
supporting:

``` text
numbers
+
-
*
/
parentheses
```

Example:

``` text
2 + 3 * 4
```

Expected:

``` text
14
```

The purpose is not to reproduce JavaScript.

The purpose is to understand:

``` text
tokenization
precedence
associativity
parsing
evaluation
```

This is an excellent preparation for understanding how JavaScript
engines process source code.

------------------------------------------------------------------------

# 93. Implementation Progression

### Guided

Implement:

``` text
number literals
+
-
*
/
()
```

### Partially Guided

Add:

``` text
unary minus
operator precedence
left associativity
```

### No Reference

Implement a parser/evaluator from scratch.

### Edge-Case Hardened

Test:

``` text
1 - 2 - 3
2 ** 3 ** 2
-(2 + 3)
2 * (3 + 4)
```

### Production-Grade

Add:

-   lexical validation,
-   syntax errors,
-   diagnostics,
-   tests,
-   performance measurements,
-   limits against pathological expressions.

------------------------------------------------------------------------

# 94. Implementation Exercise --- Defaulting Utility

Implement:

``` js
function defaultNullish(value, fallback) {
  // ...
}
```

Then compare it against:

``` js
value || fallback
```

Build tests for:

``` text
undefined
null
0
false
""
NaN
"hello"
```

The goal is to understand the semantic difference, not to recreate the
built-in operator.

------------------------------------------------------------------------

# 95. Debugging Exercises

## Exercise 1

Explain:

``` js
0 || 10
```

and:

``` js
0 ?? 10
```

## Exercise 2

Explain:

``` js
false && expensive();
```

## Exercise 3

Explain:

``` js
null ?? expensive();
```

## Exercise 4

Why can this be dangerous?

``` js
delete arr[1];
```

## Exercise 5

Why can this be misleading?

``` js
typeof value === "object"
```

## Exercise 6

Why is:

``` js
"a" in obj
```

different from:

``` js
Object.hasOwn(obj, "a")
```

## Exercise 7

Why can:

``` js
const copy = { ...original };
```

still share nested mutable objects?

------------------------------------------------------------------------

# 96. Code Review Exercise

Review:

``` js
const port = config.port || 3000;

if (user && user.permissions && user.permissions.admin) {
  startServer();
}
```

Potential improvements:

``` js
const port = config.port ?? 3000;

if (user?.permissions?.admin) {
  startServer();
}
```

But do not stop at syntax.

A strong review also asks:

-   Is `0` a valid port value in this application's contract?
-   Should malformed configuration be rejected instead of defaulted?
-   Is `admin` truthiness sufficient for authorization?
-   Is `user.permissions` trusted application data?
-   Is the expression hiding a security policy that deserves explicit
    code?

Concise syntax is not automatically better architecture.

------------------------------------------------------------------------

# 97. Interview Questions

## Junior

1.  What is an expression?
2.  What is the difference between an expression and a statement?
3.  What is an operator?
4.  What does `&&` return?
5.  What does `||` return?
6.  What is the difference between `||` and `??`?

## Mid-Level

7.  Explain short-circuit evaluation.
8.  Explain precedence vs associativity.
9.  Explain precedence vs evaluation order.
10. What does optional chaining do?
11. What does spread syntax do?
12. What does rest syntax do?
13. What does `delete` do to an array element?
14. What is the difference between `in` and own-property checks?

## Senior

15. Why is `+` special compared with other arithmetic operators?
16. Explain how object operands can participate in arithmetic.
17. Explain why logical operators are value-producing operators.
18. Explain why logical assignment cannot always be treated as simple
    textual rewriting.
19. Explain callable vs constructable values in relation to `new`.
20. Explain reference-like behavior in assignment and property access.

## Staff / Principal

21. Review a production authorization expression and explain when
    operator precedence becomes a security risk.
22. Design coding standards for default values using `||` vs `??`.
23. When should optional chaining be discouraged in favor of explicit
    validation?
24. How would you reduce expression complexity in a safety-critical
    codebase?
25. How would you teach engineers to separate grouping, evaluation
    order, coercion, and side effects when reviewing an expression?
26. How can operator choices affect portability, performance, and
    maintainability?

------------------------------------------------------------------------

# 98. Predict-the-Output Exercises

Predict before executing.

### 1

``` js
console.log(2 + 3 * 4);
```

### 2

``` js
console.log((2 + 3) * 4);
```

### 3

``` js
console.log(2 ** 3 ** 2);
```

### 4

``` js
console.log("10" + 5);
```

### 5

``` js
console.log("10" - 5);
```

### 6

``` js
console.log(false && "hello");
```

### 7

``` js
console.log(true && "hello");
```

### 8

``` js
console.log("" || "default");
```

### 9

``` js
console.log(0 || "default");
```

### 10

``` js
console.log(0 ?? "default");
```

### 11

``` js
console.log(null ?? "default");
```

### 12

``` js
console.log(undefined ?? "default");
```

### 13

``` js
const user = null;
console.log(user?.name);
```

### 14

``` js
const user = {
  profile: {
    name: "A"
  }
};

console.log(user?.profile?.name);
```

### 15

``` js
const arr = [10, 20, 30];
delete arr[1];

console.log(arr.length);
console.log(1 in arr);
```

### 16

``` js
const obj = {
  x: 1
};

console.log("x" in obj);
console.log(Object.hasOwn(obj, "x"));
```

### 17

``` js
let x = 1;
const a = x++;
console.log(x, a);
```

### 18

``` js
let x = 1;
const a = ++x;
console.log(x, a);
```

### 19

``` js
const a = [1, 2];
const b = [...a, 3];

console.log(a);
console.log(b);
```

### 20

``` js
function test(first, ...rest) {
  return [first, rest];
}

console.log(test(1, 2, 3));
```

### 21

``` js
console.log(typeof null);
```

### 22

``` js
console.log(1n + 2n);
```

### 23

``` js
console.log(1n + 2);
```

Determine whether the expression produces a value or throws.

### 24

``` js
const value = null;
console.log(value && value.name);
```

### 25

``` js
const value = 0;
console.log(value && value.name);
```

### 26

``` js
console.log((false || true) && false);
```

------------------------------------------------------------------------

# 99. Mastery Exercises

## Exercise A --- Four-Layer Expression Analysis

For any complex expression, identify:

``` text
1. Syntax / grammar
2. Grouping / precedence
3. Evaluation order
4. Value conversion
5. Side effects
6. Final result
```

Apply this to:

``` js
a() + b() * c() ?? d
```

Then explain why explicit parentheses may be required.

## Exercise B --- Defaulting Audit

Audit a codebase for:

``` js
|| default
```

Classify each use:

``` text
correct
potential bug
requires domain review
```

## Exercise C --- Security Expression Audit

Review authorization conditions containing:

``` text
&&
||
??
?.
in
instanceof
```

For each one, document:

``` text
intended boolean logic
operand types
short-circuit behavior
inherited-property behavior
nullish behavior
security assumptions
```

## Exercise D --- Build an Expression Trace

Write a utility or manual trace format:

``` text
Expression
↓
Grouping
↓
Operand evaluation
↓
Conversions
↓
Operator application
↓
Result
```

Use it for at least ten nontrivial expressions.

## Exercise E --- Parser Design

Build the mini arithmetic parser described above and explain exactly how
precedence and associativity are represented in the grammar or parser.

------------------------------------------------------------------------

# 100. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.

## Builds Toward

-   Chapter 07 --- Type Conversion, Coercion, and Equality.
-   Chapter 08 --- Control Flow and Iteration.
-   Chapter 09 --- Functions and First-Class Behavior.
-   Chapter 14 --- `this`, Invocation, and Binding.
-   Chapter 15 --- Objects and Property Semantics.
-   Chapter 16 --- Property Keys and Enumeration.
-   Chapter 17 --- Prototypes.
-   Chapter 19 --- Proxy and Metaprogramming.
-   Chapter 20 --- Symbols and Well-Known Symbols.
-   Chapter 28 --- Serialization.
-   Chapter 42 --- Abstract Operations.
-   Chapter 43 --- Ordinary Object Internal Methods.
-   Chapter 57 --- Security Engineering.

## Related Concepts

``` text
precedence
associativity
evaluation order
short-circuiting
coercion
references
property access
call semantics
assignment
side effects
spread
rest
optional chaining
nullish coalescing
```

## Concepts Revisited Later

-   `ToPrimitive`
-   `ToBoolean`
-   `ToNumber`
-   `ToString`
-   `GetValue`
-   `PutValue`
-   `Call`
-   `Construct`
-   property lookup
-   prototype traversal
-   `Symbol.hasInstance`
-   iterable protocols

## Why This Chapter Matters Later

Operators are where many foundational concepts become observable.

A developer may know:

``` text
scope
types
objects
functions
```

and still misunderstand code because they do not correctly model:

``` text
evaluation order
coercion
short-circuiting
references
side effects
```

Mastering expressions therefore turns isolated language facts into
executable reasoning.

------------------------------------------------------------------------

# 101. Key Takeaways

1.  Expressions are evaluable pieces of JavaScript syntax that produce
    values or abrupt completions.
2.  Operators act on operands according to language-defined semantics.
3.  Precedence determines grouping.
4.  Associativity determines grouping among operators of equal
    precedence.
5.  Neither precedence nor associativity should be confused with
    evaluation order.
6.  JavaScript evaluates expressions according to explicit semantic
    rules, including left-to-right evaluation in the relevant operand
    positions and short-circuiting where defined.
7.  `+` is special because it can perform numeric addition or string
    concatenation.
8.  `&&`, `||`, and `??` are short-circuiting value-selection operators.
9.  `||` checks truthiness; `??` checks only `null`/`undefined`.
10. Optional chaining `?.` short-circuits on nullish bases.
11. Optional chaining does not suppress arbitrary exceptions.
12. Assignment is an expression and produces a value.
13. Property assignment can invoke getters/setters and therefore can
    have side effects.
14. `typeof` reports a runtime category and has historical edge cases
    such as `typeof null === "object"`.
15. `instanceof` depends on prototype relationships and can be
    customized.
16. `in` includes inherited properties; own-property checks are
    different.
17. `delete` removes properties and does not directly perform garbage
    collection.
18. Deleting an array element can create a sparse array rather than
    shifting elements.
19. Spread expands values; rest collects values.
20. Object spread is shallow, not deep cloning.
21. BigInt arithmetic has separate rules and cannot be freely mixed with
    Number arithmetic.
22. Parentheses are useful both for semantics and communication.
23. Complex expressions with hidden side effects are usually worse
    production code than explicit logic.
24. Correct operator reasoning requires analyzing syntax, grouping,
    evaluation, conversion, side effects, and final value together.

------------------------------------------------------------------------

# 102. Completion Criteria

### Understand

-   Define expressions and operators.
-   Explain major operator categories.
-   Explain precedence and associativity.
-   Explain short-circuiting.

### Explain

-   Explain `&&`, `||`, and `??`.
-   Explain optional chaining.
-   Explain spread vs rest.
-   Explain `delete`, `in`, `instanceof`, `typeof`, and `new`.
-   Explain why `+` differs from other arithmetic operators.

### Predict

-   Correctly predict complex expression grouping.
-   Correctly predict evaluation order.
-   Correctly predict short-circuit paths.
-   Correctly predict implicit conversion at a high level.

### Implement

-   Build the mini expression evaluator.
-   Build expression-tracing utilities/checklists.
-   Build defaulting and operator test suites.

### Debug

-   Diagnose incorrect defaults.
-   Diagnose short-circuit bugs.
-   Diagnose mutation caused by spread-shared nested values.
-   Diagnose inherited-property checks.
-   Diagnose sparse-array behavior after `delete`.

### Apply

-   Use `??` when nullishness---not falsiness---is the intended fallback
    condition.
-   Use optional chaining for deliberate nullish traversal.
-   Use parentheses to communicate complex logic.
-   Prefer explicit branching when expression complexity harms
    readability.

### Compare

-   `||` vs `??`
-   `&&` vs conditional statements
-   `?.` vs explicit null checks
-   `in` vs own-property checks
-   `delete` vs array removal methods
-   spread vs deep cloning
-   rest vs spread
-   prefix vs postfix increment
-   precedence vs associativity vs evaluation order

### Defend

-   Defend an operator choice in a production code review.
-   Explain the precise semantic reason an expression produces its
    result.
-   Identify when concise operator usage becomes a correctness,
    security, or maintainability risk.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 07 --- Type Conversion, Coercion,
and Equality