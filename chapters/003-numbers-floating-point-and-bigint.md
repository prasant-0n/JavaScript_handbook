
# Chapter 03 --- Numbers, Floating Point, and BigInt

> **Status:** `[+] Completed`\
> **Role in curriculum:** Establishes the complete numeric model used by
> JavaScript: `Number`, IEEE-754 binary64 behavior, precision limits,
> special numeric values, integer safety, conversions, and `BigInt`.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain the ECMAScript `Number` type.
-   Explain why JavaScript uses one primary floating-point numeric type
    for ordinary numbers.
-   Understand IEEE-754 binary64 at a practical and conceptual level.
-   Explain sign, exponent, and significand/fraction.
-   Distinguish mathematical integers from exactly representable
    JavaScript Number values.
-   Explain why decimal fractions such as `0.1` and `0.2` are not
    generally represented exactly in binary floating point.
-   Explain rounding error and accumulated floating-point error.
-   Understand `Number.EPSILON`, `Number.MAX_VALUE`, `Number.MIN_VALUE`,
    `Number.MIN_SAFE_INTEGER`, and `Number.MAX_SAFE_INTEGER`.
-   Distinguish `Number.MIN_VALUE` from the smallest safe integer and
    from the most negative Number.
-   Explain `NaN`, positive infinity, negative infinity, `Infinity`,
    `-0`, and ordinary zero.
-   Understand why `NaN !== NaN`.
-   Understand why `Object.is(-0, 0)` is `false` while `-0 === 0` is
    `true`.
-   Understand integer precision loss above `Number.MAX_SAFE_INTEGER`.
-   Explain the purpose and semantics of `BigInt`.
-   Correctly distinguish Number arithmetic from BigInt arithmetic.
-   Understand why Number and BigInt are intentionally distinct numeric
    types.
-   Use appropriate comparison, conversion, serialization, and
    validation strategies in production systems.
-   Recognize numeric bugs involving money, identifiers, timestamps,
    counters, bit operations, scientific calculations, and large
    integers.

------------------------------------------------------------------------

## 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.

Chapter 02 established that:

``` text
Number
```

and:

``` text
BigInt
```

are different ECMAScript language types.

This chapter explains exactly why that distinction matters.

------------------------------------------------------------------------

# 3. What Is the `Number` Type?

The ECMAScript `Number` type represents numeric values using the
IEEE-754 binary64 floating-point model.

At a high level, a Number can represent:

``` text
ordinary finite numbers
positive zero
negative zero
positive infinity
negative infinity
NaN
```

Examples:

``` js
42
-10
3.14
0
-0
Infinity
-Infinity
NaN
```

Despite the name `Number`, it is not an arbitrary-precision mathematical
number type.

It is a finite-precision representation with a specific range and
precision model.

That distinction is one of the most important facts in JavaScript
numerical programming.

------------------------------------------------------------------------

# 4. One Ordinary Numeric Type, Many Numeric Behaviors

JavaScript does not have separate primitive types such as:

``` text
int32
uint64
float32
double
decimal
```

for ordinary numeric literals.

For example:

``` js
1
1.5
1000000
0.000001
```

are all Number values.

A literal such as:

``` js
42
```

does not create a primitive "integer type" distinct from Number.

At the language level:

``` text
42
```

is a Number.

This makes the language convenient, but it also means the developer must
understand the precision limits of the underlying representation.

------------------------------------------------------------------------

# 5. Why Floating Point Exists

Computers must encode numeric values using finite amounts of storage.

Mathematics gives us infinitely many possible real numbers:

``` text
0
1
2
0.1
π
√2
1 / 3
...
```

A fixed-size representation cannot store all real numbers exactly.

IEEE-754 floating point provides a standardized way to represent a large
and useful subset of numeric values with finite storage.

JavaScript's ordinary Number representation uses binary floating point.

The practical consequence:

``` text
mathematical real number
        ↓
finite binary representation
        ↓
sometimes exact
sometimes rounded
```

That rounding is the source of many familiar JavaScript numeric
surprises.

------------------------------------------------------------------------

# 6. Decimal Numbers vs Binary Numbers

Humans commonly think in decimal:

``` text
0.1
0.2
0.3
```

Computers using binary floating point represent fractions using powers
of two.

For example:

``` text
1/2  = 0.5
1/4  = 0.25
1/8  = 0.125
```

These decimal values have exact finite binary representations.

But:

``` text
0.1
```

does not have a finite binary representation.

The binary expansion repeats.

So the implementation must store the closest representable binary64
value.

That means:

``` js
0.1
```

is not, at the representation level, exactly the mathematical decimal
`0.1`.

The same applies to:

``` js
0.2
```

Therefore:

``` js
0.1 + 0.2
```

cannot generally produce exactly the mathematical decimal `0.3`.

------------------------------------------------------------------------

# 7. The Famous Example

``` js
console.log(0.1 + 0.2);
```

Typical result:

``` text
0.30000000000000004
```

This is not evidence that JavaScript's arithmetic is randomly broken.

It is the visible consequence of finite binary floating-point
representation and rounding.

A useful mental model:

``` text
0.1
 ↓
nearest representable binary64 value

0.2
 ↓
nearest representable binary64 value

add them
 ↓
round result to representable binary64 value

display
 ↓
0.30000000000000004
```

The displayed decimal is an exact decimal rendering of the resulting
binary floating-point value under the formatting algorithm used.

------------------------------------------------------------------------

# 8. IEEE-754 Binary64 Structure

A binary64 value uses 64 bits.

Conceptually:

``` text
┌────────┬───────────────┬────────────────────────────────┐
│ sign   │ exponent      │ fraction / significand bits    │
│ 1 bit  │ 11 bits       │ 52 bits                         │
└────────┴───────────────┴────────────────────────────────┘
```

The 52 stored fraction bits work with an implicit leading significand
bit for normalized finite values, giving 53 bits of precision in the
significand.

The exact details differ for:

-   normal finite values,
-   subnormal values,
-   zero,
-   infinities,
-   NaN.

Understanding the bit layout is useful because it explains the precision
and range characteristics of Number.

------------------------------------------------------------------------

