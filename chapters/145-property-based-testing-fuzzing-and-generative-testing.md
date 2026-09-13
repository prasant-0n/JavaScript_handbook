# Chapter 145 — Property-Based Testing, Fuzzing & Generative Testing

> **JavaScript Mastery — Part XXVI: Advanced Verification, Reliability & Failure Discovery**
>
> **Mission:** Learn to test JavaScript systems by describing invariants, generating large and adversarial input spaces, finding minimal counterexamples, reproducing failures deterministically, and integrating generative testing into Node.js unit, integration, parser, protocol, security, and runtime workflows.
>
> **Role perspective:** Principal JavaScript Engineer · Test Infrastructure Engineer · Reliability Engineer · Security Engineer · Fuzzing Engineer · Runtime Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Core principle:** **Example-based testing asks “does this example work?” Property-based testing asks “what must remain true across a space of inputs?” Fuzzing adds aggressive input exploration and failure discovery. The hard engineering problem is not generating random data; it is generating meaningful data, shrinking failures, preserving reproducibility, and turning counterexamples into durable specifications.**

---

# 1. Learning Objectives

```text
[ ] explain property-based testing
[ ] explain generative testing
[ ] explain fuzzing
[ ] distinguish example-based tests from properties
[ ] distinguish property-based testing from mutation testing
[ ] distinguish property-based testing from random testing
[ ] distinguish fuzzing from random data generation
[ ] distinguish black-box fuzzing from white-box fuzzing
[ ] distinguish grammar-based fuzzing from byte fuzzing
[ ] distinguish coverage-guided fuzzing from naive random fuzzing
[ ] identify useful properties
[ ] identify weak properties
[ ] write invariants
[ ] write metamorphic properties
[ ] write round-trip properties
[ ] write idempotence properties
[ ] write monotonicity properties
[ ] write conservation properties
[ ] write commutativity properties
[ ] write associativity properties
[ ] write normalization properties
[ ] write serialization properties
[ ] write parser properties
[ ] write security properties
[ ] generate primitive values
[ ] generate structured objects
[ ] generate arrays
[ ] generate maps
[ ] generate sets
[ ] generate strings
[ ] generate Unicode strings
[ ] generate numbers safely
[ ] generate edge numeric values
[ ] generate dates/times
[ ] generate URLs
[ ] generate HTTP requests
[ ] generate JSON
[ ] generate binary data
[ ] generate recursive structures
[ ] generate state-machine commands
[ ] generate sequences
[ ] generate dependent values
[ ] compose generators
[ ] constrain generators
[ ] bias generators
[ ] avoid useless generators
[ ] understand distribution
[ ] measure generator quality
[ ] understand shrinking
[ ] implement shrinking
[ ] preserve failure during shrinking
[ ] minimize counterexamples
[ ] distinguish shrinker bugs from product bugs
[ ] replay counterexamples
[ ] persist seeds
[ ] persist generated cases
[ ] persist serialized inputs
[ ] replay failures across CI
[ ] control randomness
[ ] use deterministic PRNGs
[ ] record runtime/version metadata
[ ] understand randomized tests in Node.js
[ ] use node:test for generated cases
[ ] integrate generators with node:test
[ ] integrate fuzzing with child processes
[ ] isolate dangerous fuzz targets
[ ] fuzz parsers
[ ] fuzz serializers
[ ] fuzz validators
[ ] fuzz decoders
[ ] fuzz protocol handlers
[ ] fuzz HTTP handlers
[ ] fuzz CLI parsers
[ ] fuzz config loaders
[ ] fuzz security boundaries
[ ] fuzz native addons
[ ] fuzz WebAssembly boundaries
[ ] fuzz stream processing
[ ] fuzz cancellation
[ ] fuzz async state machines
[ ] test concurrency properties
[ ] test race-resistant invariants
[ ] detect crashes
[ ] detect hangs
[ ] detect timeouts
[ ] detect assertion failures
[ ] detect invariant violations
[ ] detect memory growth
[ ] detect resource leaks
[ ] understand timeouts as fuzz findings
[ ] understand malformed input
[ ] understand adversarial structure
[ ] understand pathological depth
[ ] understand pathological width
[ ] understand quadratic behavior
[ ] understand catastrophic backtracking
[ ] understand parser differentials
[ ] understand serialization ambiguities
[ ] understand Unicode edge cases
[ ] understand numeric boundary cases
[ ] understand floating-point edge cases
[ ] understand NaN
[ ] understand Infinity
[ ] understand negative zero
[ ] understand bigint boundaries
[ ] understand integer overflow at host boundaries
[ ] understand surrogate pairs
[ ] understand combining marks
[ ] understand normalization
[ ] understand bidirectional text
[ ] understand confusable identifiers
[ ] understand prototype pollution inputs
[ ] understand prototype-chain edge cases
[ ] understand object key ordering assumptions
[ ] understand sparse arrays
[ ] understand holes
[ ] understand accessors
[ ] understand proxies
[ ] understand symbols
[ ] understand getters with side effects
[ ] understand hostile iterables
[ ] understand thenables
[ ] understand promise rejection
[ ] understand async generator failures
[ ] understand abort signals
[ ] understand resource ownership
[ ] build generators from specifications
[ ] build shrinkers
[ ] build a property runner
[ ] build a failure corpus
[ ] build a replay system
[ ] build a stateful model-based fuzzer
[ ] build a differential fuzzer
[ ] build a parser fuzzer
[ ] build a protocol fuzzer
[ ] build a CI fuzzing stage
[ ] build nightly fuzzing
[ ] build regression corpus
[ ] triage fuzz failures
[ ] classify unique failures
[ ] deduplicate crashes
[ ] minimize repros
[ ] secure fuzz infrastructure
[ ] control CPU budgets
[ ] control memory budgets
[ ] control process counts
[ ] control disk growth
[ ] design fuzz campaigns
[ ] choose stopping criteria
[ ] choose seeds
[ ] choose corpus
[ ] choose mutation strategy
[ ] choose generation strategy
[ ] choose coverage strategy
[ ] choose isolation
[ ] choose CI cadence
[ ] defend test strategy in interviews
[ ] compare property testing and fuzzing
[ ] compare fuzzing approaches
[ ] reason about production trade-offs


# 2. Prerequisites

You should already understand:

```text
Chapter 29 — Testing Fundamentals
Chapter 31 — Async
Chapter 33 — Event Loop
Chapter 37 — Iterators / Generators
Chapter 42 — Data Structures
Chapter 47 — Regular Expressions
Chapter 52 — Workers / Concurrency
Chapter 63 — Diagnostics
Chapter 70 — Production Debugging
Chapter 71 — Security
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging Methodology
Chapter 101 — Production Scenarios
Chapter 125 — Promise Internals
Chapter 129 — Regular Expressions
Chapter 131 — URI/URL Semantics
Chapter 140 — Node Networking
Chapter 141 — Node Diagnostics
Chapter 142 — Permission Model
Chapter 143 — Native Addons
Chapter 144 — Node Test Runner, Mocking & Test Isolation
```

You should know:

```text
assertions
test isolation
mocking
timers
randomness
serialization
parsing
HTTP
binary data
workers
child processes
resource cleanup.
```

---

# 3. What Is Property-Based Testing?

Property-based testing verifies:

```text
general truths
```

over:

```text
many generated inputs.
```

Example:

```text
reverse(reverse(array))
=
array
```

Instead of writing:

```text
[1, 2, 3]
[-1, 0, 5]
[]
```

individually.

---

# 4. What Is a Property?

A property is:

```text
condition that should remain true
```

for:

```text
all valid inputs
```

or:

```text
a defined input domain.
```

Formally:

```text
∀ x ∈ Domain:
    P(x) = true
```

---

# 5. What Is Generative Testing?

Generative testing creates:

```text
inputs algorithmically
```

instead of:

```text
hand-authoring every case.
```

The generator is part of the test architecture.

---

# 6. What Is Fuzzing?

Fuzzing repeatedly supplies:

```text
unexpected
malformed
boundary
random
mutated
adversarial
```

inputs to a target and looks for:

```text
crash
hang
assertion failure
invariant violation
memory/resource issue
security weakness.
```

---

# 7. Property Testing vs Fuzzing

```text
Property testing
→ known invariant + generated inputs

Fuzzing
→ broad input exploration + failure detection

Generative testing
→ algorithmic input generation

