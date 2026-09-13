# Chapter 99 — Myths and Misconceptions

> **JavaScript Mastery — Part XIX: Judgment**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** A JavaScript claim is trustworthy only when you know whether it comes from ECMAScript, a host platform, an engine implementation, a library/framework convention, or an application assumption.

---

# 1. Chapter Mission

JavaScript has accumulated decades of:

- half-truths
- historical rules
- engine folklore
- browser folklore
- framework slogans
- interview shortcuts
- simplified mental models

Some are useful teaching shortcuts. Some are wrong. Some are true only under narrow conditions.

This chapter trains a stronger habit:

```text
Claim
→ Scope
→ Authority
→ Mechanism
→ Counterexample
→ Evidence
→ Production consequence
```

The goal is not to memorize a list of “myths”.

The goal is to replace slogans with models that survive real debugging, code review, architecture work, performance analysis, and interviews.

---

# 2. Learning Objectives

By the end of this chapter you should be able to:

- distinguish a myth from a simplification
- distinguish language semantics from host behavior
- distinguish host behavior from engine implementation
- distinguish implementation detail from application convention
- explain values, bindings, references, and objects precisely
- explain scope, hoisting, TDZ, and closures without folklore
- explain `this` from invocation semantics
- explain prototypes and classes accurately
- explain coercion and equality accurately
- explain arrays, Maps, Sets, Symbols, and property ordering
- explain numbers, `NaN`, `-0`, BigInt, strings, Unicode, and time
- explain Promise/async behavior
- explain jobs, microtasks, tasks, and host event loops carefully
- distinguish concurrency from parallelism
- explain cancellation and timeouts
- reason about memory leaks and garbage collection
- distinguish ECMAScript guarantees from V8 implementation behavior
- reason about performance without syntax-level folklore
- explain ESM/CommonJS/tooling distinctions
- reason about browser and Node differences
- explain common web-security misconceptions
- reason about HTTP, caching, retries, idempotency, and distributed systems
- challenge interview slogans with precise corrections
- build minimal reproductions for disputed claims
- find and cite the authoritative source for a claim
- defend a principal-level engineering decision using evidence

---

# 3. Prerequisites

Recommended:

```text
Chapter 01 — Runtime Landscape
Chapter 02 — Values / Types
Chapter 07 — Coercion / Equality
Chapter 10–14 — Scope / Hoisting / Closures / this
Chapter 15–20 — Objects / Prototypes / Classes / Symbols
Chapter 22–28 — Arrays / Iteration / Serialization
Chapter 31–40 — Async / Promises / Events / Streams / Reactive
Chapter 41–44 — Specification Architecture
Chapter 45–48 — Memory / Engines / V8
Chapter 49–57 — Browser / Security
Chapter 58–63 — Node.js
Chapter 64–70 — Modules / Tooling
Chapter 78–89 — Production / Testing / Debugging
Chapter 90–97 — Modern / Compatibility / Legacy / Wasm / Edge
Chapter 98 — Anti-Patterns / Failure Modes
```

---

# 4. Source Hierarchy

Use this hierarchy when evaluating a claim:

```text
ECMAScript specification
        ↓
host/platform specification
        ↓
engine/runtime documentation
        ↓
library/framework documentation
        ↓
application convention
        ↓
blog/forum/interview claim
```

A browser API is not automatically an ECMAScript feature.

A V8 optimization is not automatically an ECMAScript guarantee.

A framework convention is not a language rule.

---

# 5. Myth #1 — “JavaScript and ECMAScript Are Exactly the Same Thing”

Not exactly.

ECMAScript defines the language.

A host environment provides additional capabilities:

```text
DOM
fetch
timers
filesystem
sockets
workers
process
storage
crypto
```

A useful model is:

```text
ECMAScript
+
host
+
runtime implementation
=
real JavaScript environment
```

---

# 6. Myth #2 — “JavaScript Runs the Same Everywhere”

False.

Language semantics can be standardized while host capabilities differ.

Compare:

```text
browser
Node.js
Deno
Bun
edge runtime
embedded engine
```

The language layer overlaps; the host layer does not.

---

# 7. Myth #3 — “The Browser Is the JavaScript Engine”

Oversimplified.

A modern browser contains:

```text
JavaScript engine
Web APIs
rendering engine
event system
network stack
storage
security model
workers
```

The JavaScript engine is only one part.

---

# 8. Myth #4 — “V8 Is JavaScript”

False.

V8 is an implementation of JavaScript/ECMAScript used in several environments.

Other engines exist.

Therefore:

```text
V8 behavior
≠
universal ECMAScript guarantee
```

---

# 9. Myth #5 — “Everything in JavaScript Is an Object”

Not literally.

Primitive values include:

```text
undefined
null
boolean
number
bigint
string
symbol
```

Objects form a distinct category.

Primitive values may participate in object-like operations through language mechanisms, but that does not make them ordinary objects.

---

# 10. Myth #6 — “Objects Are Passed by Reference”

This is a popular shortcut and an imprecise explanation.

JavaScript passes values.

For an object value, the callee receives a value that can designate the same object.

Mutation is observable:

```js
function change(user) {
  user.name = "Ada";
}

const user = { name: "Grace" };

change(user);

console.log(user.name);
```

The important distinction is:

```text
same object reachable
```

not:

```text
caller binding passed by reference
```

---

# 11. Myth #7 — “Reassigning a Parameter Changes the Caller’s Variable”

False.

```js
function replace(user) {
  user = { name: "Ada" };
}

const user = { name: "Grace" };

replace(user);

console.log(user.name);
```

The caller still points to the original object.

Separate:

```text
mutation of object
```

from:

```text
reassignment of local binding
```

---

# 12. Myth #8 — “const Makes Objects Immutable”

False.

```js
const state = {
  count: 0
};

state.count++;
```

works.

`const` protects the binding from reassignment, not the entire reachable object graph.

---

# 13. Myth #9 — “const Means the Value Never Changes”

More precise:

> The binding cannot be reassigned after initialization.

A mutable object referenced by that binding can still change.

---

# 14. Myth #10 — “let and const Are Not Hoisted”

This wording hides the important mechanism.

Lexical bindings are established during environment setup, but access before initialization is restricted by the Temporal Dead Zone.

```js
console.log(value);
const value = 10;
```

throws.

Better language:

```text
binding exists
+
binding not initialized
+
access prohibited
```

---

# 15. Myth #11 — “Hoisting Literally Moves Code Up”

No.

“Hoisting” is a useful teaching metaphor.

The specification describes declaration instantiation and environment setup rather than a universal source-rewrite operation.

---

# 16. Myth #12 — “Closures Copy Their Variables”

Oversimplified.

A closure preserves access to lexical environment state.

```js
function makeCounter() {
  let count = 0;

  return () => ++count;
}
```

The function continues to access the relevant binding.

Think:

```text
function
+
lexical environment access
```

not:

```text
function
+
frozen copies
```

---

# 17. Myth #13 — “Closures Automatically Cause Memory Leaks”

False.

Closures can extend lifetimes when long-lived references retain environments.

That is different from saying closures inherently leak.

Ask:

```text
what is reachable?
for how long?
through which reference?
is that lifetime intended?
```

---

# 18. Myth #14 — “Garbage Collection Means There Are No Memory Leaks”

False.

A garbage collector removes objects that are no longer reachable according to its model.

A memory leak can exist when an object remains reachable longer than intended.

Typical sources:

```text
global references
listeners
timers
subscriptions
caches
closures
registrations
```

---

# 19. Myth #15 — “GC Frees Memory Immediately When Something Becomes Unused”

Not guaranteed.

Eligibility for collection and actual reclamation are different concepts.

Do not build business logic around exact GC timing.

---

# 20. Myth #16 — “WeakMap Prevents All Memory Leaks”

False.

WeakMap provides weak semantics for keys.

It does not fix every retention path in the application.

---

# 21. Myth #17 — “WeakRef Is a Better Cache”

Not automatically.

Weak references do not provide deterministic eviction.

Caches generally need explicit policies:

```text
size
TTL
LRU
invalidation
ownership
```

---

# 22. Myth #18 — “FinalizationRegistry Is a Destructor”

False.

Finalization is not deterministic resource cleanup.

Do not depend on it for:

```text
closing files
releasing locks
closing sockets
committing transactions
```

---

# 23. Myth #19 — “Single-Threaded Means There Is No Concurrency”

False.

JavaScript execution on one agent is serialized, but applications can have concurrent activities involving:

```text
I/O
workers
processes
multiple agents
host tasks
message passing
```

---

# 24. Myth #20 — “Single-Threaded Means There Are No Race Conditions”

False.

Race conditions can occur across async suspension points:

```text
read state
await
other work changes state
resume
write based on old assumption
```

No simultaneous execution of two statements on one agent is required for the logical race to exist.

---

# 25. Myth #21 — “Promises Run on Another Thread”

