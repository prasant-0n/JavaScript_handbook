# Chapter 128 — Internationalization (`Intl`) Deep Dive

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Master ECMAScript internationalization through the `Intl` APIs: locale negotiation, Unicode-aware formatting, numbering systems, currencies, dates/time zones, plural rules, lists, relative time, display names, collation, segmentation, calendars, locale-sensitive data, performance, caching, and production correctness.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript/ECMA-402 Specialist · Internationalization Engineer · Frontend Platform Engineer · Node.js Engineer · API Architect · Localization/Globalization Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Internationalization is not string translation. It is correct behavior for language, locale, numbering, dates, plural categories, ordering, segmentation, calendars, and cultural conventions.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain what ECMA-402 is
[ ] distinguish ECMAScript from ECMA-402
[ ] explain why Intl exists
[ ] explain locale identifiers conceptually
[ ] explain BCP 47 language tags
[ ] distinguish language, region, script, and extension subtags
[ ] explain Unicode locale extensions
[ ] explain locale negotiation
[ ] explain lookup matching vs best-fit concepts
[ ] explain Intl.Locale
[ ] explain resolved options
[ ] explain NumberFormat
[ ] format decimal values
[ ] format currencies
[ ] format percentages
[ ] format units
[ ] format compact notation
[ ] use formatToParts
[ ] understand sign/display options
[ ] understand numbering systems
[ ] understand currency-specific formatting
[ ] explain DateTimeFormat
[ ] format dates/times by locale
[ ] reason about time zones
[ ] distinguish instant from civil/local date-time concepts
[ ] explain calendar options
[ ] explain era/year/month/day formatting
[ ] explain DST-sensitive formatting behavior
[ ] explain RelativeTimeFormat
[ ] explain ListFormat
[ ] explain PluralRules
[ ] distinguish cardinal and ordinal pluralization
[ ] explain DisplayNames
[ ] explain Collator
[ ] explain locale-aware comparison/sorting
[ ] explain Segmenter
[ ] explain grapheme/word/sentence segmentation
[ ] explain locale-aware vs code-unit/string comparisons
[ ] reason about Unicode normalization vs locale formatting
[ ] explain supported locales
[ ] explain Intl constructors and options
[ ] explain formatter lifecycle and caching
[ ] identify performance pitfalls
[ ] identify hydration/localization pitfalls
[ ] design internationalized APIs
[ ] test locale-sensitive behavior
[ ] distinguish deterministic tests from environment-dependent output
[ ] build a localization foundation for production JavaScript systems
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 04 — Strings / Unicode / Text Semantics
Chapter 07 — Type Conversion / Coercion / Equality
Chapter 22 — Arrays
Chapter 23 — String APIs
Chapter 27 — Typed Arrays / Binary Data
Chapter 28 — JSON / Serialization
Chapter 31 — Async Fundamentals
Chapter 49 — DOM Architecture
Chapter 51 — Browser APIs
Chapter 55 — Fetch / HTTP Networking
Chapter 58 — Node.js Architecture
Chapter 64 — ES Modules
Chapter 79 — API Design
Chapter 81 — Database Integration
Chapter 83 — Observability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 90 — Modern ECMAScript Features
Chapter 92 — Temporal
Chapter 123 — Grammar / Parsing
Chapter 124 — Completion / Reference Records
Chapter 126 — Module Linking
Chapter 127 — SharedArrayBuffer / Atomics
```

Useful supporting knowledge:

```text
Unicode
BCP 47
CLDR
ICU
time zones
calendar systems
localization
internationalization
```

---

# 3. What Is Internationalization?

Internationalization (`i18n`) is designing software so that behavior and presentation can adapt to different languages, regions, writing systems, calendars, numbering systems, and cultural conventions.

Examples:

```text
en-US
en-GB
fr-FR
de-DE
ja-JP
ar-SA
hi-IN
th-TH
```

Internationalization is broader than:

```text
translate("hello")
```

It includes:

```text
number formatting
currency
dates
time
time zones
plural rules
collation
segmentation
lists
relative time
display names
calendars
numbering systems
```

---

# 4. Why `Intl` Exists

Without internationalization APIs, application developers would need to manually implement rules for:

```text
decimal separators
grouping
currency placement
plural categories
weekday/month names
sorting
grapheme boundaries
calendar display
time-zone formatting
```

Those rules are complex and culturally variable.

ECMA-402 provides a standardized JavaScript API layer for internationalization behavior.

---

# 5. ECMAScript vs ECMA-402

Keep these separate.

### ECMAScript

Defines the JavaScript language and its core built-ins.

### ECMA-402

Defines internationalization APIs layered around ECMAScript.

The practical architecture is:

```text
ECMAScript language
        +
ECMA-402 Intl
        +
Unicode / locale data
        +
runtime implementation
```

Do not assume that every detail of locale data is encoded directly in the ECMAScript language specification.

---

# 6. Mental Model

Use:

```text
application data
      ↓
locale context
      ↓
Intl constructor
      ↓
locale/options resolution
      ↓
locale-sensitive rules/data
      ↓
formatted or classified result
```

For example:

```text
1234567.89
   ↓
locale = de-DE
   ↓
NumberFormat
   ↓
"1.234.567,89"
```

The input number did not change.

The presentation did.

---

# 7. Locale Is Not Language

These are different concepts:

```text
language
region
script
locale
```

For example:

```text
en
en-US
en-GB
```

share a language but can differ in:

```text
date format
currency
number formatting
spelling conventions
week/day conventions
```

Do not model:

```text
locale = language
```

as a complete abstraction.

---

# 8. BCP 47 Language Tags

Examples:

```text
en
en-US
en-GB
sr-Cyrl
zh-Hans-CN
ar-EG
```

These can encode:

```text
language
script
region
variants
extensions
private use
```

Locale-aware JavaScript APIs use language-tag concepts as input.

---

# 9. `Intl.Locale`

`Intl.Locale` provides a structured locale representation.

Example:

```js
const locale = new Intl.Locale("hi-IN");

console.log(locale.language);
console.log(locale.region);
console.log(locale.baseName);
```

Use it when application code needs to reason about locale identity rather than repeatedly manipulating raw strings.

---

# 10. Locale Negotiation

Applications often receive:

```text
supported locales
+
requested locales
```

and need to determine:

```text
which supported locale should be used
```

Example:

```js
const supported = ["en-US", "en-GB", "fr-FR"];

