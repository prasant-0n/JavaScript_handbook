
# Chapter 04 --- Strings, Unicode, and Text Semantics

> **Status:** `[+] Completed`\
> **Role in curriculum:** Establishes the correct mental model for
> JavaScript text. Strings are not simply "arrays of characters";
> JavaScript strings are sequences of UTF-16 code units, while users
> perceive text as Unicode grapheme clusters. This distinction is
> fundamental for internationalization, validation, indexing, slicing,
> security, search, storage, APIs, and UI behavior.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain what a JavaScript String value represents.
-   Explain why JavaScript strings are sequences of UTF-16 code units.
-   Distinguish:
    -   bytes,
    -   code units,
    -   code points,
    -   grapheme clusters.
-   Explain UTF-8 vs UTF-16 without confusing encoding with JavaScript's
    String model.
-   Explain the meaning of the UTF-16 code unit.
-   Explain surrogate pairs.
-   Explain astral Unicode code points.
-   Predict the behavior of:
    -   `length`,
    -   bracket indexing,
    -   `charAt`,
    -   `charCodeAt`,
    -   `codePointAt`,
    -   `fromCharCode`,
    -   `fromCodePoint`.
-   Explain why some visible characters occupy more than one UTF-16 code
    unit.
-   Explain why `string.length` does not necessarily equal the number of
    user-perceived characters.
-   Explain combining marks.
-   Explain grapheme clusters and why they matter in user interfaces.
-   Explain Unicode normalization:
    -   NFC,
    -   NFD,
    -   NFKC,
    -   NFKD.
-   Understand when visually identical text can have different
    underlying sequences.
-   Explain Unicode-aware regular expressions at a conceptual level.
-   Understand important Unicode/security issues such as confusables and
    normalization boundaries.
-   Design safer string processing for real production systems.

------------------------------------------------------------------------

## 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.

Chapter 02 established that String is a primitive type.

This chapter adds the representation model:

``` text
String value
   ↓
sequence of UTF-16 code units
   ↓
may encode Unicode code points
   ↓
may represent grapheme clusters
   ↓
may be perceived by a user as characters
```

These layers are not interchangeable.

------------------------------------------------------------------------

# 3. What Is a JavaScript String?

A JavaScript String is a primitive value representing text as a sequence
of UTF-16 code units.

Examples:

``` js
"hello"
"नमस्ते"
"こんにちは"
"😀"
```

The source code may contain Unicode characters directly, but the
language's String model is based on UTF-16 code units.

This creates one of the most important rules in JavaScript text
processing:

> A JavaScript string's `length` is measured in UTF-16 code units, not
> in user-perceived characters.

For ASCII text, the difference is often invisible:

``` js
"hello".length
```

is:

``` text
5
```

But for Unicode text, the distinction becomes important.

------------------------------------------------------------------------

# 4. Four Different Concepts

Before working with Unicode, distinguish these four concepts.

``` text
Byte
Code unit
Code point
Grapheme cluster
```

They answer different questions.

### Byte

A byte is an 8-bit unit commonly used in encoded binary data.

Examples of encodings:

``` text
UTF-8
UTF-16
UTF-32
```

### Code unit

A fixed-size element used by a particular encoding/model.

For JavaScript Strings, a UTF-16 code unit is 16 bits.

### Code point

A Unicode scalar value / Unicode code point identifies an abstract
Unicode character or symbol within the Unicode code point space.

Examples:

``` text
U+0041  → LATIN CAPITAL LETTER A
U+1F600 → GRINNING FACE
```

### Grapheme cluster

A sequence of Unicode code points that should be treated as one
user-perceived text unit in a given segmentation model.

For example, a base character plus combining mark may visually behave as
one unit.

Therefore:

``` text
byte
  ≠
code unit
  ≠
code point
  ≠
grapheme cluster
```

This distinction solves many Unicode bugs.

------------------------------------------------------------------------

# 5. Unicode Is Not an Encoding

Unicode defines a universal character repertoire and related properties
and algorithms.

Encodings define how Unicode text is represented as bytes.

Common encodings include:

``` text
UTF-8
UTF-16
UTF-32
```

A useful model:

``` text
Abstract Unicode text
       ↓
encoding
       ↓
bytes
```

JavaScript's String model is different from a byte buffer.

A JavaScript string is not simply:

``` text
"some bytes"
```

It is a sequence of UTF-16 code units.

------------------------------------------------------------------------

# 6. UTF-8 vs UTF-16

UTF-8 and UTF-16 are both Unicode encodings.

### UTF-8

Uses 1--4 bytes per encoded Unicode code point.

Typical examples:

``` text
ASCII
→ 1 byte per character/code point in UTF-8
```

Many web protocols and file formats use UTF-8.

### UTF-16

Uses 16-bit code units.

Many Unicode code points fit in one code unit.

Code points outside the Basic Multilingual Plane generally require two
UTF-16 code units.

JavaScript's String abstraction uses UTF-16 code units.

Important:

> JavaScript String semantics are not the same thing as UTF-8 byte
> semantics used when transmitting text over a network.

------------------------------------------------------------------------

# 7. Basic Multilingual Plane

Unicode divides its code point space into planes.

The first plane is the:

``` text
Basic Multilingual Plane (BMP)
```

It covers code points from:

``` text
U+0000
```

through:

``` text
U+FFFF
```

Many commonly used characters are inside the BMP.

However, characters outside the BMP exist.

Examples include many emoji and historic scripts.

Those code points may require surrogate pairs in UTF-16.

------------------------------------------------------------------------

# 8. Surrogate Pairs

UTF-16 uses two 16-bit code units to represent a Unicode code point
outside the BMP.

Those two code units are called a:

``` text
surrogate pair
```

For example:

``` js
const text = "😀";
```

The grinning-face code point is:

``` text
U+1F600
```

But its UTF-16 representation uses two code units.

Therefore:

``` js
text.length
```

returns:

``` text
2
```

even though a user may perceive:

``` text
😀
```

as one character.

This is not contradictory.

The metrics are counting different things.

------------------------------------------------------------------------

# 9. Why `length` Can Be Surprising

Consider:

``` js
const text = "😀";

console.log(text.length);
```

Result:

``` text
2
```

A naive assumption is:

``` text
length = number of characters
```

That is false for JavaScript strings.

A better statement is:

> `String.prototype.length` gives the number of UTF-16 code units in the
> String value.

This distinction is foundational.

------------------------------------------------------------------------

# 10. Indexing Strings

Consider:

``` js
const text = "😀";

console.log(text[0]);
console.log(text[1]);
```

Neither element represents the full Unicode code point `😀`.

Each index accesses one UTF-16 code unit.

This means:

``` js
text[0]
```

and:

``` js
text[1]
```

can expose surrogate halves.

This is one reason naive string indexing is unsafe for code-point-aware
text processing.

------------------------------------------------------------------------

# 11. `charCodeAt`

`charCodeAt` returns the UTF-16 code unit value at a given index.

Example:

``` js
const text = "A";

console.log(text.charCodeAt(0));
```

Result:

``` text
65
```

because:

``` text
U+0041
```

has numeric value 65.

For a surrogate pair, `charCodeAt` returns one 16-bit unit at a time.

Therefore:

``` js
const text = "😀";

text.charCodeAt(0);
text.charCodeAt(1);
```

return the two surrogate code-unit values rather than the full code
point.

This is why `charCodeAt` is not equivalent to "give me the Unicode
character number."

------------------------------------------------------------------------

# 12. `codePointAt`

`codePointAt` exists for code-point-aware access.

Example:

``` js
const text = "😀";

console.log(text.codePointAt(0));
```

Result:

``` text
128512
```

which corresponds to:

``` text
U+1F600
```

This is different from:

``` js
text.charCodeAt(0)
```

because `charCodeAt` exposes UTF-16 code units.

So:

``` text
charCodeAt
  → code unit

codePointAt
  → code point
```

------------------------------------------------------------------------

# 13. `fromCharCode` vs `fromCodePoint`

Similarly:

``` js
String.fromCharCode(...)
```

constructs strings from UTF-16 code units.

Whereas:

``` js
String.fromCodePoint(...)
```

constructs strings from Unicode code points.

For example:

``` js
String.fromCodePoint(0x1F600);
```

produces:

``` text
😀
```

This is often the clearer choice when working with Unicode code points
directly.

------------------------------------------------------------------------

# 14. Code Point vs Grapheme Cluster

Even `codePointAt` is not the final answer to:

> "How many characters does the user see?"

Consider a letter followed by a combining mark.

Conceptually:

``` text
base character
+
combining mark
=
one visible text unit
```

There can be:

``` text
2 code points
```

but:

``` text
1 grapheme cluster
```

Therefore:

``` text
UTF-16 code units
        ↓
Unicode code points
        ↓
grapheme clusters
```

Each layer can produce a different count.

------------------------------------------------------------------------

# 15. Combining Marks

Unicode includes combining marks.

A combining mark modifies or combines with another code point.

For example, a character that visually appears as:

``` text
é
```

may be represented either as:

``` text
U+00E9
```

or as:

``` text
U+0065
+
U+0301
```

The latter means:

``` text
LATIN SMALL LETTER E
+
COMBINING ACUTE ACCENT
```

These are different code point sequences that can render equivalently in
many contexts.

This is one of the reasons text normalization exists.

------------------------------------------------------------------------

# 16. Canonically Equivalent Strings

Suppose:

``` js
const a = "\u00E9";
const b = "\u0065\u0301";
```

They can render identically.

But:

``` js
a === b
```

is:

``` text
false
```

because JavaScript compares the underlying String values; it does not
automatically normalize every string before equality comparison.

This is critical.

> Unicode normalization is not automatically applied to every string
> operation.

If your application requires canonical equivalence, normalize explicitly
at the appropriate boundary.

------------------------------------------------------------------------

# 17. `String.prototype.normalize`

JavaScript provides:

``` js
String.prototype.normalize()
```

with standard normalization forms:

``` text
NFC
NFD
NFKC
NFKD
```

Example:

``` js
const a = "\u00E9";
const b = "\u0065\u0301";

console.log(a === b);
console.log(a.normalize("NFC") === b.normalize("NFC"));
```

The first comparison is false.

The normalized comparison can become true because both sequences are
transformed to a common normalization form.

------------------------------------------------------------------------

# 18. NFC

NFC means approximately:

``` text
Canonical Decomposition
+
Canonical Composition
```

It tends to produce composed canonical forms where applicable.

It is often a reasonable normalization choice when an application needs
a canonical Unicode representation without compatibility folding.

Do not treat NFC as "remove accents."

It does not mean that.

------------------------------------------------------------------------

# 19. NFD

NFD performs:

``` text
Canonical Decomposition
```

So a precomposed character can be represented as a base plus combining
marks.

For example:

``` text
é
```

can decompose conceptually into:

``` text
e + combining acute accent
```

NFD can be useful in text-processing pipelines where decomposed
representations are desirable.

------------------------------------------------------------------------

# 20. NFKC and NFKD

The `K` forms include **compatibility decomposition**.

That means they can unify characters that are considered compatibility
equivalents.

This can be useful for certain search, comparison, and canonicalization
scenarios.

But there is a critical warning:

> Compatibility normalization can erase distinctions that matter to an
> application.

Therefore NFKC/NFKD should not be used blindly for identity, security,
or user data storage.