False.

Promises model asynchronous completion and composition.

A promise does not imply:

```text
new OS thread
```

---

# 26. Myth #22 — “async Automatically Makes CPU Work Non-Blocking”

False.

```js
async function run() {
  hugeCalculation();
}
```

The calculation still executes on the current execution agent unless explicitly moved to another execution mechanism.

---

# 27. Myth #23 — “await Blocks the JavaScript Thread”

Misleading.

`await` suspends the async function's continuation while the surrounding execution environment remains able to process other work.

Think:

```text
pause this async continuation
```

not:

```text
freeze the whole runtime
```

---

# 28. Myth #24 — “await Makes Two Operations Sequential”

Not necessarily.

These operations may already have started:

```js
const a = fetchA();
const b = fetchB();

const valueA = await a;
const valueB = await b;
```

Distinguish:

```text
operation start
```

from:

```text
await point
```

---

# 29. Myth #25 — “Promise.all Creates Parallelism”

Not exactly.

`Promise.all()` aggregates Promise outcomes.

The underlying operations may be:

```text
concurrent I/O
already complete
same-thread CPU work
worker work
host operations
```

The method itself is not a promise of OS-level parallel execution.

---

# 30. Myth #26 — “Promise.all Cancels the Remaining Operations”

False.

Aggregate rejection is not automatic cancellation.

If underlying work must stop, cancellation must be explicitly supported.

---

# 31. Myth #27 — “Promise.race Cancels the Losing Promise”

False.

It determines which Promise settles the race result.

The losing operation can continue.

---

# 32. Myth #28 — “forEach Awaits Async Callbacks”

False.

```js
await items.forEach(async item => {
  await process(item);
});
```

does not wait for callback promises as many developers expect.

Use:

```js
for (const item of items) {
  await process(item);
}
```

for sequential processing, or:

```js
await Promise.all(
  items.map(process)
);
```

for aggregate concurrency.

---

# 33. Myth #29 — “map(async ...) Returns Resolved Values”

It returns an array of promises.

```js
const results =
  items.map(async item => process(item));
```

The usual aggregation form is:

```js
const results =
  await Promise.all(
    items.map(process)
  );
```

---

# 34. Myth #30 — “Every Promise Rejection Is Automatically Handled”

No.

Promise rejection ownership should be explicit.

Possible owners:

```text
caller
boundary
retry policy
UI layer
request handler
job processor
process-level safety handler
```

---

# 35. Myth #31 — “Catching an Error Means Handling It”

Not necessarily.

This:

```js
try {
  await work();
} catch {}
```

does not define a recovery strategy.

Handling should answer:

```text
recover?
retry?
transform?
propagate?
record?
alert?
terminate?
```

---

# 36. Myth #32 — “Retrying Is Always Safe”

No.

Retries can duplicate side effects.

Classify operations as:

```text
idempotent
conditionally idempotent
non-idempotent
```

Use idempotency mechanisms when necessary.

---

# 37. Myth #33 — “Exponential Backoff Solves Retry Storms”

Backoff helps.

A robust retry strategy may also require:

```text
jitter
attempt cap
deadline
retry ownership
circuit breaking
load shedding
```

---

# 38. Myth #34 — “Timeout Means the Work Stopped”

Not necessarily.

A timeout can only tell the caller:

```text
I am no longer waiting
```

unless cancellation reaches the underlying work.

---

# 39. Myth #35 — “Promise.race Is a Cancellation API”

No.

It is an aggregation/racing mechanism.

Cancellation requires an actual cancellation protocol such as:

```text
AbortSignal
custom cooperative cancellation
worker termination
resource-specific stop
```

---

# 40. Myth #36 — “The Event Loop Is One Universal Queue”

Oversimplified.

ECMAScript has job concepts.

Hosts add their own scheduling structures.

Browser and Node scheduling should not be collapsed into a single imaginary queue.

---

# 41. Myth #37 — “setTimeout(fn, 0) Runs Immediately”

False.

It schedules a timer callback subject to host scheduling and minimum-delay semantics.

Other work may happen first.

---

# 42. Myth #38 — “Zero-Millisecond Timeout Means Exactly Zero Delay”

False.

Timer scheduling is not a precise execution-time guarantee.

---

# 43. Myth #39 — “Every Callback Is Asynchronous”

False.

An API can invoke callbacks synchronously or asynchronously depending on its contract.

Never infer callback timing from the word “callback”.

---

# 44. Myth #40 — “Every Asynchronous API Uses Promises”

False.

JavaScript ecosystems contain:

```text
callbacks
events
streams
Promises
iterators
message passing
```

---

# 45. Myth #41 — “Microtasks and Tasks Are the Same”

No.

They refer to different scheduling structures.

Use the terminology appropriate to the language and host.

---

# 46. Myth #42 — “Microtasks Can Never Starve Other Work”

A sustained chain of microtasks can delay other work.

Therefore scheduling fairness must be considered in event-loop-heavy systems.

---

# 47. Myth #43 — “One await Equals One Event-Loop Tick”

No.

There is no universal one-to-one mapping.

---

# 48. Myth #44 — “this Is Determined by Where the Function Was Written”

For ordinary functions, `this` is generally determined by invocation semantics.

Arrow functions capture lexical `this`.

---

# 49. Myth #45 — “Arrow Functions Have Their Own Dynamic this”

False.

Arrow functions do not create their own `this` binding.

---

# 50. Myth #46 — “Methods Stay Bound to Their Object”

False.

```js
const fn = obj.method;
fn();
```

can lose the original receiver.

Use deliberate binding/wrapping where needed.

---

# 51. Myth #47 — “call, apply, and bind Are Interchangeable”

Not exactly.

```text
call
→ invoke now

apply
→ invoke now with argument collection

bind
→ produce a bound function
```

The timing and resulting function identity differ.

---

# 52. Myth #48 — “JavaScript Classes Are Basically Java Classes”

False.

JavaScript classes have their own semantics and are integrated with the prototype-based object model.

Do not import assumptions blindly from other class-oriented languages.

---

# 53. Myth #49 — “Class Methods Are Recreated on Every Instance”

Ordinary methods declared in a class body are associated with the class's prototype rather than copied as independent method functions onto every instance.

Class fields are a different mechanism.

---

# 54. Myth #50 — “Private #fields Are Just Naming Conventions”

False.

Private class elements use language-level private semantics.

They are not the same as:

```js
obj._secret
```

---

# 55. Myth #51 — “Prototypes Are Just Legacy Inheritance”

No.

Prototype lookup is central to ordinary object property semantics.

Class syntax does not eliminate the prototype model.

---

# 56. Myth #52 — “Prototype Lookup Copies the Property Onto the Object”

False.

Lookup can find a property on the prototype chain without creating an own property on the receiver.

---

# 57. Myth #53 — “__proto__ Is the Normal Way to Build Inheritance”

It is a legacy-facing property accessor with special behavior.

Prefer explicit modern mechanisms and stable object construction when designing new code.

---

# 58. Myth #54 — “Spread Is a Deep Clone”

False.

```js
const copy = {
  ...original
};
```

is shallow with respect to nested object references.

---

# 59. Myth #55 — “Object.assign Is a Deep Clone”

False.

It is shallow.

---

# 60. Myth #56 — “structuredClone Clones Every JavaScript Value”

False.

Structured cloning supports a defined set of values and has explicit limits.

It is not a universal clone operation for every object identity/prototype/custom-runtime state.

---

# 61. Myth #57 — “JSON.stringify Is a Lossless Representation of JavaScript Objects”

False.

JSON has a narrower model than JavaScript.

Several JavaScript values and structures are not represented losslessly.

---

# 62. Myth #58 — “JSON.parse(JSON.stringify(x)) Is a Universal Deep Clone”

False.

It can:

```text
lose undefined
reject BigInt
lose functions
change special values
lose custom prototypes
fail on cycles
alter object semantics
```

---

# 63. Myth #59 — “Arrays Are Just Objects, So Their Performance Is the Same”

Arrays are objects but engines can use specialized representations and optimized paths for arrays.

Do not infer runtime performance from the object model alone.

---

# 64. Myth #60 — “Arrays Always Use Contiguous Memory”

Not a portable guarantee.

Engines may use:

```text
packed representations
sparse representations
specialized element kinds
other internal layouts
```

---

# 65. Myth #61 — “delete array[i] Removes the Element and Shifts Everything”

False.

`delete` creates a hole rather than performing `splice()`-style reindexing.

---

# 66. Myth #62 — “array.length Is the Number of Existing Elements”

Not necessarily.

Sparse arrays can have a large length with holes.

---

# 67. Myth #63 — “for...in Is the Standard Array Iterator”

It enumerates enumerable property keys and can include inherited properties.

Use array-specific iteration for array semantics.

---

# 68. Myth #64 — “for...of Gives Array Indexes”

