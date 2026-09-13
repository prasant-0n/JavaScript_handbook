# Chapter 137 — Browser Performance APIs & Runtime Instrumentation

> **JavaScript Mastery — Part XXIII: Browser Platform & Client State**
>
> **Mission:** Master browser performance measurement as an engineering discipline: Performance Timeline, high-resolution timing, navigation timing, resource timing, user timing, paint timing, Largest Contentful Paint, First Input Delay/Interaction to Next Paint concepts, Long Tasks, event-loop responsiveness, PerformanceObserver, layout/paint indicators, memory diagnostics, network attribution, profiling strategy, production telemetry, sampling, privacy, and actionable performance budgets.
>
> **Role perspective:** Principal JavaScript Engineer · Browser Performance Engineer · Runtime Engineer · Observability Architect · Frontend Platform Engineer · SRE · Web Vitals Specialist
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Measure the user's experience and the causal chain behind it—not just “how long a function took.” Performance instrumentation should tell you where time went, how often it happens, which users are affected, and whether the result violates a product objective.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain browser performance instrumentation
[ ] distinguish wall-clock timing from monotonic performance timing
[ ] explain performance.now()
[ ] explain the Performance Timeline
[ ] explain PerformanceEntry
[ ] explain PerformanceObserver
[ ] explain User Timing API
[ ] explain performance.mark()
[ ] explain performance.measure()
[ ] explain clearMarks()
[ ] explain clearMeasures()
[ ] explain Navigation Timing
[ ] explain Resource Timing
[ ] explain Paint Timing
[ ] explain Largest Contentful Paint
[ ] explain First Input Delay historically
[ ] explain Interaction to Next Paint
[ ] distinguish lab metrics from field metrics
[ ] explain Long Tasks
[ ] explain event-loop responsiveness
[ ] explain task duration
[ ] explain rendering opportunities
[ ] explain first paint
[ ] explain first contentful paint
[ ] explain layout shifts at a high level
[ ] understand Cumulative Layout Shift conceptually
[ ] explain network timing phases
[ ] explain DNS timing
[ ] explain connection timing
[ ] explain TLS timing
[ ] explain request timing
[ ] explain response timing
[ ] explain transferSize
[ ] explain encodedBodySize
[ ] explain decodedBodySize
[ ] explain resource timing buffer
[ ] explain cross-origin timing restrictions
[ ] explain Timing-Allow-Origin
[ ] explain navigationStart concepts
[ ] explain document load milestones
[ ] explain redirects
[ ] explain cache timing
[ ] explain worker timing entries
[ ] explain server timing
[ ] explain Server-Timing
[ ] explain performance resource marks
[ ] explain user journey instrumentation
[ ] design custom spans
[ ] design business-performance metrics
[ ] define performance budgets
[ ] define latency budgets
[ ] define responsiveness budgets
[ ] define memory budgets
[ ] define network budgets
[ ] explain sampling
[ ] explain aggregation
[ ] explain percentile metrics
[ ] explain p50
[ ] explain p75
[ ] explain p95
[ ] explain p99
[ ] understand why averages can mislead
[ ] understand population segmentation
[ ] understand device segmentation
[ ] understand network segmentation
[ ] understand route/page segmentation
[ ] explain RUM
[ ] explain synthetic monitoring
[ ] compare RUM and lab measurement
[ ] design production instrumentation
[ ] avoid high-overhead instrumentation
[ ] understand measurement perturbation
[ ] understand observer overhead
[ ] explain profiling vs metrics
[ ] explain tracing vs profiling
[ ] explain flame charts conceptually
[ ] explain Performance DevTools
[ ] explain CPU profiling
[ ] explain memory profiling
[ ] explain allocation profiling
[ ] explain heap snapshots conceptually
[ ] explain garbage-collection effects on performance
[ ] explain main-thread blocking
[ ] explain long JavaScript tasks
[ ] explain forced synchronous layout
[ ] explain layout thrashing
[ ] explain paint/compositing costs
[ ] explain script parsing/compilation cost
[ ] explain module/loading cost
[ ] connect network timing to code execution
[ ] connect frontend timing to backend timing
[ ] explain Server-Timing correlation
[ ] build performance marks
[ ] build production performance measures
[ ] build an event-loop responsiveness monitor
[ ] build a long-task observer
[ ] build a navigation telemetry collector
[ ] build a resource performance collector
[ ] build a user-journey performance trace
[ ] design privacy-safe telemetry
[ ] design sampling and rate limiting
[ ] test instrumentation
[ ] validate instrumentation correctness
[ ] avoid measuring the wrong lifecycle
[ ] distinguish correlation from causation
[ ] build performance regression gates
[ ] decide what should be measured in production


# 2. Prerequisites

You should already understand:

```text
Chapter 03 — Numbers / Floating Point
Chapter 31 — Async Fundamentals
Chapter 33 — Browser Event Loop
Chapter 49 — DOM Architecture
Chapter 50 — Browser Events
Chapter 51 — Browser Web APIs
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams / Data Flow
Chapter 55 — Fetch / HTTP Networking
Chapter 63 — Async Context / Diagnostics
Chapter 70 — Source Maps / Production Debugging
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging Methodology
Chapter 101 — Real-World Production Scenarios
Chapter 130 — Date / Clock Semantics
Chapter 132 — Browser Storage
Chapter 133 — Service Workers
Chapter 134 — Web Locks / Cross-Tab Coordination
Chapter 135 — WebRTC / P2P JavaScript
Chapter 136 — WebTransport / Modern Web Networking
```

Supporting concepts:

```text
CPU
memory
networking
rendering
event loop
GC
profiling
telemetry
percentiles
sampling
```

---

# 3. What Is Browser Performance Instrumentation?

Performance instrumentation is the deliberate collection of timing and runtime signals that explain:

```text
how fast
how often
for whom
where
why
```

an experience behaves.

A mature system measures:

```text
navigation
resources
JavaScript
rendering
interaction
network
backend
user journeys
```

not just:

```text
function execution time.
```

---

# 4. Why Performance APIs Exist

A browser contains many stages:

```text
DNS
TLS
HTTP
response
parsing
compilation
script execution
style
layout
paint
compositing
input
event handlers
animation
```

Without instrumentation:

```text
“the page feels slow”
```

is difficult to diagnose.

Performance APIs expose:

```text
structured timeline data
```

that can connect:

```text
user experience
→ browser phase
→ application operation.
```

---

# 5. Monotonic Performance Time

For elapsed measurements, browser performance timing uses a monotonic time model.

Typical API:

```js
performance.now();
```

This is different from:

```js
Date.now();
```

because `Date.now()` represents wall-clock time while `performance.now()` is intended for precise elapsed-time measurement. citeturn701019search1turn701019search6

---

# 6. Why Monotonic Matters

Suppose:

```text
wall clock:
10:00:00
10:00:01
clock correction
09:59:58
```

A duration computed from wall time can become nonsensical.

A monotonic performance clock provides:

```text
elapsed-time measurement
```

without depending on:

```text
civil-time clock corrections.
```

---

# 7. Performance Timeline

The Performance Timeline provides standardized performance entries such as:

```text
navigation
resource
mark
measure
paint
longtask
```

Applications can inspect entries through:

```js
performance.getEntries()
performance.getEntriesByType("resource")
performance.getEntriesByName("app-start")
```

and observe entries using:

```text
PerformanceObserver.
```

---

# 8. PerformanceEntry

A performance entry conceptually contains:

```text
name
entryType
startTime
duration
```

Some entry types provide additional fields.

Think:

```text
PerformanceEntry
    ↓
specific performance record
```

---

# 9. `startTime`

`startTime` is expressed relative to the performance timeline's time origin.

This allows:

```text
navigation
+
marks
+
resources
```

to be correlated on one timeline.

---

# 10. Time Origin

Each relevant browsing context has a:

```text
timeOrigin
```

which anchors:

```text
performance.now()
performance entry startTime
```

The exact relationship is defined by the performance timing model.

Use:

```js
performance.timeOrigin
```

when converting relative timing to an absolute-ish wall-clock correlation carefully.

---