const requested = ["fr-CA", "en-AU"];
```

The system may choose the best supported match according to the relevant locale-matching rules.

Do not implement locale matching by:

```js
requested[0].slice(0, 2)
```

and assume that is sufficient.

---

# 11. Supported Locales

Use:

```js
Intl.NumberFormat.supportedLocalesOf(["fr-FR", "de-DE"]);
```

to inspect which requested locales are supported by the runtime for that service.

The result can differ by runtime build/data availability.

Production systems should treat:

```text
locale data availability
```

as an environment consideration.

---

# 12. `resolvedOptions()`

Example:

```js
const formatter = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD"
});

console.log(formatter.resolvedOptions());
```

This is useful for understanding:

```text
effective locale
numbering system
style
currency
digit settings
other resolved options
```

It is a diagnostic tool, not a complete dump of all underlying locale data.

---

# 13. Number Formatting

Basic:

```js
new Intl.NumberFormat("en-US").format(1234567.89);
```

Different locale:

```js
new Intl.NumberFormat("de-DE").format(1234567.89);
```

The output can differ in:

```text
group separator
decimal separator
digit shapes
```

This is why string concatenation is not a formatting strategy.

---

# 14. Currency Formatting

```js
const formatter = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD"
});

formatter.format(1234.5);
```

Currency formatting can encode:

```text
currency symbol
placement
spacing
fraction digits
regional conventions
```

Do not manually implement:

```text
"$" + amount
```

for localized UI.

---

# 15. Currency Is Not Just a Symbol

These are not equivalent:

```text
USD
$
US$
```

Currency metadata influences:

```text
display
fraction digits
sign conventions
currency spacing
```

Also distinguish:

```text
currency code
currency amount
display formatting
```

from:

```text
financial storage/rounding rules.
```

`Intl.NumberFormat` formats a numeric value.

It is not an accounting ledger.

---

# 16. Financial Correctness Boundary

Do not use localized formatting as the monetary source of truth.

Store money using an explicit model such as:

```text
integer minor units
decimal representation
arbitrary-precision decimal strategy
```

depending on domain requirements.

Then:

```text
business amount
→ exact monetary logic
→ Intl formatting
```

Never:

```text
formatted string
→ business calculation
```

---

# 17. Percentage Formatting

```js
new Intl.NumberFormat("en-US", {
  style: "percent"
}).format(0.125);
```

The formatter interprets the value according to percent formatting semantics.

Do not accidentally assume:

```text
0.125 means "0.125%"
```

when the formatter interprets it as:

```text
12.5%
```

Understand the data contract before formatting.

---

# 18. Unit Formatting

`Intl.NumberFormat` can format units.

Example:

```js
new Intl.NumberFormat("en-US", {
  style: "unit",
  unit: "kilometer",
  unitDisplay: "long"
}).format(42);
```

This is useful for:

```text
distance
duration quantities
temperature
digital sizes where supported
other standardized units
```

Do not use localized unit strings as machine-readable identifiers.

---

# 19. Compact Notation

Example:

```js
new Intl.NumberFormat("en-US", {
  notation: "compact"
}).format(1200000);
```

May produce a compact representation such as:

```text
1.2M
```

The exact output is locale-sensitive.

Do not write tests that assume every locale uses:

```text
M
```

or:

```text
K
```

in exactly the same form.

---

# 20. Sign Display

Number formatting can control how signs are presented.

Examples of concerns include:

```text
positive
negative
zero
negative zero
explicit plus
accounting-style display
```

A production financial UI should define:

```text
semantic sign policy
```

before selecting formatting options.

---

# 21. `formatToParts`

Use:

```js
const parts = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD"
}).formatToParts(1234.5);
```

This exposes structured parts such as conceptual:

```text
currency
integer
group
decimal
fraction
literal
```

This is useful when UI needs:

```text
different styling
accessibility markup
structured rendering
```

Do not parse formatted strings with regular expressions when structured parts are available.

---

# 22. `formatRange`

Where supported by the relevant formatter:

```js
formatter.formatRange(start, end);
```

can format ranges according to locale-sensitive rules.

This is often superior to:

```text
format(start) + " – " + format(end)
```

because locales can have different range conventions and repeated-field suppression rules.

---

# 23. Numbering Systems

A locale can use numbering systems beyond:

```text
Latin digits
```

Examples include:

```text
Arabic-derived digits
Devanagari digits
other Unicode-supported numbering systems
```

Formatting is therefore not equivalent to:

```text
String(number)
```

---

# 24. Decimal Formatting vs Parsing

`Intl.NumberFormat` formats numbers.

It is not a general localized-number parser.

Do not assume:

```js
new Intl.NumberFormat("de-DE").parse("1.234,56")
```

exists as the mirror operation.

A production localized input field requires a separate parsing/validation strategy.

---

# 25. Localized Input Design

For user-entered numbers:

```text
input string
→ locale-aware parser strategy
→ normalized numeric representation
→ validation
→ business logic
```

For output:

```text
business number
→ Intl
→ localized string
```

Do not make:

```text
localized output
```

the canonical stored value.

---

# 26. Date and Time Formatting

Use:

```js
new Intl.DateTimeFormat("en-US", {
  dateStyle: "medium",
  timeStyle: "short"
}).format(date);
```

Locale controls:

```text
ordering
month/day names
12/24-hour preferences
punctuation
digit formatting
```

---

# 27. Time Zone Formatting

A `Date` represents a point in time.

Formatting can display that instant in a selected time zone:

```js
new Intl.DateTimeFormat("en-US", {
  timeZone: "Asia/Kolkata",
  dateStyle: "full",
  timeStyle: "long"
}).format(new Date());
```

The key distinction:

```text
instant
vs
time-zone-specific presentation
```

---

# 28. Never Treat Local Time as a Global Truth

Example:

```text
2026-09-11 09:00
```

without zone information is ambiguous.

Define whether it means:

```text
UTC instant
local civil time
tenant-local time
user-local time
specific named-zone time
```

before storing/transmitting it.

`Intl.DateTimeFormat` formats a temporal value.

It does not determine your application's business time model.

---

# 29. DST and Political Time Zones

Named time zones can have:

```text
daylight-saving changes
historical rule changes
political changes
offset transitions
```

Therefore:

```text
timeZone = "America/New_York"
```

is not equivalent to:

```text
UTC-5 forever
```

Avoid hardcoding offsets when a named time zone is the actual requirement.

---

# 30. DateTimeFormat and Calendars

Formatting can specify calendar systems.

Examples conceptually include:

```text
gregory
buddhist
japanese
islamic variants
```

The calendar affects:

```text
displayed year/month/day
era
```

Do not assume:

```text
year = one universal culturally independent field
```

---

# 31. Calendar Is Not Time Zone

Keep separate:

```text
calendar
time zone
locale
```

For example:

```text
locale = ar-SA
calendar = a specific Islamic calendar
timeZone = Asia/Riyadh
```

These represent different dimensions of temporal presentation.

---

# 32. DateTimeFormat Options

Important categories include:

```text
dateStyle
timeStyle
weekday
era
year
month
day
hour
minute
second
fractionalSecondDigits
timeZone
timeZoneName
hourCycle
calendar
numberingSystem
```

Use style-based options when the UI needs standard date/time presentation.

Use component options when more exact field selection is required.

---

# 33. `formatToParts` for Date/Time

Example:

```js
const parts = new Intl.DateTimeFormat("en-US", {
  dateStyle: "full"
}).formatToParts(new Date());
```

Structured parts are useful for:

```text
custom UI
accessible composition
highlighting
structured export
```

Do not depend on the exact part order across locales unless the contract explicitly expects it.

---

# 34. Relative Time Formatting

```js
new Intl.RelativeTimeFormat("en", {
  numeric: "auto"
}).format(-1, "day");
```

This can produce locale-aware forms such as:

```text
yesterday
```

rather than:

```text
1 day ago
```

depending on options.

---

# 35. Relative Time Is Presentation

The formatter does not decide:

```text
whether something is "yesterday"
```

Your application computes the semantic difference.

Then:

```text
difference
→ RelativeTimeFormat
→ localized display
```

Keep business time logic separate from presentation.

---

# 36. List Formatting

```js
new Intl.ListFormat("en", {
  style: "long",
  type: "conjunction"
}).format(["A", "B", "C"]);
```

The result can vary by locale.

This is superior to manually implementing:

```text
A, B, and C
```

because list conventions vary.

---

# 37. Conjunction vs Disjunction

List formatting supports conceptual types such as:

```text
conjunction
disjunction
```

For example:

```text
A, B, and C
```

versus:

```text
A, B, or C
```

The language's punctuation and joining conventions remain locale-sensitive.

---

# 38. PluralRules

`Intl.PluralRules` classifies a number into locale-dependent plural categories.

Example:

```js
const rules = new Intl.PluralRules("en");