It gives iterable values.

Use:

```js
array.entries()
```

when indexes and values are needed together.

---

# 69. Myth #65 — “Map Is Just a Faster Object”

No.

Map and Object differ in:

```text
key semantics
prototype interaction
API
iteration
intended use
```

---

# 70. Myth #66 — “Map Is Always Faster”

False.

Performance depends on:

```text
workload
key types
access patterns
engine
allocation
data size
```

---

# 71. Myth #67 — “Object.keys and for...in Return the Same Keys”

No.

`Object.keys()` returns own enumerable string keys.

`for...in` may include inherited enumerable keys.

---

# 72. Myth #68 — “Property Order Is Random”

Modern ECMAScript defines property-order semantics for relevant operations.

Do not rely on very old folklore.

---

# 73. Myth #69 — “Property Order Is Simply Insertion Order for Every Key”

Not universally.

Different categories of property keys have defined ordering behavior.

---

# 74. Myth #70 — “Symbols Are Not Properties”

Symbols are valid property keys.

---

# 75. Myth #71 — “Symbol Properties Are Secret”

They are not cryptographic secrets.

Symbol-aware reflection can discover symbol properties.

---

# 76. Myth #72 — “Object.freeze Deep-Freezes the Object Graph”

False.

It is shallow.

---

# 77. Myth #73 — “Mutation Is Always Bad”

No.

Mutation can be appropriate within:

```text
local state
encapsulated internals
builders
controlled caches
performance-sensitive code
```

The key concern is ownership and observability.

---

# 78. Myth #74 — “Immutability Means Nothing Can Be Shared”

False.

Immutable data can share structure safely.

Immutability concerns mutation guarantees, not mandatory duplication.

---

# 79. Myth #75 — “Functional Programming Means Never Mutating Anything”

Not necessarily.

Functional techniques often emphasize controlled effects and composability rather than an absolute ban on mutation in all implementation details.

---

# 80. Myth #76 — “0.1 + 0.2 Is a JavaScript Bug”

No.

It follows from binary floating-point representation and rounding.

---

# 81. Myth #77 — “Every Integer Is Exactly Representable by number”

False for sufficiently large integer values.

Use BigInt or another exact representation where required.

---

# 82. Myth #78 — “NaN Is Not a Number Type”

`NaN` is a special value of the `number` type.

```js
typeof NaN === "number"
```

---

# 83. Myth #79 — “NaN Equals Itself”

False.

```js
NaN === NaN
```

is false.

Use:

```js
Number.isNaN(value)
```

when appropriate.

---

# 84. Myth #80 — “+0 and -0 Are Identical Everywhere”

They compare equal under some operations but are distinguishable by others, including:

```js
Object.is(-0, 0)
```

---

# 85. Myth #81 — “null and undefined Are the Same”

False.

They are distinct values with different semantic conventions.

---

# 86. Myth #82 — “Missing Property and undefined Property Are Identical”

Not always.

```js
const a = {};
const b = { value: undefined };

"value" in a; // false
"value" in b; // true
```

---

# 87. Myth #83 — “Optional Chaining Makes the Whole Expression Safe”

No.

Optional chaining short-circuits specific access/call chains.

Other operations may still throw.

---

# 88. Myth #84 — “?? and || Mean the Same Thing”

False.

```js
0 || 10   // 10
0 ?? 10   // 0
```

`||` uses truthiness.

`??` uses nullishness.

---

# 89. Myth #85 — “=== Means Every Value Is Compared Mathematically”

Strict equality is a specific language relation with special behavior for values such as:

```text
NaN
+0
-0
```

---

# 90. Myth #86 — “Object.is Is Just a Better ===”

It is a different equality relation.

It treats:

```text
NaN as equal to NaN
+0 and -0 as different
```

---

# 91. Myth #87 — “String length Equals Human Character Count”

False.

JavaScript string indexing uses UTF-16 code units.

Unicode code points and grapheme clusters can require multiple code units.

---

# 92. Myth #88 — “One Visible Character Is Always One Code Unit”

False.

Emoji and combining sequences are common counterexamples.

---

# 93. Myth #89 — “ASCII Is the JavaScript String Model”

False.

JavaScript strings support Unicode-related data beyond ASCII.

---

# 94. Myth #90 — “Date Solves All Time Modeling”

No.

Different concepts include:

```text
instant
local date
local time
time zone
calendar date
recurrence
```

Do not force them into one model.

---

# 95. Myth #91 — “UTC Solves Every Date/Time Problem”

No.

UTC is important for instants.

It does not express every calendar/time-zone/business rule.

---

# 96. Myth #92 — “All Date Strings Parse Portably”

Do not assume arbitrary date-string formats are interpreted identically everywhere.

Use explicit, well-defined formats.

---

# 97. Myth #93 — “Regex Is the Universal Parser”

Regex is useful for many formats.

It is not automatically the right solution for complex grammars.

---

# 98. Myth #94 — “Regex Is Always Faster Than a Parser”

No.

Complex regexes can be expensive.

Benchmark the actual parser and input distribution.

---

# 99. Myth #95 — “Exceptions and Promise Rejections Are Exactly the Same”

They are related through async control flow, but a Promise rejection is an asynchronous state of a Promise.

An exception inside an async function is represented through the returned Promise.

---

# 100. Myth #96 — “try/catch Cannot Handle Promise Rejections”

It can when the rejection is awaited:

```js
try {
  await operation();
} catch (error) {
  recover(error);
}
```

---

# 101. Myth #97 — “Async Function Throwing Is Always a Synchronous Throw to Its Caller”

An async function represents failure through its returned rejected Promise.

---

# 102. Myth #98 — “Every awaitable Must Already Be a Promise”

Await processing can handle values and thenables according to the language's await semantics.

---

# 103. Myth #99 — “then Callbacks Run Immediately”

Promise reactions are scheduled asynchronously through the language's job machinery.

---

# 104. Myth #100 — “Microtasks Always Run Before Every Possible Host Action”

Host scheduling details matter.

Do not turn a useful rule of thumb into a universal theorem about every host phase.

---

# 105. Myth #101 — “setInterval Means Exactly Every N Milliseconds”

It does not guarantee exact execution cadence.

Callback runtime, scheduling, load, and host behavior matter.

---

# 106. Myth #102 — “setInterval Cannot Overlap Async Work”

The timer callback itself can complete quickly while work started inside it remains pending.

That can create overlapping in-flight operations.

---

# 107. Myth #103 — “Clearing a Timer Cancels Code Already Executing”

Clearing a timer prevents relevant future callbacks from being scheduled/fired; it cannot undo already-running synchronous work.

---

# 108. Myth #104 — “Node.js Is a Single-Threaded Process and Nothing More”

Node includes an event-driven JavaScript execution model plus runtime facilities for:

```text
I/O
worker threads
child processes
libuv-backed work
native code
```

---

# 109. Myth #105 — “Node Async I/O Means CPU Can Never Block”

CPU-heavy JavaScript still occupies the execution agent.

Synchronous APIs and poorly behaving native operations can also affect responsiveness.

---

# 110. Myth #106 — “Every Node API Is Promise-Based”

False.

Node provides:

```text
callbacks
Promises
events
streams
sync APIs
```

---

# 111. Myth #107 — “Synchronous Node APIs Are Always Wrong”

They can be appropriate for:

```text
CLI tools
startup
bootstrap
scripts
controlled administration
```

They are often inappropriate in latency-sensitive request paths.

---

# 112. Myth #108 — “process.nextTick Is Just Another Promise.then”

Not identical.

Node-specific scheduling semantics differ.

Excessive next-tick work can also delay other work.

---

# 113. Myth #109 — “CommonJS and ESM Are the Same System”

They interoperate but have distinct semantics.

Differences include:

```text
loading
resolution
exports
evaluation
caching
interop
```

---

# 114. Myth #110 — “ESM Is CommonJS With Different Syntax”

False.

The module systems have different semantics and dependency graph models.

---

# 115. Myth #111 — “Dynamic import Is Just require() With New Syntax”

`import()` is Promise-based ESM loading.

It is not merely a renamed CommonJS function.

---

# 116. Myth #112 — “Tree Shaking Removes All Unused Code Automatically”

Tree shaking depends on:

```text
static analyzability
module structure
side effects
bundler
configuration
package metadata
```

---

# 117. Myth #113 — “ESM Automatically Means a Small Bundle”

ESM enables strong static analysis, but final size still depends on dependency structure and build configuration.

---

# 118. Myth #114 — “Barrel Files Always Improve Architecture”

Barrels can improve public API organization.

They can also introduce:

```text
cycles
larger dependency surfaces
less obvious dependency relationships
```

---

# 119. Myth #115 — “Lockfiles Make Dependencies Safe”

Lockfiles improve reproducibility.

They do not prove:

```text
vulnerability absence
maintainer trust
safe build scripts
```

---