Coverage-guided fuzzing
→ input exploration guided by execution feedback.
```

They overlap:

```text
property-based fuzzing
```

but are not identical.

---

# 8. Property Testing vs Example Testing

Example:

```js
assert.equal(normalize(" A "), "A");
```

Property:

```text
normalize(normalize(x))
=
normalize(x)
```

The property can expose:

```text
many unknown counterexamples.
```

---

# 9. Property Testing vs Mutation Testing

Mutation testing asks:

```text
Can tests detect intentional code mutations?
```

Property-based testing asks:

```text
Does a general behavioral invariant hold over generated inputs?
```

Mutation testing evaluates:

```text
test strength.
```

Property testing creates:

```text
broader inputs.
```

---

# 10. Strong Property Characteristics

A strong property is:

```text
observable
general
domain-relevant
hard to satisfy accidentally
cheap enough to execute
stable under valid refactoring.
```

---

# 11. Weak Property Example

Weak:

```js
assert.equal(typeof result, "object");
```

This may pass when:

```text
behavior is completely wrong.
```

---

# 12. Stronger Property

If the contract says:

```text
decode(encode(value)) = value
```

test:

```js
assert.deepEqual(
  decode(encode(value)),
  value
);
```

for:

```text
many generated values.
```

---

# 13. Core Property Families

Learn:

```text
invariant
round-trip
idempotence
commutativity
associativity
metamorphic relation
conservation
monotonicity
oracle comparison
differential equivalence
```

---

# 14. Invariant

An invariant remains true:

```text
before
during
after
```

or:

```text
after every valid operation.
```

Example:

```text
queue size
>=
0
```

---

# 15. Round-Trip Property

Serialization:

```text
deserialize(serialize(x)) = x
```

Parsing:

```text
parse(print(ast)) = ast
```

Encoding:

```text
decode(encode(bytes)) = bytes
```

---

# 16. Idempotence

For normalization:

```text
normalize(normalize(x))
=
normalize(x)
```

For canonical sorting:

```text
sort(sort(x))
=
sort(x)
```

---

# 17. Commutativity

When an operation should be commutative:

```text
f(a, b) = f(b, a)
```

Do not assert this for operations whose contract is:

```text
order-sensitive.
```

---

# 18. Associativity

For operations such as abstract addition:

```text
f(f(a,b),c)
=
f(a,f(b,c))
```

Be careful with:

```text
floating-point arithmetic
```

because mathematical associativity may not survive:

```text
finite-precision representation.
```

---

# 19. Conservation Property

For a parser transformation:

```text
number of tokens before
=
number of expected tokens after.
```

For accounting:

```text
sum(inputs)
=
sum(outputs)
+
fees.
```

---

# 20. Monotonicity

Example:

```text
adding valid permissions
```

should not:

```text
remove previously granted access
```

when the contract is monotonic.

---

# 21. Metamorphic Testing

Sometimes there is no easy oracle for:

```text
f(x)
```

but there is a relation:

```text
f(transform(x))
=
transformResult(f(x))
```

This is a:

```text
metamorphic property.
```

---

# 22. Metamorphic Example

Image transforms:

```text
resize(resize(image, 2x), 2x)
```

may be compared through:

```text
known invariants
```

rather than:

```text
exact pixel equality.
```

For JavaScript systems, common metamorphic relations include:

```text
sort
normalize
serialize
filter
map
encode/decode.
```

---

# 23. Differential Testing

Run:

```text
implementation A
```

and:

```text
implementation B
```

on the same generated input.

Compare:

```text
output
error
normalized behavior.
```

Examples:

```text
parser v1 vs v2
native vs JS implementation
reference algorithm vs optimized algorithm.
```

---

# 24. Differential Testing Caveat

Two implementations can share:

```text
same bug.
```

Therefore:

```text
independent implementations
```

are stronger than:

```text
two copies of the same algorithm.
```

---

# 25. Oracle Problem

Fuzzing requires deciding:

```text
what counts as failure?
```

A crash is obvious.

But:

```text
wrong output
```

requires:

```text
oracle
property
metamorphic relation
reference implementation.
```

---

# 26. Oracle Hierarchy

Prefer:

```text
exact specification
→ reference implementation
→ algebraic property
→ metamorphic property
→ invariant
→ heuristic signal.
```

The lower you go:

```text
confidence may decrease.
```

---

# 27. Generator

A generator maps:

```text
random state
```

to:

```text
structured input.
```

Conceptually:

```text
Generator<T>
```

produces:

```text
T
```

plus enough information to:

```text
reproduce/shrink.
```

---

# 28. Primitive Generators

Common generators:

```text
boolean
integer
bigint
number
string
symbol-like domain
null
undefined
```

---

# 29. Number Generator

Do not generate only:

```text
0..100
```

Include:

```text
0
-0
1
-1
MAX_SAFE_INTEGER
MIN_SAFE_INTEGER
Infinity
-Infinity
NaN
fractions
very small values
very large values.
```

---

# 30. Safe Integer Domain

JavaScript's exact integer region for `Number` is bounded.

Generate:

```text
Number.MIN_SAFE_INTEGER
...
Number.MAX_SAFE_INTEGER
```

plus:

```text
outside-range values
```

when testing:

```text
serialization
validation
precision behavior.
```

---

# 31. BigInt Generator

Generate:

```text
0n
1n
-1n
large positive
large negative
boundary magnitudes.
```

Test transitions between:

```text
Number
BigInt
string
JSON-compatible representations.
```

---

# 32. Negative Zero

`-0` can behave differently from:

```text
0
```

under operations such as:

```js
Object.is(-0, 0); // false
```

Include:

```text
-0
```

in numeric generator suites.

---

# 33. NaN

`NaN` is not equal to itself with:

```js
NaN === NaN
```

.

A useful property may need:

```text
Number.isNaN
Object.is
```

depending on the contract.

---

# 34. Floating-Point Generation

Generate:

```text
normal finite
subnormal-ish values
very large
very small
fractions
special values.
```

Avoid assuming:

```text
mathematical exactness
```

where:

```text
IEEE-754 rounding
```

applies.

---

# 35. String Generator

Generate:

```text
empty
ASCII
whitespace
newlines
tabs
quotes
slashes
backslashes
control characters
Unicode.
```

---

# 36. Unicode Generator

Include:

```text
BMP
surrogate pairs
isolated surrogates
combining marks
emoji sequences
variation selectors
zero-width joiners
bidirectional controls
noncharacters where relevant.
```

---

# 37. String Length Distribution

Uniform strings are often useless.

Bias toward:

```text
0
1
2
boundary - 1
boundary
boundary + 1
large
```

because bugs often appear:

```text
near boundaries.
```

---

# 38. Structural Strings

For parsers, generate:

```text
valid token
invalid token
almost valid token
nested token
ambiguous token
escape-heavy token.
```

---

# 39. Array Generator

Include:

```text
empty
singleton
duplicates
sorted
reverse sorted
sparse
large
mixed values.
```

---

# 40. Sparse Arrays

Distinguish:

```js
[]
```

from:

```js
new Array(3)
```

and:

```js
[undefined, undefined, undefined]
```

.

These can behave differently during:

```text
iteration
map
forEach
Object.keys
serialization.
```

---

# 41. Object Generator

Generate:

```text
empty objects
deep objects
wide objects
null-prototype objects
prototype-bearing objects
duplicate semantic keys where format allows
symbols
getters
accessors
```

---

# 42. Hostile Object Keys

Include:

```text
__proto__
constructor
prototype
toString
valueOf
```

when testing:

```text
object merge
configuration
validation
serialization.
```

---

# 43. Prototype Pollution Testing

Generate objects containing:

```text
__proto__
constructor.prototype
```

and verify:

```text
no unexpected mutation of global/shared prototypes.
```

Security tests should use:

```text
isolated processes
```

when necessary.

---

# 44. Map Generator

Generate:

```text
empty
single key
duplicate replacement
object keys
NaN keys
-0 / 0 keys
large maps.
```

---

# 45. Set Generator

Generate:

```text
empty
duplicates
objects
NaN
-0 / 0
large cardinality.
```

---

# 46. Recursive Generators

Recursive structures need:

```text
depth limits
size budgets
termination probability.
```

Otherwise:

```text
generator
```

can itself become:

```text
infinite.
```

---

# 47. Depth Budget

Define:

```js
depth = 0
```

and stop expanding after:

```text
MAX_DEPTH.
```

This protects:

```text
stack
memory
runtime.
```

---

# 48. Size Budget

A generator should consume:

```text
finite generation budget.
```

A huge nested object is not automatically:

```text
better fuzzing.
```

---

# 49. Generator Distribution

Suppose:

```text
90% trivial
10% complex
```

Your generator may miss:

```text
deep structural bugs.
```

Measure:

```text
depth distribution
length distribution
branching distribution
special-case frequency.
```

---

# 50. Boundary Bias

Good generator:

```text
50% common
25% boundary
15% malformed
10% adversarial
```

The exact values depend on:

```text
risk domain.
```

---

# 51. Constraint Composition

A generator for email-like strings:

```text
local-part
@
domain
```

should generate:

```text
valid variants
boundary variants
near-valid variants.
```

Do not generate:

```text
random bytes
```

and call it:

```text
email fuzzing.
```

---

# 52. Grammar-Based Generation

For parsers:

```text
Grammar
 ↓
AST
 ↓
serialized input
```

This produces:

```text
syntax-aware inputs.
```

Much more efficient than:

```text
fully random bytes
```

for many structured languages.

---

# 53. Grammar Violations

Also generate:

```text
missing token
extra token
wrong token
deep nesting
unclosed structure
invalid escape
duplicate field
```

to explore:

```text
error paths.
```

---

# 54. AST Generation

Generate:

```text
Program
Statement
Expression
Literal
Identifier
Call
Member
Binary
Unary
Conditional
```

then:

```text
print(AST).
```

---

# 55. AST Property

If printer/parser contracts support:

```text
parse(print(ast))
```

then check:

```text
semantic equivalence.
```

Not necessarily:

```text
object identity.
```

---

# 56. Serialization Property

For supported values:

```text
decode(encode(x))
=
normalize(x)
```

when encoding is not perfectly lossless.

Always define:

```text
normalization boundary.
```

---

# 57. JSON Property

Generate values restricted to:

```text
JSON domain.
```

Then test:

```text
parse(stringify(x))
```

with a precisely defined equivalence relation.

Do not include:

```text
undefined
function
symbol
BigInt
cyclic object
```

unless the expected result is explicitly:

```text
rejection/transformation.
```

---

# 58. URL Generation

Generate:

```text
scheme
username
password
host
port
path
query
fragment
percent encoding
Unicode.
```

Also generate:

```text
invalid URL variants.
```

---

# 59. URL Properties

Possible properties:

```text
parse(serialize(x))
```

or:

```text
serialize(parse(s))
```

where canonicalization is defined.

Test:

```text
normalization
escaping
port handling
empty components.
```

---

# 60. HTTP Request Generator

Generate:

```text
method
URL
headers
body
content type
query
cookies
authorization forms
```

with:

```text
valid
boundary
malformed
oversized
duplicate.
```

---

# 61. Header Generation

Include:

```text
empty
long
duplicate names
mixed casing
whitespace
invalid bytes
separator characters
Unicode where prohibited.
```

Verify:

```text
expected reject/normalize behavior.
```

---

# 62. Protocol Fuzzing

For a protocol:

```text
message
→ parser
→ state transition
→ response.
```

Generate:

```text
valid sequence
truncated sequence
reordered sequence
duplicate message
unexpected message
oversized payload
invalid state transition.
```

---

# 63. Stateful Model-Based Testing

A stateful test generates:

```text
commands
```

rather than:

```text
isolated values.
```

Example:

```text
connect
send
receive
close
```

---

# 64. Model vs System

Maintain:

```text
reference model state
```

and:

```text
real system state.
```

After each command:

```text
compare observable state.
```

---

# 65. Stateful Counterexample

A failure may require:

```text
connect
send A
send B
close
send C
```

Shrinking should find:

```text
smallest command sequence
```

that reproduces:

```text
failure.
```

---

# 66. Stateful Fuzzing

Useful for:

```text
queues
databases
protocols
sessions
caches
state machines
distributed coordination
workers.
```

---

# 67. Async Generators

Generated operations can be:

```text
async
```

but you must preserve:

```text
deterministic sequencing
```

when the property requires:

```text
ordered steps.
```

---

# 68. Concurrent Generators

A concurrency generator may create:

```text
operation A
operation B
```

and explore:

```text
interleavings.
```

This is powerful but:

```text
combinatorially expensive.
```

---

# 69. Interleaving Budget

Do not enumerate every possible ordering blindly.

Use:

```text
bounded depth
partial-order reduction
random schedules
targeted races
```

where appropriate.

---

# 70. Shrinking

When input:

```text
huge and failing
```

shrinking finds:

```text
smaller failing input.
```

Example:

```text
1000-element array
```

becomes:

```text
[0, 0, -1]
```

because the smaller case still fails.

---

# 71. Why Shrinking Matters

Without shrinking:

```text
failure = 20 MB input
```

With shrinking:

```text
failure = 17 bytes
```

The second is:

```text
understandable
replayable
debuggable
maintainable.
```

---

# 72. Shrinker Contract

A valid shrink candidate should:

```text
remain in the generator's logical domain
```

when required, and:

```text
preserve failure
```

before accepting it as a counterexample.

---

# 73. Array Shrinking

Try:

```text
remove half
remove chunks
remove one element
replace with simpler elements.
```

---

# 74. String Shrinking

Try:

```text
empty
shorter prefix
shorter suffix
remove chunk
replace with ASCII
replace with simple characters.
```

Do not destroy:

```text
essential structural trigger
```

too early.

---

# 75. Number Shrinking

Try:

```text
0
1
-1
smaller magnitude
boundary-near values.
```

For a failure at:

```text
999999
```

try:

```text
0
1
500000
```

and:

```text
boundary.
```

---

# 76. Tree Shrinking

For recursive data:

```text
remove subtree
replace subtree with leaf
reduce depth
reduce branching.
```

---

# 77. Sequence Shrinking

For stateful command sequences:

```text
remove command
remove contiguous block
simplify command args
```

until:

```text
failure remains.
```

---

# 78. Shrinker Failure

A broken shrinker can:

```text
loop forever
consume CPU
produce invalid input
hide the minimal cause.
```

Test shrinkers separately.

---

# 79. Property Runner Algorithm

Conceptually:

```text
for seed in seeds:
    input = generate(seed)

    try:
        assert property(input)
    catch error:
        minimal = shrink(input)
        save(minimal)
        fail