rules.select(1);
rules.select(2);
```

Potential categories include:

```text
one
other
zero
two
few
many
```

Not every locale uses the same categories.

---

# 39. Cardinal vs Ordinal

Cardinal:

```text
1 item
2 items
```

Ordinal:

```text
1st
2nd
3rd
```

Use:

```js
new Intl.PluralRules("en", {
  type: "ordinal"
});
```

when the semantic question is ordinal.

Do not reuse:

```text
cardinal rules
```

for:

```text
ordinal wording.
```

---

# 40. Plural Categories Are Locale Semantics

English:

```text
1 → one
2 → other
```

Another language may have much richer distinctions.

Therefore this is unsafe:

```js
count === 1 ? "item" : "items";
```

for a general localization system.

PluralRules lets application logic branch on locale semantics.

---

# 41. Message Construction

A good architecture can be:

```text
locale
→ plural category
→ message selection
→ translated message
→ formatted substitutions
```

Do not assume pluralization alone translates a sentence.

Internationalization requires:

```text
linguistic message design
+
locale-specific rules
```

---

# 42. DisplayNames

`Intl.DisplayNames` can provide localized human-readable names for standardized codes such as:

```text
language
region
currency
calendar
date-time fields
```

Conceptually:

```js
new Intl.DisplayNames("en", {
  type: "region"
}).of("IN");
```

This is useful for:

```text
country selectors
language selectors
currency selectors
metadata UI
```

---

# 43. Codes vs Display Names

Keep:

```text
"IN"
```

as machine-readable data.

Display:

```text
localized name
```

only at the presentation boundary.

Do not store:

```text
"India"
```

as the canonical region identifier.

---

# 44. Collator

String comparison is not equivalent to locale-aware sorting.

Example:

```js
const collator = new Intl.Collator("de");

["ä", "a", "z"].sort(collator.compare);
```

Locale-specific collation rules can differ substantially from:

```js
a < b
```

or:

```js
a.localeCompare(b)
```

with implicit/default locale.

---

# 45. `Intl.Collator` vs `localeCompare`

`String.prototype.localeCompare()` can provide locale-aware comparison.

`Intl.Collator` is useful when:

```text
many comparisons
reusable configuration
explicit locale/options
performance-sensitive sorting
```

are needed.

Create one configured collator rather than constructing a new one inside every comparison.

---

# 46. Collation Is Not Equality

Two strings can:

```text
sort equivalently
```

under a chosen collation while not being:

```text
strictly equal
```

Therefore do not use locale collation as an authentication/identifier equality mechanism.

For security identifiers, prefer:

```text
explicit canonicalization rules
```

defined for the domain.

---

# 47. Search vs Sort

User-facing search can require:

```text
collation
case sensitivity
accent sensitivity
normalization
tokenization
```

Database search and full-text search have their own semantics.

Do not assume:

```text
Intl.Collator
```

is a complete search engine.

It solves locale-aware comparison, not indexing, stemming, ranking, or typo correction.

---

# 48. Segmenter

`Intl.Segmenter` can segment text according to locale and granularity.

Examples:

```text
grapheme
word
sentence
```

Conceptually:

```js
const segmenter = new Intl.Segmenter("en", {
  granularity: "word"
});
```

This is useful when:

```text
"character"
```

must not be equated with:

```text
UTF-16 code unit
```

---

# 49. Grapheme Awareness

A user-perceived character may consist of:

```text
multiple Unicode code points
multiple UTF-16 code units
combining marks
emoji sequences
```

Therefore:

```js
"😀".length
```

does not answer:

```text
How many user-perceived characters?
```

`Intl.Segmenter` can provide a better abstraction for user-facing segmentation.

---

# 50. Word Segmentation Is Locale-Sensitive

Some languages do not separate words with spaces in the same way as English.

Therefore:

```text
string.split(" ")
```

is not a universal word-segmentation algorithm.

Use:

```text
Intl.Segmenter
```

when locale-aware word boundaries are required.

---

# 51. Sentence Segmentation

Sentence boundaries can depend on:

```text
punctuation
abbreviations
language conventions
```

Do not implement general sentence splitting with:

```js
text.split(".")
```

for internationalized text.

---

# 52. Unicode Normalization vs Intl

These are separate concepts.

### Normalization

Transforms equivalent Unicode representations into a canonical/selected normalization form.

Example:

```js
text.normalize("NFC");
```

### Internationalized services

Apply:

```text
locale-sensitive behavior
```

such as:

```text
collation
formatting
segmentation
plural rules
```

Normalization does not replace locale rules.

---

# 53. Case Conversion vs Case Mapping

Do not assume:

```js
text.toUpperCase()
```

means:

```text
locale-correct display capitalization
```

There are language-sensitive casing cases.

ECMAScript also provides locale-aware methods such as:

```js
toLocaleUpperCase()
toLocaleLowerCase()
```

But casing is still not equivalent to:

```text
title case
grammatical capitalization
human editorial rules
```

---

# 54. Locale-Sensitive Case Operations

Example:

```js
"i".toLocaleUpperCase("tr");
```

can differ from default assumptions because Turkish casing rules differ from English-like expectations.

This is a classic reminder:

```text
case mapping is not universally language-neutral
```

---

# 55. Intl Constructor Lifecycle

Common constructors include:

```text
Intl.Collator
Intl.DateTimeFormat
Intl.DisplayNames
Intl.ListFormat
Intl.NumberFormat
Intl.PluralRules
Intl.RelativeTimeFormat
Intl.Segmenter
Intl.Locale
```

Many of these objects are configured for repeated formatting/comparison work.

---

# 56. Formatter Reuse

Avoid:

```js
items.map(item =>
  new Intl.NumberFormat(locale, options).format(item.value)
);
```

when the formatter configuration is constant.

Prefer:

```js
const formatter = new Intl.NumberFormat(locale, options);