# 11. `performance.now()`

Example:

```js
const start = performance.now();

runExpensiveWork();

const elapsed = performance.now() - start;

console.log(elapsed);
```

Use this for:

```text
elapsed measurement
```

not:

```text
business timestamps
```

---

# 12. Why Not `Date.now()` for Benchmarking?

Bad:

```js
const start = Date.now();
work();
const elapsed = Date.now() - start;
```

Problems include:

```text
lower resolution
wall-clock adjustment
less precise measurement
```

Use:

```js
performance.now();
```

for browser performance timing.

---

# 13. User Timing

User Timing lets your application create custom measurements.

Basic APIs:

```js
performance.mark("checkout-start");
performance.mark("checkout-end");

performance.measure(
  "checkout",
  "checkout-start",
  "checkout-end"
);
```

This turns:

```text
business operation
```

into:

```text
performance timeline entry.
```

---

# 14. Why Marks Matter

A browser can tell you:

```text
navigation timing
```

but not automatically:

```text
checkout flow duration
```

Application marks bridge:

```text
browser
+
application domain.
```

---

# 15. Naming Marks

Prefer:

```text
checkout:start
checkout:data-ready
checkout:rendered
checkout:complete
```

rather than:

```text
start1
start2
foo
```

Good naming improves:

```text
queryability
debugging
dashboards
```

---

# 16. Measures

A measure represents:

```text
duration between timing points.
```

Example:

```js
performance.mark("cart:start");

await loadCart();

performance.mark("cart:loaded");

performance.measure(
  "cart:load",
  "cart:start",
  "cart:loaded"
);
```

---

# 17. Measure Detail

Where supported, measures can include:

```text
detail
```

metadata.

Keep detail:

```text
small
non-sensitive
high-cardinality-aware.
```

Do not stuff:

```text
user objects
```

into performance entries.

---

# 18. Clearing Entries

Long-lived applications can accumulate marks/measures.

Use:

```js
performance.clearMarks();
performance.clearMeasures();
```

or targeted names.

Production instrumentation should avoid:

```text
unbounded performance entry accumulation.
```

---

# 19. PerformanceObserver

`PerformanceObserver` lets applications react to performance entries as they are recorded.

Example:

```js
const observer =
  new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
      console.log(entry);
    }
  });

observer.observe({
  entryTypes: ["measure"]
});
```

The observer model is preferable when:

```text
continuous monitoring
```

is needed. citeturn701019search4turn701019search9

---

# 20. Observer Overhead

Instrumentation itself has a cost.

Poor instrumentation can create:

```text
extra allocations
extra callbacks
extra serialization
extra network
```

Therefore:

```text
measure enough
```

but not:

```text
everything at full fidelity.
```

---

# 21. Buffered Observation

Some performance entry types support:

```text
buffered observation
```

so an observer can receive entries recorded before it was attached.

This is important for:

```text
navigation
paint
early lifecycle events.
```

---

# 22. `PerformanceObserver` Support

Different entry types may have different browser support.

Use:

```js
PerformanceObserver.supportedEntryTypes
```

where available to detect supported entry categories.

Do not assume:

```text
every browser
```

supports:

```text
every PerformanceEntry type.
```

---

# 23. Navigation Timing

Navigation Timing provides data about:

```text
document navigation lifecycle
```

through:

```js
performance.getEntriesByType("navigation");
```

A navigation entry can help analyze:

```text
redirects
DNS
connection
TLS
request
response
DOM processing
load events.
```

---

# 24. Navigation Timeline

Conceptually:

```text
navigation
 ↓
redirect
 ↓
DNS
 ↓
TCP
 ↓
TLS
 ↓
request
 ↓
response
 ↓
DOM
 ↓
load
```

Not every phase occurs for every navigation.

For example:

```text
cached connection
```

may make a timing phase effectively zero/absent.

---

# 25. Navigation Performance Diagnosis

Suppose:

```text
TTFB high
DOMContentLoaded fast
```

Potential issue:

```text
server/network.
```

Suppose:

```text
TTFB low
DOMContentLoaded high
```

Potential issue:

```text
client-side parsing/execution.
```

Suppose:

```text
DOMContentLoaded low
LCP high
```

Potential issue:

```text
rendering/resource bottleneck.
```

These are diagnostic hypotheses—not proof.

---

# 26. Resource Timing

Resource Timing measures fetches for resources such as:

```text
scripts
stylesheets
images
fonts
fetch/XHR
```

Example:

```js
performance.getEntriesByType("resource");
```

Useful fields include:

```text
startTime
duration
name
initiatorType
responseStart
responseEnd
transferSize
encodedBodySize
decodedBodySize
```

---

# 27. Resource Initiator Type

Examples can identify:

```text
script
link
img
fetch
xmlhttprequest
css
```

This allows segmentation:

```text
slow images
slow APIs
slow JS
slow fonts
```

---

# 28. DNS Timing

Resource entries can expose DNS timing where applicable.

Conceptually:

```text
domain lookup
```

can contribute to:

```text
request startup latency.
```

A slow DNS phase can indicate:

```text
DNS resolver
network
cold connection
```

issues.

---

# 29. Connection Timing

Timing can include:

```text
connectStart
connectEnd
```

where applicable.

Use this to reason about:

```text
connection establishment
```

rather than assuming:

```text
server response latency
```

explains all network delay.

---

# 30. TLS Timing

For secure resources, connection timing can include:

```text
secureConnectionStart
```

and related phases.

This allows analysis of:

```text
TLS handshake cost
```

especially for:

```text
new connections
```

---

# 31. Request Timing

Key phases include:

```text
requestStart
responseStart
responseEnd
```

From these you can derive:

```text
request transmission
TTFB-like interval
response transfer time.
```

---

# 32. Transfer Size

Resource timing can expose:

```text
transferSize
```

which can help distinguish:

```text
network bytes
```

from:

```text
decoded resource size.
```

Be careful interpreting:

```text
zero
```

because caching and browser privacy rules can affect exposed values.

---

# 33. Encoded vs Decoded Body Size

```text
encodedBodySize
```

reflects the encoded response body size.

```text
decodedBodySize
```

reflects decoded size.

This helps reason about:

```text
compression ratio
```

and:

```text
download vs memory footprint.
```

---

# 34. Resource Timing Buffer

Browsers maintain a Resource Timing buffer.

If too many entries are collected without management:

```text
new observations
```

may be affected by buffer limits.

Applications can use:

```js
performance.setResourceTimingBufferSize(...)
```

and:

```js
performance.clearResourceTimings()
```

where appropriate.

---

# 35. Cross-Origin Resource Timing

Cross-origin resource timing data may be restricted unless the remote server opts in appropriately.

A key mechanism is:

```text
Timing-Allow-Origin
```

which controls exposure of detailed timing information for cross-origin resources.

This is a:

```text
privacy/security boundary.
```

---

# 36. Why Timing Is Sensitive

Detailed cross-origin timing can reveal:

```text
resource state
cache state
network behavior
user-specific differences.
```

Therefore browsers do not expose unlimited cross-origin timing detail.

---

# 37. Paint Timing

Paint Timing exposes key rendering milestones such as:

```text
first-paint
first-contentful-paint
```

Example:

```js
performance.getEntriesByType("paint");
```

These help measure:

```text
when pixels begin reaching the user.
```

---

# 38. First Paint

First Paint conceptually represents:

```text
first browser paint after navigation.
```

It may include:

```text
background
non-content pixels.
```

Do not confuse:

```text
first paint
```

with:

```text
useful content.
```

---

# 39. First Contentful Paint

FCP measures when:

```text
first content from the page is painted.
```

This is often more useful than:

```text
first paint.
```

for perceived startup.

---

# 40. Largest Contentful Paint

LCP attempts to capture when the largest relevant content element becomes visible within the viewport during loading.

It is widely used as a:

```text
user experience metric.
```

LCP is a field metric rather than a direct measure of one function's execution time.

---

# 41. LCP Is Not a Network Metric

A poor LCP can come from:

```text
slow server
slow image
render-blocking CSS
JavaScript
layout
font
resource discovery
main-thread work.
```