```

---

# 80. Reproduction Record

Persist:

```text
seed
generator version
serialized input
shrink path if useful
property name
target version
Node version
OS
architecture
configuration.
```

---

# 81. Seed Alone Is Not Always Enough

A seed depends on:

```text
generator algorithm
PRNG implementation
generator version
execution order.
```

So store:

```text
seed + final minimized input
```

whenever possible.

---

# 82. Failure Corpus

Maintain:

```text
known bad inputs
```

as a regression corpus.

Every important fuzz bug should become:

```text
permanent regression case.
```

---

# 83. Corpus Growth

A corpus can contain:

```text
minimal crashes
historical bugs
security cases
boundary cases
representative valid inputs.
```

Do not let it grow without:

```text
deduplication
retention policy.
```

---

# 84. Corpus Deduplication

Deduplicate by:

```text
input hash
normalized AST
coverage signature
failure fingerprint.
```

---

# 85. Coverage-Guided Fuzzing

The fuzzer observes:

```text
execution feedback
```

and keeps inputs that discover:

```text
new code paths
new edges
new states.
```

This can explore deeper behavior than:

```text
uniform random generation.
```

---

# 86. Coverage Is a Signal

Coverage is:

```text
guidance
```

not:

```text
proof.
```

A high-coverage fuzzer can still miss:

```text
semantic bugs.
```

---

# 87. Seed Corpus Strategy

Start with:

```text
valid minimal examples
```

then add:

```text
boundaries
invalid cases
historical bugs.
```

For parsers:

```text
small valid syntax
```

helps mutations remain close to:

```text
reachable grammar regions.
```

---

# 88. Mutation-Based Fuzzing

Take:

```text
existing input
```

and mutate:

```text
bits
bytes
tokens
AST nodes
fields
ordering.
```

---

# 89. Structure-Aware Mutation

For JSON:

```text
change field value
delete field
duplicate field
change array length.
```

For AST:

```text
replace expression
delete statement
change operator.
```

This preserves:

```text
useful structure.
```

---

# 90. Byte-Level Mutation

Useful for:

```text
binary parsers
compression formats
image/audio
wire protocols
native APIs.
```

Mutations:

```text
bit flip
byte flip
insert
delete
duplicate
splice
truncate.
```

---

# 91. Black-Box Fuzzing

Only observe:

```text
input
output
exit status
```

Useful when:

```text
implementation details unavailable.
```

---

# 92. White-Box Fuzzing

Use:

```text
internal coverage
execution state
program instrumentation.
```

Useful when:

```text
deep target exploration
```

matters.

---

# 93. Gray-Box Fuzzing

Use:

```text
limited execution feedback
```

without requiring:

```text
full semantic visibility.
```

Many practical fuzzers fit:

```text
gray-box.
```

---

# 94. Crash Fuzzing

Detect:

```text
SIGSEGV
SIGABRT
nonzero exit
uncaught exception
fatal error.
```

Native fuzzing often benefits from:

```text
child-process isolation.
```

---

# 95. Hang Fuzzing

A hang can indicate:

```text
infinite loop
deadlock
pathological backtracking
unbounded retry
resource starvation.
```

Every fuzz target needs:

```text
timeout.
```

---

# 96. Timeout as Failure

Define:

```text
target budget
```

such as:

```text
100ms
500ms
1s
```

depending on target.

Then classify:

```text
timeout
```

as:

```text
finding.
```

---

# 97. Performance Fuzzing

Generate:

```text
increasing input sizes
```

and measure:

```text
runtime
memory
allocations where measurable.
```

Look for:

```text
O(n²)
O(n³)
catastrophic growth.
```

---

# 98. Regex Fuzzing

Generate:

```text
patterns
inputs
nested quantifiers
alternation
repetition
long prefixes.
```

Measure:

```text
execution time.
```

This can expose:

```text
catastrophic backtracking.
```

---

# 99. Parser Complexity Fuzzing

Generate:

```text
deep nesting
long sequences
many alternatives
large token counts.
```

Measure:

```text
time
stack
memory.
```

---

# 100. Prototype Pollution Fuzzing

Generate nested objects containing:

```text
__proto__
constructor
prototype
```

along different merge paths.

Verify:

```text
Object.prototype
```

and other shared prototypes remain:

```text
unchanged.
```

---

# 101. Unicode Security Fuzzing

Generate:

```text
visually confusable text
mixed normalization
bidi controls
zero-width characters
combining marks
surrogates.
```

Test:

```text
validation
identifier handling
logging
authorization
canonicalization.
```

---

# 102. Path Traversal Fuzzing

Generate path components:

```text
..
.
...
encoded ..
double-encoded
mixed separators
absolute paths
UNC-like forms
NUL where relevant.
```

Verify:

```text
root confinement.
```

---

# 103. Command Injection Fuzzing

Generate:

```text
quotes
shell metacharacters
newlines
redirections
substitutions
unexpected whitespace.
```

Use only:

```text
isolated test targets.
```

Do not execute generated payloads against:

```text
production
```

or:

```text
untrusted external systems.
```

---

# 104. HTTP Parser Fuzzing

Generate:

```text
malformed request line
headers
duplicate headers
invalid content lengths
chunked framing errors
unexpected bytes
oversized inputs.
```

Verify:

```text
reject safely
close correctly
do not hang
do not desynchronize parser state.
```

---

# 105. JSON Parser Fuzzing

Generate:

```text
deep nesting
large numbers
duplicate keys
invalid escapes
unpaired surrogates
huge strings
truncated documents.
```

Compare:

```text
parse result
or
expected rejection.
```

---

# 106. URL Parser Differential Fuzzing

Compare:

```text
reference URL model
```

to:

```text
implementation.
```

Focus on:

```text
percent encoding
host parsing
credentials
ports
Unicode
dot segments.
```

---

# 107. Date/Time Fuzzing

Generate:

```text
timestamps
offsets
DST transitions
invalid dates
leap-related boundaries
timezone identifiers
```

and verify:

```text
normalization
round trips
expected rejection.
```

---

# 108. Promise/Thenable Fuzzing

Generate:

```text
normal values
thenables
throwing then getters
double resolve
resolve then reject
reject then resolve
never-settling thenables.
```

Test:

```text
assimilation
settlement
error propagation.
```

---

# 109. Iterator Fuzzing

Generate hostile iterables whose:

```text
next()
throws
returns malformed results
return()
throws
Symbol.iterator
throws.
```

This exposes:

```text
protocol error handling.
```

---

# 110. Async Iterator Fuzzing

Generate:

```text
next() resolves
next() rejects
return() resolves
return() rejects
malformed iterator results
delayed results.
```

Test:

```text
cleanup
cancellation
error propagation.
```

---

# 111. Proxy Fuzzing

Generate handlers whose:

```text
get
set
has
ownKeys
getOwnPropertyDescriptor
```

behaviors are:

```text
valid
throwing
inconsistent
side-effectful.
```

Test:

```text
invariant enforcement.
```

---

# 112. Accessor Fuzzing

Generate objects with:

```text
getters
setters
non-configurable properties
non-writable properties
```

and test:

```text
descriptor semantics
```

across:

```text
Object
Reflect
spread
destructuring
assignment.
```

---

# 113. Symbol Fuzzing

Include:

```text
well-known symbols
unique symbols
symbol-keyed properties
```

in generic object operations.

Verify:

```text
enumeration
serialization
property access
```

match the intended contract.

---

# 114. Sparse Array Fuzzing

Generate:

```text
holes
undefined values
extra properties
length changes
```

and compare:

```text
map
forEach
reduce
for...of
Object.keys
JSON.
```

---

# 115. RegExp Fuzzing

Generate:

```text
patterns
flags
Unicode escapes
lookarounds
backreferences
quantifiers
nested groups
```

and test:

```text
compile
match
replace
split
exec
test
```

for:

```text
correctness
termination
performance.
```

See:

```text
Chapter 129.
```

---

# 116. Buffer Fuzzing

Generate:

```text
empty buffers
single byte
all zero
all ff
random bytes
structured headers
truncated payloads.
```

Test:

```text
decode
offset
length
copy
slice
binary protocol.
```

---

# 117. TypedArray Fuzzing

Generate:

```text
Int8Array
Uint8Array
Int32Array
Float32Array
BigInt64Array
```

with:

```text
offset
length
misalignment
aliasing
```

where applicable.

---

# 118. ArrayBuffer Boundary Testing

Generate:

```text
0 bytes
1 byte
boundary size
oversized
detached/transfer-related states where applicable.
```

Verify:

```text
bounds checks
error behavior
data integrity.
```

---

# 119. Stream Fuzzing

Generate:

```text
chunk boundaries
empty chunks
small chunks
large chunks
early close
error
backpressure
abort.
```

A critical stream property is:

```text
chunking should not change semantic output
```

for transformations where the contract says so.

---

# 120. Chunking Metamorphic Property

Compare:

```text
process([A+B+C])
```

with:

```text
process([A], [B], [C])
```

when streaming semantics promise equivalence.

This catches:

```text
chunk-boundary bugs.
```

---

# 121. Compression Fuzzing

Generate:

```text
random bytes
repeated bytes
high-entropy bytes
truncated compressed data
invalid headers.
```

Check:

```text
decompress(compress(x))
```

and:

```text
safe rejection.
```

---

# 122. File Format Fuzzing

Generate:

```text
header
length
flags
payload
checksum
```

and mutate:

```text
length fields
reserved bits
truncation
ordering.
```

The parser must not:

```text
trust attacker-controlled lengths blindly.
```

---

# 123. Native Addon Fuzzing

Wrap native calls behind:

```text
child-process boundary
```

when a crash can kill the process.

Fuzz:

```text
Buffer sizes
offsets
types
null-like values
invalid handles
concurrent calls.
```

See:

```text
Chapter 143.
```

---

# 124. Native Crash Triage

Capture:

```text
signal
stack
input
seed
Node version
addon build
architecture.
```

Then minimize:

```text
input
```

before:

```text
root-cause debugging.
```

---

# 125. WebAssembly Boundary Fuzzing

Generate:

```text
numeric inputs
memory offsets
byte arrays
module configurations.
```

Check:

```text
trap behavior
memory safety
result consistency
```

across:

```text
reference implementation
```

when available.

---

# 126. Differential Fuzzing for Optimizations

If:

```text
slow reference implementation
```

and:

```text
fast production implementation
```

exist:

```text
generate x
→ reference(x)
→ optimized(x)
→ compare.
```

This is an excellent candidate for:

```text
nightly fuzzing.
```

---

# 127. Fuzzing Async Code

Async fuzz targets need:

```text
timeout
cancellation
resource cleanup
deterministic scheduling
```

.

Generated input may also choose:

```text
which promise resolves first.
```

---

# 128. Schedule Fuzzing

Generate:

```text
A resolves before B
B before A
A rejects before B
both reject
A times out
B cancels A
```

and test:

```text
state invariants.
```

---

# 129. Race Fuzzing

A race fuzzer manipulates:

```text
timing
interleaving
resource availability
```

to expose:

```text
non-atomic state transitions.
```

---

# 130. Deterministic Race Reproduction

Record:

```text
seed
schedule decisions
virtual times
commands.
```

Then:

```text
replay schedule.
```

Without this:

```text
race bugs
```

are difficult to debug.

---

# 131. Node `node:test` Integration

Node's built-in Test Runner is stable and supports programmatic execution through:

```js
run(...)
```

which can emit structured test events and compose reporters. citeturn802937search1turn802937search0

Use `node:test` for:

```text
assertion lifecycle
fixtures
test isolation
reporting
CI integration.
```

Use a custom generator/fuzzer for:

```text
input exploration.
```

---

# 132. Generated Cases Inside `node:test`

A simple property harness:

```js
import test from "node:test";
import assert from "node:assert/strict";

function randomArray(rng) {
  const size = rng.int(0, 100);
  const out = [];

  for (let i = 0; i < size; i++) {
    out.push(rng.int(-1000, 1000));
  }

  return out;
}

test("reverse is involutive", () => {
  for (let i = 0; i < 1000; i++) {
    const input = randomArray(rng);
    const actual =
      [...input].reverse().reverse();

    assert.deepEqual(actual, input);
  }
});
```

This is a simple starting point, but:

```text
1000 random examples
```

are not yet a mature property-testing system because:

```text
shrinking
replay
distribution control
failure corpus
```

are missing.

---

# 133. Better Generated Test Structure

Separate:

```text
generator
property
runner
shrinker
replay.
```

Architecture:

```text
Generator<T>
       ↓
Runner
       ↓
Property<T>
       ↓
Failure?
  ├─ no
  └─ yes
      ↓
   Shrinker
      ↓
   Counterexample
      ↓
   Corpus