items.map(item => formatter.format(item.value));
```

Why?

```text
construction/configuration
```

can be more expensive than:

```text
repeated formatting using one configured instance.
```

Benchmark actual workloads.

---

# 57. Caching Formatters

Production applications can cache formatter instances by:

```text
locale
service
options
```

For example:

```text
("en-US", NumberFormat, currency=USD)
```

Do not create an unbounded formatter cache keyed by arbitrary user-generated option objects.

Use:

```text
bounded cache
canonical cache key
eviction
```

where necessary.

---

# 58. Server-Side Rendering and Hydration

Locale-sensitive output can cause:

```text
server output ≠ browser output
```

because server/client environments may differ in:

```text
locale
time zone
Intl data
default options
```

This can cause hydration mismatches.

Production architecture should establish:

```text
request locale
display time zone
```

explicitly rather than relying on host defaults.

---

# 59. Deterministic Server Rendering

For stable server HTML:

```text
explicit locale
explicit time zone
explicit formatting options
```

are safer than:

```js
new Intl.DateTimeFormat().format(date);
```

because the latter depends on environment defaults.

---

# 60. User Locale vs Tenant Locale

Applications may have multiple relevant locale dimensions:

```text
browser/user locale
tenant locale
organization locale
document locale
billing locale
content locale
```

Do not assume there is exactly one global locale.

Define:

```text
which locale owns each presentation decision.
```

---

# 61. Locale Context Propagation

For a request, carry explicit context such as:

```text
locale
timeZone
currency
calendar
```

where relevant.

Then:

```text
request
→ service
→ formatting boundary
```

can use consistent presentation rules.

Avoid hidden globals such as:

```js
currentLocale = ...
```

in concurrent server systems.

---

# 62. API Design for Internationalization

A useful API contract may contain:

```json
{
  "locale": "en-IN",
  "timeZone": "Asia/Kolkata"
}
```

But do not blindly trust client-provided locale as an authorization or tenant configuration source.

Distinguish:

```text
requested presentation preference
```

from:

```text
server-authoritative business setting.
```

---

# 63. HTTP and Content Negotiation

Browsers and clients can communicate language preferences through headers such as:

```text
Accept-Language
```

An API can use that as one input to locale negotiation.

But production systems may also use:

```text
user profile
tenant settings
URL
cookie
application preferences
```

Define precedence explicitly.

---

# 64. Locale Precedence Example

A production policy might be:

```text
explicit user setting
→ tenant default
→ request preference
→ application default
```

The exact hierarchy is a product decision.

Document it.

Do not let infrastructure accidentally define product behavior.

---

# 65. Date Storage vs Date Display

Store:

```text
canonical instant / temporal representation
```

then display:

```text
user/tenant locale
+
user/tenant time zone
```

Do not store:

```text
"11/09/2026"
```

as your canonical timestamp unless the domain explicitly means a civil date.

---

# 66. Civil Date vs Instant

These are different domain concepts:

```text
Instant:
one point on the global timeline

Civil date:
calendar date without an instant

Local date-time:
calendar date + local clock time, possibly without zone

Zoned date-time:
local date/time associated with a zone
```

Intl formats temporal values.

Your domain model must decide which concept you actually have.

---

# 67. Temporal Connection

Chapter 92 covered Temporal.

Use Temporal-style domain thinking:

```text
Instant
PlainDate
PlainTime
PlainDateTime
ZonedDateTime
Duration
```

then use:

```text
Intl
```

for localized presentation.

This separation prevents:

```text
formatting concerns
```

from becoming:

```text
time arithmetic concerns.
```

---

# 68. Calendar and Business Logic

Never assume:

```text
calendar display
```

automatically changes:

```text
business calendar rules
```

For example:

```text
work week
holiday schedule
billing period
fiscal year
```

may require domain-specific configuration.

Intl is not a complete enterprise calendar engine.

---

# 69. Week Information

Applications often need:

```text
first day of week
minimal days
weekend
```

These are locale-sensitive/business-sensitive concerns.

Do not hardcode:

```text
Monday
```

or:

```text
Sunday
```

as a universal cultural rule.

Use appropriate locale data/features where available and separate:

```text
display convention
```

from:

```text
business calendar policy.
```

---

# 70. Time Zone Identifiers

Use canonical named time zones such as:

```text
Asia/Kolkata
Europe/London
America/New_York
UTC
```

Avoid arbitrary abbreviations:

```text
IST
CST
EST
```

because abbreviations can be ambiguous.

Current ECMAScript time-zone semantics integrate with named time-zone data in time-zone-aware implementations, including IANA database identifiers under the ECMA-402 rules. citeturn278112search1turn278112search3

---

# 71. Locale Data Is Data

Locale-sensitive formatting depends on locale data.

This means runtime builds can differ in:

```text
supported locales
available time zones
data version
formatting details
```

Therefore exact snapshots can change when:

```text
runtime
ICU/data
locale data
```

changes.

Treat locale output as a version-sensitive integration boundary when exact textual snapshots matter.

---

# 72. CLDR / ICU Mental Model

In common JavaScript implementations, locale-sensitive behavior is supported by large internationalization data sets and libraries such as:

```text
CLDR
ICU
```

The ECMAScript/ECMA-402 API defines the observable contract.

Underlying data/library details remain implementation concerns.

Do not couple application correctness to undocumented ICU internals.

---

# 73. `Intl` and Deterministic Tests

Bad test:

```js
expect(new Intl.DateTimeFormat().format(date))
  .toBe("9/11/2026");