Therefore:

```text
LCP
```

must be decomposed into its causal chain.

---

# 42. Largest Contentful Paint Attribution

A useful mental pipeline:

```text
navigation
→ resource discovery
→ resource request
→ resource download
→ decode
→ main-thread availability
→ render
→ LCP
```

The metric gives:

```text
outcome
```

while timing entries help investigate:

```text
cause.
```

---

# 43. Interaction Responsiveness

Modern browser performance emphasizes:

```text
interaction responsiveness
```

rather than only page load.

A user may see:

```text
page loaded
```

yet experience:

```text
button freezes
input lag
```

because:

```text
main thread is busy.
```

---

# 44. First Input Delay

First Input Delay was an important historical responsiveness metric.

It measured:

```text
delay from first user interaction
→ event handler begins processing.
```

It has been superseded in Core Web Vitals by:

```text
Interaction to Next Paint (INP).
```

Use current Core Web Vitals definitions for modern product monitoring.

---

# 45. Interaction to Next Paint

INP evaluates responsiveness across user interactions over the page lifecycle.

Conceptually:

```text
interaction
→ event processing
→ rendering
→ next visible update
```

This captures:

```text
user-perceived responsiveness.
```

Current Core Web Vitals guidance identifies INP as the responsiveness metric and LCP/CLS alongside it for loading and visual stability. citeturn701019search7turn701019search10

---

# 46. Long Tasks

A Long Task is a main-thread task that blocks the browser long enough to be considered harmful to responsiveness.

A common threshold is:

```text
> 50 ms
```

The Long Tasks API exposes relevant entries where supported. citeturn701019search0turn701019search3

---

# 47. Why 50 ms Matters

Humans may perceive:

```text
input lag
animation jank
page freeze
```

when the browser cannot respond promptly.

The 50 ms threshold is a useful diagnostic boundary, not:

```text
“every 49 ms task is fine.”
```

---

# 48. Long Task Observer

Example:

```js
const observer =
  new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
      console.log("Long task:", entry.duration);
    }
  });

observer.observe({
  type: "longtask",
  buffered: true
});
```

Use feature detection because:

```text
Long Tasks support
```

can vary.

---

# 49. Attributing Long Tasks

A long task tells you:

```text
main-thread work was long.
```

It does not automatically tell you:

```text
which business operation caused it.
```

Add:

```text
User Timing marks
```

around important application operations.

---

# 50. Combined Instrumentation

Example:

```js
performance.mark("search:start");

renderLargeResultSet();

performance.mark("search:end");

performance.measure(
  "search:render",
  "search:start",
  "search:end"
);
```

Then combine with:

```text
Long Task entries.
```

This reveals:

```text
business operation
+
main-thread blocking
```

together.

---

# 51. Event Loop Responsiveness

A simple responsiveness monitor can periodically schedule work:

```js
setTimeout(() => {
  const delay = performance.now() - scheduledAt;
}, 0);
```

Large delay can indicate:

```text
main-thread contention.
```

But timer delay is affected by:

```text
browser scheduling
throttling
background state.
```

Do not treat it as a perfect event-loop utilization metric.

---

# 52. Long Task API vs Timer Probe

### Long Task API

```text
browser-observed long main-thread tasks
```

### Timer probe

```text
application-observed scheduling delay
```

Both are useful:

```text
different perspectives
```

on responsiveness.

---

# 53. Layout Thrashing

A common anti-pattern:

```js
for (const item of items) {
  item.style.width = ...;
  console.log(item.offsetWidth);
}
```

This can alternate:

```text
write
→ forced layout
→ write
→ forced layout
```

creating:

```text
layout thrashing.
```

---

# 54. Batch DOM Writes

Prefer:

```text
read phase
→ calculate
→ write phase
```

rather than:

```text
read/write/read/write
```

This reduces unnecessary synchronous layout work.

---

# 55. Forced Synchronous Layout

Reading layout properties such as:

```text
offsetWidth
offsetHeight
getBoundingClientRect()
```

can require up-to-date layout.

If preceded by DOM/style mutations, the browser may need to:

```text
flush pending style/layout.
```

This can create unexpected main-thread cost.

---

# 56. Performance Instrumentation for Layout

Use:

```text
DevTools Performance panel
```

and:

```text
User Timing marks
```

to correlate:

```text
application action
→ layout
→ paint.
```

---

# 57. Memory and Performance

Memory pressure can affect:

```text
GC
allocation
page responsiveness
tab survival.
```

Performance instrumentation should sometimes correlate:

```text
slow interaction
+
allocation spike
+
GC pause.
```

Not every browser exposes identical memory metrics.

---

# 58. `performance.measureUserAgentSpecificMemory`

Some browsers have exposed experimental/user-agent-specific memory measurement facilities over time.

Treat these as:

```text
optional diagnostics
```

and verify support before building production assumptions.

Do not use one memory API as:

```text
universal browser memory truth.
```

---

# 59. JS Profiling vs Performance Entries

Performance entries:

```text
structured time measurements
```

are good for:

```text
production monitoring.
```

CPU profiles/flame charts:

```text
call-stack sampling
```

are better for:

```text
root-cause analysis.
```

Use both.

---

# 60. Flame Charts

A flame chart conceptually shows:

```text
call stack
+
time
```

Long wide blocks suggest:

```text
expensive functions
```

or:

```text
long call chains.
```

Performance entries tell:

```text
what happened when.
```

Flame charts help tell:

```text
what code consumed that time.
```

---

# 61. Production Profiling Trade-Off

Full profiling in every production session can be:

```text
expensive
privacy-sensitive
large
```

Prefer:

```text
sampling
targeted sessions
remote diagnostic triggers
```

for high-detail profiling.

---

# 62. Real User Monitoring

RUM collects:

```text
real-user browser performance
```

from production traffic.

It answers:

```text
what actually happened to users?
```

including:

```text
devices
networks
geographies
browser versions
```

that lab environments may not reproduce.

---

# 63. Synthetic Monitoring

Synthetic monitoring runs:

```text
controlled browser tests
```

against predictable environments.

It is useful for:

```text
regression detection
deployment checks
performance budgets
availability.
```

---

# 64. RUM vs Synthetic

### RUM

```text
real-world variance
high population
messy
```

### Synthetic

```text
controlled
repeatable
diagnostic
```

Use both.

---

# 65. Field Segmentation

A performance problem may affect:

```text
mobile users
```

but not:

```text
desktop users.
```

Segment by:

```text
device class
network type
browser
OS
country/region
route
feature
application version
```

---

# 66. Percentiles

Average latency can hide tail problems.

Example:

```text
p50 = 200 ms
p95 = 2 s
p99 = 8 s
```

This means:

```text
median experience is good
tail experience is poor.
```

Production performance should usually monitor:

```text
p50
p75
p95
p99
```

as appropriate.

---

# 67. Why p75 Often Matters

A product may define user-facing targets around:

```text
p75
```

because it captures a meaningful majority of users without being dominated by extreme outliers.

The exact percentile should come from:

```text
product SLO
```

not habit.

---

# 68. Performance Budgets

A performance budget defines a limit such as:

```text
LCP < target
INP < target
JS transfer < target
main-thread blocking < target
API p95 < target
```

Budgets turn:

```text
performance
```

into:

```text
an enforceable engineering requirement.
```

---

# 69. Loading Budget

Example:

```text
critical JS ≤ 200 KB compressed
critical CSS ≤ 50 KB
LCP ≤ 2.5 s
```

The exact values should reflect:

```text
business needs
device population
network.
```

---

# 70. Responsiveness Budget

Example:

```text
no application task should normally exceed 50 ms
interactive critical path p75 ≤ target
```

But a single threshold does not replace:

```text
INP.
```

---

# 71. Network Budget

Track:

```text
request count
critical bytes
image bytes
JS bytes
CSS bytes
font bytes
API latency
```

Network performance often determines:

```text
startup
```

before CPU becomes the bottleneck.

---

# 72. JavaScript Budget

Track:

```text
transfer size
parse/compile
execution time
long tasks
heap allocation
```

A small compressed bundle can still have:

```text
huge runtime execution cost.
```