# 120. Myth #116 — “Fewer Dependencies Always Means More Security”

Not necessarily.

A mature dependency can reduce implementation risk.

The right question is:

```text
which dependency
what trust
what attack surface
what maintenance burden
```

---

# 121. Myth #117 — “Writing It Yourself Is Always Safer”

Not for complex/security-sensitive functionality.

Reimplementing:

```text
cryptography
parsers
protocols
serialization
```

can introduce more bugs.

---

# 122. Myth #118 — “eval Is Just a Slow Function”

`eval` changes the code-execution model and can create serious security risks with untrusted input.

---

# 123. Myth #119 — “Base64 Is Encryption”

False.

Base64 is encoding.

---

# 124. Myth #120 — “HTTPS Makes the Application Secure”

HTTPS protects communication properties.

It does not automatically prevent:

```text
XSS
authorization flaws
injection
SSRF
business-logic errors
```

---

# 125. Myth #121 — “CORS Is Authentication”

False.

CORS is a browser cross-origin access-control mechanism.

It does not identify or authorize a user.

---

# 126. Myth #122 — “CORS Protects the Server From Non-Browser Clients”

No.

CORS is enforced by browsers.

Servers still need authentication and authorization.

---

# 127. Myth #123 — “Same-Origin Policy Means Cross-Origin Requests Never Happen”

Cross-origin requests can occur.

The browser controls access to results and related capabilities according to web security rules.

---

# 128. Myth #124 — “Client Validation Is a Security Boundary”

Client validation is useful for UX.

The server must enforce security and business rules.

---

# 129. Myth #125 — “TypeScript Types Validate Runtime JSON”

Normally they do not.

A static type:

```ts
type User = {
  id: string;
};
```

does not validate arbitrary runtime input.

---

# 130. Myth #126 — “TypeScript Prevents Runtime Type Errors”

Static checking can eliminate many classes of mistakes before runtime.

External inputs still require runtime validation when correctness/security requires it.

---

# 131. Myth #127 — “JSON.parse Produces Values Matching Your Type”

No.

It parses JSON syntax.

It does not prove application-level schema correctness.

---

# 132. Myth #128 — “CSP Replaces Secure Coding”

CSP is a defense layer.

It does not replace safe data handling and secure architecture.

---

# 133. Myth #129 — “Sanitization Works the Same for Every Output Sink”

No.

Encoding and sanitization are context-specific:

```text
HTML
attribute
URL
CSS
JavaScript
SQL
shell
```

---

# 134. Myth #130 — “Prototype Pollution Is Only a Legacy Browser Problem”

Unsafe merge/path handling can create prototype-pollution risk in modern JavaScript applications.

---

# 135. Myth #131 — “Object.create(null) Makes Code Secure”

It creates an object without the ordinary Object prototype.

It does not solve every security issue.

---

# 136. Myth #132 — “Symbols Are Security Secrets”

No.

Symbol keys reduce accidental collisions; they do not provide confidentiality.

---

# 137. Myth #133 — “Private Fields Are a Cryptographic Security Boundary”

Private fields provide language-level encapsulation.

They do not protect a compromised process or hostile runtime environment.

---

# 138. Myth #134 — “Front-End Secrets Can Be Hidden With Obfuscation”

If a secret is delivered to the client, assume it can be recovered.

Obfuscation is not secret protection.

---

# 139. Myth #135 — “Source Maps Should Always Be Public”

Source maps are useful for debugging but may expose source details.

Exposure is a deployment/security decision.

---

# 140. Myth #136 — “Database Data Is Automatically Trusted”

Persisted data can be stale, corrupt, migrated, manually altered, or attacker-influenced.

Validate assumptions where they matter.

---

# 141. Myth #137 — “Awaiting a Database Update Makes It Atomic”

No.

Atomicity belongs to the appropriate database transaction/locking mechanism.

---

# 142. Myth #138 — “A Database Transaction Makes the Whole Distributed Workflow Atomic”

A local transaction cannot automatically make:

```text
database
+
email
+
queue
+
external API
```

atomic as one operation.

---

# 143. Myth #139 — “Timestamp Gives Total Order in Distributed Systems”

A timestamp can help correlate events.

It does not automatically create causality or total ordering.

---

# 144. Myth #140 — “Exactly-Once Delivery Means Exactly-Once Business Effect”

Transport semantics and business-effect semantics are distinct.

Idempotency can turn duplicate deliveries into one logical effect.

---

# 145. Myth #141 — “A Queue Guarantees Reliability”

Queues can provide useful buffering and decoupling.

They can also suffer:

```text
backlog
poison messages
partitions
expiry
consumer failures
```

---

# 146. Myth #142 — “A Queue Automatically Guarantees Ordering”

Ordering depends on:

```text
queue model
partitioning
consumer behavior
parallelism
```

---

# 147. Myth #143 — “Event-Driven Architecture Eliminates Coupling”

It can reduce direct coupling while introducing:

```text
schema coupling
ordering
delivery semantics
temporal coupling
```

---

# 148. Myth #144 — “Events Are Always Better Than Function Calls”

No.

Use the mechanism that matches the relationship.

---

# 149. Myth #145 — “Every System Should Be Event-Driven”

No.

Event-heavy systems can become difficult to reason about when direct calls would be clearer.

---

# 150. Myth #146 — “Caching Always Improves Performance”

Caching can hurt when:

```text
invalidations are expensive
data is rarely reused
memory pressure rises
stale data is dangerous
cache access is remote
```

---

# 151. Myth #147 — “An In-Memory Cache Is Free”

It consumes:

```text
memory
CPU
invalidation complexity
operational attention
```

---

# 152. Myth #148 — “Memoization Always Makes Code Faster”

Memoization can cost more than the computation when reuse is low.

Unbounded memoization can also create memory pressure.

---

# 153. Myth #149 — “Debouncing Makes Everything Faster”

Debouncing changes frequency and timing.

It can lower work while increasing perceived delay.

---

# 154. Myth #150 — “Throttling and Debouncing Are the Same”

Different goals:

```text
debounce
→ wait for quiet

throttle
→ limit frequency
```

---

# 155. Myth #151 — “Lazy Loading Always Improves Performance”

It may reduce initial cost while increasing later latency or creating request waterfalls.

Measure the full user journey.

---

# 156. Myth #152 — “Code Splitting Always Reduces Total Cost”

It can improve initial transfer while increasing request/coordination complexity.

---

# 157. Myth #153 — “A Bigger Bundle Automatically Means a Slower App”

Bundle size is one dimension among:

```text
download
parse
compile
execute
memory
render
network
```

---

# 158. Myth #154 — “A Smaller Bundle Automatically Means a Faster App”

An app can have a small bundle and still be dominated by:

```text
database
network
images
rendering
serialization
```

---

# 159. Myth #155 — “Big-O Completely Determines Performance”

Big-O describes growth.

Real performance also depends on:

```text
constant factors
allocation
memory locality
I/O
branching
implementation
workload
```

---

# 160. Myth #156 — “for Is Always Faster Than forEach”

Not a universal law.

If it matters, benchmark the real workload.

---

# 161. Myth #157 — “Arrow Functions Are Always Faster”

False.

Syntax choice alone does not establish performance.

---

# 162. Myth #158 — “Classes Are Always Slower Than Functions”

No universal rule.

Measure actual workload and runtime.

---

# 163. Myth #159 — “V8 Hidden Classes Are an ECMAScript Feature”

No.

They are implementation concepts.

They should not be used as language-level correctness rules.

---

# 164. Myth #160 — “Changing a Type Deopts a Function Forever”

Optimization behavior is dynamic and implementation-specific.

Avoid simplistic permanent-deoptimization stories.

---

# 165. Myth #161 — “You Must Write Every Object in One Exact Property Order”

Stable shapes can matter in some hot paths.

Do not contort ordinary application code around undocumented optimization folklore without measurement.

---

# 166. Myth #162 — “Property Lookup Is Always a Hash Table Lookup”

That is an implementation simplification.

Engines use specialized representations and caches.

---

# 167. Myth #163 — “Map Lookup Is Guaranteed Constant-Time in Every Real Scenario”

Complexity analysis and implementation behavior are not identical.

Use realistic workload measurement.

---

# 168. Myth #164 — “Object Pooling Is Always a Performance Win”

Pooling can reduce allocation in selected systems.

It can also add:

```text
retention
reset bugs
complexity
```

---

# 169. Myth #165 — “Garbage Collection Is Free”

GC consumes resources and can affect latency/memory behavior.

---

# 170. Myth #166 — “Allocations Are Always Bad”

Allocation is normal.

Optimize allocation when:

```text
profiling shows it matters
lifetime is problematic
throughput requires it
```

---

# 171. Myth #167 — “A Heap Increase Means a Memory Leak”

Not automatically.

Investigate:

```text
retention
allocation rate
GC behavior
cache growth
buffers
native memory
```