```

This silently depends on:

```text
machine locale
machine time zone
runtime data
```

Better:

```js
const formatter = new Intl.DateTimeFormat("en-US", {
  timeZone: "UTC",
  year: "numeric",
  month: "2-digit",
  day: "2-digit"
});
```

Then the test controls the key environment dimensions.

---

# 74. Exact-Output vs Semantic Tests

Use exact-output snapshots when:

```text
the exact presentation contract matters
```

Use semantic assertions when:

```text
locale-specific variation is expected
```

For example, verify:

```text
contains currency part
```

instead of:

```text
exact string
```

when many locale variants are intentionally valid.

---

# 75. Accessibility Considerations

Localized formatting affects:

```text
screen-reader output
number readability
date ambiguity
currency interpretation
directionality
```

Avoid UI that relies only on visual punctuation.

For important dates/numbers, provide semantic context.

---

# 76. RTL and Bidirectional Text

Internationalized UIs may involve:

```text
Arabic
Hebrew
mixed LTR/RTL content
```

Formatting and rendering must consider:

```text
directionality
bidirectional isolation
embedding user-generated text
```

Do not assume:

```text
English punctuation layout
```

will remain visually correct in RTL interfaces.

---

# 77. Currency and RTL

Currency symbols, numbers, and text can interact with bidirectional rendering.

Use:

```text
Intl formatting
+
correct UI directionality
```

rather than constructing strings manually.

For mixed-direction content, inspect rendered output and assistive technology behavior.

---

# 78. Security Considerations

Internationalization can intersect with security through:

```text
confusable characters
locale-sensitive comparisons
Unicode normalization
identifier display
phishing-resistant UI
log analysis
canonicalization
```

Do not use localized display strings as:

```text
security identifiers
```

---

# 79. Confusables

Two visually similar characters can be different Unicode code points.

Examples conceptually:

```text
Latin a
vs
Cyrillic а
```

Internationalized systems should distinguish:

```text
display similarity
```

from:

```text
code-point identity
```

For security-sensitive names:

```text
canonicalization
allowlists
restricted character sets
explicit comparisons
```

may be more appropriate than user-friendly locale display.

---

# 80. Locale-Sensitive Authorization Mistake

Never do:

```js
if (username.toLocaleLowerCase(locale) === "admin") {
  allow();
}
```

for security authorization.

Authentication/authorization identifiers need:

```text
well-defined canonical comparison rules
```

not display-oriented locale transformations.

---

# 81. Localization Injection

Localized messages may contain user-controlled substitutions.

Example:

```text
"Welcome, {name}"
```

The message itself may be trusted while:

```text
name
```

is untrusted.

Keep:

```text
message selection
```

separate from:

```text
HTML rendering
```

and apply appropriate output encoding.

---

# 82. Performance Considerations

Potential costs include:

```text
formatter construction
locale negotiation
complex formatting
collation
segmentation
large locale data
```

Measure:

```text
construction cost
per-call cost
cache hit rate
memory
request latency
```

Do not assume:

```text
Intl is always slow
```

or:

```text
Intl is free
```

without measurement.

---

# 83. Formatter Pooling

In high-throughput systems:

```text
request
→ obtain cached formatter
→ format
→ return
```

can be effective.

Possible cache key:

```text
service
+
canonical locale
+
canonical options
```

Bound the cache.

---

# 84. Canonicalizing Locale Inputs

A cache key should not treat:

```text
en-us
```

and:

```text
en-US
```

as unrelated purely because casing differs in the input representation.

Use:

```js
new Intl.Locale(input).toString()
```

or another carefully defined canonicalization strategy.

But remember:

```text
locale canonicalization
```

does not mean:

```text
all user preference variants are semantically identical.
```

---

# 85. `Intl.Locale` Extensions

Locale identifiers can carry extensions affecting services such as:

```text
calendar
numbering system
hour cycle
```

For example conceptually:

```text
en-US-u-ca-gregory
```

Do not strip every substring after:

```text
"-"
```

and assume you still have a complete locale model.

---

# 86. Hour Cycles

Date/time formatting can depend on:

```text
hourCycle
```

Examples conceptually:

```text
h11
h12
h23
h24
```

Do not assume:

```text
24-hour clock
```

or:

```text
12-hour clock
```

is universally correct.

---

# 87. Weekday/Month Names

Use:

```js
new Intl.DateTimeFormat(locale, {
  weekday: "long",
  month: "long"
});
```

rather than maintaining:

```js
["January", "February", ...]
```

manually.

Month names can differ with:

```text
language
calendar
grammatical context
formatting style
```

---

# 88. Format Context Matters

A month may have different grammatical forms depending on how it appears.

This is one reason:

```text
hardcoded translation arrays
```

can be insufficient for full internationalization.

Use the locale-aware formatter appropriate to the presentation context.

---

# 89. Display Names and UI Choices

For language selectors, consider:

```text
self-name
localized name
region
script
```

A code such as:

```text
fr
```

can be displayed differently depending on whether the user is selecting:

```text
language
language + region
writing system
```

Define the product requirement first.

---

# 90. Segmenter and UI Limits

For text limits such as:

```text
"maximum 20 characters"
```

define what “character” means.

Possible meanings:

```text
UTF-16 code units
code points
grapheme clusters
bytes
```

For user-facing text limits, grapheme-aware measurement may be the correct requirement.

---

# 91. Storage Limits vs Display Limits

Do not apply:

```text
UI character count
```

to:

```text
database byte length
```

without considering encoding and storage constraints.

A production text contract should define:

```text
semantic length
storage length
transport size
display length
```

separately.

---

# 92. Locale and Caching Correctness

If cached rendered output depends on:

```text
locale
time zone
currency
calendar
```

those inputs must be part of the cache identity.

Otherwise:

```text
User A → cached en-US result
User B → receives en-US result
```

when B expects another locale.

This is both a correctness and privacy issue.

---

# 93. API Caching and Varying Locale

If HTTP responses vary by:

```text
Accept-Language
```

or another locale preference, caching infrastructure must account for that variation according to the API's actual contract.

Do not rely on application-level cache keys while forgetting:

```text
HTTP intermediary caches
CDNs
browser caches
```

---

# 94. Database Sorting vs Intl.Collator

Do not assume:

```text
ORDER BY
```

in your database has the same ordering as:

```js
Intl.Collator
```

Database collation and application collation can differ.

Define:

```text
where canonical ordering is determined.
```

For large result sets, sorting should usually happen near the data source when possible, under the intended collation.

---

# 95. Search Index and Locale

Localized search may require:

```text
language-specific analyzers
stemming
tokenization
collation
normalization
accent handling
```

`Intl` helps with some presentation/comparison concerns.

It is not a replacement for:

```text
search infrastructure
```

---

# 96. Production Usage

`Intl` is useful for:

```text
dashboards
billing
e-commerce
analytics
date/time UIs
financial displays
country/language selectors
notifications
accessibility
search/sort
mobile/web interfaces
multi-region services
```

A principal engineer should define:

```text
locale ownership
time-zone ownership
currency ownership
formatting boundaries
canonical storage formats
```

---

# 97. Internationalized API Architecture

Recommended flow:

```text
domain model
    ↓
