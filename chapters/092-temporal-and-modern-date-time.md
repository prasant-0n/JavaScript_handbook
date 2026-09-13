# Chapter 92 — Temporal

> **JavaScript Mastery — Part XVII: Modern ECMAScript & Language Evolution**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Current verification:** 2026-09-10
>
> **Current standards state:** Temporal is listed by the official TC39 proposals repository as a finished / Stage 4 proposal with expected publication year 2026. The official Temporal repository states that it is Stage 4, and the current proposal draft is dated July 27, 2026.
>
> **Runtime note:** The Temporal repository currently documents shipping support in Firefox 139, Chrome 144, and Node.js 26. Verify the exact versions and toolchain used by your production fleet before adopting it.

---

# 0. Chapter Mission

Temporal is not merely “a better `Date` API.” It is a type system for representing different temporal facts explicitly.

The core question is:

```text
What temporal fact do I actually know?
              ↓
Choose the matching Temporal type
              ↓
Perform the operation in that domain
              ↓
Convert only when the domain transition is explicit
              ↓
Serialize with enough information to preserve meaning
```

The chapter develops Temporal from problem → type model → invariants → arithmetic → time zones → DST → calendars → parsing → serialization → interoperability → performance → production engineering → specification reasoning.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- Explain why JavaScript `Date` is frequently insufficient for modern date/time work.
- Explain Temporal’s core design principles.
- Distinguish `Temporal.Instant`, `Temporal.ZonedDateTime`, `Temporal.PlainDateTime`, `Temporal.PlainDate`, `Temporal.PlainTime`, `Temporal.PlainYearMonth`, `Temporal.PlainMonthDay`, and `Temporal.Duration`.
- Explain exact time versus wall-clock time.
- Explain time zones and daylight-saving transitions.
- Explain elapsed time versus calendar time.
- Explain DST gaps and overlaps and disambiguation policy.
- Explain calendar-aware arithmetic.
- Parse and serialize Temporal values safely.
- Convert between `Date` and Temporal intentionally.
- Explain Temporal precision and the role of `BigInt`-style integer representations.
- Use `Temporal.Now` appropriately.
- Distinguish business timestamps from monotonic performance clocks.
- Design database/API representations for temporal values.
- Test DST, leap years, precision, localization, and serialization boundaries.
- Read Temporal specification text and reason about its internal semantics.
- Review Temporal architecture at principal-engineer level.

---

# 2. Prerequisites

Recommended previous chapters:

```text
03 — Numbers, Floating Point, BigInt
04 — Strings, Unicode, Text Semantics
07 — Type Conversion, Coercion, Equality
15 — Objects, Property Semantics
20 — Symbols and Well-Known Symbols
28 — JSON, Serialization, Structured Clone
41 — Spec Architecture
42 — Abstract Operations
43 — Ordinary Object Internal Methods
63 — Async Context and Diagnostics
79 — API Design
81 — Database Integration
84 — Reliability
85 — Performance
90 — Modern ECMAScript Features
91 — TC39 Proposal Tracking
```

---

# 3. What Problem Does Temporal Solve?

Date/time is not one problem.

A production system can need to answer all of these:

```text
What exact instant happened?
What local calendar date is it?
What time does a shop open every day?
What local date/time did the user enter?
What instant does that local time represent in a named zone?
How long did an operation take?
What is the next calendar month?
What is the user's birthday?
```

These are different semantic domains.

A single legacy object encourages accidental conversion between them.

Temporal instead provides specialized types so the application can preserve the original meaning.

---

# 4. The `Date` Problem

Legacy `Date` remains important for compatibility, but its API combines concepts that modern applications often need to separate.

Common sources of bugs include:

- mutation through setters,
- local-zone assumptions,
- UTC/local conversions hidden in helpers,
- millisecond precision,
- confusing parsing expectations,
- inability to express date-only and time-only concepts directly,
- weak representation of named time-zone context,
- arithmetic that encourages fixed-duration thinking for calendar problems.

The Temporal proposal documentation explicitly calls out `Date` limitations involving mutability, time zones, date-only/time-only use cases, and API ergonomics.

---

# 5. Temporal’s Core Design Principles

Temporal emphasizes:

- immutable temporal values,
- distinct temporal domains,
- first-class time-zone support,
- DST-aware operations,
- strictly specified string forms,
- calendar support,
- explicit conversions,
- high precision for exact time.

The most useful mental model is:

```text
Semantics first.
Representation second.
Formatting last.
```

---

# 6. The Temporal Type System

```text
Temporal.Instant
    exact point on the timeline

Temporal.ZonedDateTime
    exact time + named time zone + calendar

Temporal.PlainDateTime
    local date + time, no time zone

Temporal.PlainDate
    calendar date, no time zone

Temporal.PlainTime
    clock time, no date or time zone

Temporal.PlainYearMonth
    year + month

Temporal.PlainMonthDay
    month + day

Temporal.Duration
    temporal amount / period
```

Do not memorize names without understanding the domains.

---

# 7. `Temporal.Instant`

An `Instant` represents one exact point on the timeline.

```js
const instant = Temporal.Instant.from("2020-01-01T00:00+05:30");

console.log(instant.toString());
// 2019-12-31T18:30:00Z
```

An `Instant` does not contain a time zone or calendar. A zone is applied when the instant is interpreted as local civil time.

Conceptually:

```text
Timeline
──────────────────────────────────────────→
                    ●
                 Instant
```

The current Temporal documentation describes `Instant` as exact time with nanosecond precision and an integer representation based on nanoseconds since the Unix epoch, ignoring leap seconds.

---

# 8. Exact Time vs Wall-Clock Time

### Exact time

A unique point on the global timeline.

```text
2026-09-10T18:00:00Z
```

### Wall-clock time

A civil/local representation.

```text
2026-09-11 00:00 in Asia/Kolkata
```

One instant can have many wall-clock representations:

```text
Instant
  ├─ Asia/Kolkata
  ├─ Asia/Tokyo
  └─ America/New_York
```

---

# 9. `Temporal.ZonedDateTime`

`ZonedDateTime` combines:

```text
exact time
+
named time zone
+
calendar
```

Example:

```js
const zdt = Temporal.ZonedDateTime.from(
  "2026-09-10T23:30+05:30[Asia/Kolkata]"
);
```

It can answer both:

```text
What instant is this?
What local calendar/clock representation applies in this zone?
```

---

# 10. Offset vs Named Time Zone

These are not interchangeable.

```text
+05:30
```

is an offset.

```text
Asia/Kolkata
```

is a named IANA time zone.

A named zone supplies a ruleset for interpreting civil time across dates. An offset is a displacement applicable to a particular temporal context.

Principal rule:

```text
If the business meaning depends on the region's time-zone rules,
preserve the named zone.
```

---

# 11. `Temporal.PlainDateTime`