```

---

# 134. Minimal Generator Interface

Conceptually:

```js
function generate(seed, size) {
  return {
    value,
    seed,
    metadata
  };
}
```

Metadata can include:

```text
depth
size
category
generation decisions.
```

---

# 135. Minimal Shrinker Interface

```js
function* shrink(value) {
  yield simplerCandidate1;
  yield simplerCandidate2;
  yield simplerCandidate3;
}
```

Runner checks:

```text
does failure persist?
```

---

# 136. Simplest Shrinking Algorithm

```js
async function shrinkFailure(
  value,
  stillFails
) {
  let current = value;
  let changed = true;

  while (changed) {
    changed = false;

    for (const candidate of shrink(current)) {
      if (await stillFails(candidate)) {
        current = candidate;
        changed = true;
        break;
      }
    }
  }

  return current;
}
```

This is:

```text
greedy shrinking.
```

It is easy to understand but not necessarily:

```text
globally minimal.
```

---

# 137. Shrink Order Matters

A shrinker can try:

```text
large simplifications first
```

then:

```text
fine simplifications.
```

Better ordering can dramatically reduce:

```text
debugging cost.
```

---

# 138. Generator Metadata

Useful metadata:

```text
generator version
size
depth
category
seed
path decisions.
```

This helps answer:

```text
why was this case generated?
```

---

# 139. Counterexample Serialization

Serialize minimal failures into:

```text
JSON
binary
text
AST
command sequence.
```

Choose a format that is:

```text
stable
human-readable where possible
versioned
portable.
```

---

# 140. Counterexample Versioning

A future generator may no longer reproduce:

```text
old seed.
```

The saved counterexample must still be runnable against:

```text
current code.
```

This is why:

```text
corpus
```

matters more than:

```text
seed alone.
```

---

# 141. Regression Corpus

Every confirmed bug should become:

```text
unit example
+
counterexample corpus entry
+
property test coverage.
```

This creates:

```text
knowledge accumulation.
```

---

# 142. Corpus Labels

Label cases:

```text
bug-id
security
parser
performance
regression
platform
native
```

for targeted runs.

---

# 143. Fuzz Corpus Storage

For each case store:

```text
id
hash
input
classification
introduced version
fixed version
target
notes.
```

---

# 144. Corpus Governance

Avoid:

```text
unbounded binary dump storage.
```

Use:

```text
retention
deduplication
compression
priority
provenance.
```

---

# 145. Fuzz Campaign

A campaign defines:

```text
target
generator
seed corpus
mutation strategy
time budget
memory budget
parallelism
timeout
coverage signal
artifact policy
stop condition.
```

---

# 146. Campaign Time Budget

Examples:

```text
PR gate: 30 seconds
pre-merge: 5 minutes
nightly: 1 hour
weekly: 24 hours
```

Adjust to:

```text
risk
cost
target complexity.
```

---

# 147. Campaign CPU Budget

Do not run:

```text
unlimited workers
```

because:

```text
CI contention
```

can affect:

```text
production builds
```

or:

```text
other jobs.
```

---

# 148. Campaign Memory Budget

Limit:

```text
input size
process count
worker count
corpus retention
```

so a fuzz campaign cannot:

```text
consume the CI host.
```

---

# 149. Campaign Disk Budget

Fuzzing can create:

```text
many artifacts.
```

Store:

```text
unique failures
selected corpus
periodic summaries
```

rather than:

```text
every generated case.
```

---

# 150. Campaign Parallelism

Parallel fuzz workers should have:

```text
independent seeds
```

or:

```text
partitioned seed space.
```

Avoid accidental duplicate:

```text
work.
```

---

# 151. Fuzz Worker Architecture

```text
Supervisor
 ├── Worker 1
 ├── Worker 2
 ├── Worker 3
 └── Worker N
       ↓
    target
       ↓
  result stream
       ↓
 deduplication
       ↓
 corpus
```

---

# 152. Supervisor Responsibilities

The supervisor handles:

```text
timeouts
worker crashes
restart
budget
aggregation
artifact collection
```

while fuzz workers focus on:

```text
generation
execution.
```

---

# 153. Worker Restart

If a fuzz target crashes:

```text
restart worker
```

with:

```text
new process.
```

Do not let:

```text
one native crash
```

terminate the entire campaign.

---

# 154. Fuzzing Process Isolation

Use:

```text
child processes
```

for:

```text
native
crash-prone
signal-sensitive
memory-corruption
```

targets.

---

# 155. JavaScript-Only Fuzzing

For pure JS:

```text
same-process
```

can be acceptable for:

```text
fast property tests.
```

But still isolate:

```text
global state
timers
environment
module state
```

where required.

---

# 156. Fuzzing and `node:test` Isolation

Use:

```text
node:test process isolation
```

for test-file boundaries.

For:

```text
one dangerous fuzz target
```

use:

```text
additional child-process boundary.
```

---

# 157. Fuzzing Test Timeout

A fuzzer should bound:

```text
single input execution time.
```

Otherwise:

```text
one pathological case
```

can consume:

```text
entire campaign.
```

---

# 158. Timeout Classification

Differentiate:

```text
expected slow input
```

from:

```text
unbounded behavior.
```

Use:

```text
size-vs-time measurement
```

for performance-sensitive targets.

---

# 159. Fuzzing Resource Leaks

A fuzz target can accidentally leak:

```text
Buffer
timer
socket
worker
file
process.
```

Track:

```text
resource count
```

across repeated cases.

---

# 160. Fuzzing GC Noise

GC can create:

```text
runtime variability.
```

Do not classify one noisy:

```text
RSS spike
```

as:

```text
memory leak
```

without:

```text
repeatable growth.
```

---

# 161. Memory Growth Campaign

Run:

```text
same shape input
repeatedly
```

and measure:

```text
RSS
heap used
external
array buffer
```

for:

```text
trend.
```

---

# 162. Differential Failure Classification

If:

```text
A(x) !== B(x)
```

classify:

```text
A wrong
B wrong
both wrong
undefined contract.
```

Never automatically mark:

```text
one implementation
```

as correct.

---

# 163. Reference Models

For difficult systems:

```text
simple slow model
```

acts as:

```text
oracle.
```

Example:

```text
Set-based model
```

for:

```text
optimized cache/index.
```

---

# 164. Model-Based Testing Example

Real:

```text
LRU cache
```

Model:

```text
Map + explicit ordering
```

Generate:

```text
put
get
delete
evict.
```

Compare:

```text
keys
values
size
```

after every command.

---

# 165. Queue Model

Real:

```text
concurrent queue.
```

Model:

```text
array.
```

Generate:

```text
enqueue
dequeue
peek
clear.
```

Property:

```text
observable order
```

matches:

```text
model
```

under defined synchronization.

---

# 166. Database Model

Generate:

```text
insert
update
delete
query
transaction
rollback
commit.
```

Compare:

```text
real DB
```

to:

```text
reference model
```

where semantics can be modeled.

---

# 167. API Model

For REST APIs:

```text
create
read
update
delete
```

Generate:

```text
valid sequences
invalid sequences
duplicate operations
concurrent requests.
```

Properties:

```text
idempotence where promised
authorization invariants
referential integrity.
```

---

# 168. Authentication Property Testing

Generate:

```text
valid token
expired token
malformed token
wrong issuer
wrong audience
wrong signature
missing token
```

Properties:

```text
invalid token never grants protected access.
```

---

# 169. Authorization Property Testing

Generate:

```text
principal
resource
action
permissions
tenant
```

Property:

```text
decision
```

matches:

```text
authorization model.
```

---

# 170. Multi-Tenant Security Fuzzing

Generate:

```text
tenant A identity
tenant B identity
resource owner
resource ID
```

and assert:

```text
A cannot access B resource
```

unless:

```text
explicitly authorized.
```

---

# 171. Input Validation Properties

For validators:

```text
accepted input
→ satisfies validator contract