---

# 172. Myth #168 — “Setting a Reference to null Releases Memory”

Only when that reference was a relevant retaining path and other conditions permit reclamation.

---

# 173. Myth #169 — “Heap Snapshot Size Alone Diagnoses the Leak”

Leak investigation often requires:

```text
retainers
snapshots over time
allocation profiling
runtime/native metrics
```

---

# 174. Myth #170 — “Async Recursion Cannot Create Resource Problems”

It may avoid synchronous stack growth while still creating:

```text
unbounded work
queue growth
memory growth
```

---

# 175. Myth #171 — “Recursion Is Always Slower Than Iteration”

No universal guarantee.

Choose based on algorithm, depth, clarity, and measured constraints.

---

# 176. Myth #172 — “Tail Calls Are Always Optimized”

Do not assume universal proper-tail-call optimization in common production engines.

---

# 177. Myth #173 — “Regexes Are Recompiled Every Time the Line Runs”

Do not use this simplistic statement as a universal model of engine compilation.

---

# 178. Myth #174 — “String Concatenation Is Always Slow”

Modern engines optimize common string operations.

Workload matters.

---

# 179. Myth #175 — “Template Literals Are Always Slower Than +”

No universal rule.

---

# 180. Myth #176 — “Destructuring Is Always a Performance Problem”

Source syntax alone is not sufficient evidence of a bottleneck.

---

# 181. Myth #177 — “Optional Chaining Is Always Slower”

Again, measure actual hot code if this matters.

---

# 182. Myth #178 — “Code Comments Are Free”

Comments can drift.

Executable tests and contracts provide stronger guarantees.

---

# 183. Myth #179 — “100% Test Coverage Means 100% Correctness”

Coverage measures exercised structure.

It does not prove semantic completeness.

---

# 184. Myth #180 — “More Mocks Make Tests More Reliable”

Over-mocking can create tests that validate mocks rather than real behavior.

---

# 185. Myth #181 — “A Snapshot Replaces Assertions”

Snapshots can be useful but may hide the exact behavior that matters.

---

# 186. Myth #182 — “Sleeping in Tests Is the Easiest Way to Wait”

```js
await delay(1000);
```

is usually brittle.

Prefer explicit synchronization, controlled clocks, or condition-based waiting with deadlines.

---

# 187. Myth #183 — “Retrying a Flaky Test Fixes It”

Retries can hide a race rather than remove it.

---

# 188. Myth #184 — “Unit Tests Prove Integration”

Unit tests isolate.

Integration tests validate boundaries.

End-to-end tests validate larger workflows.

Use each layer for its intended question.

---

# 189. Myth #185 — “Production Debugging Is Just More console.log”

Good debugging uses evidence:

```text
logs
metrics
traces
profiles
heap snapshots
deployment metadata
request samples
```

---

# 190. Myth #186 — “More Logs Always Improve Debugging”

Large unstructured logs can lower signal-to-noise and increase cost/privacy risk.

---

# 191. Myth #187 — “A Stack Trace Is the Root Cause”

A stack trace shows where a failure became observable.

The root cause may be upstream state, timing, input, configuration, or dependency behavior.

---

# 192. Myth #188 — “Low CPU Means the System Is Healthy”

The bottleneck can be:

```text
I/O
memory
database
connections
locks
queues
rate limits
```

---

# 193. Myth #189 — “High CPU Automatically Means a Performance Bug”

High CPU can represent productive work or expected utilization.

Measure against service objectives.

---

# 194. Myth #190 — “Every Slow Request Is a JavaScript Problem”

The dominant cost may be:

```text
database
network
serialization
external API
cold start
```

---

# 195. Myth #191 — “JIT Makes Every Function Fast”

Optimization is selective and workload-dependent.

---

# 196. Myth #192 — “The JIT Can Fix Bad Algorithms”

A JIT cannot generally turn poor application-level algorithmic complexity into the intended algorithm.

---

# 197. Myth #193 — “WebAssembly Is Always Faster Than JavaScript”

Not universally.

Boundary crossing, copying, startup, workload shape, and algorithm matter.

---

# 198. Myth #194 — “Wasm Has No Interop Overhead”

Interoperability has a cost.

The entire pipeline must be measured.

---

# 199. Myth #195 — “Native Addons Are Automatically Faster”

Native boundaries, memory management, build complexity, and data conversion can dominate.

---

# 200. Myth #196 — “Serverless Means No Server”

It changes the operational model; it does not eliminate infrastructure.

---

# 201. Myth #197 — “Serverless Is Stateless”

Instances can have transient in-memory state.

It should not be treated as durable shared state.

---

# 202. Myth #198 — “Warm Serverless Instances Are Guaranteed”

Warm reuse is not a correctness guarantee.

---

# 203. Myth #199 — “Edge Means Zero Latency”

Edge compute can reduce some network distance.

It cannot remove downstream latency, database cost, serialization, or coordination.

---

# 204. Myth #200 — “The Closest Compute Location Is Always Fastest”

Data location and compute location are different dimensions.

---

# 205. Myth #201 — “A Global Edge Cache Automatically Preserves Authorization”

Personalized content requires careful cache-keying and policy.

---

# 206. Myth #202 — “Provider Compatibility Means Full API Compatibility”

Compatibility layers usually expose a defined subset.

Read the runtime's compatibility contract.

---

# 207. Myth #203 — “Stage 3 Means Production-Ready Everywhere”

Proposal stage is not a fleet-wide availability guarantee.

---

# 208. Myth #204 — “Latest ECMAScript Means Latest Browser Support”

Specification publication and runtime implementation timelines differ.

---

# 209. Myth #205 — “Polyfills Are Always Harmless”

Polyfills can add:

```text
bundle size
startup
global behavior
maintenance
conflicts
```

---

# 210. Myth #206 — “Transpilation and Polyfilling Are the Same”

They solve different problems:

```text
transpilation
→ transform syntax/code

polyfill
→ supply runtime capability
```

---

# 211. Myth #207 — “Babel Makes Every Modern Feature Work Everywhere”

Only transformable features and appropriately supported runtime capabilities can be covered.

---

# 212. Myth #208 — “Source Maps Change Runtime Performance”

They primarily affect developer tooling and source mapping.

---

# 213. Myth #209 — “User-Agent Detection Is Always Wrong”

It is often brittle for capability detection.

There are still cases where platform/version information is relevant for diagnostics or known bugs.

---

# 214. Myth #210 — “Feature Detection Is Perfect”

Feature presence does not always prove identical behavior.

Sometimes known version-specific workarounds remain necessary.

---

# 215. Myth #211 — “A Polyfill Makes the Platform Feature Identical”

A polyfill approximates a capability within what can be implemented in the available environment.

---

# 216. Myth #212 — “Fetch Rejects for HTTP 404”

A normal HTTP error response is still a response.

Application code usually needs to inspect status or `ok`.

---

# 217. Myth #213 — “404 Is a Network Error”

It is an HTTP status response.

Network failure and HTTP error response are different categories.

---

# 218. Myth #214 — “Timeout Means the Server Did Nothing”

The server may already have completed or accepted the operation.

This is a major retry/idempotency consideration.

---

# 219. Myth #215 — “Client Timeout Automatically Cancels Server Work”

No distributed cancellation occurs unless the protocol/system explicitly implements it.

---

# 220. Myth #216 — “POST Is Never Retryable”

Retry safety depends on the application's semantics and idempotency controls.

---

# 221. Myth #217 — “A GET Is Always Safe to Repeat”

HTTP method semantics provide useful defaults, but application side effects and broken implementations can violate assumptions.

---

# 222. Myth #218 — “JWT Automatically Means Stateless Authentication”

JWT is a token format/representation.

Systems can still maintain:

```text
sessions
revocation
user state
authorization state
key rotation
```

---

# 223. Myth #219 — “JWT Is More Secure Because It Is a Token”

Security comes from:

```text
validation
keys
expiry
audience
issuer
storage
transport
authorization
```

---

# 224. Myth #220 — “Hiding a Button Is Authorization”

No.

The actual protected operation must enforce authorization.

---

# 225. Myth #221 — “Checking User ID Is Enough for Authorization”

Authorization can depend on:

```text
resource
tenant
ownership
role
policy
operation
state
```

---

# 226. Myth #222 — “Database Transactions Eliminate Race Conditions”

They address specific consistency/atomicity problems.

Application-level races can remain.

---

# 227. Myth #223 — “Read Replicas Are Transparent”

Replication can introduce lag and consistency differences.

---

# 228. Myth #224 — “More Database Connections Increase Throughput Forever”

Beyond capacity, more connections can increase contention and latency.

---

# 229. Myth #225 — “Indexes Always Make Queries Faster”

Indexes can also increase:

```text
write cost
storage
maintenance
planning complexity
```

---

# 230. Myth #226 — “N+1 Happens Only in the Frontend”