Normalization must be chosen according to the semantic goal.

------------------------------------------------------------------------

# 21. Canonical vs Compatibility Equivalence

A simplified distinction:

### Canonical equivalence

Different sequences represent the same abstract text in a canonical
sense.

### Compatibility equivalence

Different representations are treated as equivalent for broader
compatibility purposes, even when distinctions may matter in some
contexts.

Therefore:

``` text
NFC/NFD
  → canonical forms

NFKC/NFKD
  → compatibility-aware forms
```

Use the weakest transformation that meets the requirement.

------------------------------------------------------------------------

# 22. Unicode-Aware Regular Expressions

JavaScript regular expressions can operate with Unicode semantics
through the `u` flag.

Example:

``` js
/\u{1F600}/u
```

The `u` flag changes how the pattern is interpreted with respect to
Unicode code points and syntax.

Modern JavaScript also provides the `v` flag for enhanced Unicode-set
behavior.

This matters for advanced Unicode pattern matching.

However:

> Unicode-aware regex is not automatically grapheme-cluster-aware.

Regex and text segmentation are related but distinct problems.

------------------------------------------------------------------------

# 23. The `u` Flag

Without Unicode mode, some regex operations can behave according to
UTF-16 code-unit-oriented semantics.

With:

``` js
/u
```

the engine interprets relevant syntax using Unicode code point
semantics.

Example:

``` js
const re = /^\u{1F600}$/u;

console.log(re.test("😀"));
```

This is a code-point-oriented pattern.

That does not mean every regex task suddenly becomes "user character
aware."

Grapheme segmentation is a separate concern.

------------------------------------------------------------------------

# 24. The `v` Flag

The `v` regular-expression flag builds on Unicode-aware
regular-expression processing and enables more expressive Unicode set
operations and related behavior.

Modern Unicode-heavy validation may benefit from it, but support is
runtime/version dependent.

For production code, always verify compatibility with your target
environments before using newer regex features.

This is another example of the Chapter 01 rule:

``` text
language feature
+
runtime support
```

must both be considered.

------------------------------------------------------------------------

# 25. Unicode Property Escapes

JavaScript supports Unicode property escapes in Unicode-aware regular
expressions.

Example:

``` js
/^\p{Letter}+$/u
```

This can match sequences classified as Unicode letters.

Similarly:

``` js
/^\p{Number}+$/u
```

can target Unicode numeric characters.

This is much more robust for international text than assuming:

``` text
[A-Za-z]
```

represents all letters.

It does not.

------------------------------------------------------------------------

# 26. Why ASCII-Only Thinking Breaks

A simplistic validator:

``` js
/^[A-Za-z]+$/
```

only accepts a narrow Latin subset.

Real-world text may include:

``` text
Latin
Cyrillic
Greek
Arabic
Devanagari
Han
Kana
Hangul
and many others
```

International software therefore needs explicit text requirements.

Do not call an input "letters only" unless you have defined what counts
as a letter.

------------------------------------------------------------------------

# 27. Grapheme Clusters

A grapheme cluster is a text-segmentation unit designed to correspond
more closely to what users perceive as a character.

A single grapheme cluster can consist of multiple code points.

Examples can include:

``` text
base + combining mark
emoji + skin-tone modifier
emoji sequences joined by ZWJ
regional-indicator sequences
certain Indic text sequences
```

Therefore:

``` text
visible character count
```

can differ from:

``` text
code point count
```

and:

``` text
UTF-16 code unit count
```

This distinction is essential for UI limits and cursor movement.

------------------------------------------------------------------------

# 28. `Intl.Segmenter`

Modern JavaScript provides:

``` js
Intl.Segmenter
```

which can segment text using locale-sensitive segmentation strategies,
including grapheme segmentation.

Conceptual usage:

``` js
const segmenter = new Intl.Segmenter(undefined, {
  granularity: "grapheme"
});

const segments = [...segmenter.segment("😀")];

console.log(segments.length);
```

The key idea is:

``` text
String indexing
  → code units

codePointAt
  → code points

Intl.Segmenter
  → higher-level text segmentation
```

This is much closer to the problem users mean when they ask:

> "How many characters are in this message?"

------------------------------------------------------------------------

# 29. Why UI Character Limits Are Hard

Suppose a product requirement says:

> "Username maximum length: 20 characters."

What does "character" mean?

Possible interpretations:

``` text
20 UTF-16 code units
20 Unicode code points
20 grapheme clusters
20 normalized characters
20 bytes when encoded as UTF-8
```

These are not equivalent.

A production requirement must define the unit.

For user-facing input limits, grapheme-based counting is often closer to
user expectations, but product/domain requirements decide the actual
contract.

------------------------------------------------------------------------

# 30. Truncation Is Even More Dangerous

A naive implementation:

``` js
text.slice(0, 10)
```

counts UTF-16 code units.

This can split a surrogate pair.

It can also split a grapheme cluster.

Therefore:

``` text
naive slicing
```

can produce malformed or visually broken user text.

For user-visible truncation, choose an appropriate segmentation
strategy.

------------------------------------------------------------------------

# 31. Combining-Mark Truncation

Suppose:

``` text
base
+
combining mark
```

forms one visual character.

If you cut between them, the result may render unexpectedly.

Therefore truncation should consider grapheme boundaries when
user-visible text integrity matters.

This is especially important for:

-   usernames,
-   chat messages,
-   search labels,
-   headlines,
-   UI buttons,
-   notifications.

------------------------------------------------------------------------

# 32. Zero Width Joiner

Unicode emoji and other sequences may use:

``` text
ZERO WIDTH JOINER
```

to combine multiple code points into one displayed sequence.

This creates cases where:

``` text
many code points
```