# 9. Sign

The sign bit distinguishes positive and negative values.

For ordinary finite nonzero values:

``` text
sign = 0 → positive
sign = 1 → negative
```

But the sign bit also creates an important special case:

``` text
+0
-0
```

JavaScript preserves signed zero semantics at the language level.

This will matter later.

------------------------------------------------------------------------

# 10. Exponent

The exponent controls the scale of the number.

Very roughly:

``` text
significand × 2^exponent
```

This allows a finite number of bits to represent values across a very
large magnitude range.

The exponent field also has special encodings for:

``` text
zero/subnormal region
normal finite values
infinity
NaN
```

The full IEEE-754 encoding is more precise than this simplified picture,
but the simplified model is enough to understand why floating point
provides both range and precision.

------------------------------------------------------------------------

# 11. Significand / Fraction

The significand carries the meaningful precision bits.

For normalized binary64 values, there is effectively one leading
significant bit in addition to the 52 stored fraction bits.

Therefore ordinary Number values provide approximately:

``` text
53 bits of binary integer precision
```

This is the origin of JavaScript's famous safe integer boundary.

------------------------------------------------------------------------

# 12. Precision vs Range

Floating point gives you both:

``` text
large numeric range
```

and:

``` text
finite precision
```

These are not the same thing.

A Number can represent extremely large magnitudes:

``` js
1e308
```

But that does not mean it can represent every integer anywhere near that
magnitude.

As numbers grow, the spacing between adjacent representable values
increases.

This is one of the most important floating-point ideas:

``` text
small magnitude
  → representable values are more densely spaced

large magnitude
  → representable values are more widely spaced
```

------------------------------------------------------------------------

# 13. Safe Integers

JavaScript defines the safe integer range:

``` js
Number.MIN_SAFE_INTEGER
Number.MAX_SAFE_INTEGER
```

Values are:

``` text
-9007199254740991
+9007199254740991
```

The absolute limit is:

``` text
2^53 - 1
```

Within this integer range, every integer is represented exactly and
integer comparisons/arithmetic behave predictably under the safe-integer
assumptions.

Example:

``` js
Number.MAX_SAFE_INTEGER === 9007199254740991;
```

is:

``` text
true
```

------------------------------------------------------------------------

# 14. Why `2^53 - 1`?

The significand provides 53 bits of precision for ordinary normalized
binary64 values.

That means consecutive integers can be represented exactly up to:

``` text
2^53 - 1
```

Once you move beyond that point, there are integers that cannot each
receive a unique Number representation.

This is not a JavaScript-specific arbitrary limit.

It follows from the finite precision of binary64.

------------------------------------------------------------------------

# 15. The Precision Trap

Consider:

``` js
const x = Number.MAX_SAFE_INTEGER;

console.log(x);
console.log(x + 1);
console.log(x + 2);
```

You might expect:

``` text
9007199254740991
9007199254740992
9007199254740993
```

But the last value is not exactly representable as a distinct Number.

This can lead to behavior such as:

``` js
Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2
```

producing:

``` text
true
```

This is one of the strongest demonstrations that:

``` text
mathematical integers
```

and:

``` text
exactly representable Number integers
```

are different concepts.

------------------------------------------------------------------------

# 16. `Number.isSafeInteger`

Use:

``` js
Number.isSafeInteger(value)
```

to check whether a value is an integer within JavaScript's safe integer
range.

Examples:

``` js
Number.isSafeInteger(42);
// true

Number.isSafeInteger(Number.MAX_SAFE_INTEGER);
// true

Number.isSafeInteger(Number.MAX_SAFE_INTEGER + 1);
// false
```

This is useful for validation at API and data boundaries.

It does not magically make an unsafe numeric value recoverable if
precision has already been lost.

------------------------------------------------------------------------

# 17. `Number.EPSILON`

`Number.EPSILON` is a small value representing the difference between 1
and the next larger representable Number.

Conceptually:

``` text
1
↓
next representable Number
```

The gap is approximately:

``` text
2^-52
```

and is exposed as:

``` js
Number.EPSILON
```

It is commonly used as a reference when discussing floating-point
comparison.

However, an important warning:

> `Number.EPSILON` is not a universal tolerance for every floating-point
> comparison.

Why?

Because representable-number spacing depends on magnitude.

A tolerance appropriate near:

``` text
1
```

may be completely inappropriate near:

``` text
1e12
```

or:

``` text
1e-12
```

------------------------------------------------------------------------

# 18. Floating-Point Equality

This is risky:

``` js
a === b
```

when `a` and `b` come from independent floating-point calculations that
should be mathematically equal.

Example:

``` js
0.1 + 0.2 === 0.3
```

Result:

``` text
false
```

The correct comparison strategy depends on the domain.

A common approximate strategy is:

``` js
Math.abs(a - b) <= tolerance
```

But production code often benefits from a combined absolute/relative
comparison:

``` js
function nearlyEqual(a, b, absTol = 1e-12, relTol = 1e-12) {
  const diff = Math.abs(a - b);

  if (diff <= absTol) {
    return true;
  }

  return diff <= Math.max(Math.abs(a), Math.abs(b)) * relTol;
}
```

The correct tolerance depends on the numerical domain and error budget.

Do not cargo-cult `Number.EPSILON`.

------------------------------------------------------------------------

# 19. `Number.MIN_VALUE` Is Commonly Misunderstood

A major interview trap:

``` js
Number.MIN_VALUE
```

is **not** the most negative Number.

It is a small positive Number: the smallest positive nonzero Number
representable by the Number format.

This distinction is critical.

Compare:

``` js
Number.MIN_VALUE
```

with:

``` js
-Number.MAX_VALUE
```

The first is a tiny positive magnitude.

The second is the most negative finite magnitude.

Therefore:

``` text
MIN_VALUE
  → smallest positive nonzero magnitude

-MAX_VALUE
  → most negative finite Number
```

The naming is historical and easy to misunderstand.

------------------------------------------------------------------------

# 20. `Number.MAX_VALUE`

`Number.MAX_VALUE` is approximately:

``` text
1.7976931348623157 × 10^308
```

It represents the largest finite Number magnitude.