rejected input
→ cannot cross trust boundary.
```

Do not assume:

```text
validator = security boundary
```

without testing:

```text
downstream interpretation.
```

---

# 172. Canonicalization Fuzzing

Generate multiple representations:

```text
encoded
decoded
double encoded
mixed case
Unicode variants.
```

Property:

```text
authorization is evaluated on canonical meaning.
```

---

# 173. Parser Differential Security

Compare:

```text
frontend parser
```

and:

```text
backend parser
```

for:

```text
same input.
```

A dangerous mismatch can cause:

```text
request smuggling
filter bypass
validation bypass.
```

---

# 174. Serialization Differential

Test:

```text
producer
```

against:

```text
consumer.
```

Generate:

```text
optional fields
unknown fields
nulls
large numbers
duplicate keys
```

and verify:

```text
contract.
```

---

# 175. Fuzzing Logs

Generated data can contain:

```text
newlines
control chars
Unicode
terminal escape sequences.
```

Check that:

```text
logging
```

does not:

```text
forge misleading entries
```

or:

```text
break observability pipelines.
```

---

# 176. Fuzzing Error Messages

Properties can test:

```text
error is classified correctly
```

and:

```text
error does not leak secrets.
```

Generate:

```text
attacker-controlled inputs
```

and inspect:

```text
logs
responses
exceptions.
```

---

# 177. Fuzzing Configuration

Generate:

```text
missing
empty
null
wrong type
unknown key
duplicate key
out-of-range
```

and verify:

```text
safe defaults
explicit failure
```

where appropriate.

---

# 178. Configuration Merge Fuzzing

Generate nested:

```text
defaults
user config
environment config
CLI config.
```

Then assert:

```text
precedence rules
```

are deterministic.

Also test:

```text
prototype pollution.
```

---

# 179. Fuzzing CLI Parsers

Generate:

```text
flags
short flags
long flags
missing values
duplicate flags
unknown flags
quoted values
empty strings
negative numbers.
```

Property:

```text
parser never silently treats malformed syntax
as a dangerous default.
```

---

# 180. CLI Security Fuzzing

Run generated arguments in:

```text
isolated child processes.
```

Never directly feed:

```text
generated shell strings
```

into:

```text
production shell.
```

---

# 181. Fuzzing Environment Variables

Generate:

```text
missing
empty
unicode
long
unexpected separators
invalid encodings
```

and test:

```text
configuration parser
```

for:

```text
safe behavior.
```

---

# 182. Fuzzing Headers and Cookies

Generate:

```text
duplicate
empty
long
invalid
encoded
case variants.
```

Test:

```text
parser
normalizer
security checks.
```

---

# 183. Fuzzing Compression Bomb Defenses

Generate or use controlled fixtures with:

```text
high expansion ratios.
```

Verify:

```text
decompressed size limits
time limits
memory limits.
```

Run in:

```text
isolated test infrastructure.
```

---

# 184. Fuzzing Zip/File Paths

Generate:

```text
../
..\
absolute
encoded
nested
duplicate
```

and test:

```text
extraction confinement.
```

---

# 185. Fuzzing HTTP Bodies

Generate:

```text
empty
small
large
truncated
chunked
multipart
invalid encoding.
```

Test:

```text
body parser
limits
cleanup
timeouts.
```

---

# 186. Fuzzing Multipart Boundaries

Generate:

```text
boundary collisions
missing closing boundary
duplicate headers
empty parts
nested multipart.
```

This can reveal:

```text
parser state bugs
memory issues
request smuggling-like discrepancies.
```

---

# 187. Fuzzing WebSocket Protocols

Generate:

```text
connect
message
fragment
ping
pong
close
invalid frame
unexpected opcode.
```

Test:

```text
state machine invariants.
```

---

# 188. Fuzzing Worker Messages

Generate:

```text
valid message
malformed message
large payload
unexpected command
duplicate command
cancel during command.
```

Test:

```text
worker protocol.
```

---

# 189. Fuzzing Child Process Protocols

Generate:

```text
stdin
signals
arguments
environment
large output
unexpected exit.
```

Verify:

```text
supervisor cleanup.
```

---

# 190. Fuzzing Stream Backpressure

Generate:

```text
producer rate
consumer delay
buffer size
abort timing.
```

Property:

```text
memory remains bounded
```

within a defined workload.

---

# 191. Fuzzing Resource Limits

Generate near-boundary values for:

```text
body size
header size
queue depth
retry count
worker count
file size
recursion depth.
```

Verify:

```text
reject
degrade
or
bound
```

according to contract.

---

# 192. Fuzzing Error Paths

A mature fuzz campaign intentionally targets:

```text
error branches
```

rather than generating only:

```text
happy-path inputs.
```

Track:

```text
error-path coverage
```

where instrumentation supports it.

---

# 193. Fuzzing Valid Inputs Is Not Enough

A parser may handle:

```text
valid input
```

perfectly but fail on:

```text
near-valid malformed input.
```

Therefore corpus should include:

```text
valid
near-valid
invalid
adversarial.
```

---

# 194. Fuzzing Invalid Inputs Is Not Enough

Overly random invalid inputs can all fail immediately at:

```text
first parser check.
```

They reveal little.

Useful fuzzing seeks:

```text
deeply processed invalid inputs.
```

---

# 195. Reachability

A generator is valuable when it reaches:

```text
meaningful program states.
```

Not when it generates:

```text
lots of useless reject-at-byte-1 cases.
```

---

# 196. Generator Coverage

Track:

```text
validation branch
parser state
AST node type
protocol state
error category
```

reached by generated inputs.

---

# 197. Generator Quality Score

A useful internal score can combine:

```text
validity rate
structural diversity
state coverage
boundary frequency
failure yield.
```

Avoid reducing generator quality to:

```text
randomness.
```

---

# 198. Failure Yield

Measure:

```text
unique meaningful findings
/
generated executions.
```

A low yield is not always bad:

```text
mature target
```

may require:

```text
billions of inputs
```

for the next bug.

---

# 199. Unique Finding

A unique finding should be deduplicated using:

```text
stack fingerprint
failure class
input signature
coverage signature.
```

---

# 200. Fuzzing Dashboard

Track:

```text
executions/sec
unique paths
unique crashes
unique hangs
timeouts
coverage
corpus size
CPU
memory
worker health.
```

---

# 201. Corpus Lifecycle

```text
seed
→ mutate
→ execute
→ interesting?
→ retain
→ minimize
→ classify
→ regress.
```

---

# 202. Interestingness

An input may be interesting because it:

```text
finds new coverage
finds a new protocol state
triggers a new error
causes unusual resource behavior
finds a unique failure.
```

---

# 203. Minimization

After discovering:

```text
interesting input
```

minimize to:

```text
smallest useful reproducer.
```

Do this before:

```text
bug triage
```

where practical.

---

# 204. Crash Reproduction Contract

Every discovered crash should become a command such as:

```bash
node repro.js corpus/crash-001.bin
```

or:

```bash
node --test tests/regressions/crash-001.test.js
```

---

# 205. Regression Test Conversion

The ideal flow:

```text
fuzz finding
→ minimized input
→ human diagnosis
→ regression test
→ bug fix
→ rerun corpus.
```

---

# 206. Fuzzing in Pull Requests

PR fuzzing should be:

```text
bounded
fast
deterministic
```

with:

```text
fixed seeds
small budgets
known corpus.
```

---

# 207. Nightly Fuzzing

Nightly campaigns can use:

```text
many seeds
large budgets
more workers
deeper inputs
broader corpus.
```

They should publish:

```text
findings
coverage trends
new corpus.
```

---

# 208. Long-Running Fuzzing

For sustained campaigns:

```text
persist workers
rotate seeds
checkpoint corpus
monitor health
restart hung workers.
```

---

# 209. Fuzzing in CI Failure Policy

A CI fuzz failure should record:

```text
test/target
seed
input
Node version
commit
worker
timeout/crash details.
```

and:

```text
fail reproducibly.
```

---

# 210. Reproducibility Across Machines

A fuzz result can differ because of:

```text
Node version
OS
architecture
locale
timezone
PRNG
floating-point behavior
CPU timing
worker scheduling.
```

Therefore record:

```text
environment metadata.
```

---

# 211. Version-Pinned Fuzzing

Pin:

```text
Node
dependencies
generator
seed/corpus
runtime flags
```

for deterministic regression runs.

---

# 212. Fuzzing Node Upgrades

When upgrading Node:

```text
replay regression corpus
```

and:

```text
replay important saved seeds
```

to detect:

```text
runtime-specific changes.
```

---

# 213. Fuzzing Engine Behavior

Engine-focused fuzzing can generate:

```text
syntax
expressions
objects
proxies
iterators
async flows.
```

Targets may include:

```text
parser
interpreter
JIT
runtime built-ins.
```

For application engineers:

```text
differential behavior
```

is often more useful than:

```text
engine internals.
```

---

# 214. Fuzzing Specification Conformance

If a behavior is defined by:

```text
ECMAScript
```

build:

```text
input generator
+
expected semantic result
```

or compare against:

```text
reference model.
```

Distinguish:

```text
spec behavior
vs
host-specific behavior.
```

---

# 215. Fuzzing JavaScript APIs

Generate:

```text
valid
invalid
weird
hostile
boundary
```

arguments.

Test:

```text
return
throw
side effects
state transitions
```

against documented contracts.

---

# 216. Built-In API Differential Testing

Compare:

```text
API result
```

against:

```text
independent implementation
```

where feasible.

Examples:

```text
URL encoding
base64
date normalization
compression
serialization.
```

---

# 217. Security Fuzzing Principle

Never assume:

```text
“fuzzer input is random”
```

means:

```text
“fuzzer input is safe.”
```

Generated data can contain:

```text
shell syntax
network targets
filesystem paths
resource-exhaustion payloads.
```

Sandbox the target.

---

# 218. Fuzzing Permission Model

Use Node's Permission Model to restrict:

```text
filesystem
network
child process
native addon
```

access during fuzzing when compatible with the target.

This reduces:

```text
blast radius.
```

---

# 219. Fuzzing Native Code Security

For native fuzzing:

```text
process isolation
resource limits
sanitizers
minimal permissions
artifact capture
```

are stronger than:

```text
same-process brute force.
```

---

# 220. Fuzzing Dependency Supply Chain

Before fuzzing third-party binaries:

```text
verify provenance
```

and:

```text
avoid executing untrusted fuzz targets
```

inside:

```text
high-privilege CI.
```

---

# 221. Generator as Specification

A generator is also:

```text
executable domain model.
```

A weak generator means:

```text
weak input model.
```

Review generators like:

```text
production code.
```

---

# 222. Generator Review Questions

```text
What domain does it cover?
What domain does it exclude?
Where are the boundaries?
What distributions are biased?
Can it terminate?
Can it create pathological size?
Can it reproduce?
Can it shrink?
Can it generate security-sensitive payloads?
```

---

# 223. Property Review Questions

```text
What contract does the property represent?
Could a broken implementation still satisfy it?
Does the property overfit implementation?
Does it hold for all supported inputs?
Does it define normalization?
Does it handle errors?
Does it cover security behavior?
```

---

# 224. Shrinker Review Questions

```text
Does every candidate terminate?
Can shrink preserve the failure?
Can shrink generate invalid metadata?
Can shrink lose required structure?
Can shrink become O(n²) itself?
```

---

# 225. Fuzzer Review Questions

```text
What counts as interesting?
What counts as a failure?
How are timeouts classified?
How are crashes deduplicated?
How are seeds recorded?
How is corpus persisted?
How is worker isolation enforced?
How are secrets prevented from leaking?
```

---

# 226. Example: Sorting Property

Property:

```text
sort(output)
=
output
```

plus:

```text
multiset(output)
=
multiset(input)
```

This is stronger than:

```text
output is an array.
```

---

# 227. Sorting Generator

Generate:

```text
empty
duplicates
negative
large
already sorted
reverse sorted
NaN
Infinity
-0
```

if the sorting contract permits those values.

---

# 228. Sorting Metamorphic Property

For a valid comparator:

```text
sort(x + y)
```

should contain the same multiset as:

```text
sort(x) + sort(y)
```

after sorting the combined list where the contract allows.

Be explicit about:

```text
stability
comparator consistency
special values.
```

---

# 229. Example: Cache Property

Generate:

```text
put
get
delete
expire
```

Properties:

```text
get(after put) returns value
delete removes value
size matches number of stored entries
expiration removes inaccessible entries.
```

---

# 230. Cache Model

Reference model:

```js
Map
```

Real system:

```text
TTL/LRU cache.
```

Compare:

```text
observable reads
```

after:

```text
generated command sequence.
```

---

# 231. Example: Parser Property

For valid strings:

```text
parse(print(parse(x)))
```

should:

```text
retain semantic meaning.
```

For malformed strings:

```text
parser fails safely
```

without:

```text
hang
crash
unexpected state mutation.
```

---

# 232. Example: Validator Property

For every generated:

```text
value
```

if validator returns:

```text
valid
```

then:

```text
downstream invariant
```

must hold.

This is stronger than:

```text
validator returns boolean.
```

---

# 233. Example: Authorization Property

Generate:

```text
principal
resource
action
tenant
role
```

Property:

```text
cross-tenant access
```

is rejected unless:

```text
explicit cross-tenant permission exists.
```

---

# 234. Example: API Idempotence

If endpoint promises:

```text
PUT
```

idempotence:

```text
PUT(x); PUT(x)
```

should produce:

```text
same observable final state
```

as:

```text
PUT(x)
```

---

# 235. Example: Retry Property

Generate:

```text
failure sequence
```

such as:

```text
fail, fail, success
```

and verify:

```text
attempt count
backoff bounds
deadline
final state
```

---

# 236. Example: Stream Property

Generate arbitrary:

```text
chunk partition.
```

Then compare:

```text
one chunk
```

with:

```text
many chunks.
```

This exposes:

```text
state retained incorrectly across chunks.
```

---

# 237. Example: Encoding Property

For supported code points:

```text
decode(encode(x))
=
x
```

Include:

```text
ASCII
BMP
surrogate pairs
combining marks
boundary code points.
```

---

# 238. Example: URL Property

For canonical URLs:

```text
serialize(parse(url))
```

should equal:

```text
canonical(url)
```

rather than:

```text
original string
```

when normalization is intentional.

---

# 239. Example: File Path Property

For confined storage:

```text
resolve(root, generatedPath)
```

must remain:

```text
within root
```

after canonicalization.

Test:

```text
dot segments
encoding
symlinks
absolute paths
platform separators
```

where the target supports them.

---

# 240. Example: HTTP Parser Property

Property:

```text
valid request framing
```

must produce:

```text
exactly one interpretation.
```

Ambiguous:

```text
length headers
chunking
duplicate headers
```

must be:

```text
rejected or normalized deterministically.
```

---

# 241. Property Vacuity

A dangerous property can become trivially true.

Example:

```js
if (!isValid(x)) return;
assert(...)
```

If generator produces:

```text
99.99% invalid
```

the property can pass without testing:

```text
real behavior.
```

Measure:

```text
validity rate.
```

---

# 242. Conditional Properties

When using:

```text
precondition
```

track:

```text
accepted cases
rejected cases.
```

Otherwise:

```text
discarding
```

can hide:

```text
generator quality problems.
```

---

# 243. Better Than Rejection Sampling

Instead of:

```text
generate arbitrary
→ reject if invalid
```

prefer:

```text
generate directly from constrained domain
```

when possible.

This improves:

```text
coverage efficiency.
```

---

# 244. Generator Bias

Bias toward:

```text
high-risk states
```

not simply:

```text
uniform values.
```

Security fuzzing often benefits from:

```text
near-boundary
near-valid
deeply structured
```

inputs.

---

# 245. Generator Independence

Independent generators increase:

```text
combination space
```

but can produce:

```text
nonsensical combinations.
```

Domain-aware generators produce:

```text
meaningful interactions.
```

---

# 246. Cross-Field Constraints

For:

```text
request
```

generate:

```text
content-length
body
```

consistently for valid cases.

Then separately generate:

```text
mismatched values
```

for invalid cases.

---

# 247. Correlated Generation

Examples:

```text
expiry > issue time
start <= end
offset matches timezone
length matches payload
```

Generate:

```text
related fields
```

together.

---

# 248. Shrinking Correlated Data

When shrinking:

```text
start/end
length/body
offset/timezone
```

must remain:

```text
internally coherent
```

unless the test intentionally explores:

```text
invalid correlation.
```

---

# 249. Fuzzing and State Invariants

After every generated state transition, check:

```text
invariants.
```

Do not wait until:

```text
entire sequence completes.
```

Earlier detection produces:

```text
smaller command sequence
```

and:

```text
better diagnosis.
```

---

# 250. Property Granularity

Test properties at:

```text
function
module
service
protocol
system.
```

The correct level depends on:

```text
oracle availability
cost
risk.
```

---

# 251. Fuzz Target Selection

Highest-value targets often include:

```text
parsers
decoders
validators
protocol boundaries
security filters
serialization
native interfaces
complex state machines.
```

Pure deterministic arithmetic may have:

```text
lower fuzzing ROI
```

unless:

```text
algorithmic complexity
```

is the risk.

---

# 252. Fuzz Target Stability

A target should have:

```text
clear input boundary
clear result/failure signal
bounded runtime
bounded resource usage
repeatable environment.
```

---

# 253. Fuzz Target Adapter

Wrap production code:

```js
function fuzzTarget(input) {
  const result = target(input);

  assertInvariant(result);
}
```

Do not put:

```text
environment setup
```

inside:

```text
every fuzz iteration
```

unless required.

---

# 254. Harness Overhead

Fuzzing speed depends on:

```text
target cost
generator cost
serialization cost
process startup
coverage instrumentation.
```

Optimize:

```text
hot path
```

before adding:

```text
more workers.
```

---

# 255. In-Process vs Out-of-Process Fuzzing

In-process:

```text
fast
low overhead
shared memory
```

but:

```text
crash risk
state leakage.
```

Out-of-process:

```text
isolation
crash containment
clean state
```

but:

```text
IPC/startup cost.
```

---

# 256. Fuzzing Worker Pools

For CPU-bound JS fuzzing:

```text
Worker Threads
```

may increase throughput.

For crash-prone/native fuzzing:

```text
child processes
```

provide stronger containment.

---

# 257. Fuzzing and Event Loop

Do not run a huge synchronous fuzz loop on:

```text
main application event loop
```

inside:

```text
integration environment.
```

It can:

```text
starve
```

other work.

---

# 258. Fuzzing and Backpressure

When fuzzing async targets, bound:

```text
in-flight inputs
```

so:

```text
generated workload
```

does not become:

```text
unbounded promise accumulation.
```

---

# 259. Fuzzing Concurrency

Use:

```text
bounded concurrency
```

and measure:

```text
throughput vs contention.
```

---

# 260. Fuzzing Under Node Test Runner

Use:

```text
node:test
```

for:

```text
regression corpus
property smoke tests
small deterministic property runs.
```

Use:

```text
dedicated fuzz process
```

for:

```text
long-running campaigns.
```

This keeps:

```text
normal CI
```

fast.

---

# 261. Fuzz Smoke Tests

Every PR can run:

```text
100–10,000 generated cases
```

with:

```text
fixed seed
```

to catch:

```text
obvious regressions.
```

The exact count depends on:

```text
target speed.
```

---

# 262. Nightly Corpus Refresh

Nightly:

```text
generate broadly
→ discover cases
→ minimize
→ update corpus
```

then:

```text
commit/publish curated regressions.
```

---

# 263. Seed Rotation

Rotate seeds because:

```text
one fixed seed
```

eventually becomes:

```text
a fixed example suite.
```

Use:

```text
fixed seeds for regression
+
rotating seeds for discovery.
```

---

# 264. Corpus Plus Randomness

Best practice:

```text
known corpus
+
new generated cases
```

provides:

```text
memory
+
exploration.
```

---

# 265. Mutation Operators

Useful operators:

```text
delete
duplicate
replace
swap
truncate
insert
splice
increment
decrement
boundary substitution
nest
unnest.
```

Select operators based on:

```text
input structure.
```

---

# 266. Grammar-Aware Mutation

AST mutation can:

```text
replace operator
replace literal
delete statement
duplicate expression
swap branches.
```

This creates:

```text
semantic diversity.
```

---

# 267. Fuzzing with Existing Production Traffic

Anonymized production examples can seed:

```text
corpus
```

but require:

```text
privacy
secret scrubbing
PII removal
stable retention policy.
```

---

# 268. Fuzzing Realistic Inputs

Start from:

```text
realistic valid corpus
```

because:

```text
deep system paths
```

often require:

```text
structurally valid prefixes.
```

Then mutate:

```text
near boundaries.
```

---

# 269. PII-Safe Corpus

Never store:

```text
real tokens
passwords
private keys
customer data
session cookies.
```

Use:

```text
redaction
tokenization
synthetic replacement.
```

---

# 270. Secrets in Counterexamples

A failing generated input can accidentally include:

```text
environment values
```

or:

```text
credentials.
```

Run a:

```text
secret scanner
```

before:

```text
artifact publication.
```

---

# 271. Fuzzing External Services

Do not fuzz:

```text
public third-party APIs
```

without:

```text
explicit authorization.
```

Prefer:

```text
local simulator
sandbox
owned service.
```

---

# 272. Fuzzing Rate Limits

Generated retries can accidentally trigger:

```text
provider bans
```

.

Keep external integration fuzzing:

```text
local
rate-limited
sandboxed.
```

---

# 273. Fuzzing Database Boundaries

Generate:

```text
queries
filters
sorts
pagination
nulls
large values
```

but prefer:

```text
local test database
```

rather than:

```text
production.
```

---

# 274. SQL Fuzzing

Use a parser-aware generator for:

```text
expressions
predicates
ordering
joins.
```

The goal may be:

```text
query planner robustness
```

rather than:

```text
SQL injection
```

against:

```text
real infrastructure.
```

---

# 275. ORM Fuzzing

Generate:

```text
filter combinations
relations
null handling
pagination
sorting.
```

Compare:

```text
ORM result
```

against:

```text
reference queries
```

or:

```text
known domain model.
```

---

# 276. Fuzzing Authentication Middleware

Generate:

```text
authorization header
cookie
session
token
origin
method
path
```

and property:

```text
protected path
```

never bypasses:

```text
authorization policy.
```

---

# 277. Fuzzing CSRF/Origin Logic

Generate:

```text
same-origin
cross-origin
missing origin
null origin
weird casing
redirect contexts.
```

Verify:

```text
policy.
```

---

# 278. Fuzzing Content-Type Parsing

Generate:

```text
mixed case
parameters
quoted values
duplicate params
invalid syntax
missing boundary.
```

Check:

```text
deterministic interpretation.
```

---

# 279. Fuzzing Multipart/File Upload Validation

Generate:

```text
filename
content type
size
magic bytes
extension
path.
```

Property:

```text
accepted content
```

must satisfy:

```text
all downstream constraints.
```

---

# 280. Fuzzing Compression Ratios

Generate:

```text
highly repetitive input
```

and track:

```text
compressed/uncompressed ratio
```

for:

```text
resource protection.
```

---

# 281. Fuzzing DoS Resistance

Measure:

```text
input size
→ time
→ memory
```

to find:

```text
superlinear growth
```

before attackers do.

---

# 282. Property Testing for Performance

Property:

```text
runtime growth
```

should stay under:

```text
specified complexity envelope
```

for representative ranges.

Do not use:

```text
single fixed millisecond threshold
```

across all CI machines.

---

# 283. Complexity Regression Harness

Generate:

```text
n = 10
20
40
80
160
...
```

Measure:

```text
time(n)
```

and inspect:

```text
growth factor.
```

---

# 284. Fuzzing Allocation Behavior

Generate:

```text
small
medium
large
deep
wide
```

inputs.

Measure:

```text
RSS
heap
external
```

to detect:

```text
unexpected superlinear memory.
```

---

# 285. Test Flakiness Under Generative Testing

A property suite can itself be flaky if:

```text
seed not recorded
```

or:

```text
concurrent generation nondeterministic.
```

Use:

```text
seed
generated input
environment metadata.
```

---

# 286. Randomized Node Test Runner

Current Node v26 includes:

```bash
--test-randomize
--test-random-seed
```

for randomized test execution order. The seed controls ordering across test files and queued tests, and the CLI prints the seed for reproduction. citeturn802937search6

This complements:

```text
input generation randomness
```

but is:

```text{not the same randomness source}.
```

---

# 287. Test Order vs Input Randomness

There are two different variables:

```text
test execution order
```

and:

```text
generated input.
```

Record both.

---

# 288. Reproduction Matrix

A failure may require:

```text
test-order seed
+
property seed
+
minimized input
```

.

Record all three when available.

---

# 289. Test Runner Isolation for Fuzz Regressions

Current Node Test Runner process isolation runs each test file in a separate child process by default when `--test` is used. This can contain global/module state leakage between files. citeturn802937search6

Use:

```text
process isolation
```

for:

```text
security-sensitive regressions
native crash repros
stateful fixtures
```

where appropriate.

---

# 290. Programmatic Fuzz Runner

The Node test runner's `run()` API exposes a test stream suitable for custom orchestration and reporters. citeturn802937search0

A custom platform can conceptually:

```text
run regression tests
→ collect test events
→ attach fuzz metadata
→ publish artifacts.
```

---

# 291. Fuzzing and Reporters

Use a machine-readable channel for:

```text
seed
input ID
failure class
duration
worker
target.
```

Human output should remain:

```text
diagnostic and concise.
```

---

# 292. Fuzz Failure Report

Minimum:

```text
Target:
Commit:
Node:
Platform:
Seed:
Input:
Input hash:
Failure:
Stack:
Duration:
Worker:
Timeout:
```

---

# 293. Counterexample Readability

A counterexample should be:

```text
small
formatted
named
replayable.
```

Prefer:

```text
commands
AST
JSON
```

over:

```text
opaque binary
```

when possible.

---

# 294. Binary Counterexamples

For binary protocols, include:

```text
hex dump
decoded summary
raw bytes
```

so humans can inspect:

```text
structure.
```

---

# 295. Fuzz Triage Workflow

```text
DETECT
 ↓