---

# 73. Parse/Compile Cost

Large JS can cost:

```text
download
parse
compile
execute
memory.
```

Compression only addresses:

```text
network transfer
```

not:

```text
CPU work.
```

---

# 74. Code Splitting

Use:

```text
critical path
+
lazy modules
```

where appropriate.

But excessive code splitting can increase:

```text
request count
latency
coordination complexity.
```

Measure:

```text
real critical-path cost.
```

---

# 75. Dynamic Import Timing

Instrument:

```js
performance.mark("editor:import:start");

const editor =
  await import("./editor.js");

performance.mark("editor:import:end");

performance.measure(
  "editor:import",
  "editor:import:start",
  "editor:import:end"
);
```

This connects:

```text
feature interaction
→ module loading
```

to user experience.

---

# 76. Resource Discovery

A resource can be slow because:

```text
browser discovers it late.
```

For images/fonts/scripts:

```text
preload
priority
HTML ordering
CSS dependency
```

can affect:

```text
discovery time.
```

Performance measurement should distinguish:

```text
slow download
```

from:

```text
late discovery.
```

---

# 77. Server-Timing

Servers can expose backend timing information through:

```text
Server-Timing
```

headers.

A browser can surface corresponding timing data in performance entries.

This allows:

```text
browser
+
backend
```

correlation.

---

# 78. Server-Timing Example

Server:

```http
Server-Timing:
  db;dur=35,
  cache;dur=4,
  app;dur=18
```

Browser can correlate:

```text
frontend request
```

with:

```text
database/app timings.
```

Do not expose sensitive backend details.

---

# 79. Full Request Trace

Ideal chain:

```text
user click
 ↓
client mark
 ↓
fetch
 ↓
resource timing
 ↓
Server-Timing
 ↓
backend trace
 ↓
DB timing
```

This is:

```text
distributed tracing for user experience.
```

---

# 80. Trace/Correlation IDs

Use a request identifier such as:

```text
traceparent
```

or an application-level correlation ID.

Then connect:

```text
frontend event
→ network request
→ backend span
→ database operation.
```

Do not leak:

```text
private identifiers
```

into user-visible URLs.

---

# 81. Business Performance

Technical timing is not enough.

Measure:

```text
checkout completion
search success
time-to-first-result
time-to-interactive-feature
```

A slower technical metric can be acceptable if:

```text
business completion stays strong.
```

---

# 82. User Journey Measurement

Example:

```text
search submitted
→ result visible
→ result selected
→ detail usable
```

Instrument:

```text
search:submit
search:first-result
search:interactive
```

Then measure:

```text
submit → usable.
```

---

# 83. Avoid Over-Instrumentation

Bad:

```text
mark every function
measure every event
send every entry.
```

Costs include:

```text
memory
CPU
network
cardinality
```

Instrument:

```text
critical journeys
major resources
known bottlenecks
SLO-related milestones.
```

---

# 84. Sampling

For common events:

```text
100% sampling
```

may be unnecessary.

Use:

```text
1%
5%
10%
```

or adaptive strategies for:

```text
high-volume diagnostics.
```

Keep:

```text
100%
```

for:

```text
critical low-volume failures
```

where valuable.

---

# 85. Adaptive Sampling

Example:

```text
normal sessions → 1%
slow sessions → 50%
failed sessions → 100%
```

This provides:

```text
high detail where it matters
```

without:

```text
huge telemetry cost.
```

---

# 86. High Cardinality

Avoid labels like:

```text
userId
request URL with random IDs
search query
```

in unbounded metric dimensions.

Prefer:

```text
route template
feature
browser class
version
region
```

---

# 87. Privacy

Performance data can contain:

```text
URLs
query strings
resource names
timing
device info
network info
user behavior.
```

Apply:

```text
redaction
minimization
sampling
retention policy
```

before sending telemetry.

---

# 88. Sensitive URLs

Never blindly send:

```js
entry.name
```

to analytics if the URL may contain:

```text
tokens
PII
query secrets
```

Normalize:

```text
origin
route template
safe resource class
```

as appropriate.

---

# 89. Measurement Perturbation

Instrumentation changes the program.

Example:

```text
logging every frame
```

can reduce:

```text
frame rate.
```

This is:

```text
observer effect.
```

Benchmark with:

```text
instrumented
vs
uninstrumented
```

to measure overhead.

---

# 90. Production Instrumentation Overhead

A telemetry SDK should have:

```text
bounded queue
sampling
batching
compression
backoff
drop policy
```

and:

```text
never block the critical user path.
```

---

# 91. Telemetry Transport

Performance telemetry can use:

```text
sendBeacon
fetch({ keepalive: true })
```

or another ingestion mechanism where appropriate.

Batch:

```text
multiple records
```

instead of:

```text
one request per metric.
```

---

# 92. Page Unload

Telemetry sent during page termination is difficult because:

```text
page may disappear quickly.
```

Use appropriate browser lifecycle-friendly delivery mechanisms.

Do not rely on:

```text
await fetch(...)
```

during unload.

---

# 93. Long-Lived SPA

A single-page app can remain open:

```text
hours/days.
```

Therefore monitor:

```text
memory growth
performance entry growth
listener growth
queue growth
```

over time.

---

# 94. Performance Memory Leak

A telemetry SDK can itself leak by retaining:

```text
PerformanceEntry
DOM node
event handler
route object
user context
```

Use:

```text
weak references where appropriate
bounded buffers
cleanup
```

and test:

```text
long-lived sessions.
```

---

# 95. PerformanceObserver Lifecycle

Disconnect observers when:

```text
feature ends
page module unloads
test completes
```

Use:

```js
observer.disconnect();
```

to avoid:

```text
unnecessary callbacks
```

and:

```text
retained state.
```

---

# 96. Resource Timing Collection

A production collector can:

```text
observe resource entries
→ classify
→ sample
→ aggregate
→ send batch.
```

Do not send:

```text
every image request
```

unless the product actually needs it.

---

# 97. Navigation Timing Collection

Collect:

```text
navigation type
redirects
TTFB-like timing
DOM milestones
load
```

and:

```text
LCP/INP/CLS
```

as separate experience signals.

---

# 98. SPA Navigation

A single-page application does not perform full document navigation on every route.

Therefore:

```text
Navigation Timing
```

does not automatically measure:

```text
client-side route transitions.
```

Instrument route changes manually:

```text
route:start
route:data-ready
route:rendered
route:interactive
```

---

# 99. SPA Route Metric

Example:

```text
click dashboard
→ route start
→ data ready
→ component rendered
→ usable
```

Measure:

```text
route transition duration
```

not:

```text
document navigation.
```

---

# 100. View Transitions and Performance

Modern browser UI transitions can add:

```text
capture
animation
render
compositing
```

cost.

Measure:

```text
transition latency
```

with:

```text
User Timing
+
DevTools
```

rather than assuming:

```text
animation = free.
```

---

# 101. Animation Performance

For smooth rendering, common targets include:

```text
~16.7 ms/frame at 60 Hz
```

but modern displays can use:

```text
90 Hz
120 Hz
144 Hz
```

so:

```text
frame budget depends on refresh rate.
```

Do not hard-code:

```text
16.67 ms
```

as the universal rendering budget.

---

# 102. `requestAnimationFrame`

Use:

```js
requestAnimationFrame(() => {
  // visual update
});
```

to align visual work with rendering.

For measurement, record:

```text
start
work
next frame
```

carefully.

---

# 103. Frame Drops

Dropped frames can come from:

```text
long JS task
layout
paint
compositing
GPU pressure
image decode
browser contention.
```

Do not assume:

```text
dropped frame = slow JavaScript.
```

---

# 104. Interaction Instrumentation

For a user interaction:

```text
input
→ event handler
→ async work
→ DOM update
→ next paint.
```

Instrument:

```text
interaction:start
interaction:logic-end
interaction:rendered
```

This provides a conceptual decomposition of:

```text
INP-like experience.
```

---

# 105. Async Gaps

Suppose:

```text
click
→ await fetch
→ render
```

The user-perceived delay includes:

```text
network wait
```