Beyond the finite range:

``` js
Number.MAX_VALUE * 2
```

produces:

``` text
Infinity
```

This is a range overflow, not a precision issue.

------------------------------------------------------------------------

# 21. Subnormal Numbers

Very small positive Numbers can enter the subnormal range.

Subnormal values allow the representation to approach zero more
gradually instead of abruptly underflowing from the smallest normal
value to zero.

Conceptually:

``` text
normal numbers
      ↓
subnormal numbers
      ↓
0
```

Subnormals trade some relative precision for gradual underflow.

This detail is more important in numerical computing than in typical
CRUD application development, but it explains part of the IEEE-754
behavior of Number.

------------------------------------------------------------------------

# 22. Zero Is More Than One Value

JavaScript has:

``` text
+0
-0
```

They compare equal under `===`:

``` js
0 === -0
```

Result:

``` text
true
```

But they are not identical under `Object.is`:

``` js
Object.is(0, -0)
```

Result:

``` text
false
```

This difference exists because JavaScript preserves signed-zero behavior
from the underlying floating-point model.

------------------------------------------------------------------------

# 23. Why Signed Zero Matters

Consider:

``` js
1 / 0;
```

Result:

``` text
Infinity
```

But:

``` js
1 / -0;
```

Result:

``` text
-Infinity
```

The sign of zero therefore affects some operations.

This can matter in:

-   numerical algorithms,
-   complex mathematical transformations,
-   reciprocal calculations,
-   certain graphics/scientific computations.

For ordinary business logic, the distinction may rarely matter, but it
should be understood.

------------------------------------------------------------------------

# 24. Detecting Negative Zero

One classic technique:

``` js
Object.is(value, -0)
```

Example:

``` js
Object.is(-0, -0); // true
Object.is(0, -0);  // false
```

Another mathematical observation:

``` js
1 / -0
```

produces:

``` text
-Infinity
```

but `Object.is` is the clearer semantic test when negative zero itself
is what you care about.

------------------------------------------------------------------------

# 25. `NaN`

`NaN` means:

``` text
Not-a-Number
```

despite being a value of the Number type.

Examples:

``` js
Number("hello");
0 / 0;
Math.sqrt(-1);
```

may produce:

``` text
NaN
```

`NaN` represents an invalid or undefined numeric result within the
Number model.

------------------------------------------------------------------------

# 26. `NaN` Is Still a Number

This is counterintuitive:

``` js
typeof NaN
```

returns:

``` text
"number"
```

That is because `NaN` is a special value of the Number type.

It does not mean that the numeric operation produced a meaningful
ordinary number.

It means:

``` text
Number type
+
special NaN value
```

------------------------------------------------------------------------

# 27. Why `NaN !== NaN`

This is specified behavior:

``` js
NaN === NaN
```

is:

``` text
false
```

And:

``` js
NaN !== NaN
```

is:

``` text
true
```

Why?

`NaN` represents an unordered/invalid numeric result, and the equality
semantics intentionally do not consider NaN equal to itself under
ordinary numeric equality.

To test for NaN reliably, use:

``` js
Number.isNaN(value)
```

Example:

``` js
Number.isNaN(NaN);      // true
Number.isNaN("hello");  // false
```

------------------------------------------------------------------------

# 28. `Number.isNaN` vs Global `isNaN`

Prefer:

``` js
Number.isNaN(value)
```

when you mean:

> "Is this value actually the NaN value?"

The older global:

``` js
isNaN(value)
```

performs coercion before checking.

For example:

``` js
isNaN("hello");
```

returns:

``` text
true
```

because `"hello"` is coerced in the process and becomes an invalid
Number.

By contrast:

``` js
Number.isNaN("hello");
```

returns:

``` text
false
```

because the input is a String, not the actual NaN value.

This is an early example of why coercion rules matter.

------------------------------------------------------------------------

# 29. Infinity

The Number type includes:

``` js
Infinity
-Infinity
```

Examples:

``` js
1 / 0;
-1 / 0;
```

produce:

``` text
Infinity
-Infinity
```

Infinity is not an exception in ordinary numeric division.

It is a special Number value.

You can test finiteness with:

``` js
Number.isFinite(value)
```

Prefer the Number-specific form when you do not want implicit coercion.

------------------------------------------------------------------------

# 30. `Number.isFinite` vs Global `isFinite`

Compare:

``` js
Number.isFinite("10");
```

Result:

``` text
false
```

while:

``` js
isFinite("10");
```

may return:

``` text
true
```

because the global function performs numeric coercion.

For validation, `Number.isFinite` is usually the clearer API because it
asks the direct question:

> Is this already a finite Number value?

------------------------------------------------------------------------

# 31. Integer Conversion Does Not Create Exact Integers Automatically

JavaScript provides integer-related operations such as:

``` js
Math.floor
Math.ceil
Math.round
Math.trunc
```

These operate on Number values.

They do not change the underlying Number type into a special integer
type.

For example:

``` js
Math.floor(3.9)
```

returns:

``` text
3
```

which is still a Number.

Likewise:

``` js
Math.trunc(3.9)
```

returns:

``` text
3
```

as a Number.

------------------------------------------------------------------------

# 32. Bitwise Operators and 32-Bit Integer Semantics

JavaScript bitwise operators have special numeric semantics.

For example:

``` js
5 | 2
```

does not perform general arbitrary-width integer arithmetic.

Bitwise operations conceptually operate using signed 32-bit integer
representations.

This is an important reason why:

``` js
Number
```

should not be equated with:

``` text
unbounded integer
```

Bitwise operators are useful, but they come with their own conversion
rules and limits.

Later chapters will treat them in the broader operators/coercion
context.

------------------------------------------------------------------------

# 33. Why Large Numeric IDs Are Dangerous

Suppose a backend receives an identifier:

``` text
9007199254740993
```

and parses it as a Number.

The exact integer may not be preserved.

That can be catastrophic if the number is an identifier rather than a
quantity.

For example:

``` text
database ID
order number
large account identifier
external system identifier
cryptographic integer
```

should not automatically be represented as Number merely because they
contain digits.

If an identifier does not need arithmetic, a string is often safer:

``` js
const id = "9007199254740993";
```

The correct choice depends on the data contract.

------------------------------------------------------------------------

# 34. Money Is Another Classic Problem

Do not assume this is safe:

``` js
const total = 0.1 + 0.2;
```

For financial calculations, binary floating-point rounding must be
handled deliberately.

Common strategies include:

### Integer smallest units

Store:

``` text
₹10.99
```

as:

``` text
1099
```

paise, assuming the domain defines paise as the unit.

Then arithmetic remains integer-based within the domain's safe range.

### Decimal arithmetic library

Use a decimal representation when financial precision and arbitrary
decimal scale are required.

### Database decimal type

Use an appropriate exact/decimal database representation and preserve
the semantics across API boundaries.

The choice must be driven by the domain's precision requirements.

------------------------------------------------------------------------

# 35. Why Scaling Money to Integers Works

Suppose a currency always has two decimal places.

Instead of:

``` text
12.34
```

use:

``` text
1234
```

representing the smallest unit.

Then:

``` js
1234 + 200
```

is exact as long as the resulting integer remains in the safe integer
range.

This strategy is not universally sufficient.

It fails if the domain requires:

-   arbitrary decimal scale,
-   fractions smaller than the chosen unit,
-   huge values beyond safe integer range.

Always define the monetary model explicitly.

------------------------------------------------------------------------

# 36. Parsing Numeric Strings

JavaScript provides multiple parsing/conversion tools:

``` js
Number("42")
parseInt("42", 10)
parseFloat("42.5")
```

They do not all have identical semantics.

For strict numeric validation:

``` js
const value = Number(input);
```

followed by explicit checks can be clearer.

Example:

``` js
const value = Number(input);

if (!Number.isFinite(value)) {
  throw new Error("Expected a finite number");
}
```

For integer validation:

``` js
if (!Number.isSafeInteger(value)) {
  throw new Error("Expected a safe integer");
}
```

The correct conversion strategy depends on what input grammar your
application accepts.

------------------------------------------------------------------------

# 37. `BigInt` --- The Second Numeric Type

`BigInt` exists for arbitrary-precision integers.

Example:

``` js
const n = 123456789012345678901234567890n;
```

Its value is an integer without the binary64 precision limit of Number.

The essential distinction is:

``` text
Number
  → binary64 floating-point numeric model

BigInt
  → arbitrary-precision integer model
```

BigInt does not replace Number.

They solve different numerical problems.

------------------------------------------------------------------------

# 38. BigInt Literals

BigInt literals use the `n` suffix:

``` js
0n
1n
42n
9007199254740993n
```

Example:

``` js
const value = 9007199254740993n;
console.log(value);
```

This preserves the integer exactly.

Compare with Number:

``` js
const value = 9007199254740993;
```

This cannot preserve the exact mathematical integer if it is outside the
safe integer range.

------------------------------------------------------------------------

# 39. BigInt Arithmetic

BigInt arithmetic works with BigInt operands:

``` js
10n + 20n
```

Result:

``` text
30n
```

Similarly:

``` js
100n * 5n
```

returns:

``` text
500n
```

The result remains BigInt.

------------------------------------------------------------------------

# 40. BigInt Does Not Use Decimal Fractions

BigInt represents integers.

Therefore:

``` js
10n / 3n
```

produces an integer result with the fractional part discarded according
to BigInt division semantics.

It does not produce:

``` text
3.333333...
```

There is no fractional BigInt.

If your domain requires decimal fractions, BigInt alone is not the
solution.

------------------------------------------------------------------------

# 41. Number and BigInt Cannot Be Freely Mixed

This is intentionally invalid:

``` js
1n + 2
```

It throws a `TypeError`.

JavaScript avoids silently guessing whether the programmer wanted:

``` text
Number arithmetic
```

or:

``` text
BigInt arithmetic
```

This protects against accidental precision-changing conversions.

Convert explicitly when appropriate:

``` js
BigInt(2) + 1n
```

or:

``` js
Number(1n) + 2
```

But conversions can lose information, especially when converting large
BigInts to Number.

------------------------------------------------------------------------

# 42. Explicit BigInt-to-Number Conversion Can Be Dangerous

Example:

``` js
const big = 9007199254740993n;

Number(big);
```

The conversion produces a Number that cannot represent every integer
exactly in this range.

Therefore:

``` text
BigInt → Number
```

may lose integer precision.

The reverse:

``` text
Number → BigInt
```

also requires that the Number be an integer, and the semantic meaning
must be carefully considered.

Do not convert numeric types merely to silence a type error.

------------------------------------------------------------------------

# 43. BigInt Equality

Some equality comparisons between Number and BigInt can be true
depending on the equality algorithm.

For example:

``` js
1n == 1
```

is:

``` text
true
```

while:

``` js
1n === 1
```

is:

``` text
false
```

Why?

Strict equality requires compatible types for this comparison.

Loose equality has coercion rules that can compare numeric values across
these types.

This is one of many reasons equality and coercion deserve their own deep
chapter.

------------------------------------------------------------------------

# 44. BigInt and Relational Comparison

Relational operations can compare Number and BigInt values:

``` js
2n < 3
```

and:

``` js
3n > 2
```

can produce boolean results.

The semantic rules differ from arithmetic mixing.

Therefore do not generalize:

``` text
"Number and BigInt cannot interact"
```

The precise rule is:

``` text
some operations permit interaction,
some explicitly reject mixing,
some perform defined cross-type comparisons.
```

------------------------------------------------------------------------

# 45. BigInt and JSON

Standard `JSON.stringify` does not serialize BigInt values directly.

For example:

``` js
JSON.stringify(1n);
```

throws a `TypeError`.

This matters in:

-   REST APIs,
-   logging,
-   cache serialization,
-   frontend/backend boundaries,
-   database drivers.

You must define an explicit representation such as:

``` json
"9007199254740993"
```

or another protocol-specific representation.

Do not assume every JSON consumer can safely understand arbitrary
integers.

------------------------------------------------------------------------

# 46. BigInt Is Not Automatically Better

BigInt is appropriate when you need:

-   exact large integers,
-   counters beyond safe Number range,
-   certain financial/integer-unit domains,
-   large integer algorithms,
-   integer identifiers that require arithmetic.

BigInt is not appropriate merely because:

``` text
"large number sounds safer"
```

Trade-offs include:

-   different operator semantics,
-   no fractional values,
-   interoperability concerns,
-   serialization requirements,
-   potential performance differences,
-   explicit conversion boundaries.

Choose based on the domain.

------------------------------------------------------------------------

# 47. Number vs BigInt vs String

This is a common production decision.

  ---------------------------------------------------------------------
  Requirement                        Good candidate
  ---------------------------------- ----------------------------------
  General scientific/engineering     Number
  numeric values                     

  Ordinary safe integer              Number

  Large exact integer arithmetic     BigInt

  Large numeric identifier with no   String
  arithmetic                         

  Exact decimal financial arithmetic Decimal representation / integer
                                     smallest-unit model / decimal
                                     library

  JSON-compatible large identifier   String is often simplest

  Bitwise operations                 Number with explicit 32-bit
                                     semantics, or appropriate typed
                                     integer abstraction
  ---------------------------------------------------------------------

There is no universal "best numeric type."

------------------------------------------------------------------------

# 48. Number Conversion Rules to Remember

Some common conversions:

``` js
Number("42");       // 42
Number("");         // 0
Number("   ");      // 0
Number("42x");      // NaN
Number(null);       // 0
Number(undefined);  // NaN
Number(true);       // 1
Number(false);      // 0
```

These are coercion rules.

Do not memorize isolated cases without understanding the conversion
algorithms.

Chapter 07 will derive them systematically.

------------------------------------------------------------------------

# 49. Numeric Literals

JavaScript supports multiple numeric literal syntaxes.

Decimal:

``` js
123
123.45
```

Binary:

``` js
0b1010
```

Octal:

``` js
0o755
```

Hexadecimal:

``` js
0xff
```

Numeric separators:

``` js
1_000_000
```

BigInt forms can use the same radix styles with `n`:

``` js
0xffn
0b1010n
0o755n
```

These are syntax-level representations of values, not different Number
subtypes.

------------------------------------------------------------------------

# 50. Scientific Notation

JavaScript supports exponential notation:

``` js
1e3
```

means:

``` text
1000
```

and:

``` js
1.5e3
```

means:

``` text
1500
```

Likewise:

``` js
1e-3
```

means:

``` text
0.001
```

Scientific notation is useful for very large and very small numbers.

------------------------------------------------------------------------

# 51. Rounding

JavaScript provides:

``` js
Math.round
Math.floor
Math.ceil
Math.trunc
```

They solve different problems.

### `Math.floor`

Moves toward negative infinity.

``` js
Math.floor(1.9);   // 1
Math.floor(-1.1);  // -2
```

### `Math.ceil`

Moves toward positive infinity.

``` js
Math.ceil(1.1);    // 2
Math.ceil(-1.9);   // -1
```

### `Math.trunc`

Removes the fractional part toward zero.

``` js
Math.trunc(1.9);   // 1
Math.trunc(-1.9);  // -1
```

### `Math.round`

Rounds according to JavaScript's specified rounding semantics, including
its behavior around half values and signed zero.

Do not use the function names as if they were interchangeable.

------------------------------------------------------------------------

# 52. Floating-Point Error Accumulation

Even small rounding errors can accumulate:

``` js
let total = 0;

for (let i = 0; i < 10; i++) {
  total += 0.1;
}
```

The result may not be exactly:

``` text
1
```

depending on representation and arithmetic sequence.

This is a numerical-analysis problem, not merely a JavaScript syntax
problem.

For large numerical systems, engineers may need:

-   compensated summation,
-   domain-specific rounding,
-   decimal arithmetic,
-   integer scaling,
-   numerical error budgets,
-   exact arithmetic libraries.

------------------------------------------------------------------------

# 53. Catastrophic Cancellation --- Conceptual Warning

Floating-point algorithms can lose significant precision when
subtracting nearly equal quantities.

For example:

``` text
a ≈ b
a - b
```

can discard meaningful significant digits.

This is called cancellation and is a general floating-point
numerical-analysis issue.

The solution is algorithm-specific.

Do not assume changing JavaScript syntax or using `Number.EPSILON` fixes
numerical instability.

------------------------------------------------------------------------

# 54. Numeric Stability Is Different From Numeric Correctness

A program can be:

``` text
syntactically correct
```

and:

``` text
language-semantically correct
```

yet:

``` text
numerically unstable
```

For example, a mathematically valid algorithm can amplify rounding
errors.

Therefore a serious numerical code review asks:

``` text
Is the representation appropriate?
Is the algorithm numerically stable?
What precision is required?
What is the error tolerance?
What happens at boundaries?
```

------------------------------------------------------------------------

# 55. Performance Considerations

Number arithmetic is heavily optimized in modern JavaScript engines.

However, performance depends on:

-   value patterns,
-   type stability,
-   arithmetic operations,
-   allocations,
-   conversions,
-   JIT optimization,
-   deoptimization,
-   engine implementation,
-   algorithmic complexity.

BigInt operations can have different cost characteristics because
arbitrary precision means the computational cost can grow with operand
size.

Therefore:

``` text
Number
  → fixed-width floating-point representation

BigInt
  → variable-size integer representation
```

This often produces different performance profiles.

Never assume BigInt is a drop-in performance-equivalent replacement for
Number.

------------------------------------------------------------------------

# 56. Memory Considerations

A Number's language-level semantics do not require you to reason about a
particular heap allocation strategy.

Engines may represent certain Numbers efficiently without allocating
ordinary heap objects for every value.

Similarly, BigInt values may require storage proportional to their
magnitude.

The language specification does not prescribe one physical memory layout
for these values.

Therefore:

``` text
Number / BigInt type
```

should not be directly translated into:

``` text
exact number of machine words in every engine
```

That is implementation-specific.

------------------------------------------------------------------------

# 57. Security Considerations

Numeric errors can become security issues.

Examples:

### Authorization

``` js
if (userBalance >= requestedAmount) {
  // ...
}
```