`PlainDateTime` represents a local date and time without a time zone.

```js
const dt = Temporal.PlainDateTime.from("2026-09-10T14:30");
```

Useful for:

- user-entered local date/time,
- schedule templates,
- meeting drafts before a zone is assigned,
- values intentionally independent of a time zone.

It does not identify a unique instant.

---

# 12. Why `PlainDateTime` Is Not an Instant

A local date/time may be:

```text
unambiguous
ambiguous
nonexistent
```

depending on the zone to which it is later assigned.

For example, a DST overlap can cause the same wall-clock time to occur twice, while a DST gap can create a local time that never occurs.

Therefore:

```text
PlainDateTime
    ↓ assign zone + policy
ZonedDateTime
    ↓ exact interpretation
Instant
```

---

# 13. `Temporal.PlainDate`

Use `PlainDate` for a calendar date.

```js
const birthday = Temporal.PlainDate.from("1998-06-14");
```

Typical uses:

- birthday,
- invoice date,
- due date,
- accounting date,
- holiday date.

Do not invent UTC midnight just to represent a date-only business concept.

---

# 14. `Temporal.PlainTime`

Use `PlainTime` for a clock time without date or zone.

```js
const opening = Temporal.PlainTime.from("09:30");
```

Examples:

- store opening time,
- recurring alarm time,
- shift start template,
- daily business cutoff.

---

# 15. `Temporal.PlainYearMonth`

Represents a year/month value.

```js
const period = Temporal.PlainYearMonth.from("2026-09");
```

Useful for:

- reporting periods,
- monthly accounting,
- subscription periods,
- expiration month concepts.

It is not automatically a timestamp for the first day of the month.

---

# 16. `Temporal.PlainMonthDay`

Represents month + day without a year.

```js
const anniversary = Temporal.PlainMonthDay.from("--06-14");
```

Good for recurring annual concepts where the year itself is not part of the stored fact.

---

# 17. `Temporal.Duration`

A `Duration` represents an amount of temporal units.

```js
const duration = Temporal.Duration.from({
  days: 3,
  hours: 4
});
```

Units can include:

```text
years months weeks days
hours minutes seconds
milliseconds microseconds nanoseconds
```

Critical distinction:

```text
Duration
≠
fixed number of milliseconds in every context
```

---

# 18. Calendar Duration vs Elapsed Duration

Consider:

```text
1 day
```

In ordinary elapsed-time arithmetic this may suggest:

```text
24 hours
```

But in a named local time zone, a civil day around a DST transition may represent 23 or 25 elapsed hours.

Therefore the correct question is:

```text
Is the business rule about calendar progression?
Or about elapsed time?
```

---

# 19. Immutability

Temporal objects are immutable.

```js
const d = Temporal.PlainDate.from("2026-09-10");
const next = d.add({ days: 1 });

console.log(d.toString());
// 2026-09-10

console.log(next.toString());
// 2026-09-11
```

Why this matters:

- easier reasoning,
- safer sharing,
- fewer aliasing bugs,
- easier testing,
- better functional composition.

---

# 20. DST Fundamentals

DST creates two important classes of local-time problems.

## Gap

A clock moves forward and some local times do not exist.

```text
01:59:59
→
03:00:00
```

A local value such as `02:30` can be nonexistent.

## Overlap

A clock moves backward and some local times occur twice.

```text
02:59:59
→
02:00:00
```

A local value such as `02:30` can be ambiguous.

---

# 21. DST Disambiguation

Temporal supports explicit disambiguation policies for local-to-zoned conversions. The principal choices are conceptually:

```text
compatible
earlier
later
reject
```

Example:

```js
const zdt = Temporal.ZonedDateTime.from(
  {
    timeZone: "America/New_York",
    year: 2026,
    month: 11,
    day: 1,
    hour: 1,
    minute: 30
  },
  {
    disambiguation: "later"
  }
);
```

For financial or contractual systems, silently resolving an invalid or ambiguous civil time may be unacceptable; `reject` can be appropriate when the input must be unambiguous.

---

# 22. Converting `Instant` to a Zone

```js
const instant = Temporal.Instant.from("2026-09-10T18:00Z");

const kolkata = instant.toZonedDateTimeISO("Asia/Kolkata");
const tokyo = instant.toZonedDateTimeISO("Asia/Tokyo");
```

The instant is the same.

The local fields differ.

```text
same instant
     ↓
 different zones
     ↓
 different wall-clock values
```

---

# 23. Converting Zoned Time to Instant

```js
const zdt = Temporal.ZonedDateTime.from(
  "2026-09-10T23:30+05:30[Asia/Kolkata]"
);

const instant = zdt.toInstant();
```

The result is the unique exact point represented by the zoned value.

---

# 24. Plain-to-Zoned Conversion

```js
const local = Temporal.PlainDateTime.from("2026-09-10T23:30");
const zoned = local.toZonedDateTime("Asia/Kolkata");
```

This is a semantic conversion, not just formatting.

You are saying:

> Interpret this local date/time using this time-zone rule set.

---

# 25. Zoned-to-Plain Conversion

```js
const zdt = Temporal.ZonedDateTime.from(
  "2026-09-10T23:30+05:30[Asia/Kolkata]"
);

const date = zdt.toPlainDate();
const time = zdt.toPlainTime();
const dateTime = zdt.toPlainDateTime();
```

These operations intentionally discard some information.

For example:

```text
ZonedDateTime
     ↓
PlainDateTime
```

loses zone/instant context.

---

# 26. `withTimeZone()` Mental Model

```text
same instant
     ↓
new zone
     ↓
new local representation
```

Example:

```js
const original = Temporal.ZonedDateTime.from(
  "2026-09-10T23:30+05:30[Asia/Kolkata]"
);

const tokyo = original.withTimeZone("Asia/Tokyo");
```

Do not expect the local hour to remain 23:30. The instant is preserved, not the wall-clock fields.

---

# 27. Same Clock Time vs Same Instant

These are opposite transformation goals.

### Preserve instant

```text
withTimeZone()
```

### Preserve local fields and reinterpret in a different zone

This is a different domain transformation.

A principal engineer should state which invariant is being preserved before writing the conversion.

---

# 28. Calendar Arithmetic

```js
const date = Temporal.PlainDate.from("2026-01-31");
const next = date.add({ months: 1 });
```

The operation is calendar-aware.

Do not reason as though “one month” is a fixed number of seconds.

The exact result depends on the applicable calendar and overflow policy.

---

# 29. Overflow

Temporal can expose policies such as:

```text
constrain
reject
```

Example:

```js
Temporal.PlainDate.from(
  { year: 2026, month: 2, day: 31 },
  { overflow: "reject" }
);
```

For strict data validation, rejecting invalid calendar input is often preferable to silently constraining it.

---

# 30. `add()` / `subtract()`

Examples:

```js
const d = Temporal.PlainDate.from("2026-09-10");

const nextWeek = d.add({ weeks: 1 });
const previousDay = d.subtract({ days: 1 });
```

The original remains unchanged.

For `ZonedDateTime`, the operation can involve both calendar arithmetic and time-zone transition rules.

---

# 31. `with()`

Use `with()` to replace selected fields without mutating the original.

```js
const d = Temporal.PlainDate.from("2026-09-10");
const changed = d.with({ day: 20 });
```

This supports explicit value transformations.

---

# 32. `until()` and `since()`

Temporal supports difference calculations:

```js
const start = Temporal.PlainDate.from("2026-09-10");
const end = Temporal.PlainDate.from("2026-09-25");

const difference = start.until(end);
```

Difference semantics can depend on:

- type/domain,
- largest unit,
- smallest unit,
- rounding,
- calendar,
- time-zone context.

---

# 33. Why `86400000` Is a Dangerous Calendar Abstraction

This code:

```js
const days = Math.floor(
  (end.epochMilliseconds - start.epochMilliseconds) / 86400000
);
```

can answer an elapsed-time question approximately.

It does not automatically answer:

> How many local calendar days separate these values?

Choose calendar-aware operations for calendar questions.

---

# 34. Elapsed Time vs Calendar Time

Example conceptual situation:

```text
A = local midnight before a DST spring-forward
B = local midnight the next day
```

Calendar difference:

```text
1 day
```

Elapsed exact time can be:

```text
23 hours
```

The correct result depends on the question.

---

# 35. Rounding

Temporal has defined rounding operations and rounding modes.

Typical concepts include:

```text
smallestUnit
largestUnit
roundingIncrement
roundingMode
relativeTo
```

Rounding calendar quantities requires more semantic context than rounding a plain floating-point number.

---

# 36. `Temporal.Now`

Temporal provides current-time helpers including forms such as:

```js
Temporal.Now.instant();
Temporal.Now.plainDateISO();
Temporal.Now.plainTimeISO();
Temporal.Now.plainDateTimeISO();
Temporal.Now.zonedDateTimeISO();
Temporal.Now.timeZoneId();
```

The exact method set and behavior should be verified against the runtime/spec version being targeted.

---

# 37. Wall Clock vs Monotonic Clock

This is essential for production systems.

```text
Temporal.Instant
= exact civil/timeline timestamp

performance.now()
= elapsed-time measurement using a monotonic clock
```

Use exact timestamps for business events.

Use a monotonic clock for latency/SLA measurements where wall-clock adjustment could distort the measurement.

---

# 38. `Date` Interoperability

Legacy systems will still contain `Date`.

Conceptually:

```js
const date = new Date();
const instant = date.toTemporalInstant();
```

A reverse conversion may use millisecond representation.

Important warning:

```text
Temporal nanoseconds
        ↓
Date milliseconds
        ↓
possible precision loss
```

Do not round-trip through `Date` when sub-millisecond precision matters.

---

# 39. Precision

Temporal `Instant` uses nanosecond precision; `Date` is millisecond-based.

This creates a direct connection to `Number` and `BigInt`:

```text
high-precision epoch integer
          ↓
BigInt semantics
```

Store or transmit the precision your domain actually guarantees. Do not claim nanosecond precision when the database or wire format only preserves milliseconds.

---

# 40. BigInt Connection

A large integer count of nanoseconds can exceed exact-integer limits of ordinary `Number` values.

This is why exact epoch representations can use `BigInt`.

Principal lesson:

```text
Time precision is part of the data contract.
```

---

# 41. Time Zones Are Data

A zone identifier such as:

```text
America/New_York
```

is associated with a ruleset whose interpretation depends on time-zone data.

Rules can change due to real-world policy and database updates.

Therefore:

```text
time-zone conversion
=
temporal semantics + external zone data
```

Long-lived schedules must account for this.

---

# 42. Time-Zone Data Versioning

A future local schedule can be affected by a future time-zone database change.

Therefore decide whether a schedule should:

```text
follow future civil-time rule changes
```

or:

```text
be frozen to already-resolved instants
```

This is a product and architecture decision.

---

# 43. Scheduling a Daily Local Event

Requirement:

> Run every day at 09:00 in New York.

Do not store only:

```text
13:00Z
```

Store the semantic rule:

```text
local time = 09:00
zone = America/New_York
recurrence = daily
```

Then resolve each occurrence using the time-zone rules.

---

# 44. Scheduling vs Fixed Duration

These two requirements are different:

```text
Every day at 09:00 local time
```

versus:

```text
Every 24 elapsed hours
```

The first is calendar/civil-time semantics.

The second is elapsed-duration semantics.

Never encode one as the other accidentally.

---

# 45. Example — E-Commerce Orders

Use:

```text
createdAt → Instant
paidAt → Instant
shippedAt → Instant
```

But:

```text
expectedDeliveryDate → PlainDate
```

and:

```text
storeOpeningTime → PlainTime
```

A single workflow can legitimately use several Temporal types.

---

# 46. Example — Hotel Booking

Possible fields:

```text
checkInDate → PlainDate
checkOutDate → PlainDate
arrivalInstant → Instant
hotelCheckInTime → PlainTime
hotelZone → IANA zone identifier
```

This is clearer than one generic “booking date” field.

---

# 47. Example — Payments

A payment capture event is an exact event:

```text
capturedAt → Instant
```

An expiry condition can also be an exact instant.

Do not transform those into local date/time fields merely because users view them in a local zone later.

---

# 48. Example — Subscription Renewal

Requirement:

> Renew monthly at 10:00 in the customer's local time.

Model:

```text
recurrence rule
+
PlainTime
+
zone
```

Do not approximate “monthly” as a fixed 30-day duration.

---

# 49. Example — Birthday

A birthday generally represents:

```text
month + day
```

rather than a global instant.

A `PlainMonthDay` can express the concept directly when the year is not part of the domain.

---

# 50. Example — Age

Age is calendar-oriented.

Avoid:

```js
Math.floor(elapsedMilliseconds / approximateYearLength);
```

Instead model the birth date and current date as calendar values and calculate according to the business definition.

---

# 51. Example — SLA

Requirement:

> Finish within 500 ms.

Use a monotonic timing mechanism for measurement.

The request's audit timestamp can still be an `Instant`.

This is a key separation:

```text
When did it happen?
→ Instant

How long did it take?
→ monotonic timer
```

---

# 52. Example — Reporting Month

Requirement:

> September 2026 report.

Model:

```js
Temporal.PlainYearMonth.from("2026-09");
```

Do not invent a timestamp if the day and instant have no business meaning.

---

# 53. `Intl` Integration

Temporal and `Intl` solve complementary problems.

```text
Temporal
= representation + calculation

Intl
= locale-aware presentation
```