not merely:

```text
render execution.
```

Performance traces should preserve:

```text
causal waiting time.
```

---

# 106. Parallel vs Sequential Fetches

Compare:

```js
await fetchA();
await fetchB();
```

with:

```js
await Promise.all([
  fetchA(),
  fetchB()
]);
```

Performance instrumentation should reveal:

```text
sequential critical path
```

vs:

```text
parallel overlap.
```

Do not optimize based only on:

```text
individual request duration.
```

---

# 107. Critical Path

Overall user-visible time is often:

```text
max dependency chain
```

rather than:

```text
sum of every operation in the system.
```

A principal performance engineer identifies:

```text
critical path
```

before optimizing.

---

# 108. Queueing Delays

An operation may be slow because it waits in:

```text
connection pool
task queue
server queue
database queue
browser scheduler
```

not because its execution is intrinsically expensive.

Telemetry should separate:

```text
queue time
processing time
```

when possible.

---

# 109. Server Timing Correlation

If backend emits:

```text
Server-Timing: db;dur=40,app;dur=20
```

and browser sees:

```text
TTFB = 100 ms
```

the gap may include:

```text
network
queue
proxy
TLS/connection
```

Do not assume:

```text
100 - 60 = network.
```

without understanding all phases.

---

# 110. Resource Priorities

Browser scheduling may prioritize:

```text
CSS
fonts
hero image
scripts
```

differently.

Resource Timing tells:

```text
what happened
```

while:

```text
fetch priority
preload
HTML structure
```

influence:

```text
when it happens.
```

---

# 111. Cache Attribution

A fast resource may be:

```text
memory cache
HTTP cache
service worker cache
network
```

Resource timing can provide clues, but interpretation must account for:

```text
browser cache behavior
service worker behavior
privacy restrictions.
```

---

# 112. Service Worker Resource Timing

A service worker can change:

```text
network path
response source
cache behavior.
```

Therefore performance analysis of:

```text
resource timing
```

should include:

```text
service worker version/state
```

where applicable.

---

# 113. WebTransport Performance

For long-lived WebTransport sessions, measure:

```text
connect latency
ready latency
reconnect
stream throughput
datagram rate
```

Use:

```text
application marks
```

plus:

```text
transport-specific metrics.
```

---

# 114. WebRTC Performance

For WebRTC, use:

```text
getStats()
```

for:

```text
RTT
jitter
loss
bitrate
```

and combine with:

```text
User Timing
```

for:

```text
call setup
first media
reconnect.
```

Chapter 135 covers this deeply.

---

# 115. Storage Performance

For IndexedDB/storage operations:

```text
operation start
request complete
transaction complete
```

measure separately.

A transaction may be slow because of:

```text
browser storage
serialization
contention
quota
large data
```

---

# 116. Worker Performance

Measure:

```text
worker startup
message latency
compute duration
transfer/copy time
```

when using workers.

Do not assume:

```text
worker = automatically faster.
```

Communication and serialization cost matter.

---

# 117. Structured Clone Cost

Sending a large object through:

```text
postMessage
```

can incur:

```text
serialization/copy cost
```

unless the API/path uses transferables appropriately.

Measure:

```text
payload size
clone/transfer time
```

in worker-heavy applications.

---

# 118. Performance.mark Across Workers

Different execution contexts can have:

```text
separate time origins
```

and timelines.

Do not assume:

```text
performance.now() values
```

from different contexts can be compared directly without synchronization/correlation.

Use:

```text
message timestamps
shared identifiers
time-origin correlation
```

as appropriate.

---

# 119. Clock Correlation

For cross-context traces:

```text
context A
context B
worker C
server D
```

you need:

```text
trace/span IDs
```

and:

```text
timestamp semantics
```

to reconstruct causality.

Do not infer:

```text
event order
```

from raw timestamps alone.

---

# 120. Performance Trace Model

A useful event:

```js
{
  traceId,
  spanId,
  parentSpanId,
  name,
  startTime,
  duration,
  attributes
}
```

This moves User Timing toward:

```text
distributed tracing.
```

---

# 121. High-Fidelity Traces

For debugging:

```text
many spans
```

may be useful.

For production:

```text
sampling
```

is usually required to keep:

```text
cost
memory
privacy
```

manageable.

---

# 122. Performance Regression Detection

Compare:

```text
version N
vs
version N+1
```

for:

```text
LCP
INP
CLS
route duration
API latency
JS execution
```

Use:

```text
statistical confidence
```

rather than declaring regression from:

```text
one run.
```

---

# 123. Lab Variance

Browser performance has noise from:

```text
CPU contention
network
cache
browser state
OS
thermal throttling
background tasks.
```

Therefore:

```text
repeat
control environment
use distributions.
```

---

# 124. A/B Performance Testing

A feature can improve:

```text
conversion
```

while worsening:

```text
LCP
```

or:

```text
INP.
```

Track:

```text
performance
+
business outcome
```

together.

---

# 125. Field Metric Segmentation

Always inspect:

```text
overall
mobile
desktop
slow CPU
slow network
regions
versions
```

A global average can hide:

```text
severe subgroup regression.
```

---

# 126. Performance Budgets in CI

CI can gate:

```text
bundle size
Lighthouse metrics
script execution
resource count
```

but CI cannot perfectly predict:

```text
real-world RUM.
```

Use:

```text
lab gates
+
field monitoring.
```

---

# 127. Budget Selection

Do not create:

```text
100 performance budgets
```

because nobody can own them.

Choose:

```text
5–10 meaningful budgets
```

linked to:

```text
product outcomes.
```

---

# 128. Performance Ownership

Each important metric should have:

```text
owner
target
data source
alert
runbook.
```

Performance without ownership becomes:

```text
dashboard decoration.
```

---

# 129. Alerts

Alert on:

```text
sustained p75 regression
p95 regression
availability impact
```

not:

```text
one outlier.
```

Use:

```text
baseline
seasonality
release correlation.
```

---

# 130. Release Correlation

Record:

```text
deployment version
commit
feature flag
```

with performance telemetry.

Then a regression can be correlated:

```text
version 842
→ LCP +20%
```

instead of:

```text
“something became slow.”
```

---

# 131. Feature Flag Correlation

Track:

```text
flag state
```

in performance dimensions, but avoid high-cardinality explosion.

Prefer:

```text
small finite flag groups
```

or:

```text
experiment assignment ID
```

with controlled retention.

---

# 132. Performance Error Budget

A team can treat performance as:

```text
SLO
```

such as:

```text
p75 LCP ≤ target
p75 INP ≤ target
```

Then:

```text
performance regression
=
release risk
```

rather than subjective feedback.

---

# 133. Network Information

Browser network APIs can expose some connection hints, but they are:

```text
heuristic
privacy-constrained
browser-dependent.
```

Do not use them as exact:

```text
bandwidth measurements.
```

Actual resource timing is usually more actionable.

---

# 134. Performance and Privacy Budget

Modern browsers may reduce precision or exposure of some timing data to limit:

```text
fingerprinting
side channels.
```

Therefore:

```text
higher precision
```

is not always:

```text
more information you can access.
```

---

# 135. Timing Precision

`performance.now()` provides a high-resolution monotonic timer, but browsers may coarsen timing precision under certain privacy/security conditions.

Do not assume:

```text
microsecond precision
```

is always available.

The useful principle is:

```text
measurement precision is browser policy-dependent.
```

---

# 136. Spectre-Related Timing Concerns

High-resolution timing can be useful for:

```text
side-channel attacks.
```

Browser security designs therefore constrain:

```text
timing precision
cross-origin isolation
shared-memory/timing combinations.
```

This connects to:

```text
Chapter 127
```

and:

```text
Chapter 56/57 security.
```

---

# 137. PerformanceObserver Error Handling

Your observer callback should be small:

```js
for (const entry of list.getEntries()) {
  queue(entry);
}
```

Do not perform:

```text
large serialization
network requests
DOM work
```

inside the observer callback.

---

# 138. Batch Telemetry

Use:

```text
in-memory queue
→ bounded buffer
→ periodic batch
→ sendBeacon/fetch keepalive
```