Precision or parsing bugs can produce incorrect decisions.

### Rate limits

Counters or timestamps that overflow, round, or parse incorrectly can
weaken enforcement.

### Financial systems

Floating-point inaccuracies can create monetary discrepancies.

### Integer parsing

Very large externally supplied integers can be truncated when converted
to Number.

### Resource limits

Unsafe numeric parsing may cause incorrect memory or computation limits.

At trust boundaries, define:

``` text
accepted grammar
range
precision
type
representation
```

Do not merely call `Number()` and assume the input is safe.

------------------------------------------------------------------------

# 58. Production Validation Pattern

A useful pattern for a finite safe integer:

``` js
function assertSafeInteger(value, name = "value") {
  if (!Number.isSafeInteger(value)) {
    throw new TypeError(`${name} must be a safe integer`);
  }

  return value;
}
```

For a finite Number:

``` js
function assertFiniteNumber(value, name = "value") {
  if (!Number.isFinite(value)) {
    throw new TypeError(`${name} must be a finite number`);
  }

  return value;
}
```

For BigInt:

``` js
function assertBigInt(value, name = "value") {
  if (typeof value !== "bigint") {
    throw new TypeError(`${name} must be a BigInt`);
  }

  return value;
}
```

The important engineering principle:

> Validate according to the domain contract, not merely according to
> JavaScript's ability to parse the input.

------------------------------------------------------------------------

# 59. Debugging Workflow for Numeric Bugs

When a numeric bug appears:

### Step 1

Print the exact value.

### Step 2

Check the type:

``` js
typeof value
```

### Step 3

Check finiteness:

``` js
Number.isFinite(value)
```

### Step 4

Check integer status:

``` js
Number.isInteger(value)
```

### Step 5

If integer precision matters:

``` js
Number.isSafeInteger(value)
```

### Step 6

Check whether the input was converted from a string.

### Step 7

Check whether Number and BigInt were mixed.

### Step 8

Check rounding and comparison strategy.

### Step 9

Check serialization boundaries.

### Step 10

Check the mathematical algorithm itself.

A correct diagnosis requires distinguishing representation error from
algorithmic error.

------------------------------------------------------------------------

# 60. Common Misconceptions

## Misconception 1 --- "JavaScript Number is a normal integer type."

False.

Number is IEEE-754 binary64 floating point.

------------------------------------------------------------------------

## Misconception 2 --- "All integers are exact in JavaScript."

False.

Only integers within the safe integer range are guaranteed to be exactly
represented as distinct integers under the relevant assumptions.

------------------------------------------------------------------------

## Misconception 3 --- "`Number.MAX_VALUE` is the largest safe integer."

False.

`MAX_VALUE` is the largest finite Number magnitude.

The largest safe integer is:

``` js
Number.MAX_SAFE_INTEGER
```

------------------------------------------------------------------------

## Misconception 4 --- "`Number.MIN_VALUE` is the smallest Number."

Misleading.

It is the smallest positive nonzero Number, not the most negative finite
Number.

------------------------------------------------------------------------

## Misconception 5 --- "`Number.EPSILON` fixes floating-point comparisons."

False.

It can be useful as a reference near 1, but real comparisons require an
appropriate tolerance model.

------------------------------------------------------------------------

## Misconception 6 --- "`NaN` is not a Number."

False.

It is a special value of the Number type.

------------------------------------------------------------------------

## Misconception 7 --- "BigInt should replace Number everywhere."

False.

BigInt represents arbitrary-precision integers, not fractions or general
floating-point quantities.

------------------------------------------------------------------------

## Misconception 8 --- "A huge identifier should be stored as a Number because it contains digits."

False.

If arithmetic is unnecessary and exact preservation matters, a String
may be more appropriate.

------------------------------------------------------------------------

# 61. Common Mistakes

### Mistake: Money with raw floating-point arithmetic

``` js
price * quantity
```

may be insufficient for exact financial requirements.

### Mistake: Large ID as Number

``` js
const id = Number("9007199254740993");
```

can lose precision.

### Mistake: Blind BigInt conversion

``` js
BigInt(number)
```

does not recover precision already lost by an unsafe Number.

### Mistake: Comparing floats with exact equality

``` js
a === b
```

can reject mathematically equal values produced through different
floating-point calculations.

### Mistake: Using global `isNaN`

It coerces input.

Prefer `Number.isNaN` for strict validation.

### Mistake: Confusing zero values

``` text
0
-0
NaN
Infinity
```

have different semantics.

------------------------------------------------------------------------

# 62. Comparison --- Number vs BigInt

  -----------------------------------------------------------------------
  Property                Number                  BigInt
  ----------------------- ----------------------- -----------------------
  Type                    `number`                `bigint`

  Representation model    binary64 floating point arbitrary-precision
                                                  integer

  Fractions               Yes                     No

  Large exact integers    Limited by safe range   Yes

  `typeof`                `"number"`              `"bigint"`

  `1 + 2`                 `3`                     `3n` with BigInt
                                                  operands

  Mixed arithmetic        ---                     Number + BigInt throws

  JSON.stringify direct   Yes for supported       Direct BigInt
  support                 Number values           serialization throws

  Typical use             general numeric         exact large integer
                          computing               arithmetic
  -----------------------------------------------------------------------

The table is a decision aid, not a replacement for understanding
semantics.

------------------------------------------------------------------------

# 63. Conceptual IEEE-754 Example

Imagine a simplified binary representation:

``` text
sign | exponent | significand
```

A finite value is conceptually close to:

``` text
(-1)^sign × significand × 2^exponent
```

This explains why the representation is powerful:

``` text
exponent
  → range

significand
  → precision
```

The trade-off is unavoidable:

``` text
fixed number of bits
+
huge numeric range
=
finite precision
```

This is the core floating-point trade-off.

------------------------------------------------------------------------

# 64. Execution Walkthrough --- `0.1 + 0.2`

Conceptual process:

### Step 1

Evaluate:

``` js
0.1
```

A Number value is produced.

Its exact mathematical decimal value cannot be represented finitely in
binary64.

### Step 2

Evaluate:

``` js
0.2
```