Use Temporal for domain semantics and `Intl` for user-facing formatting.

---

# 54. Formatting vs Serialization

Formatting is for humans:

```text
10 Sep 2026, 11:30 PM
```

Serialization is for machines:

```text
2026-09-10T23:30:00+05:30[Asia/Kolkata]
```

Never use presentation formatting as your canonical wire format.

---

# 55. Parsing

Parse according to the intended type:

```js
Temporal.PlainDate.from("2026-09-10");
Temporal.Instant.from("2026-09-10T18:00:00Z");
Temporal.PlainTime.from("09:30");
```

Do not infer semantic type only from the fact that the value is a string.

---

# 56. API Schema Design

Bad:

```ts
type Payload = {
  date: string;
  timestamp: string;
  expires: string;
};
```

Better conceptual contract:

```text
createdAt = exact timestamp
invoiceDate = calendar date
openingTime = local clock time
scheduledAt = local date/time + named zone
```

The JSON strings can still be strings at the wire level; the semantics must be explicit in the schema.

---

# 57. Database Mapping

Typical mappings:

```text
Instant
→ database timestamp/instant type

PlainDate
→ SQL DATE

PlainTime
→ SQL TIME

PlainYearMonth
→ year/month fields or canonical string

ZonedDateTime
→ usually Instant + zone identifier
```

The exact mapping depends on the database and driver.

---

# 58. Recommended Zoned Storage

For events where both the exact instant and zone context matter, store something like:

```text
instant = 2026-09-10T18:00:00Z
zone = Asia/Kolkata
```

This separates:

```text
when it happened
```

from:

```text
which civil-time context matters
```

An offset alone may not preserve the latter.

---

# 59. Precision at Boundaries

Audit every boundary:

```text
Temporal
→ database
→ driver
→ JSON
→ frontend
```

Ask:

```text
What precision survives each step?
```

A nanosecond-capable source is not enough if the database stores milliseconds.

---

# 60. JSON Interoperability

A strong pattern is:

```js
const payload = {
  createdAt: instant.toString()
};
```

Then revive according to the schema:

```js
const createdAt = Temporal.Instant.from(payload.createdAt);
```

Do not guess whether a value is an `Instant`, `PlainDate`, or `ZonedDateTime` from the string alone when an explicit schema can tell you.

---

# 61. Distributed Systems

Temporal timestamps are useful, but timestamps do not establish causality.

```text
timestamp order
≠
causal order
```

For distributed ordering, use domain mechanisms such as:

- sequence numbers,
- message offsets,
- database ordering,
- logical clocks,
- tracing context.

---

# 62. Clock Skew

Different machines can have different wall-clock values.

Use `Instant` for exact event records, but do not treat timestamps as a universal distributed-consensus mechanism.

---

# 63. Caching and Expiry

A stored expiration can be:

```text
expiresAt → Instant
```

For internal control loops, use appropriate monotonic elapsed-time mechanisms where required.

Keep business timestamps separate from measurement clocks.

---

# 64. Logging and Auditing

Distributed logs should normally contain an unambiguous machine timestamp.

Prefer:

```json
{
  "timestamp": "2026-09-10T18:00:00Z"
}
```

over a local timestamp with no zone.

For audits, preserve the information required by the business to reconstruct meaning.

---

# 65. Security Considerations

Temporal improves semantic clarity but does not remove security problems.

Review:

- untrusted date strings,
- time-zone identifiers,
- authorization windows,
- replay windows,
- token expiry,
- clock skew,
- future schedule behavior,
- inconsistent browser/server interpretations,
- denial-of-service through pathological parsing.

---

# 66. Authorization Example

Requirement:

> Coupon valid until September 10.

This is ambiguous until policy defines whether it means:

```text
start of local day
end of local day
specific instant
merchant-local cutoff
UTC cutoff
```

Temporal can represent the chosen rule. It cannot invent the business rule for you.

---

# 67. Token Expiry

Protocol expiry generally represents an exact instant.

Use:

```text
Instant
```

rather than interpreting an expiration field as a local calendar date unless the protocol says so.

---

# 68. Input Validation

Validate at the boundary:

```js
function parseDueDate(value) {
  if (typeof value !== "string") {
    throw new TypeError("dueDate must be a string");
  }

  return Temporal.PlainDate.from(value);
}
```

Then the domain layer can operate on a typed value rather than repeatedly parsing arbitrary strings.

---

# 69. Error Handling

Temporal parsing can fail.

Treat malformed user data as an input/domain error.

```js
try {
  return Temporal.Instant.from(input);
} catch (error) {
  throw new Error("Invalid timestamp", { cause: error });
}
```

Do not catch every error and silently convert it to `null`.

---

# 70. Performance Considerations

Temporal objects have object and calculation costs.

Potential sources include:

- object allocation,
- parsing,
- zone rule lookup,
- calendar operations,
- formatting,
- repeated conversion.

Do not assume Temporal is always faster or slower than legacy alternatives.

Benchmark real workloads.

---

# 71. Performance Strategy

Prefer boundary conversion:

```text
wire string
   ↓
parse once
   ↓
Temporal value
   ↓
domain logic
   ↓
serialize once
```

Avoid repetitive conversion inside hot loops.

---

# 72. Memory Considerations

Immutability means transformations create new values.

```js
const d2 = d1.add({ days: 1 });
const d3 = d2.add({ days: 1 });
```

This is excellent for correctness, but allocation pressure can still matter in extreme workloads.

Profile before optimizing.

---

# 73. Time-Zone Performance

Named-zone work can involve rule lookup and offset calculations.

For high-volume applications:

- avoid redundant conversions,
- reuse stable formatter/configuration objects,
- batch work where appropriate,
- convert at boundaries.

Do not create complicated caches before profiling.

---

# 74. Testing Strategy

Temporal systems need boundary-focused tests.

Include:

```text
normal date
month boundary
year boundary
leap day
DST gap
DST overlap
multiple zones
invalid input
precision conversion
serialization round-trip
Date interop
calendar behavior
```

---

# 75. DST Test Matrix

For each supported region, consider:

```text
before transition
at transition
after transition
ambiguous local value
nonexistent local value
next-day arithmetic
```

Do not test only UTC.

Representative examples can include:

```text
UTC
Asia/Kolkata
Asia/Tokyo
America/New_York
Europe/Berlin
Australia/Lord_Howe
```

Select the real matrix from your customer/domain requirements.

---

# 76. Deterministic Time in Tests

Inject a clock instead of reading current time everywhere.

```js
class AppClock {
  constructor(nowInstant) {
    this.nowInstant = nowInstant;
  }

  now() {
    return this.nowInstant;
  }
}
```

Then tests can use a fixed instant.

---

# 77. Clock Injection vs Global Mutation

Prefer:

```text
dependency injection
```

over global mutation.

A service depending on `clock.now()` is easier to:

- test,
- simulate,
- reason about,
- replay.

---

# 78. Debugging Workflow

When a temporal bug appears, ask in this order:

```text
1. What temporal domain is this value?
2. What information does it contain?
3. What information was discarded?
4. Which zone is involved?
5. Which calendar is involved?
6. Is the operation elapsed-time or calendar-time?
7. Was the local time ambiguous or nonexistent?
8. What serialization occurred?
9. What precision was lost?
10. What runtime/time-zone data version was involved?
```

This sequence catches conceptual mistakes before implementation details.

---

# 79. Debugging Exercise — Wrong Birthday Type

```js
const birthday = Temporal.Instant.from("1998-06-14T00:00:00Z");
```

Why can this be wrong?

Because a birthday is generally a calendar concept rather than one universal instant.

A better domain model may be:

```js
Temporal.PlainDate
```

or `Temporal.PlainMonthDay` when year is not part of the concept.

---

# 80. Debugging Exercise — Wrong Day Arithmetic

```js
const days = Math.floor(
  (end.epochMilliseconds - start.epochMilliseconds) / 86400000
);
```

Question:

> Does this always compute local calendar days?

No. It computes an elapsed-millisecond approximation.

---

# 81. Debugging Exercise — Zone Conversion

```js
const tokyo = kolkata.withTimeZone("Asia/Tokyo");
```

Why can the hour change?

Because the operation preserves the exact instant while changing the local representation.

---

# 82. Debugging Exercise — Precision Loss

```js
const instant = Temporal.Instant.from(
  "2026-09-10T18:00:00.123456789Z"
);

const date = new Date(Number(instant.epochMilliseconds));
```

What may be lost?

Sub-millisecond precision.

---

# 83. Code Review Exercise

Review:

```js
function isCouponValid(expiresAt) {
  return new Date() < new Date(expiresAt);
}
```

Potential problems:

- unclear input contract,
- implicit parsing,
- legacy semantics,
- hidden time-zone assumptions,
- untestable current-time dependency,
- repeated parsing,
- no explicit temporal domain.

---

# 84. Better Service Shape

```js
class CouponService {
  constructor(clock) {
    this.clock = clock;
  }

  isValid(expiresAt) {
    return Temporal.Instant.compare(
      this.clock.now(),
      expiresAt
    ) < 0;
  }
}
```

The type of the value and the source of current time are explicit.

---

# 85. Implementation — Guided

Build a small `TemporalEvent` value object.

```js
class TemporalEvent {
  constructor(instant, metadata = {}) {
    if (!(instant instanceof Temporal.Instant)) {
      throw new TypeError("instant must be Temporal.Instant");
    }

    this.instant = instant;
    this.metadata = { ...metadata };
  }
}
```

Add:

```text
toJSON()
fromJSON()
toZone()
isBefore()
isAfter()
```

---

# 86. Implementation — Partially Guided

Build:

```js
class LocalSchedule {
  constructor({ date, time, timeZone }) {
    // validate types
  }

  resolve(options = {}) {
    // resolve local date/time in zone
  }
}
```

Support explicit DST disambiguation.

---

# 87. Implementation — No Reference

Build a CLI:

```bash
node schedule.mjs add
node schedule.mjs next
node schedule.mjs inspect
node schedule.mjs convert
```

It should:

- parse Temporal values,
- resolve a zone,
- show local representation,
- show exact instant,
- expose ambiguity policy,
- serialize output.

---

# 88. Implementation — Edge-Case Hardened

Support and test:

```text
invalid zone
invalid date
invalid time
DST gap
DST overlap
precision conversion
calendar mismatch
Date interop
serialization failure
```

---

# 89. Implementation — Production Grade

A production temporal layer can look like:

```text
domain/
  temporal-clock.js
  temporal-parser.js
  temporal-serializer.js
  temporal-policy.js
  temporal-adapter.js
```

Architecture:

```text
external strings
      ↓
validation/parser
      ↓
Temporal value
      ↓
domain policy
      ↓
storage/wire boundary
```

Do not pass arbitrary date strings throughout the business domain.

---

# 90. Specification Perspective

Temporal is an excellent application of earlier specification chapters.

The specification defines and composes:

- built-in objects,
- internal slots,
- abstract operations,
- calendar behavior,
- time-zone behavior,
- parsing algorithms,
- rounding rules,
- conversion algorithms,
- serialization behavior.

Canonical specification:

https://tc39.es/proposal-temporal/

For exact semantics, read the algorithms rather than relying solely on examples.

---

# 91. Internal Data Model — Conceptual

Think of the values as:

```text
Instant
  → exact epoch-nanosecond quantity

PlainDate
  → calendar + date fields

PlainTime
  → time fields

PlainDateTime
  → calendar + date + time

ZonedDateTime
  → exact time + time-zone identifier + calendar

Duration
  → individual temporal unit fields
```

This is a reasoning model, not a replacement for the exact specification.

---

# 92. Abstract Operations

Temporal relies on numerous specification algorithms for:

```text
ISO date calculations
calendar operations
time-zone transitions
parsing
validation
rounding
duration balancing
comparison
conversion
```

Connect this chapter directly to:

```text
Chapter 42 — Abstract Operations
```

---

# 93. Calendar Protocol

Temporal supports calendar-aware behavior rather than hard-coding every concept as a Gregorian-only string manipulation.

For principal-level study, inspect:

```text
calendar identifiers
calendar slots
calendar operations
calendar protocol behavior
```

Then reason about how the built-ins obtain and validate calendar operations.

---

# 94. Time-Zone Protocol

Time-zone semantics require more than storing an offset.

They involve determining offsets and transitions for particular temporal contexts.

This is what makes the following possible:

```text
DST
historical transitions
future rule changes
ambiguous local times
nonexistent local times
```

---

# 95. Principal-Level Temporal Model

A production system benefits from four layers:

```text
Layer 1 — Domain meaning
"birthday", "payment captured", "due date"

Layer 2 — Temporal representation
PlainDate / Instant / ZonedDateTime / Duration

Layer 3 — Wire/storage representation
JSON / SQL / message schemas

Layer 4 — Presentation
Intl / locale / UI
```

Do not collapse all four into one helper object.

---

# 96. When Not to Use `ZonedDateTime`

Do not use the richest type by default.

If a field is simply:

```text
invoice date
```

then `PlainDate` is clearer.

If a field is:

```text
exact event timestamp
```

then `Instant` is clearer.

More information is not always better if it creates unnecessary coupling.

---

# 97. When Not to Use `Instant`

Do not turn:

```text
birthday
opening time
reporting month
```

into instants unless the business rule truly defines an instant.

Precision does not compensate for semantic mismatch.

---

# 98. When Not to Use `PlainDateTime`

Do not use `PlainDateTime` when the domain requires a unique point in time.

Typical exact-event fields:

```text
payment captured
request received
token expires
order created
```

These should normally be represented as exact timestamps.

---

# 99. When Not to Use `Duration`

Do not substitute:

```js
Temporal.Duration.from({ days: 30 })
```

for every “one month” business rule.

Calendar quantities and fixed elapsed quantities are different semantic classes.

---

# 100. Principal Selection Rule

Before choosing a Temporal type, ask:

```text
What fact does this field represent?

Does it identify an exact instant?
Does it identify a calendar date?
Does it identify a local time?
Does it represent local date + time?
Does it require a named zone?
Is it an amount rather than a point?
```

Then choose the type.

---

# 101. Production Policy

A useful internal rule can be:

```text
Exact event
→ Instant

Date-only business fact
→ PlainDate

Time-only business rule
→ PlainTime

Local date/time without zone
→ PlainDateTime

Local date/time tied to region
→ ZonedDateTime

Calendar or elapsed amount
→ Duration
```

The policy should be reviewed against actual product semantics.

---

# 102. API Review Questions

For every temporal API field, the reviewer should know:

```text
What does this field mean?
What Temporal type represents it?
What wire format is used?
What zone is authoritative?
What precision is guaranteed?
What happens at DST transitions?
What calendar assumptions exist?
```

If any answer is unclear, the API contract is incomplete.

---

# 103. Migration From `Date`

Migration should be semantic, not mechanical.

Workflow:

```text
1. Inventory Date usage
2. Classify each value by domain
3. Define API/database contracts
4. Introduce Temporal at boundaries
5. Migrate date-only logic
6. Migrate exact timestamps
7. Migrate local scheduling
8. Remove ambiguous helper utilities
```

Do not globally replace every `Date` with `Instant`.

---

# 104. Migration Inventory

Search for:

```text
new Date()
Date.now()
getTime()
setDate()
setMonth()
setHours()
toISOString()
toLocaleString()
/ 86400000
manual UTC offsets
```

Classify every occurrence.

---

# 105. Migration Example

Legacy:

```js
function getInvoiceDate() {
  return new Date().toISOString().slice(0, 10);
}
```

This derives a date through UTC formatting.

If the business rule is:

> today's date in the store's local zone

then the implementation can be semantically wrong.

A Temporal design should establish the relevant zone and derive the date explicitly.

---

# 106. Feature Detection and Compatibility

Because Temporal is Stage 4, standards risk is very different from early proposals. But target runtime support remains a deployment concern.

A basic capability check can be:

```js
const hasTemporal = typeof globalThis.Temporal !== "undefined";
```

Centralize compatibility policy rather than scattering checks throughout the application.

---

# 107. Compatibility Layer

Use:

```text
Application
   ↓
Temporal adapter
   ↓
native Temporal or supported fallback
```

This makes runtime migration explicit.

---

# 108. Polyfill Caveats

A polyfill provides compatibility for environments without native support, but it is not automatically identical to a native engine implementation in performance, integration, or every operational detail.

Before depending on one, verify:

```text
semantic conformance
precision
bundle impact
runtime support
tooling
migration path
```

---

# 109. Current Standards/Runtime Snapshot — 2026-09-10

Official sources currently show:

```text
Temporal proposal status: Stage 4 / finished
Expected publication year: 2026
Current spec draft: July 27, 2026
Firefox: Temporal shipped in Firefox 139
Chrome/V8: Temporal shipped in Chrome 144
Node.js: Temporal shipped in Node 26
```

These values are date-sensitive. Re-check the official sources before making a deployment decision.

---

# 110. Post-Stage-4 Reality

Stage 4 does not mean all ecosystem work has ended.

The Temporal repository continues to contain post-Stage-4 work involving issues such as:

- Test262 coverage,
- documentation,
- web-platform integration,
- specification-related changes,
- interoperability with related standards work.

The correct interpretation is:

```text
standards completion
≠
ecosystem completion
```

---

# 111. Principal Challenge — Global Commerce Platform

Design a temporal architecture for a platform with:

```text
customers in 20+ time zones
user birthdays
order timestamps
local store hours
scheduled deliveries
recurring subscriptions
payment expirations
audit logs
SLA measurements
monthly reports
internationalized UI
```

Your design must define:

```text
Temporal type per field
API schema
Database schema
Time-zone ownership
DST policy
Calendar policy
Precision guarantees
Serialization
Clock abstraction
Monotonic timing
Testing
Time-zone data updates
Migration from Date
Observability
Security
Rollback behavior
```

---

# 112. Principal Decision Matrix

| Requirement | Preferred model | Reason |
|---|---|---|
| Audit event | `Instant` | exact point |
| User birthday | `PlainMonthDay` / `PlainDate` | calendar concept |
| Invoice due date | `PlainDate` | date only |
| Store opening time | `PlainTime` | daily clock value |
| User-entered local date/time | `PlainDateTime` | no zone yet |
| Meeting in named region | `ZonedDateTime` | local + zone + exact time |
| Request latency | monotonic timer | elapsed measurement |
| Subscription renewal | rule + local time + zone | calendar semantics |
| Reporting month | `PlainYearMonth` | month period |

---

# 113. Principal Anti-Patterns

## Everything is `Instant`

Invents instants for values that do not have one.

## Everything is `ZonedDateTime`

Adds unnecessary zone semantics.

## Everything is a string

Defers semantic validation.

## Everything is milliseconds

Confuses exact time with all temporal quantities.

## Offset-only storage

Can lose named-zone meaning.

## Server-local time as business time

Makes behavior environment-dependent.

---

# 114. Common Misconceptions

### “Temporal is just a wrapper around Date.”

No. It provides distinct temporal types and explicit semantics.

### “UTC solves every date problem.”

UTC is excellent for exact instants, not a replacement for calendar and civil-time concepts.

### “One day is always 24 hours.”

False for some local civil-time calculations.

### “A local timestamp is enough.”

Not when a unique instant or named zone is required.

### “Temporal means every date bug disappears.”

No. Ambiguous business requirements can still produce ambiguous software.

---

# 115. Interview Questions — Fundamentals

1. What problem does Temporal solve?
2. Why is `Date` insufficient for many modern domains?
3. What is `Temporal.Instant`?
4. What is `Temporal.PlainDate`?
5. What is `Temporal.ZonedDateTime`?
6. What is `Temporal.Duration`?
7. Why are Temporal objects immutable?
8. What is exact time?
9. What is wall-clock time?
10. What is a named time zone?

---

# 116. Interview Questions — Senior

1. Why is `09:00 America/New_York` different from “every 24 hours”?
2. What happens during a DST gap?
3. What happens during a DST overlap?
4. When would you use `PlainDateTime` rather than `Instant`?
5. Why should birthdays usually be dates rather than instants?
6. Why does `Date` interop risk precision loss?
7. What information is lost when converting a `ZonedDateTime` to `PlainDateTime`?
8. Why should distributed logs use exact timestamps?
9. Why is `performance.now()` still relevant?
10. How would you migrate a Date-heavy codebase?