This reduces:

```text
network chatter
```

and:

```text
observer overhead.
```

---

# 139. Backpressure in Telemetry

If telemetry ingestion is slow:

```text
queue grows.
```

Set:

```text
max queue size
```

and:

```text
drop policy.
```

Never let:

```text
performance monitoring
```

become:

```text
application memory leak.
```

---

# 140. Telemetry Failure

If monitoring endpoint fails:

```text
application must continue working.
```

Telemetry is:

```text
best-effort.
```

Do not:

```text
block checkout
```

because:

```text
metrics server is down.
```

---

# 141. Performance SDK Architecture

```text
Browser APIs
    ↓
Collector
    ↓
Normalizer
    ↓
Sampler
    ↓
Aggregator
    ↓
Batcher
    ↓
Transport
    ↓
Ingestion
```

Each stage should have:

```text
bounded memory
failure handling
versioning.
```

---

# 142. Entry Normalization

Convert entries into a stable schema:

```js
{
  type,
  name,
  startTime,
  duration,
  route,
  appVersion
}
```

Do not send the entire native entry object blindly.

---

# 143. Route Template Normalization

Instead of:

```text
/users/12345/orders/89123
```

store:

```text
/users/:userId/orders/:orderId
```

This keeps metrics:

```text
low-cardinality
```

and avoids:

```text
PII/identifier leakage.
```

---

# 144. Resource URL Normalization

Instead of:

```text
https://cdn.example.com/assets/user123/hash789.js
```

extract:

```text
resourceHost
resourceType
route
cacheClass
```

and retain raw URL only in:

```text
high-detail controlled debugging.
```

---

# 145. Performance Event Schema

Example:

```js
{
  schemaVersion: 1,
  traceId: "abc",
  route: "/checkout",
  metric: "checkout:interactive",
  durationMs: 820,
  sampleRate: 0.1,
  appVersion: "2026.09.11"
}
```

---

# 146. Data Validation

Validate telemetry before sending:

```text
finite numbers
bounded strings
allowed enum values
max attribute count
max payload size
```

Telemetry inputs can be polluted by:

```text
user-controlled URL fields
```

so treat them as:

```text
untrusted data.
```

---

# 147. Performance API Feature Detection

Use:

```js
if ("PerformanceObserver" in globalThis) {
  // setup
}
```

and:

```js
const types =
  PerformanceObserver.supportedEntryTypes ?? [];
```

Feature detection prevents:

```text
old browser failures.
```

---

# 148. Fallback Instrumentation

If:

```text
Long Tasks
```

are unavailable:

```text
timer probes
```

may provide coarse responsiveness signals.

If:

```text
LCP API
```

is unavailable:

```text
custom user timing
```

can measure:

```text
known application milestones.
```

Do not label:

```text
custom metric
```

as:

```text
official LCP
```

unless it actually is.

---

# 149. Lab vs Field Naming

Keep metrics distinct:

```text
lab.lcp
field.lcp
custom.checkoutInteractive
```

Do not merge them into:

```text
lcp
```

without source context.

---

# 150. Performance Metric Contract

For every metric document:

```text
definition
start point
end point
population
unit
percentile
sampling rate
browser support
known exclusions
```

This prevents:

```text
metric drift.
```

---

# 151. Performance Debugging Workflow

Use:

```text
1. Confirm regression
2. Segment affected users
3. Identify affected lifecycle phase
4. Inspect browser timeline
5. Correlate application marks
6. Correlate network/backend timing
7. profile CPU/memory if needed
8. reproduce
9. change one thing
10. verify field + lab recovery
```

---

# 152. Do Not Optimize Blindly

Bad:

```text
“Let's add memoization.”
```

before measuring.

Better:

```text
measure
→ identify bottleneck
→ form hypothesis
→ optimize
→ remeasure.
```

---

# 153. Root Cause Hierarchy

For a slow feature, ask:

```text
Is it network?

Is it server?

Is it waiting?

Is it parsing?

Is it JavaScript?

Is it layout?

Is it paint?

Is it memory/GC?

Is it browser scheduling?

Is it coordination?
```

Only then choose:

```text
optimization.
```

---

# 154. Performance vs Correctness

Never trade:

```text
correctness
```

for:

```text
small benchmark improvement
```

without explicit business approval.

Example:

```text
dropping state updates
```

may improve responsiveness but:

```text
break user correctness.
```

Performance decisions require:

```text
trade-off analysis.
```

---

# 155. Performance vs Observability

Detailed tracing can cost:

```text
CPU
memory
network
```

Therefore observability itself belongs in the:

```text
performance budget.
```

---

# 156. Production Instrumentation Checklist

```text
[ ] entry types feature-detected
[ ] observers lightweight
[ ] observer callbacks bounded
[ ] telemetry queue bounded
[ ] sampling enabled
[ ] batching enabled
[ ] sensitive URL data redacted
[ ] route templates normalized
[ ] high-cardinality fields controlled
[ ] send failures tolerated
[ ] no critical-path blocking
[ ] observers disconnected when unnecessary
[ ] performance entry buffers managed
[ ] metrics versioned
[ ] release/feature correlation available
```

---

# 157. Implementation From Scratch — Performance SDK

Build:

```text
MarkManager
MeasureManager
ObserverRegistry
Normalizer
Sampler
Aggregator
Batcher
Transport
```

Objective:

```text
learn how performance telemetry systems should remain cheap,
bounded, reliable, and privacy-safe.
```

---

# 158. Implementation Milestone 1 — Mark Helper

```js
function mark(name, detail) {
  performance.mark(name, { detail });
}
```

Add:

```text
validation
namespace
cleanup
```

---

# 159. Implementation Milestone 2 — Measure Helper

Create:

```js
function measure(name, start, end) {
  performance.measure(name, {
    start,
    end
  });
}
```

Then normalize:

```text
name
startTime
duration
```

into your own event schema.

---

# 160. Implementation Milestone 3 — Observer Registry

Create:

```text
register(type, callback)
unregister(type)
disconnectAll()
```

Track:

```text
observer lifecycle
```

so features can:

```text
start/stop instrumentation.
```

---

# 161. Implementation Milestone 4 — Sampler

Implement:

```text
always sample
random sample
rate sample
slow-event sample
error sample
```

Example policy:

```text
normal < 1s → 1%
slow ≥ 1s → 25%
critical failure → 100%
```

---

# 162. Implementation Milestone 5 — Bounded Queue

Implement:

```text
maxEvents
dropOldest
dropNewest
priority
```

and verify:

```text
memory remains bounded.
```

---

# 163. Implementation Milestone 6 — Batch Transport

Batch:

```text
20 events
```

or:

```text
5 seconds
```

whichever comes first.

Use:

```text
sendBeacon
```

or:

```text
fetch keepalive
```

as appropriate.

---

# 164. Implementation Milestone 7 — Navigation Collector

Collect:

```text
navigation type
TTFB-like timing
DOMContentLoaded
load
redirect
```

and:

```text
app version
route
```

---

# 165. Implementation Milestone 8 — Resource Collector

Collect only:

```text
scripts
CSS
fonts
API
hero image
```

initially.

Then add:

```text
all resources
```

only when required.

---

# 166. Implementation Milestone 9 — Long Task Monitor

Observe:

```text
longtask
```

and aggregate:

```text
count
total blocking duration
max task
p95 task duration
```

---

# 167. Implementation Milestone 10 — Route Transition

Instrument:

```text
route:start
route:data-ready
route:rendered
route:interactive
```

and report:

```text
transition duration
```

---

# 168. Implementation Milestone 11 — Backend Correlation

Read:

```text
Server-Timing
```

and combine:

```text
frontend request
+
backend phases.
```

Do not expose:

```text
sensitive backend information
```

to untrusted users.

---

# 169. Implementation Milestone 12 — Budget Gate

Fail CI when:

```text
bundle > limit
```

or:

```text
synthetic performance > threshold
```

but keep:

```text
field monitoring
```

separate.

---

# 170. Debugging Exercises

## Exercise A — Slow Function, Fast User Experience

A function takes:

```text
80 ms
```

but the page feels instant.