REPRODUCE
 ↓
MINIMIZE
 ↓
CLASSIFY
 ↓
ROOT CAUSE
 ↓
FIX
 ↓
REGRESSION
 ↓
CORPUS
 ↓
VERIFY FIX
```

---

# 296. Root Cause Categories

```text
correctness
parser
boundary
state machine
resource
concurrency
performance
security
runtime
native
platform
dependency.
```

---

# 297. Security Finding Escalation

A fuzz finding becomes security-sensitive when it can imply:

```text
bypass
data exposure
code execution
memory corruption
denial of service
privilege escalation.
```

Handle according to:

```text
security response process.
```

---

# 298. Fuzzing and Disclosure

Do not publish:

```text
unfixed security payloads
```

in:

```text
public corpus
```

until:

```text
responsible disclosure
```

is complete.

---

# 299. Implementation From Scratch — Mini Property Framework

Build:

```text
Generator
Arbitrary
Shrink
Check
Runner
Replay
Corpus
```

---

# 300. Milestone 1 — Seeded PRNG

Implement:

```js
function createRng(seed) {
  let state = seed >>> 0;

  return {
    next() {
      state =
        (Math.imul(state, 1664525) + 1013904223) >>> 0;
      return state / 2 ** 32;
    }
  };
}
```

Understand that:

```text
PRNG algorithm
```

becomes:

```text
part of reproduction semantics.
```

---

# 301. Milestone 2 — Integer Generator

Implement:

```js
function int(rng, min, max) {
  return min +
    Math.floor(
      rng.next() * (max - min + 1)
    );
}
```

Then add:

```text
biased boundaries
```

for:

```text
0
1
-1
min
max.
```

---

# 302. Milestone 3 — Array Generator

Implement:

```text
size generator
+
element generator.
```

Add:

```text
metadata.
```

---

# 303. Milestone 4 — Shrinking

Implement:

```text
array chunk deletion
+
element shrinking.
```

Confirm:

```text
same failure
```

with:

```text
smaller input.
```

---

# 304. Milestone 5 — Property Runner

Implement:

```text
N cases
→ property
→ first failure
→ shrink
→ report.
```

---

# 305. Milestone 6 — Replay File

Write:

```json
{
  "seed": 123,
  "input": [1, 2, 3]
}
```

Then:

```text
replay without regeneration.
```

---

# 306. Milestone 7 — Regression Corpus

Store:

```text
every confirmed bug.
```

Run corpus:

```text
before fuzz discovery
```

so:

```text
known bugs
```

remain permanently guarded.

---

# 307. Milestone 8 — Stateful Commands

Implement:

```text
Command
precondition
execute
model update
postcondition
```

.

Generate:

```text
command sequences.
```

---

# 308. Milestone 9 — Differential Fuzzer

Implement:

```text
reference(input)
optimized(input)
compare.
```

Add:

```text
normalization.
```

---

# 309. Milestone 10 — Worker Supervisor

Create:

```text
parent supervisor
child fuzz worker.
```

Supervisor handles:

```text
timeout
crash
restart
artifact.
```

---

# 310. Milestone 11 — Corpus Deduplication

Use:

```text
SHA-256 hash
```

for exact duplicate elimination.

Then add:

```text
semantic/failure fingerprint
```

for deeper deduplication.

---

# 311. Milestone 12 — Fuzz Dashboard

Track:

```text
cases/sec
unique findings
timeouts
crashes
coverage proxy
corpus count
worker status.
```

---

# 312. Milestone 13 — Node Test Integration

Run:

```text
regression corpus
+
fixed seed smoke properties
```

through:

```text
node:test
```

and publish:

```text
standard CI result.
```

---

# 313. Milestone 14 — Nightly Campaign

Add:

```text
rotating seeds
larger budget
more workers
corpus persistence
failure notifications.
```

---

# 314. Milestone 15 — Security Fuzz Target

Pick:

```text
parser
validator
auth boundary
```

and create:

```text
isolated fuzz harness
```

with:

```text
resource limits
secret filtering
artifact policy.
```

---

# 315. Debugging Exercises

## Exercise A — Find an Invariant

Given a cache:

```text
put
get
delete
```

write at least:

```text
5 invariants.
```

---

## Exercise B — Build a Generator

Generate:

```text
nested objects
```

with:

```text
depth
width
special keys.
```

---

## Exercise C — Build a Shrinker

Create a failure from:

```text
large array
```

and shrink it to:

```text
minimal counterexample.
```

---

## Exercise D — Reproduce by Seed

Record:

```text
seed
```

cause a failure, then:

```text
replay same seed.
```

---

## Exercise E — Corpus Regression

Turn one generated failure into:

```text
regression fixture
```

and ensure:

```text
future fixes cannot reintroduce it.
```

---

## Exercise F — Differential Fuzzer

Compare:

```text
simple sort
```

with:

```text
production sort.
```

---

## Exercise G — Parser Fuzzer

Generate:

```text
valid
near-valid
invalid
```

JSON-like inputs.

Verify:

```text
no hang
no crash
correct rejection.
```

---

## Exercise H — Stateful Fuzzer

Model:

```text
LRU cache
```

and fuzz:

```text
put
get
delete
```

---

## Exercise I — Security Fuzzer

Generate:

```text
__proto__
constructor.prototype
```

shapes and test:

```text
merge utility.
```

---

## Exercise J — Complexity Fuzzer

Generate:

```text
n = 10, 20, 40, 80, ...
```

and measure:

```text
runtime growth.
```

---

# 316. Code Review Exercise — Random Loop

```js
for (let i = 0; i < 100000; i++) {
  const value =
    Math.floor(Math.random() * 100);

  runTest(value);
}
```

Find:

```text
no seed
no replay
no distribution visibility
no shrink
no corpus
no timeout
```

---

# 317. Code Review Exercise — Rejection Sampling

```js
let value;