can produce:

``` text
one apparent symbol
```

Therefore even code-point-aware counting is not sufficient for all
user-visible text operations.

Again:

``` text
code point ≠ grapheme cluster
```

------------------------------------------------------------------------

# 33. Regional Indicator Sequences

Flags may be represented using pairs of regional-indicator symbols.

The resulting visual flag behaves as a single user-perceived unit in
ordinary text segmentation even though multiple code points are
involved.

This is another example of why:

``` js
str.length
```

is not a universal "character counter."

------------------------------------------------------------------------

# 34. String Iteration

One of the most useful distinctions:

``` js
for (const item of text) {
  ...
}
```

uses the string iterator.

The string iterator processes Unicode code points rather than simply
exposing UTF-16 code units one by one.

Therefore:

``` js
[..."😀"].length
```

can differ from:

``` js
"😀".length
```

This is an important language-level improvement over raw indexing for
code-point-oriented iteration.

But it still does not guarantee grapheme-cluster iteration.

------------------------------------------------------------------------

# 35. Code Unit Iteration vs Code Point Iteration

Compare:

``` js
const text = "😀";

console.log(text.length);
console.log([...text].length);
```

Conceptually:

``` text
String.length
  → UTF-16 code units

spread/string iteration
  → Unicode code points
```

For `😀`:

``` text
length → 2
code points → 1
grapheme clusters → 1
```

This three-level comparison is worth memorizing.

------------------------------------------------------------------------

# 36. Escape Sequences

JavaScript supports several forms of expressing Unicode text.

Examples:

``` js
"\u0041"
```

for a BMP code unit/value.

Code point escapes:

``` js
"\u{1F600}"
```

when using the modern code-point escape syntax.

Surrogate code units can also be represented explicitly:

``` js
"\uD83D\uDE00"
```

which forms the same UTF-16 sequence as the grinning face.

The important point:

``` text
source representation
```

and:

``` text
runtime String value
```

are different layers.

------------------------------------------------------------------------

# 37. Lone Surrogates

A JavaScript string can contain lone UTF-16 surrogate code units.

For example:

``` js
const s = "\uD800";
```

This is a valid JavaScript String value.

However, a lone surrogate does not represent a Unicode scalar value on
its own.

This matters at encoding boundaries because:

``` text
JavaScript String
```

and:

``` text
well-formed Unicode text
```

are not always identical concepts.

Modern JavaScript provides APIs such as:

``` js
String.prototype.isWellFormed()
String.prototype.toWellFormed()
```

for dealing with lone-surrogate situations explicitly.

------------------------------------------------------------------------

# 38. Well-Formedness

A string containing unpaired surrogates can be valid as a JavaScript
String while not being well-formed Unicode scalar-value text.

This distinction matters when sending text to systems that require valid
Unicode encoding.

For example, before crossing certain serialization or encoding
boundaries, it can be appropriate to make well-formedness explicit.

The broader engineering rule:

> Validate the representation required by the boundary, not merely
> whether the JavaScript language accepts the value.

------------------------------------------------------------------------

# 39. Encoding Strings as UTF-8

When text crosses a byte-oriented boundary, JavaScript commonly uses
APIs such as:

``` js
new TextEncoder().encode(text);
```

This produces UTF-8 bytes.

The conceptual pipeline is:

``` text
JavaScript String
   ↓
Unicode text representation
   ↓
UTF-8 encoding
   ↓
Uint8Array bytes
```

The reverse direction can use:

``` js
new TextDecoder().decode(bytes);
```

This distinction becomes essential in networking, files, cryptography,
and binary protocols.

------------------------------------------------------------------------

# 40. String vs Byte Data

Do not confuse:

``` js
const text = "hello";
```

with:

``` js
const bytes = new Uint8Array(...);
```

The first is a String value.

The second is binary data.

They require an encoding boundary when converted.

Chapter 27 will connect this to typed arrays and binary data in depth.

------------------------------------------------------------------------

# 41. Strings Are Immutable

Strings are primitive immutable values.

Therefore:

``` js
let text = "hello";

text[0] = "H";
```

does not mutate the original string into:

``` text
Hello
```

Operations produce new string values.

For example:

``` js
text = text.toUpperCase();
```

reassigns the binding to a new String value.

The distinction remains:

``` text
string immutability
+
binding reassignment
```

------------------------------------------------------------------------

# 42. Concatenation

String concatenation can produce new string values:

``` js
const a = "hello";
const b = "world";

const result = a + " " + b;
```

The result is:

``` text
"hello world"
```

The exact internal allocation strategy is engine-specific.

Do not assume that every `+` operation necessarily creates an eagerly
materialized flat character array in memory.

Modern engines may use optimized representations internally.

This is an implementation detail.

------------------------------------------------------------------------

# 43. Template Literals

Template literals provide convenient string construction:

``` js
const name = "A";
const message = `Hello ${name}`;
```

They also support tagged templates.

Template literal syntax is language-level.

But interpolated expressions still evaluate using normal JavaScript
semantics.

This becomes important for:

-   escaping,
-   injection prevention,
-   localization,
-   structured output.

A template literal is not automatically a security boundary.

------------------------------------------------------------------------

# 44. Tagged Templates

Example:

``` js
function tag(strings, ...values) {
  return { strings, values };
}

const name = "A";

tag`Hello ${name}`;
```

Tagged templates let a function receive structured pieces of the
template rather than simply receiving one prebuilt string.

They can be useful for:

-   SQL-safe abstractions,
-   HTML escaping systems,
-   localization,
-   DSLs,
-   structured logging.

However, the tag implementation must enforce the intended security
semantics.

------------------------------------------------------------------------

# 45. Unicode and Case Conversion

Operations such as:

``` js
toUpperCase()
toLowerCase()
```

have Unicode-aware semantics, but "uppercase" and "lowercase" are not
universally one-to-one transformations.

Some languages have locale-specific casing rules.

For locale-sensitive behavior, the `Intl` APIs can matter.

Do not assume:

``` text
lowercase(string)
```

is a universal normalization step.

Case conversion, case folding, normalization, and locale-sensitive
comparison are separate concepts.

------------------------------------------------------------------------

# 46. Unicode Normalization Is Not Case Folding

These are different:

``` text
Normalization
Case conversion
Case folding
```

Normalization addresses canonical/compatibility representations.

Case conversion changes letter case.

Case folding is a comparison-oriented operation intended to reduce
certain case distinctions.

A robust internationalized search system may need to define all of these
separately.

------------------------------------------------------------------------

# 47. Locale-Sensitive Comparison

Do not assume:

``` js
a < b
```

implements human-language sorting.

For locale-aware comparison, use:

``` js
new Intl.Collator(locale)
```

or relevant `Intl` APIs.

Example:

``` js
const collator = new Intl.Collator("de");

collator.compare("ä", "z");
```

The result depends on locale and options.

Sorting for display is a different requirement from binary or code-point
ordering.

------------------------------------------------------------------------

# 48. Unicode Security --- Confusables

Unicode contains visually similar characters.

For example, characters from different scripts can appear similar enough
to confuse users.

This can affect:

-   usernames,
-   domain names,
-   account names,
-   authorization labels,
-   phishing detection,
-   search,
-   logs.

A string that looks like:

``` text
admin
```

may contain characters that are not the ASCII letters you expect.

Therefore:

> Visual similarity is not string equality.

Security-sensitive identifiers need explicit policies.

------------------------------------------------------------------------

# 49. Unicode Security --- Mixed Scripts

An application may choose to restrict certain identifiers to:

-   ASCII,
-   a defined Unicode subset,
-   a single script,
-   an allowed script combination.

There is no universally correct rule.

The correct rule depends on the security boundary.

For example:

``` text
login username
```

may have stricter requirements than:

``` text
free-form display name
```

This is a product/security design decision.

------------------------------------------------------------------------

# 50. Normalization at Security Boundaries

Suppose an authorization system compares identifiers.

If one component normalizes text and another does not, semantically
equivalent forms can produce inconsistencies.

Potential pipeline:

``` text
Input
 ↓
validation
 ↓
normalization policy
 ↓
canonical identity representation
 ↓
comparison/storage
```

The key principle is consistency.

Do not normalize in one service while leaving another service to compare
raw strings unless the system contract explicitly accounts for the
difference.

------------------------------------------------------------------------

# 51. Homoglyph Problems

Homoglyphs are visually similar characters.

A hostile identifier might exploit:

``` text
Latin
vs
Cyrillic
vs
Greek
```

or other visually related characters.

This can cause:

``` text
look-alike account names
look-alike domains
misleading logs
confusing approvals
```

Mitigations may involve:

-   allowed-character policies,
-   script restrictions,
-   confusable detection,
-   trusted display mechanisms,
-   canonicalization,
-   security review.

Do not attempt to solve the entire problem with a simple regex.

------------------------------------------------------------------------

# 52. Performance Considerations

String performance depends on:

-   length,
-   representation,
-   concatenation pattern,
-   slicing,
-   searching,
-   normalization,
-   regex complexity,
-   segmentation,
-   encoding/decoding,
-   engine implementation.

Unicode-aware operations can be more computationally expensive than
simple ASCII operations.

Normalization and grapheme segmentation are not free.

Therefore:

``` text
correctness requirement
+
measured workload
```

should determine optimization decisions.

Do not prematurely optimize away required Unicode correctness.

------------------------------------------------------------------------

# 53. Memory Considerations

A string's logical content can require significant memory.

Important factors include:

-   number of UTF-16 code units,
-   temporary strings,
-   concatenation,
-   normalization output,
-   encoded byte buffers,
-   retained substrings/slices depending on implementation strategy.

Again:

``` text
language String length
```

does not directly provide an exact physical memory cost in bytes on
every engine.

An engine may use internal optimizations such as compact
representations.

For memory investigations, use runtime-specific profiling tools rather
than assuming a fixed layout.

------------------------------------------------------------------------

# 54. Security Considerations

Unicode bugs often occur at boundaries:

``` text
input
→ validation
→ normalization
→ authorization
→ storage
→ display
→ logging
```

A secure design asks:

``` text
Which exact text forms are accepted?
Which representations are equivalent?
When is normalization applied?
What scripts are permitted?
What is the canonical identity form?
How is text encoded on output?
```

This is especially important for:

-   authentication,
-   identifiers,
-   URLs,
-   filenames,
-   policy names,
-   access-control subjects,
-   user-generated content.

------------------------------------------------------------------------

# 55. Production Usage --- Usernames

A production username system should explicitly define:

``` text
allowed characters
maximum grapheme count or other length unit
normalization policy
case policy
confusable policy
storage representation
comparison semantics
display rules
```

For example, a product may choose:

``` text
ASCII lowercase usernames only
```

while allowing arbitrary Unicode in:

``` text
displayName
```

This may be more secure and operationally predictable.

The correct choice is a product/security decision, not a JavaScript
requirement.

------------------------------------------------------------------------

# 56. Production Usage --- Search

Search often requires multiple layers:

``` text
raw input
 ↓
normalization
 ↓
case policy
 ↓
tokenization/segmentation
 ↓
index/search
```

Blindly lowercasing and stripping characters can produce incorrect
behavior in international text.

The system must define what "equivalent" means for its search domain.

------------------------------------------------------------------------

# 57. Production Usage --- Database Equality