canonical values
    ↓
presentation context
    ├─ locale
    ├─ time zone
    ├─ currency
    └─ calendar
    ↓
Intl formatter
    ↓
localized output
```

Do not contaminate:

```text
domain state
```

with:

```text
localized display strings
```

---

# 98. Internationalization Service Layer

A production application may centralize formatting through a service:

```js
i18n.number(amount, options);
i18n.date(instant, options);
i18n.relative(value, unit);
i18n.list(items, options);
i18n.message(key, variables);
```

Benefits:

```text
consistent locale context
formatter caching
testability
observability
migration
```

Avoid creating an abstraction that merely wraps every Intl method without adding policy/value.

---

# 99. Testing Strategy

Test:

```text
locale negotiation
number formatting
currency
dates
time zones
plural categories
lists
collation
segmentation
fallback
unsupported locale
invalid options
server/client consistency
```

Include representative locales such as:

```text
en-US
en-GB
de-DE
fr-FR
ja-JP
ar-SA
hi-IN
```

Add other locales based on actual product markets.

---

# 100. Differential Locale Testing

For each service compare:

```text
expected semantic behavior
```

rather than assuming:

```text
exact string is universal.
```

For example:

```text
currency part exists
decimal quantity is represented correctly
plural category matches rule
date fields correspond to intended instant/time zone
```

---

# 101. Invalid Options

Intl constructors validate options and can throw for invalid combinations.

Examples may include:

```text
invalid currency code
invalid style combination
invalid time zone
invalid option value
```

Treat these as configuration/API validation errors.

Do not discover them only after deployment.

---

# 102. Error Handling

A formatting failure can originate from:

```text
invalid locale
invalid option
invalid currency
invalid time zone
unsupported configuration
```

Distinguish:

```text
programmer/configuration error
```

from:

```text
user preference fallback.
```

A user requesting an unsupported locale should generally have a fallback strategy.

A service configured with an invalid production time zone should generally fail fast.

---

# 103. Fallback Strategy

A robust locale pipeline can be:

```text
requested locale
→ normalize
→ supported-locale negotiation
→ fallback locale
→ formatter
```

For example:

```text
user preference
→ tenant preference
→ platform default
```

Document the policy.

---

# 104. Bundle / Runtime Considerations

Browser applications may have different locale-data footprints and runtime capabilities than Node.js services.

For large client applications consider:

```text
bundle size
locale data
lazy loading
server formatting
client formatting
```

Do not assume moving all formatting to the client is always cheaper.

---

# 105. Server vs Client Formatting

## Server

Advantages:

```text
consistent output
controlled locale/time zone
SEO/server rendering
```

Costs:

```text
must know user context
hydration consistency
```

## Client

Advantages:

```text
local preferences
interactive formatting
```

Costs:

```text
runtime/environment differences
hydration timing
```

Choose deliberately.

---

# 106. Hydration Safety Pattern

For server-rendered date/time:

```text
explicit locale
+
explicit time zone
+
explicit options
```

For truly client-local values:

```text
render placeholder
→ client format
```

or:

```text
server sends canonical value
→ client formats intentionally
```

Do not accidentally format with different defaults on each side.

---

# 107. Observability

Useful metrics:

```text
locale negotiation fallback rate
unsupported locale requests
formatter cache hit rate
formatting failures
request locale distribution
time-zone distribution
SSR hydration mismatch rate
```

Avoid logging unnecessary:

```text
personal preferences
exact user profile details
sensitive location data
```

---

# 108. Performance Experiment

Benchmark:

```text
new NumberFormat per item
```

vs:

```text
reuse one NumberFormat
```

Then measure:

```text
N = 10
N = 1,000
N = 100,000
```

Also benchmark:

```text
Collator
Segmenter
DateTimeFormat
```

under realistic workloads.

Record:

```text
construction
per-call formatting
memory
throughput
```

---

# 109. Implementation From Scratch — Locale Context

Create:

```js
function createLocaleContext({
  locale,
  timeZone,
  currency,
  calendar
}) {
  // validate and normalize
}
```

Requirements:

```text
validate locale
normalize supported values
define defaults
expose immutable context
```

---

# 110. Implementation — Formatter Registry

Build:

```js
registry.getNumberFormatter(key)
registry.getDateFormatter(key)
registry.getCollator(key)
registry.getSegmenter(key)
```

Requirements:

```text
canonical cache keys
bounded growth
cache metrics
clear/invalidation
```

---

# 111. Implementation — Money Formatter

Build:

```js
formatMoney(amount, {
  locale,
  currency
});
```

Separate:

```text
money representation
```

from:

```text
localized output
```

Test:

```text
USD
EUR
INR
JPY
```

and multiple locales.

---

# 112. Implementation — Relative Time Service

Build:

```js
formatRelativeTime(value, unit, context)
```

Support:

```text
numeric
numeric: "auto"
```

and document that:

```text
semantic time-difference computation
```

is outside the formatter itself.

---

# 113. Implementation — Plural Message Selection

Build:

```js
selectPluralMessage(locale, count, messages)
```

Example:

```js
{
  one: "...",
  other: "..."
}
```

Validate that the message map handles the categories required by the selected locale.

Do not assume:

```text
one/other
```

is sufficient for every locale.

---

# 114. Implementation — Text Segmentation

Build a utility:

```js
segmentText(text, {
  locale,
  granularity
});
```

Support:

```text
grapheme
word
sentence
```

Use it for:

```text
user-facing character counting
excerpting
cursor-safe limits
```

---

# 115. Debugging Exercises

## Exercise A — Wrong Currency

The UI shows:

```text
$1,000
```

for users in a region expecting:

```text
local currency
```

Diagnose:

```text
currency source
locale source
presentation policy
cache key
```

---

## Exercise B — Hydration Mismatch

Server renders:

```text
11/09/2026
```

Browser renders:

```text
09/11/2026
```

Identify:

```text
locale mismatch
```

and possibly:

```text
formatting default mismatch.
```

---

## Exercise C — Time Zone Bug

The backend sends:

```text
2026-09-11T00:00:00Z
```

User expects:

```text
their local calendar date.
```

Explain the difference between:

```text
instant
```

and:

```text
local display date.
```

---

## Exercise D — “Character Count” Bug

A UI allows:

```text
10 characters
```

but an emoji sequence appears to consume:

```text
multiple characters.
```

Diagnose:

```text
UTF-16 code units
vs
grapheme clusters.
```

---

# 116. Code Review Exercise

Review:

```js
function formatDate(date) {
  return `${date.getMonth() + 1}/${date.getDate()}/${date.getFullYear()}`;
}
```

Questions:

```text
Which locale is assumed?