---

# 117. Interview Questions — Principal

1. Design a temporal model for a global commerce platform.
2. How would you model “every weekday at 09:00 customer local time”?
3. How would you store an audit event and its civil-time context?
4. How would you handle DST gaps and overlaps in a booking system?
5. How would you respond to a time-zone database update?
6. How would you guarantee precision across DB/API boundaries?
7. How would you test scheduling across many zones?
8. When would you reject a Stage 4 Temporal deployment?
9. How would you build a Temporal compatibility layer?
10. Why does “UTC everywhere” fail to model every business date/time requirement?

---

# 118. Predict-the-Result Exercises

Predict before reading your notes.

## Exercise 1 — Immutability

```js
const d = Temporal.PlainDate.from("2026-09-10");
const next = d.add({ days: 1 });

console.log(d.toString());
console.log(next.toString());
```

Expected:

```text
2026-09-10
2026-09-11
```

Rule:

```text
Temporal values are immutable.
```

## Exercise 2 — Same Instant, Different Zone

```js
const instant = Temporal.Instant.from("2026-09-10T18:00Z");

console.log(instant.toZonedDateTimeISO("Asia/Kolkata").hour);
console.log(instant.toZonedDateTimeISO("Asia/Tokyo").hour);
```

Predict first, then execute. The important invariant is that the instant remains identical.

## Exercise 3 — Plain Date

```js
const date = Temporal.PlainDate.from("2026-09-10");
console.log(date.toString());
```

Expected:

```text
2026-09-10
```

No time zone is invented.

---

# 119. Mastery Exercise — Temporal Type Classifier

Implement:

```js
function chooseTemporalType(requirement) {
  // return recommended temporal domain
}
```

Test it with:

```text
customer birthday
payment captured
store opening time
meeting local date/time
meeting date/time in New York
subscription month
request elapsed time
invoice due date
```

Defend each choice.

---

# 120. Mastery Exercise — Global Scheduler

Build a scheduler accepting:

```json
{
  "date": "2026-11-01",
  "time": "09:00",
  "timeZone": "America/New_York"
}
```

It must:

```text
validate
resolve
report ambiguity
show Instant
serialize
```

Add transition-specific tests.

---

# 121. Mastery Exercise — Legacy Migration

Take 20 `Date` usages from a real or sample project.

Classify each as:

```text
Instant
PlainDate
PlainTime
PlainDateTime
ZonedDateTime
Duration
not actually temporal domain logic
```

Rewrite five and document the information gained/lost.

---

# 122. Mastery Exercise — Storage Model

Design mappings for:

```text
birthday
orderCreatedAt
storeOpeningTime
subscriptionRenewal
monthlyReport
requestLatency
```

For every field specify:

```text
Temporal type
DB type
wire type
zone requirement
precision
```

---

# 123. Mastery Exercise — Time-Zone Update Drill

Imagine the time-zone rules for a region change.

Answer:

```text
Which stored records remain fixed?
Which schedules must be recomputed?
Which audit records must remain immutable?
Which future events can change local interpretation?
```

---

# 124. Spaced Retrieval Schedule

### Day 0

Memorize the Temporal type model.

### Day 1

Explain `Instant` vs `ZonedDateTime`.

### Day 3

Explain DST gap/overlap behavior.

### Day 7

Explain elapsed time vs calendar time.

### Day 14

Design storage for five temporal domains.

### Day 30

Review a `Date` migration.

### Day 60

Build a global scheduler without notes.

### Day 90

Defend a principal-level temporal architecture.

---

# 125. Retrieval Prompts

Answer without notes:

1. Why is `Date` insufficient?
2. What is `Instant`?
3. What is `PlainDate`?
4. What is `PlainTime`?
5. What is `PlainDateTime`?
6. What is `ZonedDateTime`?
7. What is `Duration`?
8. Why are Temporal values immutable?
9. What is a DST gap?
10. What is a DST overlap?
11. Why is one local day not always 24 elapsed hours?
12. Why is `withTimeZone()` different from changing local fields?
13. Why does `Instant` not contain a zone?
14. Why is a monotonic timer useful?
15. How would you migrate a legacy scheduling service?

---

# 126. Dependency Graph

```text
Chapter 03 — Numbers / BigInt
          │
          └────→ Instant precision

Chapter 04 — Strings / Unicode
          │
          └────→ Temporal parsing / serialization

Chapter 28 — JSON / Serialization
          │
          └────→ Temporal wire formats

Chapter 41 — Spec Architecture
Chapter 42 — Abstract Operations
          │
          └────→ Temporal specification reasoning

Chapter 90 — Modern ECMAScript
Chapter 91 — TC39 Proposal Tracking
          │
          └────→ Chapter 92 — Temporal
                            │
                            ├────→ Chapter 93 — Decorators
                            └────→ Chapter 94 — Compatibility Engineering
```

---

# 127. Concept Connections

## Depends On

- Numbers and `BigInt`.
- Strings and parsing.
- Objects and internal slots.
- Serialization.
- ECMAScript specification.
- TC39 proposal process.
- `Intl`.

## Builds Toward

- Compatibility engineering.
- API design.
- Database integration.
- Reliability.
- Performance.
- Testing.
- Production JavaScript architecture.

## Related Concepts

- time zones,
- DST,
- calendars,
- localization,
- distributed clocks,
- monotonic timers,
- scheduling.

## Concepts Revisited

- immutability,
- abstract operations,
- domain modeling,
- error handling,
- standardized serialization.

## Why This Chapter Matters Later

Date/time bugs are usually domain-modeling bugs rather than syntax bugs.

Temporal gives JavaScript a richer vocabulary for expressing the domain directly.

---

# 128. Production Checklist

```text
[ ] Every temporal field has an explicit semantic type
[ ] Wire format documented
[ ] Storage precision documented
[ ] Time-zone ownership documented
[ ] DST behavior tested
[ ] Calendar assumptions documented
[ ] Current-time access injectable
[ ] Monotonic timing used for latency where appropriate
[ ] Serialization round-trips tested
[ ] Date interoperability tested
[ ] Runtime baseline verified
[ ] Tooling baseline verified
[ ] Time-zone data update process documented
[ ] Security validation performed
[ ] Rollback strategy defined
```

---

# 129. Principal Review Template

```md
# Temporal Design Review

## Domain
- Field:
- Business meaning:

## Representation
- Temporal type:
- Why:

## Time Zone
- Required?:
- Source of authority:

## Calendar
- Calendar:
- Business rules:

## Precision
- Source precision:
- Storage precision:
- Wire precision:

## Arithmetic
- Calendar or elapsed:
- Rounding policy:

## DST
- Gap policy:
- Overlap policy:

## Testing
- Zones:
- Boundaries:
- Serialization:

## Security
- Input validation:
- Expiry/replay policy:

## Operational
- Runtime baseline:
- Time-zone data update policy:
- Rollback:
```