Do not assume the database and JavaScript compare text identically.

Possible differences include:

-   Unicode normalization,
-   collation,
-   case sensitivity,
-   locale,
-   indexing rules.

Therefore a full production text comparison may involve:

``` text
JavaScript semantics
+
database collation
+
storage encoding
+
application normalization policy
```

This is especially important for unique constraints.

------------------------------------------------------------------------

# 58. Production Usage --- API Boundaries

JSON APIs carry Unicode text, but the actual transport is bytes.

A robust pipeline may be:

``` text
HTTP bytes
 ↓
UTF-8 decoding
 ↓
JSON parsing
 ↓
JavaScript String
 ↓
validation / normalization
 ↓
business logic
 ↓
serialization
 ↓
UTF-8 encoding
```

A bug can occur at any boundary.

This is why text bugs should be diagnosed by layer rather than by
staring only at the String value.

------------------------------------------------------------------------

# 59. Common Misconceptions

## Misconception 1 --- `length` means character count.

False.

It counts UTF-16 code units.

## Misconception 2 --- UTF-16 code units are Unicode characters.

False.

A surrogate pair represents one code point using two code units.

## Misconception 3 --- A code point is always one visible character.

False.

A grapheme cluster may consist of multiple code points.

## Misconception 4 --- UTF-8 and UTF-16 are different character sets.

False.

They are encodings of Unicode.

## Misconception 5 --- `charCodeAt` returns the Unicode character.

Not always.

It returns one UTF-16 code unit.

## Misconception 6 --- `codePointAt` gives user-perceived characters.

No.

It gives code points. Grapheme segmentation is a higher-level problem.

## Misconception 7 --- Visually identical strings are always `===`.

False.

Different code point sequences can render similarly.

## Misconception 8 --- Normalization is automatic.

False.

Explicit normalization is required when an application needs normalized
comparison/storage.

## Misconception 9 --- Unicode-aware regex solves all Unicode text problems.

False.

Regex, code-point handling, normalization, and grapheme segmentation
solve different problems.

------------------------------------------------------------------------

# 60. Common Mistakes

### Mistake: truncating with `slice()` for user-visible text

It can split surrogate pairs or grapheme clusters.

### Mistake: using `[A-Za-z]` for international letters

It excludes most scripts.

### Mistake: comparing raw strings when canonical equivalence matters

Normalize according to the domain contract.

### Mistake: lowercasing as a substitute for all text normalization

Case conversion and normalization are different operations.

### Mistake: assuming `String.length` is suitable for product limits

Define the correct measurement unit first.

### Mistake: assuming every JavaScript String is well-formed Unicode scalar-value text

Lone surrogates can exist.

------------------------------------------------------------------------

# 61. Comparison Table

  --------------------------------------------------------------------------------------------------
  Operation / Concept                               Unit or Semantics     Example
  ------------------------------------------------- --------------------- --------------------------
  `string.length`                                   UTF-16 code units     `"😀".length === 2`

  `string[i]`                                       UTF-16                can expose surrogate
                                                    code-unit-oriented    halves
                                                    indexing              

  `charCodeAt()`                                    one UTF-16 code unit  `charCodeAt(0)`

  `codePointAt()`                                   Unicode code point    `codePointAt(0)`

  String iterator                                   code-point-oriented   `[..."😀"].length === 1`
                                                    iteration             

  `Intl.Segmenter(..., {granularity:"grapheme"})`   grapheme segmentation user-oriented text units

  `normalize("NFC")`                                canonical             canonical composition
                                                    normalization         

  `normalize("NFD")`                                canonical             decomposed canonical form
                                                    decomposition         

  `normalize("NFKC")`                               compatibility         broader equivalence
                                                    normalization         

  UTF-8 encoding                                    bytes                 `TextEncoder`

  UTF-16 code unit                                  16 bits               JavaScript String model
  --------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 62. Execution Walkthrough --- Emoji

Consider:

``` js
const text = "😀";
```

At a conceptual level:

### Step 1

The source represents the Unicode code point:

``` text
U+1F600
```

### Step 2

The JavaScript String value uses UTF-16 code units.

### Step 3

Because the code point is outside the BMP, it is represented using a
surrogate pair.

### Step 4

Therefore:

``` js
text.length
```

returns:

``` text
2
```

### Step 5

The String iterator recognizes the surrogate pair as one Unicode code
point.

Therefore:

``` js
[...text].length
```

returns:

``` text
1
```

### Step 6

A grapheme segmenter can also treat the emoji as one grapheme cluster.

The three counts are therefore:

``` text
UTF-16 code units → 2
code points        → 1
grapheme clusters  → 1
```

------------------------------------------------------------------------

# 63. Execution Walkthrough --- Combining Sequence

Consider conceptually:

``` js
const a = "\u00E9";
const b = "\u0065\u0301";
```

### Step 1

`a` contains a precomposed character.

### Step 2

`b` contains a base code point plus combining mark.

### Step 3

They can render equivalently.

### Step 4

The String values are still different sequences.

Therefore:

``` js
a === b
```

is false.

### Step 5

Normalize both to NFC:

``` js
a.normalize("NFC")
b.normalize("NFC")
```

They can then produce equivalent canonical sequences.

The lesson:

``` text
visual equivalence
≠
raw String equality
```

------------------------------------------------------------------------

# 64. Execution Walkthrough --- Code Units vs Code Points

Consider:

``` js
const s = "😀";
```

Then:

``` js
s.length
```

counts:

``` text
2 code units
```

while:

``` js
s.codePointAt(0)
```

returns the full code point.

And:

``` js
[...s]
```

iterates one code point.

This is why the same String can be observed through multiple valid
abstraction levels.

------------------------------------------------------------------------

# 65. Implementation From Scratch --- Code Point Counter