do {
  value = randomValue();
} while (!isInteresting(value));

test(value);
```

Problems:

```text
unbounded generation
unknown acceptance rate
possible starvation
poor distribution.
```

---

# 318. Code Review Exercise — Giant Counterexample

A fuzz failure stores:

```text
20 MB JSON
```

but no:

```text
seed
hash
shrinker
generator version.
```

Find why:

```text
the failure is expensive to debug and reproduce.
```

---

# 319. Code Review Exercise — Retry as “Fix”

```text
fuzz finding failed once
→ rerun 10 times
→ it passes
→ delete finding.
```

Correct response:

```text
reproduce
classify nondeterminism
record environment
do not delete evidence.
```

---

# 320. Code Review Exercise — Unsafe External Fuzzing

```text
for (;;) {
  await fetch(randomUrl());
}
```

Problems:

```text
unauthorized targets
network abuse
rate limits
data leakage
resource explosion.
```

---

# 321. Predict-the-Output / Behavior Exercises

### Exercise 1

Generate:

```text
-0
```

Predict:

```js
Object.is(value, -0)
```

---

### Exercise 2

Generate:

```text
NaN
```

Predict:

```js
value === value
```

---

### Exercise 3

Generate:

```text
new Array(3)
```

Predict:

```text
Object.keys(value).length
```

---

### Exercise 4

Generate:

```text
[undefined, undefined]
```

Compare:

```text
JSON.stringify(...)
```

with:

```text
JSON.stringify(new Array(2))
```

---

### Exercise 5

Generate a nested object containing:

```text
__proto__
```

Predict:

```text
which merge implementations become dangerous.
```

---

### Exercise 6

Generate:

```text
start
end
```

where:

```text
start > end.
```

Predict whether:

```text
range(start,end)
```

should:

```text
swap
reject
return empty.
```

This is an:

```text
API contract decision.
```

---

### Exercise 7

Generate:

```text
1000 random cases
```

with:

```text
90% empty arrays.
```

Predict:

```text
what your apparent passing rate says about test quality.
```

---

### Exercise 8

A failure requires:

```text
seed 42
```

but generator version changed.

Predict:

```text
whether seed 42 alone still guarantees the same input.
```

---

### Exercise 9

A fuzz target crashes after:

```text
50 ms
```

and usually runs in:

```text
1 ms.
```

Predict:

```text
whether you should investigate the outlier.
```

---

### Exercise 10

Two implementations agree on:

```text
10 million inputs
```

but both share:

```text
same specification mistake.
```

Predict:

```text
why differential agreement is not proof.
```

---

# 322. Interview Questions

### Fundamentals

```text
1. What is property-based testing?
2. What is a property?
3. What is fuzzing?
4. How is fuzzing different from random testing?
5. What is generative testing?
```

### Properties

```text
6. What is an invariant?
7. What is a round-trip property?
8. What is idempotence?
9. What is a metamorphic property?
10. What is differential testing?
```

### Generators

```text
11. What makes a good generator?
12. Why bias toward boundaries?
13. Why is uniform random data often inefficient?
14. How do you generate recursive structures safely?
15. What is rejection sampling and why can it be dangerous?
```

### Shrinking

```text
16. Why is shrinking important?
17. How would you shrink an array?
18. How would you shrink a tree?
19. How would you shrink a command sequence?
20. What makes a shrinker incorrect?
```

### Reproducibility

```text
21. Why is a seed not enough?
22. Why store the minimized input?
23. What metadata belongs in a repro?
24. How do you reproduce async race fuzzing?
```

### Fuzzing

```text
25. What is coverage-guided fuzzing?
26. What is gray-box fuzzing?
27. What is grammar-based fuzzing?
28. How do you detect hangs?
29. How do you deduplicate crashes?
```

### Security

```text
30. How would you fuzz a parser securely?
31. How would you fuzz auth middleware?
32. How would you fuzz prototype pollution?
33. How would you fuzz path traversal?
34. How do you prevent fuzz payloads from harming external systems?
```

### Node

```text
35. How would you integrate property tests with node:test?
36. When would you use Worker Threads?
37. When would you use child processes?
38. How would Permission Model help fuzzing?
39. How would you capture fuzz findings from node:test?
```

### Principal

```text
40. Design a fuzzing platform for a Node monorepo.
41. How would you combine PR fuzzing and nightly fuzzing?
42. How would you manage a corpus across teams?
43. How would you prevent flaky fuzz failures?
44. How would you fuzz native addons safely?
45. How would you use model-based testing for an API?
46. How would you decide which code deserves fuzzing?
47. How would you measure fuzzing ROI?
48. How would you defend “high coverage but low bug yield”?
```

---

# 323. Mastery Exercises

### Exercise 1 — Property Portfolio

For one production feature, define:

```text
5 invariants
3 round-trips
2 metamorphic properties
1 differential property.
```

### Exercise 2 — Generator

Build generators for:

```text
user
order
HTTP request
URL
nested config.
```

### Exercise 3 — Shrinker

Implement shrinkers for:

```text
number
string
array
object
command sequence.
```

### Exercise 4 — Stateful Model

Create:

```text
reference model
```

for:

```text
cache
queue
or session state machine.
```

### Exercise 5 — Differential Fuzzer

Compare:

```text
reference
vs
optimized.
```

### Exercise 6 — Security Campaign

Fuzz:

```text
auth
path
config
merge
```

through:

```text
isolated process.
```

### Exercise 7 — CI Integration

Build:

```text
PR smoke fuzz
+
nightly campaign
+
corpus
+
artifact publishing.
```

### Exercise 8 — Crash Minimizer

Given:

```text
large failing input,
```

produce:

```text
minimal replayable case.
```

### Exercise 9 — Flake-Safe Fuzzer

Generate:

```text
random input
```

with:

```text
fixed seed
```

and prove:

```text
same seed
→ same input
```

under:

```text
same generator version.
```

### Exercise 10 — Principal Design

Design a system supporting:

```text
10,000 fuzz targets
1,000 workers
shared corpus
failure deduplication
security isolation
nightly runs
PR smoke runs.
```

---

# 324. Track A — Core Theory

Master:

```text
properties
invariants
metamorphic testing
differential testing
generators
distribution
shrinkers
counterexamples
corpus
coverage-guided fuzzing
grammar-based fuzzing
stateful testing
model-based testing
fuzz campaigns
reproducibility
deduplication
security isolation.
```

Deliverable:

```text
Explain how generated inputs become minimized evidence.
```

---

# 325. Track B — Implementation

Build:

```text
PRNG
integer generator
structured generator
shrinker
property runner
replay system
corpus
state machine fuzzer
differential fuzzer
worker supervisor
failure collector
nightly campaign.
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

# 326. Track C — Interview / Reasoning

Practice:

```text
“What makes a property strong?”

“Why is shrinking necessary?”

“Why is seed-only reproduction insufficient?”

“Why can random tests have low value?”

“When would you choose grammar-based fuzzing?”

“How do you fuzz async state machines?”

“When should a fuzz target run out-of-process?”

“How do you distinguish crash uniqueness from input uniqueness?”

“How do you build a regression corpus?”

“How do you prevent fuzzing from becoming infrastructure abuse?”
```

Answer using:

```text
contract
generator
oracle
determinism
shrink
isolation
resource budget
evidence
trade-offs.
```

---

# 327. Principal Decision Framework

For every generative-testing system ask:

```text
1. What behavior matters?
2. What is the domain?
3. What input states are high risk?
4. What property proves the behavior?
5. Is there an exact oracle?
6. If not, is there a reference model?
7. If not, is there a metamorphic relation?
8. What generator reaches meaningful states?
9. What distribution should it use?
10. Which boundaries are intentionally over-sampled?
11. Can generation terminate?
12. Can generated data become dangerously large?
13. How will failures be detected?
14. How will hangs be detected?
15. How will crashes be isolated?
16. How will failures be shrunk?
17. How will counterexamples be serialized?
18. Is seed reproducibility sufficient?
19. Where will minimized inputs live?
20. How are findings deduplicated?
21. How are security findings protected?
22. What Node/runtime version is pinned?
23. What process/worker isolation is required?
24. What is the CPU budget?
25. What is the memory budget?
26. What is the disk budget?
27. What is the PR budget?
28. What is the nightly budget?
29. What is the stop condition?
30. What is the regression policy?
31. How will corpus quality be measured?
32. How will generator quality be reviewed?
33. How will fuzz infrastructure failures be distinguished from target failures?
34. What evidence reaches CI?
35. What is the operational maintenance cost?
```

---

# 328. Production Property-Test Checklist

```text
[ ] properties documented
[ ] generators documented
[ ] domain boundaries documented
[ ] boundary bias intentional
[ ] generator termination guaranteed
[ ] generator size bounded
[ ] correlated fields generated coherently
[ ] shrinkers tested
[ ] seed recorded
[ ] minimized input stored
[ ] regression corpus exists
[ ] failures deduplicated
[ ] timeouts enforced
[ ] dangerous targets isolated
[ ] resource budgets enforced
[ ] secrets scrubbed
[ ] CI smoke fuzzing configured
[ ] nightly fuzzing configured
[ ] Node version pinned
[ ] test runner configuration pinned
[ ] security findings governed
```

---

# 329. Generator Quality Checklist

```text
[ ] common cases
[ ] edge cases
[ ] boundaries
[ ] malformed cases
[ ] near-valid cases
[ ] adversarial structures
[ ] deep structures
[ ] wide structures
[ ] Unicode
[ ] numeric extremes
[ ] special values
[ ] cross-field constraints
[ ] stateful sequences
[ ] realistic corpus seeds
[ ] measurable distribution
```

---

# 330. Shrinker Quality Checklist

```text
[ ] terminates
[ ] preserves target domain where required
[ ] tries coarse simplification first
[ ] tries fine simplification later
[ ] preserves failure
[ ] preserves relevant metadata
[ ] handles recursive structures
[ ] handles correlated values
[ ] handles command sequences
[ ] produces readable cases
```