---

# 130. Final Mental Model

```text
Instant
    = exact point

PlainDate
    = calendar date

PlainTime
    = clock time

PlainDateTime
    = local date + time

ZonedDateTime
    = exact time + named zone + calendar

Duration
    = temporal quantity
```

Then ask:

```text
Am I measuring elapsed time?
Am I describing a calendar date?
Am I describing a local schedule?
Am I identifying an exact event?
Am I preserving named time-zone context?
```

The correct Temporal type follows from the answer.

---

# 131. Final Principal Rule

> **Do not start with the Temporal API. Start with the temporal fact.**

First define:

```text
What does this field mean?
```

Then define:

```text
Temporal type
conversion rules
storage
serialization
time-zone policy
calendar policy
precision
tests
```

Only then write the code.

---

# 132. Chapter Quality Gate

```text
[ ] All core Temporal types understood
[ ] Exact vs wall-clock distinction understood
[ ] DST gap/overlap explained
[ ] Calendar vs elapsed arithmetic understood
[ ] Precision model understood
[ ] Date interoperability understood
[ ] API/storage design practiced
[ ] Deterministic clock strategy practiced
[ ] Scheduler implemented
[ ] DST tests implemented
[ ] One `Date` migration completed
[ ] Principal temporal design defended
```

---

# Chapter 92 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I explain all Temporal types without notes? [ ]
- Could I distinguish Instant vs ZonedDateTime? [ ]
- Could I distinguish PlainDate vs PlainDateTime? [ ]
- Could I explain DST gaps and overlaps? [ ]
- Could I explain calendar vs elapsed arithmetic? [ ]
- Could I explain precision loss through Date? [ ]
- Could I design temporal API fields? [ ]
- Could I design DB mappings? [ ]
- Could I debug a timezone bug? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 92 — Canonical References and Source Discipline

Primary references:

1. Temporal proposal repository — https://github.com/tc39/proposal-temporal
2. Temporal specification — https://tc39.es/proposal-temporal/
3. TC39 finished proposals — https://github.com/tc39/proposals/blob/main/finished-proposals.md
4. TC39 process — https://tc39.es/process-document/
5. Temporal documentation — https://github.com/tc39/proposal-temporal/tree/main/docs
6. Temporal Instant documentation — https://github.com/tc39/proposal-temporal/blob/main/docs/instant.md
7. Test262 — https://github.com/tc39/test262
8. ECMAScript specification — https://tc39.es/ecma262/
9. ECMA-402 — https://tc39.es/ecma402/
10. MDN Temporal reference — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal

### Verification Notes — 2026-09-10

- TC39's finished-proposals list identifies Temporal as finished / Stage 4 with expected publication year 2026.
- The Temporal repository identifies the proposal as Stage 4.
- The current Temporal specification page is labeled “Stage 4 Draft / July 27, 2026”.
- The Temporal repository currently documents shipping support in Firefox 139, Chrome 144, and Node.js 26.
- The Temporal repository continues to contain post-Stage-4 issues and integration work.

Because time-zone, runtime, and ecosystem state can change, re-check primary sources before production adoption.

---

# Chapter 92 — Completion Snapshot

```text
Track A — Core Theory
[ ] Explain the Temporal domain model
[ ] Explain every core Temporal type
[ ] Explain exact vs wall-clock time
[ ] Explain named time zones
[ ] Explain DST gaps and overlaps
[ ] Explain calendars
[ ] Explain precision
[ ] Explain Temporal specification architecture

Track B — Implementation
[ ] Build Temporal parsing layer
[ ] Build temporal API boundary
[ ] Build scheduler
[ ] Add DST tests
[ ] Add serialization
[ ] Add Date interoperability
[ ] Add time-zone handling
[ ] Build production adapter

Track C — Interview / Reasoning
[ ] Select correct Temporal type
[ ] Defend temporal data model
[ ] Explain DST correctness
[ ] Compare elapsed vs calendar arithmetic
[ ] Explain migration from Date
[ ] Design storage mappings
[ ] Design global scheduling
[ ] Defend principal-level architecture

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# Completion Criteria

Do not mark this chapter mastered because you can repeat the type names.

You are ready to move forward when you can independently:

1. Classify a temporal requirement by domain.
2. Select the correct Temporal type.
3. Explain why the selected type is semantically correct.
4. Explain exact time versus local date/time.
5. Explain why named zones are richer than offsets.
6. Explain DST gaps and overlaps.
7. Explain calendar arithmetic versus elapsed arithmetic.
8. Explain why monotonic clocks remain relevant.
9. Design a temporal JSON/API schema.
10. Design a database mapping.
11. Design deterministic temporal tests.
12. Debug precision loss.
13. Debug zone-conversion mistakes.
14. Migrate legacy `Date` logic safely.
15. Explain Temporal specification concepts at an abstract-operation level.
16. Evaluate runtime/tooling compatibility before production adoption.
17. Defend a global scheduling architecture.

---

# Principal Challenge

Produce a **Principal Temporal Architecture Review** for a global commerce platform.

Your review must defend:

```text
Temporal type choices
API contracts
DB mapping
Time-zone ownership
DST policies
Calendar policies
Precision
Serialization
Testing
Clock abstraction
Time-zone data update strategy
Migration from Date
Observability
Security
Performance
Rollback
```

Final verdict:

```text
CORRECT
SAFE
PERFORMANT
MAINTAINABLE
OBSERVABLE
SCALABLE
```

---

# Compact Reference Card

```text
Temporal.Instant
= exact timeline point
= nanosecond-scale exact representation
= no zone/calendar attached

Temporal.ZonedDateTime
= instant + named zone + calendar

Temporal.PlainDateTime
= local date + time
= no zone

Temporal.PlainDate
= calendar date

Temporal.PlainTime
= clock time

Temporal.PlainYearMonth
= year + month

Temporal.PlainMonthDay
= month + day

Temporal.Duration
= temporal amount

Business event?
→ Instant

Calendar date?
→ PlainDate

Clock time?
→ PlainTime

Local date/time?
→ PlainDateTime

Local date/time + named zone?
→ ZonedDateTime

Amount/period?
→ Duration

Latency?
→ monotonic timer
```

---

## File Metadata

```text
Chapter: 92
Title: Temporal
Part: XVII — Modern ECMAScript & Language Evolution
Format: Standalone Markdown chapter
Verification date: 2026-09-10
Standards state verified: Stage 4
Status: [ ] Not Started
```

> **Mastery reminder:** Reading this chapter does not mark it complete. You must retrieve, implement, debug, apply, compare, and defend the concepts.