It can occur in:

```text
ORM
database
HTTP services
microservices
GraphQL
```

---

# 231. Myth #227 — “Parallel Requests Always Beat Batching”

Batching can reduce round trips.

Parallelism can overload downstream systems.

---

# 232. Myth #228 — “Serialization Is Free”

It consumes:

```text
CPU
memory
bytes
latency
```

---

# 233. Myth #229 — “Compression Fixes an Inefficient API”

Compression reduces bytes.

It does not eliminate unnecessary calls or a poor data model.

---

# 234. Myth #230 — “Caching Fixes a Slow Database Permanently”

Caching can reduce reads.

It can also introduce stale-data and invalidation problems.

---

# 235. Myth #231 — “Object Pools Are the Best Cure for Allocation”

Only when profiling supports the trade-off.

---

# 236. Myth #232 — “A Queue Removes Backpressure”

A queue is one mechanism for absorbing/controlling work.

If producers create work faster than consumers can process it, backlog grows.

---

# 237. Myth #233 — “Backpressure Means the Consumer Becomes Faster”

Backpressure controls flow.

It does not create capacity.

---

# 238. Myth #234 — “Streams Automatically Prevent Memory Growth”

Streaming can reduce materialization.

Bad buffering or queueing can still exhaust memory.

---

# 239. Myth #235 — “Node Streams and Web Streams Are Identical”

They solve related problems through distinct APIs and integration models.

---

# 240. Myth #236 — “DOM Operations Are Always Cheap”

DOM work can trigger broader rendering consequences such as:

```text
style recalculation
layout
paint
compositing
```

depending on the operation.

---

# 241. Myth #237 — “requestAnimationFrame Is Exactly 16.67 ms”

Cadence depends on display, browser, load, and scheduling.

---

# 242. Myth #238 — “requestIdleCallback Is Guaranteed to Run”

Idle work is opportunistic.

Do not rely on it for required persistence.

---

# 243. Myth #239 — “localStorage Is a Database”

It is browser storage with synchronous access and substantial limitations.

Use an appropriate storage system for the workload.

---

# 244. Myth #240 — “Cookies Are Just Small localStorage”

Cookies interact with HTTP and have security attributes, server visibility, and different lifecycle semantics.

---

# 245. Myth #241 — “Service Workers Are General-Purpose Background Threads”

They are specialized web platform workers with a managed lifecycle.

---

# 246. Myth #242 — “Shadow DOM Is a Security Boundary”

Shadow DOM provides encapsulation behavior, not a general security boundary.

---

# 247. Myth #243 — “Workers Share Ordinary JavaScript Objects”

Workers have isolated execution contexts and communicate through defined messaging/transfer/shared-memory mechanisms.

---

# 248. Myth #244 — “SharedArrayBuffer Shares Arbitrary Objects”

Shared memory concerns shared buffer data, not arbitrary object identity.

---

# 249. Myth #245 — “Atomics Solves Concurrency Automatically”

Atomics supplies primitives.

Correctness still depends on the synchronization protocol.

---

# 250. Myth #246 — “EventEmitter Provides Exactly-Once Delivery”

An event emitter is not a durable distributed message system.

---

# 251. Myth #247 — “Events Guarantee Ordering Across the Entire Architecture”

Local emitter order is not the same as distributed causal ordering.

---

# 252. Myth #248 — “A Global Event Bus Reduces All Coupling”

It can replace visible coupling with hidden temporal dependency.

---

# 253. Myth #249 — “A Singleton Is Always an Anti-Pattern”

A singleton can be appropriate when one process-level resource genuinely exists.

The problem is often hidden mutable global state, not the word singleton.

---

# 254. Myth #250 — “A Singleton Is Always the Best Way to Share State”

No.

Explicit dependency ownership is often easier to test and reason about.

---

# 255. Myth #251 — “DRY Means Never Duplicate Code”

Some duplication is cheaper than a wrong abstraction.

---

# 256. Myth #252 — “More Abstraction Means Better Architecture”

More abstraction also means more indirection.

The value is in isolating meaningful variation and stable boundaries.

---

# 257. Myth #253 — “A Generic Utility Is Automatically More Reusable”

Generic APIs can accumulate accidental assumptions.

---

# 258. Myth #254 — “Clean Architecture Requires Many Layers”

Architecture principles are about dependency direction and boundaries, not mandatory layer counts.

---

# 259. Myth #255 — “Dependency Injection Requires a Framework”

Explicit constructor/function arguments are already dependency injection.

---

# 260. Myth #256 — “Dependency Injection Means Everything Must Be Configurable”

Over-configurability increases state-space and complexity.

Inject meaningful variation.

---

# 261. Myth #257 — “Design Patterns Are Recipes”

Patterns are reusable structures/vocabulary, not mandatory recipes.

---

# 262. Myth #258 — “Microservices Automatically Scale Better”

Service splitting does not automatically remove bottlenecks.

---

# 263. Myth #259 — “Microservices Automatically Improve Team Ownership”

Ownership still requires:

```text
clear domains
deployment autonomy
operational practices
```

---

# 264. Myth #260 — “A Service Mesh Eliminates Network Failures”

It can improve traffic management and visibility.

The network remains fallible.

---

# 265. Myth #261 — “An API Gateway Eliminates Coupling”

It can centralize concerns while still leaving domain and data dependencies.

---

# 266. Myth #262 — “A Rewrite Is Always Better Than Incremental Migration”

Rewrites can remove historical constraints while also discarding operational knowledge.

Choose according to risk and business value.

---

# 267. Myth #263 — “Legacy Code Must Be Rewritten”

Stable, low-risk, well-tested legacy can be cheaper to keep.

---

# 268. Myth #264 — “Modern Syntax Automatically Means Modern Architecture”

Syntax does not determine architecture.

---

# 269. Myth #265 — “Latest Technology Automatically Reduces Technical Debt”

New technology can create new debt.

---

# 270. Myth #266 — “Best Practice Means One Best Implementation”

A best practice is usually a strong default under stated conditions, not a universal law.

---

# 271. Myth #267 — “Code Smell Means Code Is Definitely Wrong”

A smell is a reason to investigate, not a proof of failure.

---

# 272. Myth #268 — “All Technical Debt Should Be Removed Immediately”

Debt reduction is prioritization under risk and resource constraints.

---

# 273. Myth #269 — “If Everyone Believes a Rule, It Is Probably True”

Popularity is not evidence.

---

# 274. Myth #270 — “An Expert Should Never Say I Don't Know”

Principal engineers distinguish:

```text
known
unknown
assumed
measured
specified
implementation-specific
```

---

# 275. Myth #271 — “Confidence Is Evidence”

No.

Evidence includes:

```text
specification
reproduction
benchmark
profile
test
production telemetry
```

---

# 276. Myth #272 — “You Should Memorize the Entire ECMAScript Spec”

No.

You should understand its structure and know how to navigate relevant concepts.

---

# 277. Myth #273 — “The Spec Describes the Exact Machine Instructions”

No.

The specification defines semantics, not a universal JIT code-generation recipe.

---

# 278. Myth #274 — “Anything Not in ECMAScript Is Undefined”

Too broad.

A behavior can be defined by:

```text
Web Platform
Node.js
another host specification
implementation contract
library
```

---

# 279. Myth #275 — “Implementation-Defined Means Random”

No.

It means an implementation selects among permitted choices according to the relevant specification/contract.

---

# 280. Myth #276 — “Engine-Specific Means Useless”

Engine-specific knowledge can be extremely useful for:

```text
profiling
performance
memory diagnosis
debugging
```

It simply must be labeled correctly.

---

# 281. Myth #277 — “Framework Behavior Is JavaScript Behavior”

Frameworks add their own abstractions and semantics.

---

# 282. Myth #278 — “Framework Abstraction Eliminates Platform Reality”

HTTP, memory, scheduling, security, and network failure still exist underneath.

---

# 283. Myth #279 — “Framework Magic Means It Is Free”

Convenience can hide:

```text
allocation
renders
network calls
subscriptions
serialization
```

---

# 284. Myth #280 — “A Compiler Fixes Architecture”

Compilation transforms code.

It does not automatically fix:

```text
N+1
wrong data model
bad retries
poor authorization
```

---

# 285. Myth #281 — “Minification Hides Sensitive Information”

It does not provide confidentiality.

---

# 286. Myth #282 — “Tree Shaking Means Unused Dependencies Cost Nothing”

Dependencies can still affect:

```text
install
audit
build
maintenance
source ownership
```

---

# 287. Myth #283 — “Dynamic Import Automatically Unloads Modules”

Loading and module lifetime semantics do not imply simple unload behavior.

---

# 288. Myth #284 — “Deleting a Reference Unloads a Module”

Not necessarily.

Module systems can retain module records/caches independently of one local reference.

---

# 289. Myth #285 — “A Heap Snapshot Shows Every Resource”