---

# 331. Fuzz Campaign Checklist

```text
[ ] target
[ ] corpus
[ ] generator
[ ] mutation strategy
[ ] seed policy
[ ] timeout
[ ] CPU budget
[ ] memory budget
[ ] worker count
[ ] isolation
[ ] artifact policy
[ ] deduplication
[ ] minimization
[ ] regression conversion
[ ] security handling
[ ] monitoring
```

---

# 332. Failure Reproduction Checklist

```text
[ ] Node version
[ ] package lock
[ ] OS
[ ] architecture
[ ] runtime flags
[ ] test order seed
[ ] generator seed
[ ] minimized input
[ ] generator version
[ ] corpus version
[ ] target version
[ ] environment summary
[ ] failure fingerprint
```

---

# 333. Current Platform Notes

As of September 2026, the official Node.js v26 documentation describes:

```text
node:test
```

as:

```text
Stable.
```

The current v26 CLI includes:

```text
--test-isolation
--test-name-pattern
--test-randomize
--test-random-seed
--test-shard
--test-rerun-failures
--test-reporter
--test-reporter-destination
--experimental-test-coverage
--test-global-setup
--experimental-test-module-mocks
--experimental-test-tag-filter
```

with different stability levels. In particular:

```text
test runner
→ stable

test randomization
→ available in v26

random seed
→ available in v26

test sharding
→ available

rerun failures
→ available

coverage
→ experimental CLI integration

module mocking
→ early development

tag filter
→ early development

global setup
→ early development.
```

The exact flags and stability classifications are version-sensitive and should be verified against the pinned Node version used by the repository. citeturn802937search6turn802937search8

---

# 334. Source Discipline

Use this hierarchy:

```text
ECMAScript specification
↓
Node.js documentation
↓
engine/runtime documentation
↓
library documentation
↓
application contract
```

For fuzzing specifically distinguish:

```text
language semantics
host behavior
Node behavior
implementation-specific behavior
security assumptions.
```

When documenting a feature, label:

```text
stable
experimental
early development
version-specific
proposal
application-level convention.
```

Current official Node docs are the authoritative source for the v26 test-runner capabilities used in this chapter. citeturn802937search6turn802937search1

---

# 335. Performance Considerations

Fuzzing performance depends on:

```text
generation
serialization
target execution
coverage instrumentation
IPC
shrinking
artifact handling.
```

Measure:

```text
executions/sec
```

but also:

```text
unique useful findings/sec.
```

The fastest useless generator is:

```text
not
```

the best fuzzer.

---

# 336. Memory Considerations

Control:

```text
input size
corpus size
worker count
concurrent inputs
artifact retention
coverage data.
```

A generator that occasionally creates:

```text
gigabyte-sized nested objects
```

can destroy:

```text
CI reliability.
```

---

# 337. Security Considerations

Generated inputs can become:

```text
shell syntax
filesystem traversal
network abuse
resource exhaustion
credential-shaped data.
```

Therefore fuzz infrastructure should use:

```text
minimal permissions
isolated processes
sandbox/test services
secret scrubbing
resource budgets.
```

---

# 338. Common Misconceptions

### Misconception 1

```text
“Property-based testing is just random testing.”
```

Reality:

```text
properties + generators + shrinking + reproduction
```

make it a richer methodology.

### Misconception 2

```text
“More random values automatically mean better coverage.”
```

Reality:

```text
distribution and reachability matter.
```

### Misconception 3

```text
“Seed = full reproduction.”
```

Reality:

```text
generator/runtime/version can change the sequence.
```

### Misconception 4

```text
“If the fuzzer did not crash, the code is safe.”
```

Reality:

```text
oracle weakness can hide semantic/security bugs.
```

### Misconception 5

```text
“Coverage means correctness.”
```

Reality:

```text
coverage is guidance, not proof.
```

### Misconception 6

```text
“A giant crash input is useful evidence.”
```

Reality:

```text
minimal counterexamples are usually far more actionable.
```

### Misconception 7

```text
“Nightly fuzzing replaces unit and integration tests.”
```

Reality:

```text
generative testing complements, not replaces, deterministic regression tests.
```

---

# 339. Common Mistakes

```text
[ ] no property, only randomness
[ ] generator almost always produces trivial values
[ ] no boundary bias
[ ] no shrinking
[ ] seed not recorded
[ ] minimized input not saved
[ ] no regression corpus
[ ] rejection sampling can starve
[ ] recursive generator unbounded
[ ] correlated fields generated inconsistently
[ ] huge inputs consume CI
[ ] no timeout
[ ] no crash isolation
[ ] no security sandbox
[ ] no deduplication
[ ] corpus unbounded
[ ] fuzz findings not converted into tests
[ ] differential test uses non-independent implementations
[ ] property is vacuous
[ ] invalid cases dominate without intent
[ ] valid cases are missing
```

---

# 340. Final Generative Testing Mental Model

```text
DOMAIN
 ↓
PROPERTY / ORACLE
 ↓
GENERATOR
 ↓
INPUT
 ↓
TARGET
 ↓
OBSERVATION
 ↓
FAILURE?
 ├─ no → next input
 └─ yes
      ↓
    SHRINK
      ↓
 COUNTEREXAMPLE
      ↓
 REPRODUCE
      ↓
 CLASSIFY
      ↓
 REGRESSION
      ↓
 CORPUS
```

---

# 341. Fuzzing Mental Model

```text
CORPUS
 +
GENERATION
 +
MUTATION
 +
EXECUTION FEEDBACK
 ↓
INTERESTING INPUT?
 ↓
RETAIN
 ↓
MINIMIZE
 ↓
RUN REGRESSION
```

---

# 342. Property Mental Model

```text
INPUT SPACE
      ↓
      x
      ↓
PROPERTY
      ↓
TRUE / FALSE
      ↓
COUNTEREXAMPLE
      ↓
MINIMAL CASE
      ↓
HUMAN UNDERSTANDING
```

---

# 343. Shrinking Mental Model

```text
LARGE FAILURE
     ↓
DELETE
     ↓
SIMPLIFY
     ↓
REDUCE
     ↓
RETRY
     ↓
FAILURE REMAINS?
     ├─ yes → continue
     └─ no → keep previous
```

---

# 344. Reproduction Mental Model

```text
FAILURE
 ↓
seed
 +
environment
 +
generator version
 +
input
 ↓
replay
 ↓
same failure
```

The strongest evidence is:

```text
minimized serialized input
```

plus:

```text
repro metadata.
```

---

# 345. Fuzz ROI Model

```text
value
=
risk covered
×
state space explored
×
failure detectability
×
reproducibility
÷
infrastructure cost
```

Optimize:

```text
useful exploration
```

rather than:

```text
raw randomness.
```

---

# 346. Dependency Graph

```text
Chapter 29
Testing Fundamentals
        ↓
Chapter 31
Async
        ↓
Chapter 33
Event Loop
        ↓
Chapter 37
Generators
        ↓
Chapter 42
Data Structures
        ↓
Chapter 47
Regular Expressions
        ↓
Chapter 52
Workers
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Debugging
        ↓
Chapter 71
Security
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
Chapter 125
Promise Internals
        ↓
Chapter 129
Regex Semantics
        ↓
Chapter 131
URL Semantics
        ↓
Chapter 140
Node Networking
        ↓
Chapter 141
Node Diagnostics
        ↓
Chapter 142
Permission Model
        ↓
Chapter 143
Native Addons
        ↓
Chapter 144
Node Test Runner / Mocking / Isolation
        ↓
Chapter 145
Property-Based Testing, Fuzzing & Generative Testing
```

---

# 347. Concept Connections

## Depends On

```text
testing
assertions
randomness
data generation
async control
isolation
diagnostics
performance
security
Node runtime.
```

## Builds Toward

```text
determinism engineering
flaky-test engineering
platform testing
security verification
parser hardening
runtime conformance
large-scale CI
production confidence.
```

## Related Concepts

```text
property
generator
shrinker
oracle
fuzzer
corpus
seed
metamorphic test
differential test
model-based test
coverage-guided fuzzing
state-machine testing.
```

## Concepts Revisited

```text
Randomness
Timers
Promises
Workers
Child Processes
HTTP
URL
Regex
Serialization
Diagnostics
Permission Model
Native Addons
Performance
Security
```

## Why This Chapter Matters

Example tests cover:

```text
known cases.
```

Generative testing explores:

```text
unknown combinations.
```

Fuzzing aggressively searches:

```text
unexpected and adversarial states.
```

Shrinking turns:

```text
unknown failure
```

into:

```text
small understandable evidence.
```

The result is not merely:

```text
more tests.
```

It is:

```text
better failure discovery
+
better specifications
+
better regression memory.
```

---

# 348. Retrieval Record

```md
# Chapter 145 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Property Testing
-

## Generators
-

## Distribution
-

## Shrinking
-

## Reproduction
-

## Metamorphic Testing
-

## Differential Testing
-

## Stateful Testing
-

## Coverage-Guided Fuzzing
-

## Grammar-Based Fuzzing
-

## Security Fuzzing
-

## Performance Fuzzing
-

## Corpus
-

## Deduplication
-

## Node:test Integration
-

## Process Isolation
-

## Worker / Child Strategy
-

## CI
-

## Nightly Campaign
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

# 349. Spaced Retrieval Schedule

### Day 0

Explain:

```text
property
generator
shrinker
oracle
fuzzer
```

without notes.

### Day 1

Write:

```text
10 properties
```

for:

```text
array
cache
URL
parser
API.
```

### Day 3

Build:

```text
integer generator
array generator
shrinker
```

from scratch.

### Day 7

Build:

```text
stateful model-based test
```

and:

```text
differential test.
```

### Day 14

Build:

```text
parser fuzz target
+
corpus
+
replay.
```

### Day 21

Build:

```text
multi-worker fuzz supervisor
```

with:

```text
timeouts
deduplication
artifact collection.
```

### Day 30

Design:

```text
enterprise fuzzing platform
```

without notes.

---

# 350. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
write simple properties and generators.
```

Mark:

```text
[?] Needs Revision
```

when:

```text
your properties are weak
your generator is trivial
you cannot reproduce failures
shrinkers are unreliable
```

.

Mark:

```text
[+] Completed
```

when you can:

```text
build generators
write meaningful properties
shrink failures
persist counterexamples
integrate with node:test
```

.

Mark:

```text
[*] Mastered
```

only when you can:

```text
design and operate a generative testing platform
for parsers, protocols, security boundaries,
stateful services, native code, and performance-sensitive
systems with deterministic reproduction, shrinking,
corpus governance, isolation, and CI/nightly strategy.
```

Reading alone does not mark mastery.

---

# 351. Final Principal Principle

> **The purpose of generative testing is not to generate as much randomness as possible. The purpose is to explore meaningful state space, expose violated contracts, minimize the evidence, reproduce the failure, and permanently convert that evidence into engineering knowledge.**

The principal workflow is:

```text
DEFINE DOMAIN
→ DEFINE PROPERTY / ORACLE
→ GENERATE MEANINGFUL INPUT
→ EXPLORE BOUNDARIES
→ EXECUTE SAFELY
→ DETECT FAILURE
→ SHRINK
→ REPRODUCE
→ CLASSIFY
→ REGRESS
→ RETAIN IN CORPUS
→ MEASURE COVERAGE / YIELD
→ EXPAND CAMPAIGN
```

Remember:

```text
randomness ≠ coverage

coverage ≠ correctness

seed ≠ complete reproduction

generator ≠ specification automatically

fuzzing ≠ production attack

retry ≠ diagnosis

large failure ≠ useful failure

crash-free ≠ bug-free

property ≠ assertion of one example

shrinking ≠ optional debugging decoration

corpus ≠ dumping ground

nightly fuzzing ≠ replacement for deterministic tests

more workers ≠ more useful exploration
```

The principal question is:

```text
“What meaningful state space are we exploring, what exact
property tells us the system is correct, how will we generate
high-value inputs, how will we minimize the first counterexample,
how will we reproduce it across environments, and how will we
turn every important discovery into permanent regression knowledge?”
```

That is property-based testing and fuzzing engineering.