Which time zone is assumed?

Which calendar is assumed?

What happens in RTL languages?

What happens outside the Gregorian presentation expectation?

What happens during SSR?

How would you test it?
```

Then replace it with a policy-driven design.

---

# 117. Interview Questions

### Fundamentals

```text
1. What is ECMA-402?
2. What problem does Intl solve?
3. What is a locale?
4. What is a BCP 47 language tag?
5. What is Intl.Locale?
```

### Number / Date

```text
6. Why should currency be formatted with Intl?
7. Why is localized formatting not a financial calculation engine?
8. What is NumberFormat.formatToParts?
9. How does DateTimeFormat use time zones?
10. Why are named time zones better than fixed offsets for civil-time display?
```

### Text / Language

```text
11. What does PluralRules do?
12. Why is `count === 1` not universal pluralization?
13. What does Collator do?
14. What does Segmenter solve?
15. Why isn't string length equal to user-visible character count?
```

### Principal

```text
16. How would you design locale context propagation across a Node.js service?
17. How would you prevent formatter caches from growing without bound?
18. How would you prevent SSR/client locale mismatch?
19. How would you decide whether sorting belongs in the database or application?
20. How would you design internationalized financial/date APIs without leaking presentation concerns into the domain model?
```

---

# 118. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
new Intl.NumberFormat("en-US").format(1234567.89);
```

Explain which dimensions are locale-sensitive.

### Exercise 2

```js
new Intl.NumberFormat("de-DE").format(1234567.89);
```

Compare with Exercise 1.

### Exercise 3

```js
new Intl.PluralRules("en").select(1);
new Intl.PluralRules("en").select(2);
```

### Exercise 4

```js
new Intl.RelativeTimeFormat("en", {
  numeric: "auto"
}).format(-1, "day");
```

### Exercise 5

```js
new Intl.ListFormat("en", {
  type: "disjunction"
}).format(["A", "B", "C"]);
```

### Exercise 6

```js
const segmenter = new Intl.Segmenter("en", {
  granularity: "grapheme"
});
```

Explain what abstraction the segmenter is providing.

---

# 119. Mastery Exercises

### Exercise 1 — Locale Negotiator

Implement:

```text
requested locales
→ supported locales
→ chosen locale
```

with explicit fallback.

### Exercise 2 — Formatter Cache

Implement a bounded cache for:

```text
NumberFormat
DateTimeFormat
Collator
```

### Exercise 3 — i18n Context

Propagate:

```text
locale
timeZone
currency
```

through a request.

### Exercise 4 — Locale Test Matrix

Create automated tests for:

```text
en-US
en-GB
de-DE
fr-FR
ja-JP
ar-SA
hi-IN
```

covering:

```text
numbers
money
dates
plural
lists
```

### Exercise 5 — SSR Consistency

Build server/client rendering that guarantees:

```text
same locale
same time zone
same options
```

for a deterministic date output.

---

# 120. Track A — Core Theory

Master:

```text
ECMA-402
locale
BCP 47
Intl.Locale
locale negotiation
supported locales
NumberFormat
DateTimeFormat
PluralRules
RelativeTimeFormat
ListFormat
DisplayNames
Collator
Segmenter
calendars
numbering systems
time zones
Unicode segmentation
locale-sensitive casing
```

Deliverable:

```text
explain why a localized result is correct for the selected locale/context.
```

---

# 121. Track B — Implementation

Build:

```text
locale context
formatter registry
bounded Intl cache
money formatter
date/time formatter
plural service
relative-time service
text segmentation service
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

Deliverable:

```text
internationalization becomes a reusable platform capability.
```

---

# 122. Track C — Interview / Reasoning

Practice:

```text
"Why is manual date formatting dangerous?"

"Why can't string length measure user-visible characters?"

"Why should locale be explicit in SSR?"

"Why is sorting different from equality?"

"Why isn't Intl a parser?"

"Where should locale belong in an API?"

"What is the difference between instant, calendar, and time zone?"
```

Deliverable:

```text
precise internationalization reasoning.
```

---

# 123. Specification / Runtime Source Discipline

Prefer:

```text
1. ECMA-402 specification
2. ECMAScript specification where integrated
3. Unicode/CLDR terminology and data model
4. runtime implementation documentation
5. browser/Node behavior
```

The ECMAScript specification delegates `Date` internationalization behavior to ECMA-402 when the implementation includes that API. citeturn278112search1

The ECMAScript time-zone model also integrates with ECMA-402 time-zone requirements for time-zone-aware implementations. citeturn987626search0turn987626search2

When discussing:

```text
ICU data
CLDR versions
runtime locale-data footprint
```

label those as implementation/runtime concerns rather than pretending they are all ECMAScript language rules.

---

# 124. Common Failure Modes

```text
Failure 1:
Treating locale as language only.