Implement:

``` js
function countCodePoints(text) {
  // ...
}
```

Do not use:

``` js
[...text].length
```

for the first implementation.

Use the String iteration rules or explicit surrogate-pair logic.

Then compare:

``` text
UTF-16 code-unit count
code-point count
```

for a multilingual test corpus.

------------------------------------------------------------------------

# 66. Implementation --- Grapheme Counter

Implement:

``` js
function countGraphemes(text) {
  // ...
}
```

Use a standards-based segmentation strategy such as `Intl.Segmenter`
where supported.

Then test:

``` text
ASCII
emoji
combining marks
emoji sequences
Indic text
CJK
Arabic
```

The goal is to discover that "character count" is not one universal
measurement.

------------------------------------------------------------------------

# 67. Implementation Progression

### Guided

Create:

``` js
inspectText(text)
```

that reports:

``` text
UTF-16 code units
code points
grapheme clusters
```

### Partially Guided

Add:

``` text
normalization forms
well-formedness
UTF-8 byte length
```

### No Reference

Build a text-analysis utility suitable for debugging multilingual input.

### Edge-Case Hardened

Test:

``` text
empty string
ASCII
accented precomposed text
combining sequences
emoji
ZWJ emoji
regional indicators
lone surrogates
large strings
mixed scripts
```

### Production-Grade

Add:

-   explicit contracts,
-   normalization policy,
-   security checks,
-   performance measurement,
-   test corpus,
-   API documentation.

------------------------------------------------------------------------

# 68. Debugging Exercises

## Exercise 1

Why does this return 2?

``` js
"😀".length
```

## Exercise 2

Why can this return an incomplete-looking value?

``` js
"😀"[0]
```

## Exercise 3

What is the difference between:

``` js
"😀".charCodeAt(0)
```

and:

``` js
"😀".codePointAt(0)
```

## Exercise 4

Why can these look equal but compare false?

``` js
"\u00E9" === "\u0065\u0301"
```

## Exercise 5

Why might this truncate text incorrectly?

``` js
text.slice(0, 10)
```

## Exercise 6

Why is this validator incomplete?

``` js
/^[A-Za-z]+$/
```

## Exercise 7

Why should you not call `text.length` a user-visible character count?

------------------------------------------------------------------------

# 69. Code Review Exercise

Review:

``` js
function limitName(name) {
  return name.slice(0, 20);
}
```

A strong review asks:

-   What does "20" measure?
-   UTF-16 code units?
-   Code points?
-   Grapheme clusters?
-   Bytes?
-   Is the limit a storage requirement or a UI requirement?
-   Could truncation split a surrogate pair?
-   Could truncation split a grapheme cluster?
-   Is the string normalized?
-   Does the database apply additional collation or length constraints?

A better implementation cannot be chosen until the contract is defined.

This is a principal-level engineering lesson:

> **Do not optimize an implementation before defining the semantic unit
> the requirement actually refers to.**

------------------------------------------------------------------------

# 70. Interview Questions

## Junior

1.  What does `String.prototype.length` count?
2.  What is UTF-16?
3.  What is a surrogate pair?
4.  Why is `"😀".length` 2?
5.  What is the difference between `charCodeAt` and `codePointAt`?

## Mid-Level

6.  What is the difference between code units and code points?
7.  What is a grapheme cluster?
8.  Why can two visually identical strings compare unequal?
9.  What does `normalize()` do?
10. Why is `[A-Za-z]` insufficient for international text?

## Senior

11. Explain UTF-8 vs UTF-16.
12. Why is `slice()` dangerous for user-visible Unicode truncation?
13. Explain NFC vs NFD.
14. Explain NFKC/NFKD and why they can be dangerous if applied blindly.
15. How would you design a Unicode-aware username validator?
16. What is a lone surrogate?
17. How can Unicode issues create security vulnerabilities?

## Staff / Principal

18. Design a global username/identity system and specify its
    normalization policy.
19. How would you define "length" in an internationalized product
    requirement?
20. How would you ensure consistent Unicode handling across frontend,
    Node.js, database, and search infrastructure?
21. How would you investigate a bug where visually identical usernames
    are treated as different accounts?
22. How would you defend an ASCII-only identity policy while still
    supporting Unicode display names?
23. What are the trade-offs of normalization at input time vs comparison
    time vs storage time?

------------------------------------------------------------------------

# 71. Predict-the-Output Exercises

Predict before executing.

### 1

``` js
console.log("hello".length);
```

### 2

``` js
console.log("😀".length);
```

### 3

``` js
console.log([..."😀"].length);
```

### 4

``` js
console.log("😀".charCodeAt(0));
```

### 5

``` js
console.log("😀".codePointAt(0));
```

### 6

``` js
console.log("\u00E9" === "\u0065\u0301");
```

### 7

``` js
console.log(
  "\u00E9".normalize("NFC") ===
  "\u0065\u0301".normalize("NFC")
);
```

### 8

``` js
console.log("A".codePointAt(0));
```

### 9

``` js
console.log(
  String.fromCodePoint(0x1F600)
);
```

### 10

``` js
console.log(
  String.fromCharCode(0xD83D, 0xDE00)
);
```

### 11

``` js
console.log("\uD800".length);
```

### 12

``` js
console.log(
  typeof "hello"
);
```

------------------------------------------------------------------------

# 72. Mastery Exercises

## Exercise A --- Explain the Text Stack

Explain:

``` text
bytes
→ encoding
→ UTF-16 code units
→ code points
→ grapheme clusters
→ rendered glyphs
```

Make clear that these are different layers.

## Exercise B --- Build a Unicode Inspector

Create a utility that, for each input string, reports:

``` text
UTF-16 code-unit count
code-point count
grapheme count
UTF-8 byte count
normalization comparison
well-formedness
```

## Exercise C --- Design Username Semantics

Define:

``` text
allowed scripts
normalization
case sensitivity
length unit
confusable policy
storage form
comparison form
```

## Exercise D --- Design a Safe Truncation Function

Implement:

``` js
truncateGraphemes(text, max)
```

with predictable behavior around:

``` text
emoji
combining marks
ZWJ sequences
regional indicators
```

## Exercise E --- Security Review

Analyze a login system where:

``` text
usernames are arbitrary Unicode strings
```

List attack classes involving:

``` text
confusables
mixed scripts
normalization
case
logging/display
database uniqueness
cross-service comparison
```

------------------------------------------------------------------------

# 73. Production Text Decision Framework

For any text-processing requirement, ask:

``` text
1. Is this user-facing or machine-facing?
2. Do we care about bytes, code units, code points, or graphemes?
3. What encoding is used at the boundary?
4. Is normalization required?
5. Which normalization form?
6. Is case sensitivity required?
7. Is locale involved?
8. Is the text security-sensitive?
9. Are confusables relevant?
10. Are mixed scripts allowed?
11. What are the storage/database semantics?
12. What are the API interoperability requirements?
```

The right answer is a system contract.

------------------------------------------------------------------------

# 74. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.

## Builds Toward

-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.
-   Chapter 16 --- Property Keys and Enumeration.
-   Chapter 23 --- String APIs.
-   Chapter 27 --- Typed Arrays and Binary Data.
-   Chapter 28 --- Serialization.
-   Chapter 51 --- Browser APIs.
-   Chapter 55 --- Fetch and HTTP Networking.
-   Chapter 56 --- Browser Security.
-   Chapter 57 --- JavaScript Security Engineering.
-   Chapter 81 --- Database Integration.
-   Chapter 82 --- API Architecture.
-   Chapter 94 --- Compatibility Engineering.

## Related Concepts

``` text
Unicode
UTF-8
UTF-16
code units
code points
grapheme clusters
normalization
regular expressions
internationalization
encoding
security
collation
text boundaries
```

## Concepts Revisited Later

-   String methods and search.
-   Regular expressions.
-   Serialization.
-   Binary encoding.
-   HTTP.
-   Security and canonicalization.
-   Database collation.
-   Performance/memory of large strings.

## Why This Chapter Matters Later

Text is one of the most deceptively complex data types in software.

A system that treats:

``` text
UTF-16 code units
```

as:

``` text
user-perceived characters
```

can produce broken UI, incorrect limits, corrupted identifiers,
inconsistent equality, and security vulnerabilities.

Understanding the text stack early prevents these bugs from being
repeated throughout the rest of the curriculum.

------------------------------------------------------------------------

# 75. Key Takeaways

1.  JavaScript String values are sequences of UTF-16 code units.
2.  `string.length` counts UTF-16 code units, not user-perceived
    characters.
3.  UTF-8 and UTF-16 are Unicode encodings, not different character
    sets.
4.  Unicode code points and UTF-16 code units are different concepts.
5.  Code points outside the BMP can require surrogate pairs.
6.  `charCodeAt` exposes UTF-16 code units.
7.  `codePointAt` exposes a Unicode code point.
8.  String iteration is code-point-oriented.
9.  Code points still do not equal grapheme clusters.
10. A single grapheme cluster may contain multiple code points.
11. Combining marks create visually equivalent sequences with
    potentially different underlying code points.
12. JavaScript does not automatically normalize strings before equality
    comparison.
13. `normalize()` provides NFC, NFD, NFKC, and NFKD.
14. Compatibility normalization can erase distinctions and should be
    used deliberately.
15. Unicode-aware regex with `u`/modern `v` support is not the same as
    grapheme segmentation.
16. `Intl.Segmenter` can provide higher-level text segmentation.
17. JavaScript Strings can contain lone surrogates.
18. Text and byte data are different abstractions and require an
    encoding boundary.
19. Unicode bugs frequently appear at validation, normalization,
    storage, comparison, display, and serialization boundaries.
20. "Character count" is a product requirement that must define its
    measurement unit.
21. Security-sensitive identifiers need explicit policies for
    normalization, scripts, case, and confusables.
22. Correct Unicode handling is a systems-design concern, not merely a
    string-method concern.

------------------------------------------------------------------------

# 76. Completion Criteria

### Understand

-   Explain UTF-16 code units.
-   Explain Unicode code points.
-   Explain grapheme clusters.
-   Explain normalization.

### Explain

-   Explain why `"😀".length === 2`.
-   Explain why visually identical strings can compare unequal.
-   Explain UTF-8 vs UTF-16.
-   Explain surrogate pairs and combining marks.

### Predict

-   Predict `length`, indexing, `charCodeAt`, `codePointAt`, iteration,
    and normalization behavior.

### Implement

-   Implement code-point and grapheme-aware utilities.
-   Build a Unicode inspection tool.

### Debug

-   Diagnose incorrect truncation.
-   Diagnose broken Unicode indexing.
-   Diagnose normalization inconsistencies.
-   Diagnose cross-service string comparison problems.

### Apply

-   Define the correct text unit for user-facing and machine-facing
    requirements.

### Compare

-   Code units vs code points.
-   Code points vs grapheme clusters.
-   NFC vs NFD.
-   NFC/NFD vs NFKC/NFKD.
-   UTF-8 vs UTF-16.
-   raw comparison vs normalized comparison.

### Defend

-   Defend a Unicode normalization strategy.
-   Defend a username policy.
-   Defend a grapheme-based UI limit.
-   Defend an ASCII-only security-sensitive identifier policy where
    appropriate.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 05 --- Variables, Declarations,
and Assignment