Not necessarily.

Native resources and external systems can require other diagnostics.

---

# 290. Myth #286 — “JavaScript Heap Is All Process Memory”

A process can use:

```text
JavaScript heap
native memory
buffers
stacks
code
allocator overhead
```

---

# 291. Myth #287 — “GC Tuning Fixes Leaks”

Tuning may change collection behavior.

It does not fix unintended retention.

---

# 292. Myth #288 — “Performance Regression Means Runtime Regression”

Regression sources include:

```text
application change
dependency update
data change
environment
runtime
```

---

# 293. Myth #289 — “Runtime Upgrade Is Only a Performance Change”

It can affect:

```text
security
APIs
V8
memory
module behavior
compatibility
```

---

# 294. Myth #290 — “Lockfile Means Perfect Reproducibility”

It helps.

Full reproducibility can still depend on:

```text
OS
architecture
toolchain
native dependencies
build scripts
external inputs
```

---

# 295. Myth #291 — “Containerization Makes Everything Deterministic”

Containers improve environmental consistency but do not eliminate:

```text
time
randomness
network
external services
build inputs
```

---

# 296. Myth #292 — “Testing Everything Means Production Cannot Fail”

Tests reduce risk.

They cannot exhaust all real-world states.

---

# 297. Myth #293 — “A Production Incident Means Someone Was Careless”

Complex systems have interacting failure modes.

Good incident reviews examine controls and system conditions, not only individual blame.

---

# 298. Myth #294 — “Every Incident Has One Root Cause”

Large incidents often have:

```text
trigger
contributing factors
latent conditions
missing controls
```

---

# 299. Myth #295 — “Fixing the Root Cause Means Changing One Line”

Systemic causes may require:

```text
code
configuration
architecture
observability
process
training
```

---

# 300. Myth #296 — “The Oldest Code Is the Most Dangerous”

Age is not risk.

Poorly understood new code can be more dangerous than stable legacy code.

---

# 301. Myth #297 — “New Code Is Automatically Better”

Age and quality are different dimensions.

---

# 302. Myth #298 — “Principal Engineers Know Everything”

Principal engineering is about:

```text
knowing what matters
knowing what to verify
knowing where to look
making sound decisions under uncertainty
```

---

# 303. The Anti-Myth Method

When someone says:

```text
“JavaScript always...”
```

ask:

```text
Always where?
Always according to whom?
Always in which version?
Always in which host?
Always for which input?
Always for which operation?
```

Then classify:

```text
Guaranteed
Conditional
Implementation-specific
Historical
Convention
False
Unknown
```

---

# 304. Claim Verification Worksheet

```md
## Claim

-

## Authority Claimed

-

## Actual Authority

- ECMAScript
- Web Platform
- Node.js
- Engine
- Library
- Framework
- Application convention

## Scope

-

## Conditions

-

## Counterexample

-

## Reproduction

-

## Corrected Statement

-

## Confidence

- low
- medium
- high
```

---

# 305. Better Explanation Template

Use:

```text
Claim
→ precise correction
→ scope
→ counterexample
→ mechanism
→ authority
→ production implication
```

Example:

```text
Claim:
“async makes code non-blocking.”

Correction:
async/await changes asynchronous control flow; it does not automatically move CPU work to another thread.

Mechanism:
CPU-heavy work still occupies its execution agent.

Production implication:
Move suitable CPU-heavy work to workers/processes or redesign the algorithm.
```

---

# 306. Three Levels of Truth

```text
Level 1 — Language
What ECMAScript guarantees.

Level 2 — Host
What browser/Node/edge APIs and scheduling guarantee.

Level 3 — Implementation
What a particular engine/runtime currently does.

Level 4 — Application
What your architecture assumes and promises.
```

---

# 307. Common Bad Explanations

Avoid:

```text
“JavaScript is weird.”
“V8 does magic.”
“Promises run on another thread.”
“const means immutable.”
“objects are passed by reference.”
“hoisting moves declarations.”
“the event loop is one queue.”
“GC handles memory for you.”
“async means parallel.”
```

Replace each with an actual mechanism.

---

# 308. Track A — Core Theory

Master:

```text
language vs host
values vs bindings
references
scope
hoisting
TDZ
closures
this
objects
prototypes
classes
coercion
equality
numbers
Unicode
time
promises
jobs
event loops
cancellation
memory
GC
engines
modules
security
HTTP
workers
Wasm
compatibility
architecture
```

---

# 309. Track B — Implementation

Build:

```text
myth-lab
scheduler experiments
equality suite
prototype explorer
promise lab
memory lab
module graph analyzer
compatibility matrix
security reproductions
benchmark harness
```

Every experiment should record:

```text
hypothesis
environment
input
prediction
observation
explanation
authority
```

---

# 310. Track C — Interview / Reasoning

Use:

```text
myth
→ corrected statement
→ counterexample
→ mechanism
→ trade-off
→ production implication
```

Principal-level answer additions:

```text
authority
confidence
measurement plan
migration consequence
```

---

# 311. Mastery Gate

```text
Understand
→ Explain
→ Predict
→ Implement
→ Debug
→ Apply
→ Compare
→ Defend
```

Reading alone does not mark mastery.

---

# 312. Debugging Exercise — Bindings

Predict:

```js
function change(user) {
  user.name = "Ada";
  user = { name: "Lin" };
}

const user = { name: "Grace" };

change(user);

console.log(user);
```

Explain mutation versus reassignment.

---

# 313. Debugging Exercise — Scope

Predict:

```js
const jobs = [];

for (var i = 0; i < 3; i++) {
  jobs.push(() => i);
}

console.log(jobs.map(fn => fn()));
```

Then rewrite with lexical bindings.

---

# 314. Debugging Exercise — Equality

Predict:

```js
console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));
console.log(Object.is(-0, 0));
```

Explain the equality relations.

---

# 315. Debugging Exercise — Async

Predict the behavior of:

```js
const a = workA();
const b = workB();

console.log("before");

console.log(await a);
console.log(await b);
```

Trace:

```text
operation start
await
settlement
continuation
```

---

# 316. Debugging Exercise — Promise.race

Given:

```js
await Promise.race([
  slowWork(),
  timeout(1000)
]);
```

Question:

> What additional mechanism is required if `slowWork()` must actually stop?

Answer:

```text
cooperative cancellation
```

---

# 317. Debugging Exercise — Event Loop

Create a minimal browser and Node experiment containing:

```text
sync log
Promise.then
queueMicrotask
setTimeout
I/O where available
```

Do not merely memorize one ordering.

Explain which parts are:

```text
language
host scheduling
runtime-specific
```

---

# 318. Debugging Exercise — Memory

Build a leak using:

```text
listener
timer
closure
unbounded Map
```

Then identify the retaining path.

---

# 319. Debugging Exercise — Performance

Benchmark:

```text
for
for...of
forEach
map
```

with:

```text
small input
large input
cheap callback
expensive callback
```

Report variance and conclusions.

---

# 320. Debugging Exercise — Time

Model:

```text
instant
calendar date
local time
timezone conversion
DST
```

Explain the semantic category before choosing an API.

---

# 321. Debugging Exercise — Security

Create:

```text
untrusted input
→ unsafe HTML sink
```

Then replace it with a safe design.

---

# 322. Interview Drill

Correct these:

```text
JavaScript is single-threaded.
Objects are passed by reference.
const means immutable.
Promises run on threads.
await blocks.
Promise.all means parallel.
Promise.race cancels.
forEach awaits.
GC means no leaks.
Map is always faster.
TypeScript is runtime validation.
CORS is authentication.
HTTPS means secure.
JWT means stateless auth.
Wasm is always faster.
Edge means zero latency.
```

Each response must include:

```text
precise correction
scope
reason
```

---

# 323. Principal Interview Drill

Explain:

> “The event loop is a JavaScript feature.”

Your answer should separate:

```text
ECMAScript job semantics
host scheduling
browser behavior
Node behavior
```

---

# 324. Principal Interview Drill

Explain:

> “V8 deoptimizes when types change.”

Replace the slogan with:

```text
runtime observations
optimization assumptions
implementation-specific behavior
profiling evidence
```

---

# 325. Principal Interview Drill

Explain:

> “Promises are parallel.”

Distinguish:

```text
Promise abstraction
concurrency
parallelism
underlying work
host
```

---

# 326. Principal Interview Drill

Explain:

> “Garbage collection prevents memory leaks.”

Distinguish:

```text
reachability
retention
GC
resource lifecycle
application leaks
```

---

# 327. Principal Interview Drill

Explain:

> “Map is faster than Object.”

Answer with:

```text
semantics
workload
implementation
measurement
```

---

# 328. Mastery Project — Myth Laboratory

Create:

```text
myth-lab/
├── 01-values/
├── 02-scope/
├── 03-closures/
├── 04-this/
├── 05-prototypes/
├── 06-coercion/
├── 07-promises/
├── 08-event-loop/
├── 09-memory/
├── 10-performance/
├── 11-security/
├── 12-modules/
├── 13-browser/
├── 14-node/
├── 15-wasm/
└── 16-compatibility/
```

Each experiment should contain:

```text
experiment
prediction
observed result
trace
explanation
source
```

---

# 329. Mastery Project — Authority Mapper

Build a table:

| Claim | Correct? | Authority | Conditions | Counterexample |
|---|---|---|---|---|
| `const` means immutable | No | ECMAScript | binding vs object | object mutation |
| `await` blocks the thread | No | ECMAScript/host | continuation semantics | other work proceeds |
| Map is always faster | No | implementation/workload | actual workload | Object may win |
| CORS authenticates users | No | Web security/HTTP | origin policy | non-browser client |

Add at least 100 claims.

---

# 330. Mastery Project — Runtime Comparison

Compare:

```text
Browser
Node.js
Edge runtime
```

for:

```text
timers
fetch
streams
workers
modules
filesystem
crypto
process
event scheduling
```

Classify each:

```text
standardized
host-defined
runtime-specific
```

---

# 331. Mastery Project — Benchmark Discipline

For a disputed performance claim:

```text
hypothesis
workload
warmup
measurement
variance
result
limitations
conclusion
```

Reject conclusions that cannot be reproduced.

---

# 332. Mastery Project — Security Claim Validation

For five security myths:

```text
claim
threat model
attacker capability
counterexample
defense
residual risk
```

---

# 333. Mastery Project — Production Myth Review

Take one production architecture and search for statements like:

```text
“This can't happen.”
“This is always fast.”
“The cache makes it safe.”
“The queue guarantees delivery.”
“The runtime handles it.”
```

Challenge each statement with:

```text
assumption
evidence
counterexample
control
```

---

# 334. Spaced Retrieval Schedule

### Day 0

```text
language vs host
value vs binding
mutation vs reassignment
```

### Day 1

```text
hoisting
TDZ
closures
this
```

### Day 3

```text
Promises
await
jobs
event loop
```

### Day 7

```text
GC
memory
WeakMap
performance
```

### Day 14

```text
security
modules
compatibility
```

### Day 30

Correct ten performance myths.

### Day 60

Conduct a runtime comparison.

### Day 90

Conduct a principal-level myth review.

---

# 335. Retrieval Prompts

Without notes:

```text
What is guaranteed by ECMAScript?
What is host-defined?
What is engine-specific?
Why is “objects are passed by reference” imprecise?
Why is “const means immutable” false?
Why does hoisting need a better explanation?
Why do closures not simply copy values?
Why does GC not eliminate memory leaks?
Why does async not equal parallelism?
Why does Promise.race not cancel?
Why does forEach not await?
Why can microtasks affect responsiveness?
Why is this invocation-sensitive?
Why are classes not Java classes?
Why is spread shallow?
Why is JSON not lossless?
Why is Date not a universal temporal model?
Why is CORS not authentication?
Why is TypeScript not runtime validation?
Why is Map vs Object a workload question?
Why does edge not mean zero latency?
Why does Wasm not automatically win?
```

---

# 336. Dependency Graph

```text
Chapter 01
  ↓
Chapter 02
  ↓
Chapter 07
  ↓
Chapter 10–14
  ↓
Chapter 15–20
  ↓
Chapter 22–28
  ↓
Chapter 31–40
  ↓
Chapter 41–44
  ↓
Chapter 45–48
  ↓
Chapter 49–57
  ↓
Chapter 58–70
  ↓
Chapter 78–89
  ↓
Chapter 90–97
  ↓
Chapter 98
  ↓
Chapter 99
```

---

# 337. Concept Connections

## Depends On

```text
language semantics
runtime models
async execution
memory
browser/Node
security
performance
modules
compatibility
production architecture
```

## Builds Toward

```text
Chapter 100 — Cost Model / Trade-offs
Chapter 101 — Real-World Production Scenarios
```

## Related

```text
anti-patterns
failure modes
debugging
code review
technical debt
decision quality
risk management
```

## Revisited

```text
values
scope
closures
this
objects
prototypes
promises
event loop
GC
V8
modules
HTTP
security
Wasm
edge
compatibility
```

## Why This Chapter Matters

Without myth correction, advanced engineering becomes:

```text
memorized slogans
```

With myth correction, the engineer reasons from:

```text
mechanism
authority
evidence
context
trade-off
```

---

# 338. Principal Decision Framework

For any JavaScript claim:

```text
1. State the claim.
2. Narrow its scope.
3. Identify the authority.
4. Construct a counterexample.
5. Reproduce it.
6. Separate guarantee from implementation detail.
7. Identify production impact.
8. Decide whether the simplification is useful.
```

---

# 339. Canonical References and Source Discipline

Primary references:

1. ECMAScript Language Specification  
   https://tc39.es/ecma262/

2. MDN JavaScript Reference  
   https://developer.mozilla.org/en-US/docs/Web/JavaScript

3. MDN Promise Reference  
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

4. MDN Web APIs  
   https://developer.mozilla.org/en-US/docs/Web/API

5. Node.js Documentation  
   https://nodejs.org/docs/

6. TC39 Proposals  
   https://github.com/tc39/proposals

7. WebAssembly Specifications  
   https://webassembly.org/specs/

8. OWASP  
   https://owasp.org/

Source discipline:

```text
ECMAScript semantics
→ ECMA-262

Browser behavior
→ Web Platform specifications + browser documentation

Node behavior
→ Node.js documentation

Engine behavior
→ engine documentation/source/profiling

Security
→ authoritative security guidance + threat model

Performance
→ benchmarks + profiles + production telemetry
```

Never convert an implementation observation into a language guarantee.

---

# 340. Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Language vs host [ ]
- Specification vs implementation [ ]
- Value vs binding [ ]
- Mutation vs reassignment [ ]
- Scope / hoisting / TDZ [ ]
- Closures [ ]
- this [ ]
- Prototypes [ ]
- Classes [ ]
- Coercion/equality [ ]
- Async / Promise semantics [ ]
- Event-loop reasoning [ ]
- Memory / GC [ ]
- Performance myths [ ]
- Security myths [ ]
- Module myths [ ]
- Compatibility myths [ ]
- Counterexample construction [ ]
- Source verification [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# 341. Completion Snapshot

```text
Part XIX — Judgment

Chapter 99 — Myths and Misconceptions
[ ] Not Started

Track A — Core Theory
[ ] Language vs host
[ ] Specification vs implementation
[ ] Values / bindings
[ ] Scope / hoisting / TDZ
[ ] Closures
[ ] this
[ ] Objects / prototypes
[ ] Classes
[ ] Coercion / equality
[ ] Numbers
[ ] Strings / Unicode
[ ] Date / time
[ ] Promises
[ ] Async / await
[ ] Event loop
[ ] Cancellation
[ ] Memory / GC
[ ] Engines / V8
[ ] Modules
[ ] Browser APIs
[ ] Node APIs
[ ] Security
[ ] HTTP
[ ] Workers
[ ] Wasm
[ ] Edge / serverless
[ ] Compatibility
[ ] Testing
[ ] Observability
[ ] Architecture

Track B — Implementation
[ ] Myth laboratory
[ ] Authority mapper
[ ] Runtime comparison
[ ] Benchmark discipline
[ ] Security validation
[ ] Production myth review

Track C — Interview / Reasoning
[ ] Correct slogans
[ ] State assumptions
[ ] Produce counterexamples
[ ] Identify authority
[ ] Explain mechanisms
[ ] Defend trade-offs
[ ] Separate guarantees from observations
[ ] Design verification experiments

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

Do not mark this chapter mastered because you remember a list.

You are ready to move forward when you can independently:

1. Correct a common JavaScript misconception.
2. Explain why the misconception is tempting.
3. Produce a counterexample.
4. State the precise rule.
5. Identify its authority.
6. Distinguish language from host.
7. Distinguish host from engine.
8. Distinguish implementation from convention.
9. Explain async without thread folklore.
10. Explain memory without GC folklore.
11. Explain performance without benchmark folklore.
12. Explain security without browser-magic assumptions.
13. Explain module behavior without syntax-only reasoning.
14. Explain compatibility without proposal-stage assumptions.
15. Build a minimal reproduction.
16. Locate the primary source.
17. State uncertainty when evidence is incomplete.
18. Apply the corrected model to production design.
19. Challenge a claim respectfully in review.
20. Defend a technical decision with evidence.

---

# Final Mental Model

```text
Claim
 ↓
Scope
 ↓
Authority
 ↓
Mechanism
 ↓
Counterexample
 ↓
Evidence
 ↓
Production implication
 ↓
Decision
```

> **Do not memorize myths. Replace them with precise models you can explain, predict, test, and defend.**