Again, a nearby representable binary64 value is used.

### Step 3

Perform Number addition.

### Step 4

The mathematical result is approximately:

``` text
0.3
```

### Step 5

The result must also be represented as a binary64 Number.

### Step 6

The nearest representable result is produced.

### Step 7

When displayed in decimal form, the result is commonly:

``` text
0.30000000000000004
```

The important lesson:

``` text
rounding occurs because representation is finite
```

not because `+` is unreliable.

------------------------------------------------------------------------

# 65. Execution Walkthrough --- Unsafe Integer

Consider:

``` js
const x = 9007199254740992;
const y = 9007199254740993;
```

Conceptually:

``` text
x
↓
representable Number

y
↓
mathematical integer requested

y
↓
cannot necessarily be represented as a distinct Number
```

So the source literal does not guarantee that the runtime retains the
mathematical integer exactly.

This is why large integer boundaries must be treated as representation
boundaries.

------------------------------------------------------------------------

# 66. Execution Walkthrough --- BigInt

``` js
const x = 9007199254740993n;
```

The value is a BigInt.

Its integer semantics do not depend on the Number safe-integer boundary.

Then:

``` js
x + 7n
```

produces another exact BigInt integer.

The trade-off is that BigInt is not a floating-point or decimal type.

------------------------------------------------------------------------

# 67. Implementation From Scratch --- Safe Numeric Helpers

Implement:

``` js
function isSafeFiniteNumber(value) {
  // ...
}
```

Required behavior:

``` text
42        → true
3.14      → true
Infinity  → false
NaN       → false
1n        → false
"42"      → false
```

Then implement:

``` js
function isSafeIntegerLike(value) {
  // ...
}
```

Define the contract precisely.

The exercise is designed to reinforce that:

``` text
type
+
finite
+
integer
+
safe range
```

are separate predicates.

------------------------------------------------------------------------

# 68. Implementation Progression

### Guided

Implement:

``` js
isFiniteNumber
isIntegerNumber
isSafeInteger
isBigInt
```

using the appropriate built-ins.

### Partially Guided

Create:

``` js
validateNumericInput(value, options)
```

where options can specify:

``` text
number / bigint
finite
integer
safeInteger
min
max
```

### No Reference

Create a reusable numeric validation module for an API service.

### Edge-Case Hardened

Test:

``` text
NaN
Infinity
-Infinity
0
-0
Number.MAX_VALUE
Number.MIN_VALUE
MAX_SAFE_INTEGER
MAX_SAFE_INTEGER + 1
0n
large BigInt
numeric strings
empty string
null
undefined
```

### Production-Grade

Add:

-   explicit domain contracts,
-   useful error messages,
-   tests,
-   serialization rules,
-   boundary tests,
-   documentation.

------------------------------------------------------------------------

# 69. Debugging Exercises

## Exercise 1

Why does this happen?

``` js
console.log(0.1 + 0.2 === 0.3);
```

## Exercise 2

Why can this happen?

``` js
console.log(Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2);
```

## Exercise 3

Why are these different?

``` js
console.log(0 === -0);
console.log(Object.is(0, -0));
```

## Exercise 4

Why are these different?

``` js
console.log(isNaN("hello"));
console.log(Number.isNaN("hello"));
```

## Exercise 5

Why does this throw?

``` js
console.log(1n + 1);
```

## Exercise 6

Why can this be dangerous?

``` js
const id = Number("9007199254740993");
```

------------------------------------------------------------------------

# 70. Code Review Exercise

Review:

``` js
function calculatePrice(price, quantity) {
  return price * quantity;
}
```

A senior review should ask:

-   Is `price` binary floating point?
-   Is exact decimal behavior required?
-   What is the maximum allowed value?
-   What is the input type?
-   Can `NaN` or infinity enter?
-   Is rounding defined?
-   Is currency represented in smallest units?
-   Does the database use decimal or integer representation?
-   Does JSON serialization preserve the intended semantics?

The important lesson:

> Numeric correctness is a domain contract, not merely an arithmetic
> expression.

------------------------------------------------------------------------

# 71. Interview Questions

## Junior

1.  What is the JavaScript `Number` type?
2.  Is JavaScript Number an integer type?
3.  Why is `0.1 + 0.2` not exactly `0.3`?
4.  What is `NaN`?
5.  What does `typeof NaN` return?

## Mid-Level

6.  What is `Number.MAX_SAFE_INTEGER`?
7.  Why is `Number.MIN_VALUE` often misunderstood?
8.  What is the difference between `Number.isNaN` and `isNaN`?
9.  What is negative zero?
10. Why can large integer IDs lose precision?

## Senior

11. Explain IEEE-754 binary64 at a practical level.
12. Explain precision vs range.
13. Why is `Number.EPSILON` not a universal floating-point tolerance?
14. When should you use BigInt?
15. Why should monetary calculations be designed explicitly?

## Staff / Principal

16. Design a numeric representation strategy for a payment platform.
17. Design an API contract for 64-bit database IDs.
18. When would you choose String instead of BigInt for identifiers?
19. How would you detect numeric precision loss at service boundaries?
20. How would you review a scientific algorithm for floating-point
    stability?
21. What observable differences can signed zero create?
22. How would you document numeric guarantees for a public JavaScript
    library?

------------------------------------------------------------------------

# 72. Predict-the-Output Exercises

Predict before executing.

### 1

``` js
console.log(typeof 1);
```

### 2

``` js
console.log(typeof 1n);
```

### 3

``` js
console.log(0.1 + 0.2);
```

### 4

``` js
console.log(0.1 + 0.2 === 0.3);
```

### 5

``` js
console.log(Number.MAX_SAFE_INTEGER);
```

### 6

``` js
console.log(Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2);
```

### 7

``` js
console.log(0 === -0);
```

### 8

``` js
console.log(Object.is(0, -0));
```

### 9

``` js
console.log(1 / 0);
```

### 10

``` js
console.log(1 / -0);
```

### 11

``` js
console.log(NaN === NaN);
```

### 12

``` js
console.log(Number.isNaN("hello"));
```

### 13

``` js
console.log(isNaN("hello"));
```

### 14