Explain why:

```text
function duration
```

does not automatically imply:

```text
user-visible latency.
```

---

## Exercise B — Fast API, Slow Interaction

API:

```text
100 ms
```

but button response:

```text
1.5 s
```

Find:

```text
main-thread
layout
rendering
```

bottlenecks.

---

## Exercise C — Good Lab, Bad Field

Lab:

```text
LCP = 1.8 s
```

RUM:

```text
p75 LCP = 4.2 s
```

Segment:

```text
mobile
network
region
device
```

and identify which population is responsible.

---

## Exercise D — Resource Timing Missing

A cross-origin API has:

```text
name
```

but detailed timing is restricted.

Investigate:

```text
CORS
Timing-Allow-Origin
browser privacy policy.
```

---

## Exercise E — Telemetry Slows App

Performance SDK adds:

```text
200 ms
```

to route transitions.

Profile:

```text
observer callback
serialization
network
```

and redesign:

```text
sampling
batching
queueing.
```

---

# 171. Code Review Exercise

Review:

```js
new PerformanceObserver(list => {
  navigator.sendBeacon(
    "/metrics",
    JSON.stringify(list.getEntries())
  );
}).observe({
  entryTypes: [
    "resource",
    "measure",
    "longtask"
  ]
});
```

Identify:

```text
excessive frequency
raw-entry serialization
privacy leakage
no sampling
no batching
no queue limit
no feature detection
no cleanup
critical-path network activity
high cardinality
```

Then redesign the collector.

---

# 172. Interview Questions

### Fundamentals

```text
1. Why use performance.now() instead of Date.now() for durations?
2. What is the Performance Timeline?
3. What is a PerformanceEntry?
4. What does PerformanceObserver do?
5. What is User Timing?
```

### Navigation

```text
6. What does Navigation Timing measure?
7. What is Resource Timing?
8. How do you reason about TTFB?
9. What does transferSize mean?
10. What is Timing-Allow-Origin?
```

### Responsiveness

```text
11. What is a Long Task?
12. Why is 50 ms significant?
13. What is INP?
14. What was FID?
15. How do you detect main-thread blocking?
```

### Rendering

```text
16. What is FCP?
17. What is LCP?
18. Why can LCP be slow when the API is fast?
19. What is layout thrashing?
20. What causes dropped frames?
```

### Observability

```text
21. What is RUM?
22. What is synthetic monitoring?
23. Why use p95/p99?
24. Why does sampling matter?
25. What is high cardinality?
```

### Principal

```text
26. How would you design a production browser performance SDK?
27. How would you correlate frontend and backend latency?
28. How would you detect a performance regression after a deployment?
29. How would you instrument a SPA route transition?
30. How would you balance observability detail against performance/privacy cost?
```

---

# 173. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
const start = performance.now();

setTimeout(() => {
  console.log(performance.now() - start);
}, 0);
```

Explain why the result is:

```text
not guaranteed to be exactly 0 or 1 ms.
```

---

### Exercise 2

```js
performance.mark("a");

setTimeout(() => {
  performance.mark("b");
  performance.measure("x", "a", "b");

  console.log(
    performance.getEntriesByName("x")[0].duration
  );
}, 100);
```

Predict the approximate magnitude.

---

### Exercise 3

A page performs:

```text
100 ms network
500 ms JavaScript
```

Which is more likely to dominate:

```text
main-thread responsiveness?
```

---

### Exercise 4

Two users have:

```text
p50 LCP = 1.5 s
p99 LCP = 10 s
```

Predict what an average may hide.

---

### Exercise 5

A `PerformanceObserver` callback performs:

```js
JSON.stringify(
  performance.getEntries()
);
```

for every callback.

Predict:

```text
why instrumentation itself can become expensive.
```

---

# 174. Mastery Exercises

### Exercise 1 — Performance SDK

Build:

```text
marks
measures
observers
sampling
batching
privacy normalization
```

### Exercise 2 — RUM Dashboard

Track:

```text
LCP
INP
CLS
TTFB-like timing
route duration
API p95
long-task metrics
```

Segment by:

```text
device
browser
network
version
```

### Exercise 3 — Route Tracing

Instrument:

```text
route start
data ready
render
interactive
```

and correlate with:

```text
fetch
resource
backend.
```

### Exercise 4 — Long-Task Detector

Build a detector and calculate:

```text
count
total blocking duration
max
p95.
```

### Exercise 5 — Resource Budget

Create budgets for:

```text
JS
CSS
fonts
images
API
```

and fail CI when they regress.

### Exercise 6 — Field vs Lab

Compare:

```text
synthetic
vs
RUM
```

and explain:

```text
why they disagree.
```

### Exercise 7 — Performance Incident

Simulate:

```text
release
→ INP worsens 30%
```

Perform:

```text
segmentation
timeline analysis
profiling
root cause
fix
verification
```

---

# 175. Track A — Core Theory

Master:

```text
performance.now
time origin
Performance Timeline
PerformanceEntry
PerformanceObserver
User Timing
Navigation Timing
Resource Timing
Paint Timing
LCP
INP
Long Tasks
event-loop responsiveness
rendering
network attribution
RUM
synthetic monitoring
sampling
percentiles
budgets
```

Deliverable:

```text
translate a user-visible performance problem into a measurable timeline
and identify the likely causal phase.
```

---

# 176. Track B — Implementation

Build:

```text
performance SDK
mark/measure utilities
observer registry
RUM collector
navigation collector
resource collector
long-task monitor
route-transition tracing
backend correlation
sampling
bounded queues
performance-budget gates
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```

---

# 177. Track C — Interview / Reasoning

Practice:

```text
“Why isn't Date.now() ideal for benchmarking?”

“How would you diagnose a slow LCP?”

“Why can an API be fast while INP is bad?”

“What is the difference between RUM and Lighthouse?”

“How do you correlate browser and backend timing?”

“Why is p95 more useful than an average?”

“How can telemetry itself hurt performance?”
```

Deliverable:

```text
metric
+
timeline
+
hypothesis
+
measurement
+
trade-off.
```

---

# 178. Specification / Runtime Source Discipline

Use this hierarchy:

```text
1. W3C/WHATWG Performance specifications
2. Web Vitals definitions / current web.dev guidance
3. Fetch / Resource Timing / Navigation Timing standards
4. browser implementation behavior
5. DevTools documentation
6. telemetry SDK behavior
7. application code
```

Keep these distinctions explicit:

```text
standard performance entries
vs
browser-specific diagnostics
vs
DevTools-only profiling
vs
application custom metrics
vs
third-party analytics metrics.
```

Do not claim:

```text
“my custom mark is LCP”
```

or:

```text
“timer delay is exact CPU idle time.”
```

The Performance Timeline, User Timing, Navigation Timing, Resource Timing, Long Tasks, and observer APIs are separate layers of browser performance instrumentation. citeturn701019search2turn701019search4turn701019search0

---

# 179. Current Platform Notes

As of September 2026:

```text
performance.now():
widely available.

User Timing / PerformanceObserver:
widely supported.

Navigation Timing / Resource Timing:
widely supported.

FCP/LCP:
widely used for web performance measurement.

INP:
current Core Web Vitals responsiveness metric.

FID:
historical/legacy Core Web Vital, replaced by INP.

Long Tasks:
available in modern browsers with varying support.