Failure 2:
Formatting money with string concatenation.

Failure 3:
Using localized strings as machine identifiers.

Failure 4:
Parsing localized numbers with generic Number() assumptions.

Failure 5:
Using machine default locale in deterministic tests.

Failure 6:
Using machine local time zone for business-critical rendering.

Failure 7:
Treating fixed UTC offsets as named civil time zones.

Failure 8:
Implementing pluralization with singular/plural only.

Failure 9:
Sorting identifiers with display-oriented locale rules.

Failure 10:
Using string length as user-visible character count.

Failure 11:
Creating new Intl formatters inside large loops.

Failure 12:
Building unbounded formatter caches.

Failure 13:
Ignoring SSR/client locale mismatch.

Failure 14:
Mixing presentation strings into domain models.

Failure 15:
Assuming Intl can parse every localized string.
```

---

# 125. Principal Decision Framework

When designing internationalization architecture, evaluate:

```text
Correctness
Locale coverage
Time-zone correctness
Currency correctness
Unicode correctness
Accessibility
Performance
Memory
Security
Consistency
Testing
Observability
Developer Experience
Operational Complexity
Future locale expansion
```

The central question is:

```text
Which values are canonical,
and which values are merely localized presentations?
```

---

# 126. Production Checklist

Before shipping an internationalized feature:

```text
[ ] locale source defined
[ ] locale fallback defined
[ ] time-zone source defined
[ ] currency source defined
[ ] calendar requirements defined
[ ] canonical storage defined
[ ] formatter options explicit
[ ] SSR/client behavior aligned
[ ] unsupported locale tested
[ ] representative locales tested
[ ] plural behavior tested
[ ] long text tested
[ ] RTL tested where applicable
[ ] accessibility tested
[ ] formatter caching reviewed
[ ] exact-output assumptions reviewed
[ ] security identifiers kept locale-neutral
```

---

# 127. Retrieval Record

```md
# Chapter 128 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Locale / Negotiation
-

## Number Formatting
-

## Currency
-

## Dates / Times
-

## Time Zones
-

## Plural Rules
-

## Relative Time
-

## Lists
-

## Display Names
-

## Collation
-

## Segmentation
-

## Unicode / Casing
-

## Performance
-

## Testing
-

## Strongest Areas
-

## Weakest Areas
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 128. Spaced Retrieval Schedule

### Day 0

Study:

```text
locale
NumberFormat
DateTimeFormat
PluralRules
Collator
Segmenter
```

### Day 1

Explain:

```text
locale vs language
instant vs time zone
calendar vs locale
```

without notes.

### Day 3

Build:

```text
formatter registry
```

with a bounded cache.

### Day 7

Test:

```text
en-US
en-GB
de-DE
fr-FR
ja-JP
ar-SA
hi-IN
```

### Day 14

Design an SSR-safe locale architecture.

### Day 21

Perform a production internationalization review.

### Day 30

Build a complete locale context service from scratch.

---

# 129. Dependency Graph

```text
Chapter 04
Strings / Unicode
        ↓
Chapter 23
String APIs
        ↓
Chapter 28
Serialization
        ↓
Chapter 49–55
Browser / Web Platform / Networking
        ↓
Chapter 79
API Design
        ↓
Chapter 81
Database Integration
        ↓
Chapter 85
Performance
        ↓
Chapter 86
Testing
        ↓
Chapter 92
Temporal
        ↓
Chapter 123
Grammar / Parsing
        ↓
Chapter 124
Execution / References
        ↓
Chapter 125
Promise Internals
        ↓
Chapter 126
Module Linking
        ↓
Chapter 127
Shared Memory
        ↓
Chapter 128
Internationalization / Intl
        ↓
Chapter 129
Regular Expressions
```

---

# 130. Concept Connections

## Depends On

```text
Unicode
strings
numbers
dates
time zones
Temporal
objects/options
testing
performance
API design
```

## Builds Toward

```text
global product architecture
localization systems
calendar/time correctness
search/sorting systems
accessibility
platform internationalization
```

## Related Concepts

```text
BCP 47
Unicode
CLDR
ICU
time zones
calendar systems
locale negotiation
localization
message formatting
```

## Concepts Revisited

```text
Strings
Unicode
Date
Temporal
Numbers
Arrays
sorting
performance
testing
HTTP headers
API context
```

## Why This Chapter Matters

Internationalization is where simple primitives stop being universal.

```text
number
```

is not always displayed the same way.

```text
date
```

does not imply one calendar presentation.

```text
time
```

does not imply one time zone.

```text
character
```

does not imply one UTF-16 code unit.

```text
comparison
```

does not imply code-unit ordering.

The production rule is:

```text
canonical domain value
→ explicit cultural context
→ Intl presentation
```

---

# 131. Final Principal Mental Model

Use:

```text
canonical value
      ↓
domain meaning
      ↓
presentation context
      ├── locale
      ├── region
      ├── numbering system
      ├── calendar
      ├── time zone
      ├── currency
      └── display policy
      ↓
Intl service
      ↓
localized output
```

For text:

```text
Unicode data
→ normalization if required
→ locale-sensitive operation
→ localized presentation
```

For time:

```text
instant / civil date / zoned value
→ selected time zone/calendar
→ Intl.DateTimeFormat
→ localized presentation
```

For numbers:

```text
canonical number / exact money model
→ locale/currency/unit context
→ Intl.NumberFormat
→ localized output
```

For language:

```text
locale
→ plural/list/relative-time/collation/segmentation rules
→ user-facing result
```

---

# 132. Final Principal Principle

> **Internationalization is the separation of universal data from culturally variable presentation and behavior.**

The mature architecture is:

```text
canonical state
+
explicit locale context
+
domain-specific semantics
+
Intl formatting/classification
+
deterministic testing
+
bounded formatter reuse
```

Do not design global software around:

```text
one language
one time zone
one date format
one currency
one alphabet
one plural rule
one character model
```

Design around:

```text
canonical data
→ explicit context
→ standardized internationalization services
→ localized presentation
```

That is the foundation for production-grade global JavaScript applications.