``` js
console.log(10n + 20n);
```

### 15

``` js
console.log(10n + 20);
```

### 16

``` js
console.log(1n == 1);
console.log(1n === 1);
```

### 17

``` js
console.log(Number.isSafeInteger(9007199254740991));
```

### 18

``` js
console.log(Number.isSafeInteger(9007199254740992));
```

------------------------------------------------------------------------

# 73. Mastery Exercises

## Exercise A --- Explain IEEE-754

Explain:

``` text
sign
exponent
significand
precision
range
rounding
```

without reducing the explanation to:

> "JavaScript has floating-point problems."

## Exercise B --- Safe Integer Audit

Take five real identifiers or counters from an application and determine
whether Number is safe for them.

## Exercise C --- Money Model

Design a money representation for:

``` text
amount
currency
scale
rounding
serialization
database storage
API representation
```

## Exercise D --- BigInt Boundary

Design a function that safely accepts either:

``` text
safe Number integer
```

or:

``` text
BigInt
```

and normalizes it without precision loss.

State which inputs are rejected and why.

## Exercise E --- Floating Comparison

Implement and test:

``` js
nearlyEqual(a, b)
```

with both absolute and relative tolerance.

Then explain why the tolerance values are domain-specific.

------------------------------------------------------------------------

# 74. Production Numeric Decision Framework

When choosing a representation, ask:

``` text
1. Is it a quantity or an identifier?
2. Does it require arithmetic?
3. Can it contain fractions?
4. Must every integer be exact?
5. What is the maximum magnitude?
6. What precision is required?
7. What rounding rules apply?
8. What will the database store?
9. What will JSON/API clients receive?
10. What happens across language boundaries?
11. What happens when the value is invalid?
12. What is the performance requirement?
```

This decision framework is much more valuable than memorizing a list of
Number constants.

------------------------------------------------------------------------

# 75. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.

## Builds Toward

-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.
-   Chapter 15 --- Object and Property Semantics.
-   Chapter 27 --- Typed Arrays and Binary Data.
-   Chapter 28 --- JSON and Serialization.
-   Chapter 72 --- Complexity and Cost Analysis.
-   Chapter 81 --- Database Integration.
-   Chapter 82 --- API Architecture.
-   Chapter 84 --- Reliability Engineering.
-   Chapter 85 --- Performance Engineering.
-   Chapter 96 --- WebAssembly and Native Interoperability.
-   Chapter 100 --- Cost Models and Engineering Trade-offs.

## Related Concepts

``` text
binary representation
precision
rounding
coercion
equality
serialization
data modeling
API contracts
database types
financial correctness
numerical stability
```

## Concepts Revisited Later

-   Number coercion.
-   Equality algorithms.
-   `Object.is`.
-   Bitwise conversion.
-   Typed arrays.
-   Binary data.
-   JSON serialization.
-   Memory representation.
-   Engine optimization.
-   Performance.

## Why This Chapter Matters Later

Many JavaScript bugs that look like syntax or API bugs are actually
representation bugs.

Understanding:

``` text
what value exists
+
how precisely it can be represented
+
what conversion occurred
```

makes later work with operators, objects, serialization, databases, and
production APIs substantially more rigorous.

------------------------------------------------------------------------

# 76. Key Takeaways

1.  JavaScript's ordinary `Number` type uses IEEE-754 binary64
    floating-point semantics.
2.  Number is not an arbitrary-precision integer type.
3.  Floating point trades finite precision for a large representable
    range.
4.  Decimal fractions such as `0.1` often do not have finite binary
    representations.
5.  `0.1 + 0.2` therefore illustrates representation and rounding
    behavior rather than broken arithmetic.
6.  Binary64 provides approximately 53 bits of significand precision for
    ordinary normalized values.
7.  The largest safe integer is `2^53 - 1`, exposed as
    `Number.MAX_SAFE_INTEGER`.
8.  `Number.MIN_VALUE` is the smallest positive nonzero Number, not the
    most negative Number.
9.  `NaN`, `Infinity`, `-Infinity`, `0`, and `-0` are special Number
    values.
10. `NaN !== NaN`; use `Number.isNaN` when testing for actual NaN.
11. `0 === -0` is true, while `Object.is(0, -0)` is false.
12. `Number.EPSILON` is useful but is not a universal tolerance for
    floating-point comparison.
13. Large integer identifiers can silently lose precision when converted
    to Number.
14. `BigInt` provides arbitrary-precision integer semantics.
15. BigInt does not represent fractions and should not be treated as a
    universal replacement for Number.
16. Number and BigInt arithmetic cannot be freely mixed.
17. BigInt serialization requires an explicit representation at JSON
    boundaries.
18. Numeric correctness is a domain-level engineering problem involving
    representation, validation, arithmetic, rounding, storage, and
    serialization.

------------------------------------------------------------------------

# 77. Completion Criteria

### Understand

-   Explain Number and BigInt.
-   Explain IEEE-754 binary64 at a practical level.
-   Explain safe integer limits and special Number values.

### Explain

-   Explain `0.1 + 0.2`.
-   Explain the safe integer boundary.
-   Explain signed zero.
-   Explain NaN.
-   Explain why Number and BigInt are different types.

### Predict

-   Predict floating-point comparisons.
-   Predict special-value behavior.
-   Predict Number/BigInt mixed-operation results.

### Implement

-   Implement safe numeric validators.
-   Implement an appropriate approximate floating-point comparison.
-   Implement domain-specific numeric boundary validation.

### Debug

-   Diagnose precision loss.
-   Diagnose incorrect numeric parsing.
-   Diagnose incorrect floating-point comparison.
-   Diagnose accidental Number/BigInt mixing.

### Apply

-   Choose Number, BigInt, String, integer-scaled, or decimal
    representations according to a real domain.

### Compare

-   Number vs BigInt.
-   Number vs String for identifiers.
-   Exact integer arithmetic vs floating-point arithmetic.
-   Absolute vs relative floating-point tolerance.

### Defend

-   Defend a numeric representation choice using precision, range,
    correctness, interoperability, performance, and operational
    requirements.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 04 --- Strings, Unicode, and Text
Semantics