Some detailed timing/memory APIs:
remain browser-dependent or privacy-constrained.
```

Current web.dev guidance uses:

```text
LCP
INP
CLS
```

as the Core Web Vitals, while FID has been retired in favor of INP. citeturn701019search7turn701019search10

---

# 180. Principal Decision Framework

For every performance problem ask:

```text
1. What user experience is slow?
2. What exact start and end points define “slow”?
3. Is it loading, interaction, animation, network, storage, or background work?
4. Is the issue field-only or lab-visible?
5. Which users are affected?
6. Which versions are affected?
7. Is the problem CPU, memory, network, rendering, or waiting?
8. What does the Performance Timeline show?
9. What custom marks are needed?
10. What backend timing correlates?
11. What is the critical path?
12. What is the tail latency?
13. Is instrumentation affecting the result?
14. What is the smallest safe optimization?
15. What correctness trade-off does it introduce?
16. How will the fix be verified?
17. What performance budget should prevent regression?
```

---

# 181. Production Checklist

```text
[ ] performance goals tied to user experience
[ ] metrics explicitly defined
[ ] start/end points documented
[ ] RUM implemented
[ ] synthetic monitoring implemented
[ ] percentiles monitored
[ ] meaningful segmentation available
[ ] route transitions instrumented
[ ] critical user journeys marked
[ ] observer callbacks lightweight
[ ] performance entries bounded/managed
[ ] telemetry queue bounded
[ ] batching implemented
[ ] sampling implemented
[ ] high-cardinality fields controlled
[ ] URLs redacted/normalized
[ ] sensitive data excluded
[ ] send failures tolerated
[ ] no critical-path telemetry blocking
[ ] release correlation available
[ ] feature-flag correlation available
[ ] backend timing correlated where useful
[ ] long-task monitoring enabled where supported
[ ] current Core Web Vitals tracked
[ ] browser support tested
[ ] CI budgets exist
[ ] performance incident runbook exists
```

---

# 182. Retrieval Record

```md
# Chapter 137 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## performance.now / Time Origin
-

## Performance Timeline
-

## PerformanceObserver
-

## User Timing
-

## Navigation Timing
-

## Resource Timing
-

## Paint Timing
-

## LCP
-

## INP
-

## Long Tasks
-

## Event Loop Responsiveness
-

## Rendering
-

## Network
-

## Backend Correlation
-

## RUM
-

## Synthetic
-

## Sampling
-

## Percentiles
-

## Budgets
-

## Privacy
-

## Instrumentation Overhead
-

## Testing
-

## Implementation Progress
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

# 183. Spaced Retrieval Schedule

### Day 0

Study:

```text
performance.now
Performance Timeline
marks/measures
PerformanceObserver
```

### Day 1

Explain:

```text
navigation
resource
paint
longtask
```

entries.

### Day 3

Build:

```text
route-transition instrumentation
```

### Day 7

Build:

```text
RUM collector
+
sampling
+
batching.
```

### Day 14

Diagnose:

```text
slow LCP
slow INP
```

from a supplied trace.

### Day 21

Build:

```text
backend + frontend timing correlation.
```

### Day 30

Perform a complete:

```text
browser performance architecture review
```

without notes.

---

# 184. Dependency Graph

```text
Chapter 03
Numbers
        ↓
Chapter 31
Async
        ↓
Chapter 33
Browser Event Loop
        ↓
Chapter 49
DOM
        ↓
Chapter 50
Browser Events
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 52
Workers
        ↓
Chapter 53
Streams
        ↓
Chapter 55
Fetch / HTTP
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Production Debugging
        ↓
Chapter 83
Observability
        ↓
Chapter 84
Reliability
        ↓
Chapter 85
Performance
        ↓
Chapter 86
Testing
        ↓
Chapter 88
Debugging Methodology
        ↓
Chapter 101
Production Scenarios
        ↓
Chapter 130
Clock Semantics
        ↓
Chapter 132
Storage
        ↓
Chapter 133
Service Workers
        ↓
Chapter 134
Web Locks
        ↓
Chapter 135
WebRTC
        ↓
Chapter 136
WebTransport
        ↓
Chapter 137
Browser Performance APIs / Runtime Instrumentation
```

Cross-cutting:

```text
Chapter 37 → cancellation
Chapter 38 → stream timing
Chapter 56 → privacy/security
Chapter 57 → timing side channels
Chapter 67 → dependency performance
Chapter 69 → bundle/build performance
Chapter 79 → API latency
Chapter 98 → performance anti-patterns
Chapter 129 → instrumentation payload parsing
Chapter 131 → URL privacy/canonicalization
Chapter 132 → storage latency
Chapter 134 → coordination contention
Chapter 146 → deterministic measurement/testing
```

---

# 185. Concept Connections

## Depends On

```text
monotonic timing
browser event loop
DOM/rendering
fetch
web APIs
workers
networking
observability
testing
```

## Builds Toward

```text
RUM
performance engineering
frontend observability
distributed tracing
performance budgets
incident response
platform engineering
```

## Related Concepts

```text
Performance Timeline
User Timing
Navigation Timing
Resource Timing
Paint Timing
Long Tasks
LCP
INP
CLS
Server-Timing
RUM
synthetic monitoring
DevTools profiling
```

## Concepts Revisited

```text
Date
Event Loop
Fetch
Workers
Streams
Service Workers
WebRTC
WebTransport
Storage
Observability
Reliability
Security
Testing
```

## Why This Chapter Matters

Performance is not:

```text
“make code fast.”
```

It is:

```text
measure user experience
→ identify causal path
→ find bottleneck
→ optimize
→ verify
→ prevent regression.
```

The browser provides many performance APIs because:

```text
loading
interaction
rendering
network
storage
workers
```

are separate systems.

A principal engineer must be able to move from:

```text
“users report the app feels slow”
```

to:

```text
“p75 INP regressed for low-end Android users
after version X; the dominant critical path is a 180 ms
main-thread task caused by component Y, amplified by a
late API response and an expensive layout pass.”
```

That is the difference between:

```text
performance opinion
```

and:

```text
performance engineering.
```

---

# 186. Final Principal Mental Model

Use:

```text
USER EXPERIENCE
      ↓
performance objective
      ↓
metric
      ↓
timeline
      ↓
causal decomposition
      ├── network
      ├── server
      ├── browser scheduling
      ├── JavaScript
      ├── parsing/compilation
      ├── layout
      ├── paint
      ├── memory/GC
      ├── storage
      └── coordination
      ↓
profiling / diagnostics
      ↓
hypothesis
      ↓
optimization
      ↓
verification
      ↓
performance budget
      ↓
continuous regression monitoring
```

For user journeys:

```text
interaction
→ application mark
→ async wait
→ resource timing
→ server timing
→ main-thread work
→ render
→ user-visible completion
```

For production telemetry:

```text
browser APIs
→ normalize
→ redact
→ sample
→ aggregate
→ batch
→ send
→ analyze
```

For performance incidents:

```text
confirm
→ segment
→ localize
→ profile
→ correlate
→ fix
→ verify
→ prevent recurrence
```

---

# 187. Final Principal Principle

> **You cannot optimize what you cannot explain, and you cannot explain performance with one timing number.**

The production-grade sequence is:

```text
define user-visible objective
→ choose correct metric
→ instrument critical path
→ capture browser timing
→ capture application timing
→ correlate backend timing
→ segment users
→ inspect percentiles
→ profile root cause
→ optimize smallest bottleneck
→ verify field + lab
→ enforce budget
→ monitor continuously
```

The central distinctions to internalize are:

```text
Date.now() is wall-clock time.

performance.now() is for elapsed performance measurement.

PerformanceEntry is a timeline record.

PerformanceObserver is an observation mechanism.

User Timing connects application semantics to browser timing.

Navigation Timing measures document navigation.

Resource Timing measures resource fetch behavior.

Paint timing measures rendering milestones.

LCP measures a user-visible loading outcome.

INP measures interaction responsiveness.

FID is historical; INP replaced it as the modern Core Web Vital.

Long Tasks reveal significant main-thread blocking.

Timer probes are not exact event-loop utilization.

RUM shows real users.

Synthetic monitoring shows controlled environments.

Profiling explains CPU/memory root cause.

Metrics tell you what happened.

Traces tell you how work flowed.

Sampling controls cost.

Percentiles reveal tails.

High cardinality destroys metric usability.

Telemetry can hurt performance.

Privacy constrains timing detail.

Performance budgets turn goals into engineering constraints.

```

At principal level, the key question is:

```text
“What exact user experience are we optimizing, what measurement proves
it is slow, what causal chain explains the delay, what is the smallest
safe intervention, and what instrumentation will prove that the problem
stays solved in production?”
```

Performance engineering is therefore a continuous loop:

```text
MEASURE
→ EXPLAIN
→ OPTIMIZE
→ VERIFY
→ PREVENT
```

—not a one-time benchmark.