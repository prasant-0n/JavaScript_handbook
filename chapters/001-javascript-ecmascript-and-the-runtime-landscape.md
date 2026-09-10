# Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape

> **Status:** `[+] Completed`\
> **Role in curriculum:** Foundation for every later chapter.\
> **Primary question:** What exactly is JavaScript, what does ECMAScript
> define, and what part of "JavaScript behavior" comes from the runtime
> or host?

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain the difference between JavaScript, ECMAScript, a JavaScript
    engine, a runtime, and a host environment.
-   Explain why JavaScript can run in a browser, Node.js, Deno, Bun,
    embedded systems, serverless environments, and other hosts while
    retaining a common language core.
-   Identify which behaviors are defined by ECMAScript and which are
    defined by a host.
-   Explain the role of Ecma International and TC39.
-   Explain the relationship between the ECMAScript specification,
    engine implementations, and host APIs.
-   Understand yearly ECMAScript editions versus the living
    specification.
-   Explain TC39 proposal stages and why a proposal must not be treated
    as a language standard merely because an implementation supports it.
-   Distinguish specification guarantees from implementation details
    such as V8 optimization behavior.
-   Build an accurate mental model for investigating "JavaScript
    behavior" by asking whether the cause is language, engine, host,
    library, or application code.
-   Read basic specification terminology without assuming that
    specification text is ordinary source code or an implementation
    recipe.
-   Reason about portability and compatibility across JavaScript
    environments.

------------------------------------------------------------------------

## 2. Prerequisites

This chapter is intentionally the first chapter.

Recommended background:

-   Basic programming concepts such as variables, functions, values,
    control flow, and source code.
-   No prior ECMAScript specification knowledge is required.

Later chapters will supply the technical foundations needed to
understand the deeper execution model.

------------------------------------------------------------------------

## 3. What Is JavaScript?

JavaScript is a programming language.

That statement sounds obvious, but it is the most important starting
point because the term "JavaScript" is often used to refer to several
different layers at once.

A JavaScript program is source text written according to the syntax and
semantic rules of the language. The source text is processed by an
implementation such as a JavaScript engine. That engine executes the
language semantics inside a host environment that may provide additional
capabilities such as networking, filesystems, timers, DOM APIs,
cryptography, or process management.

A useful conceptual chain is:

``` text
JavaScript source
      ↓
ECMAScript language rules
      ↓
JavaScript engine implementation
      ↓
Host/runtime environment
      ↓
Application
```

This is a conceptual model, not a literal sequence of four separate
software processes.

The key idea is that **not everything developers call "JavaScript" is
specified by the ECMAScript language specification**.

For example:

``` js
const x = 10 + 20;
```

The language rules define values, operators, evaluation, and the
resulting value.

But:

``` js
fetch("https://example.com");
```

The `fetch` API is not simply "a built-in JavaScript language feature"
in the same sense as `+`, `const`, or function syntax. `fetch` belongs
to a host/platform API specification and implementation. In browsers, it
is exposed by the web platform. Other runtimes may provide a compatible
or different implementation.

Likewise:

``` js
console.log("hello");
```

`console` is host/platform functionality. Its availability and exact
behavior should not be confused with ECMAScript core semantics.

This distinction becomes essential later when debugging, optimizing,
securing, and designing portable systems.

------------------------------------------------------------------------

# 4. Why Does ECMAScript Exist?

JavaScript originally emerged in web browsers during a period when
different vendors controlled different implementations.

A language used across different implementations needs a common
definition.

Without standardization, this situation becomes possible:

``` text
Browser A:
  let x = ...
  behavior = one thing

Browser B:
  let x = ...
  behavior = another thing

Browser C:
  let x = ...
  behavior = a third thing
```

A standard provides a shared contract.

ECMAScript is the standardized language specification for JavaScript.

The standardized name is **ECMAScript**. The term **JavaScript** remains
the common name developers use for the language and ecosystem.

The distinction is therefore approximately:

``` text
JavaScript
  = common name used for the language and ecosystem

ECMAScript
  = standardized language specification
```

That does **not** mean JavaScript and ECMAScript are two unrelated
languages.

A better mental model is:

``` text
"JavaScript"        ← language as commonly named
       │
       └── standardized through ECMAScript
```

Historically, the language grew through implementations and
standardization rather than appearing as a completely finished
specification first.

------------------------------------------------------------------------

# 5. Historical Foundation

JavaScript was created by Brendan Eich at Netscape and first appeared in
Netscape Navigator.

Microsoft also implemented a compatible scripting language known as
JScript.

The existence of multiple implementations made standardization
important.

The first edition of the ECMAScript standard was adopted by Ecma
International in June 1997.

Over time, ECMAScript became the formal standard describing the
language.

The development history matters because many "JavaScript oddities" make
more sense once you understand that the language evolved while
preserving compatibility with existing software.

Backward compatibility is one of the strongest forces shaping
JavaScript.

That is why modern JavaScript contains:

-   modern lexical declarations such as `let` and `const`,
-   legacy constructs such as `var`,
-   old coercion behavior,
-   historical object semantics,
-   newer abstractions layered onto older mechanisms.

A language with a long deployed history cannot casually remove behavior
that existing applications depend on.

------------------------------------------------------------------------

# 6. ECMAScript and ECMA-262

The main ECMAScript language specification is **ECMA-262**.

The specification defines the language itself.

At a high level, it defines areas such as:

-   lexical grammar,
-   syntactic grammar,
-   language types,
-   values,
-   operators,
-   functions,
-   objects,
-   classes,
-   prototypes,
-   built-in objects,
-   iterators,
-   promises,
-   execution contexts,
-   environments,
-   jobs,
-   realms,
-   agents,
-   internal methods,
-   abstract operations,
-   and other language semantics.

The specification is not a beginner tutorial.

It is a formal standard.

A simplified distinction is:

``` text
Tutorial:
  "Use Array.prototype.map like this..."

Specification:
  "Given this abstract state and these conditions,
   perform these defined semantic steps..."
```

The specification is designed so independent implementers can produce
compatible implementations.

------------------------------------------------------------------------

# 7. What ECMAScript Does NOT Define

This is one of the most important concepts in the entire curriculum.

ECMAScript does not define every API that happens to be available when
you write JavaScript.

For example, the following are generally host/platform concerns rather
than core ECMAScript language syntax:

``` js
document.querySelector("#app");
window.location.href;
fetch("/api/users");
setTimeout(fn, 1000);
process.exit(1);
fs.readFile(...);
```

The exact APIs and their semantics come from additional specifications
and runtime implementations.

So when you ask:

> "Is this behavior part of JavaScript?"

the correct engineering response is often:

> "Which layer are you asking about?"

Possible layers include:

1.  ECMAScript language semantics.
2.  A JavaScript engine.
3.  A host/runtime specification.
4.  A runtime's own APIs.
5.  A library or framework.
6.  Application code.

------------------------------------------------------------------------

# 8. The Language / Engine / Runtime / Host Distinction

## 8.1 Language

The language is the set of syntax and semantic rules that define what
JavaScript programs mean.

Examples:

``` js
let x = 10;
x + 5;
```

Questions such as:

-   What is `undefined`?
-   How does `===` work?
-   What does a function call do?
-   How does lexical scoping work?
-   What happens when an object property is read?

are primarily language questions.

------------------------------------------------------------------------

## 8.2 JavaScript Engine

A JavaScript engine is an implementation of ECMAScript.

Examples include:

-   V8
-   SpiderMonkey
-   JavaScriptCore

The engine takes JavaScript source and provides the machinery needed to
execute it.

Very roughly, an engine may involve:

``` text
Source text
   ↓
Parsing
   ↓
Internal representation
   ↓
Interpreter / execution
   ↓
Optimization
   ↓
Machine code and runtime support
```

The exact pipeline differs by engine and version.

The language specification describes **what the program means**.

The engine decides **how to implement that meaning efficiently**.

This distinction is fundamental.

For example, the ECMAScript specification does not require every engine
to use:

-   a particular bytecode format,
-   a particular JIT compiler,
-   hidden classes,
-   inline caches,
-   a specific garbage collector,
-   a specific machine-code optimization strategy.

Those are implementation techniques.

------------------------------------------------------------------------

## 8.3 Runtime

A runtime is the environment in which JavaScript code is executed
together with APIs and infrastructure.

Examples:

-   Browser environments
-   Node.js
-   Deno
-   Bun
-   Edge runtimes
-   Embedded JavaScript environments

A runtime typically combines some or all of:

``` text
JavaScript engine
+
host APIs
+
I/O facilities
+
timers/event mechanisms
+
module/package infrastructure
+
process/runtime lifecycle
+
security/isolation mechanisms
```

Therefore:

``` text
V8 ≠ Node.js
```

V8 is an engine.

Node.js is a runtime built around V8 plus Node-specific functionality
and other components.

Similarly:

``` text
JavaScriptCore ≠ Safari
```

The engine is one major component of a larger host/runtime architecture.

------------------------------------------------------------------------

# 9. Host Environment

A host environment provides capabilities surrounding the JavaScript
language.

In a browser, the host may provide:

``` text
DOM
Fetch
Web Streams
Web Workers
Storage
Web Crypto
History
Clipboard
Timers
Rendering integration
```

In Node.js, the environment may provide:

``` text
filesystem
networking
process
TCP/HTTP APIs
streams
worker threads
child processes
buffers
diagnostics
```

Both can execute JavaScript while exposing very different capabilities.

This is why:

``` js
document.querySelector(...)
```

can work in a browser and fail in standard Node.js code when no
DOM-compatible environment is provided.

Likewise:

``` js
process.env.PORT
```

is a Node-style capability and is not a language primitive.

------------------------------------------------------------------------

# 10. A Four-Layer Mental Model

Use this model throughout the entire curriculum.

## Layer 1 --- ECMAScript

Defines language semantics.

Examples:

``` js
const x = 1;
x === 1;
function add(a, b) {
  return a + b;
}
```

## Layer 2 --- Engine

Implements ECMAScript.

Examples:

-   parsing strategy,
-   bytecode,
-   JIT compilation,
-   optimization,
-   garbage collection implementation,
-   internal object representation.

## Layer 3 --- Host / Runtime

Provides environmental capabilities.

Examples:

``` js
fetch(...)
setTimeout(...)
document.querySelector(...)
process.nextTick(...)
fs.readFile(...)
```

Exact availability depends on the environment.

## Layer 4 --- Application

Your own code and architecture.

Examples:

``` text
controllers
services
repositories
queues
business rules
authentication
caching
database integration
```

A debugging investigation should ask which layer is responsible.

------------------------------------------------------------------------

# 11. The Specification Is a Contract, Not an Implementation

A very common beginner misconception is:

> "The ECMAScript specification is the source code used by the
> JavaScript engine."

It is not.

The specification is a formal description of language behavior.

Consider:

``` js
1 + 2
```

The specification can describe how the operands are evaluated,
converted, and combined.

An engine may internally perform these operations using optimized
machine instructions.

The implementation might effectively recognize that the operands are
small numeric values and use a very fast path.

Both can conform to the same language semantics.

Think of it like this:

``` text
Specification
    ↓
Defines observable semantic requirements

Implementation
    ↓
Chooses internal mechanisms that satisfy those requirements
```

This separation allows implementation innovation without changing
language meaning.

------------------------------------------------------------------------

# 12. Observable Behavior vs Internal Implementation

A useful engineering boundary is:

### Observable behavior

Behavior that a program can legitimately depend on because the relevant
specification guarantees it.

Examples include:

-   the result of a defined language operation,
-   whether an exception is thrown under specified conditions,
-   property enumeration ordering where specified,
-   Promise semantics defined by the language/host standards.

### Implementation detail

A mechanism an engine may change without changing language semantics.

Examples can include:

-   specific JIT tiers,
-   internal machine-code layout,
-   exact garbage-collector scheduling,
-   object memory layout,
-   internal caches,
-   particular bytecode instructions.

A production engineer must not confuse:

``` text
"I observed this in V8"
```

with:

``` text
"ECMAScript guarantees this"
```

The first is empirical implementation knowledge.

The second is a language-level claim.

They require different evidence.

------------------------------------------------------------------------

# 13. TC39

**TC39** is Ecma International's Technical Committee responsible for
evolving ECMAScript.

TC39 includes language implementers, developers, academics, and other
participants who collaborate on the language standard.

The committee works on:

-   language evolution,
-   proposals,
-   specification changes,
-   compatibility,
-   implementation experience,
-   tests,
-   and other standardization work.

TC39 is important because JavaScript is a living language.

The language does not stop evolving when a yearly edition is published.

------------------------------------------------------------------------

# 14. ECMAScript Editions and the Living Specification

ECMAScript has historically been published in yearly editions.

For example:

``` text
ECMAScript 2024
ECMAScript 2025
...
```

The formal standardized edition is a snapshot.

The TC39 specification site also maintains a current living
specification that reflects the latest specification state and completed
additions.

As of September 8, 2026:

-   ECMAScript 2025 is the latest formally published edition listed by
    Ecma International.
-   The official TC39 site provides a living specification reflecting
    newer completed work and ongoing maintenance.
-   Draft specifications for future editions exist and should not be
    confused with already-published editions.

This creates an important distinction:

``` text
Published edition
    ≠
Current living specification
    ≠
Future draft
```

When writing production documentation, always identify which one you
mean.

------------------------------------------------------------------------

# 15. Published Standard vs Living Spec vs Draft

Consider three statements:

### Statement A

> "ECMAScript 2025 defines feature X."

This is a claim about a published standard edition.

### Statement B

> "The current ECMA-262 living specification defines feature X."

This is a claim about the current specification state.

### Statement C

> "The 2027 draft includes feature X."

This is a claim about a draft and does not mean that the feature is
already standardized.

These distinctions matter when designing compatibility matrices,
libraries, runtimes, and educational material.

------------------------------------------------------------------------

# 16. TC39 Proposal Stages

TC39 proposals move through a staged process.

The commonly used progression is:

``` text
Stage 0
   ↓
Stage 1
   ↓
Stage 2
   ↓
Stage 2.7
   ↓
Stage 3
   ↓
Stage 4
```

The stages are signals about proposal maturity, implementation
experience, and specification readiness.

The exact status of an individual proposal must always be checked from
the current TC39 proposal information.

------------------------------------------------------------------------

# 17. Stage 0

Stage 0 represents an idea or proposal that is not yet a formal mature
language proposal.

A Stage 0 idea may communicate:

-   a problem,
-   a possible syntax,
-   an API concept,
-   an architectural direction.

At this stage, major details can change.

A Stage 0 idea should **not** be treated as future JavaScript that is
guaranteed to arrive.

------------------------------------------------------------------------

# 18. Stage 1

Stage 1 means the committee is exploring the problem and potential
solution more seriously.

The proposal may include:

-   motivation,
-   examples,
-   semantics,
-   an initial specification direction.

However, substantial redesign is still possible.

Engineering rule:

> Stage 1 is not a compatibility guarantee.

------------------------------------------------------------------------

# 19. Stage 2

Stage 2 means the idea has advanced significantly and typically has a
more developed specification model.

Developers may begin experimenting seriously.

But:

``` text
Stage 2 ≠ Standard
```

A feature can still change or fail to progress.

------------------------------------------------------------------------

# 20. Stage 2.7

The current TC39 process includes Stage 2.7 as a proposal checkpoint.

This stage reflects a proposal that has advanced beyond Stage 2 and has
substantial specification/implementation maturity, while still not being
Stage 3.

It is especially important for learners reading modern TC39 materials
because older tutorials may describe the process using only Stages 0--4
and omit the modern 2.7 checkpoint.

------------------------------------------------------------------------

# 21. Stage 3

Stage 3 means the proposal is considered close to completion and is
intended to receive implementation and ecosystem feedback.

This is often where runtime implementations, compatibility testing, and
real-world feedback become especially important.

However:

``` text
Stage 3 ≠ Final standard
```

A production dependency should not automatically treat every Stage 3
proposal as universally available.

------------------------------------------------------------------------

# 22. Stage 4

Stage 4 represents completion.

A Stage 4 proposal is considered finished and is incorporated into the
standardization cycle.

The current TC39 specification describes completed proposals as those
that reached Stage 4 and are implemented in several implementations,
after which they are included in the next yearly snapshot.

This is the point at which it becomes appropriate to describe a feature
as standardized rather than merely proposed.

Even then, implementation availability can still differ across
engine/runtime versions.

Therefore:

``` text
Stage 4
   ↓
Standardized
   ↓
Check runtime/version support for deployment
```

Standardization and universal runtime availability are separate
questions.

------------------------------------------------------------------------

# 23. The Most Important Proposal Rule

Never write:

> "Feature X is JavaScript."

until you know what you mean.

Instead ask:

1.  Is it in the ECMAScript standard?
2.  Which edition?
3.  Is it only in the living specification?
4.  Is it a proposal?
5.  Which stage?
6.  Which engines implement it?
7.  Which runtime versions expose it?
8.  Does the host environment affect availability?

This habit prevents a large class of technical misinformation.

------------------------------------------------------------------------

# 24. JavaScript Engines

A JavaScript engine implements ECMAScript semantics.

Major engines include:

### V8

Used by environments such as:

-   Chrome/Chromium-based browsers,
-   Node.js,
-   Deno,
-   Bun components and integrations in its ecosystem.

V8 is developed by Google.

### SpiderMonkey

Mozilla's JavaScript engine, used by Firefox.

### JavaScriptCore

Apple's JavaScript engine, used by WebKit-based environments such as
Safari.

There are additional engines and embedded implementations beyond these.

The critical lesson is not memorizing brand names.

It is understanding:

``` text
Different engines can implement the same language.
```

This is what enables JavaScript portability.

------------------------------------------------------------------------

# 25. Why Multiple Engines Matter

Suppose ECMAScript specifies:

``` js
const value = someOperation();
```

Different engines can have completely different internals while
producing the same specified result.

One engine may:

``` text
parse → bytecode → optimize → machine code
```

Another may:

``` text
parse → different IR → different optimization pipeline
```

Another embedded engine may prioritize:

``` text
small memory footprint → interpreter-heavy execution
```

The language contract remains the same where the behavior is
standardized.

This separation is a foundational example of interface vs
implementation.

------------------------------------------------------------------------

# 26. V8 Is Not "JavaScript"

A common statement is:

> "JavaScript uses V8."

That is too broad.

A more precise statement is:

> "Some JavaScript environments use V8 as their ECMAScript engine."

For example:

``` text
Node.js
  └── V8
      └── ECMAScript implementation
```

But Node.js also adds functionality outside V8.

Therefore this:

``` js
require("fs");
```

is not simply "V8 functionality."

Node provides the environment-level feature.

This distinction becomes crucial in backend development.

------------------------------------------------------------------------

# 27. Browser JavaScript

A browser generally combines:

``` text
Browser
 ├── JavaScript engine
 ├── DOM implementation
 ├── Web APIs
 ├── networking stack
 ├── rendering engine
 ├── event/event-loop infrastructure
 └── security/isolation model
```

So a browser is far more than a JavaScript engine.

For example:

``` js
document.body.textContent = "Hello";
```

The following concerns are involved:

-   JavaScript language evaluation,
-   host-provided DOM objects,
-   browser document state,
-   browser rendering integration.

A JavaScript engine alone does not need to contain an HTML DOM.

------------------------------------------------------------------------

# 28. Node.js

Node.js combines JavaScript execution with server-side and
systems-oriented capabilities.

A simplified model is:

``` text
Node.js
 ├── V8
 ├── Node runtime APIs
 ├── libuv
 ├── filesystem integration
 ├── networking
 ├── streams
 ├── process management
 ├── worker threads
 └── diagnostics/tooling
```

This is why Node.js applications can perform operations that browser
JavaScript traditionally cannot perform directly:

``` js
import fs from "node:fs/promises";

const content = await fs.readFile("data.txt", "utf8");
```

The language mechanism `await` belongs to ECMAScript.

The filesystem API belongs to Node.js.

This is a perfect example of cross-layer cooperation.

------------------------------------------------------------------------

# 29. Deno, Bun, and Other Runtimes

Different runtimes can implement the same ECMAScript language while
making different ecosystem and runtime choices.

They may differ in:

-   APIs,
-   module systems,
-   package support,
-   permissions,
-   tooling,
-   networking,
-   filesystem behavior,
-   startup characteristics,
-   compatibility,
-   runtime internals.

Therefore:

``` text
ECMAScript compatibility
        ≠
complete runtime compatibility
```

Two environments can both be JavaScript runtimes and still differ
substantially.

------------------------------------------------------------------------

# 30. Edge Runtimes

Modern edge environments often expose a programming model closer to web
platform APIs and isolate-based execution.

An edge runtime may:

-   provide `fetch`,
-   provide Web Streams,
-   restrict filesystem access,
-   restrict native modules,
-   limit long-running processes,
-   enforce resource quotas,
-   use worker-style isolation.

Therefore, code written for Node.js may not run unchanged at the edge
even if both execute JavaScript.

Again:

``` text
Same language
        ≠
same host APIs
```

------------------------------------------------------------------------

# 31. Runtime Portability

When moving JavaScript code between environments, test at three levels.

## Language portability

Does the code use standardized ECMAScript behavior?

Example:

``` js
const x = 1;
const y = x + 2;
```

Usually highly portable.

## API portability

Does the environment provide the APIs the code uses?

Example:

``` js
fetch(...)
```

Different environments may provide it with different support details.

## Operational portability

Even if the code runs, does the environment provide the same:

-   performance,
-   memory budget,
-   filesystem,
-   network model,
-   lifecycle,
-   concurrency behavior,
-   security model?

A system can be syntactically portable while operationally incompatible.

------------------------------------------------------------------------

# 32. Specification Hierarchy

Use the following source hierarchy when investigating behavior:

``` text
1. ECMAScript specification
        ↓
2. JavaScript engine documentation / implementation
        ↓
3. Host/runtime documentation
        ↓
4. Framework/library documentation
        ↓
5. Application code
```

The correct source depends on the question.

For:

> "How does `===` compare values?"

Start with ECMAScript.

For:

> "Why did V8 deoptimize this function?"

You may need V8 implementation information.

For:

> "Why does this API fail in Node?"

Check Node/runtime documentation.

For:

> "Why does our endpoint return stale data?"

Investigate the application, cache, database, and runtime layers.

------------------------------------------------------------------------

# 33. Normative vs Informative Material

Standards often distinguish normative requirements from explanatory
material.

### Normative

Defines requirements that implementations are expected to follow.

### Informative

Provides explanation, examples, or context.

This distinction becomes important when reading ECMA-262.

Do not treat every sentence in a standard as equivalent to executable
algorithmic requirements.

The specification also uses formal notation and abstract operations.

------------------------------------------------------------------------

# 34. Why the Specification Uses Abstract Operations

The specification needs a language-independent way to describe
semantics.

Instead of saying:

> "Run this C++ function..."

it describes conceptual operations such as:

``` text
ToPrimitive
ToNumber
Get
Set
Call
Construct
SameValue
HasProperty
```

These are **specification-level concepts**.

An engine may implement them differently.

For example, later chapters will explain that a simple expression may
conceptually invoke conversion steps such as `ToPrimitive`, even though
an optimized engine does not literally call a function with that name at
runtime.

This distinction is critical:

``` text
Spec algorithm
   ≠
literal engine function call
```

------------------------------------------------------------------------

# 35. Specification Terminology You Will Encounter Later

This chapter introduces the vocabulary before the deep chapters.

You will eventually encounter:

### Internal slots

Conceptual state associated with specification objects.

Examples can include things like:

``` text
[[Prototype]]
[[PromiseState]]
```

The double-bracket notation indicates specification-level internal
machinery.

### Internal methods

Conceptual operations used to manipulate objects.

Examples:

``` text
[[Get]]
[[Set]]
[[Call]]
[[Construct]]
```

### Abstract operations

Reusable specification algorithms.

Examples:

``` text
ToNumber
ToString
ToObject
Get
Set
Call
Construct
```

### Execution contexts

Specification-level representation of currently executing code.

### Jobs

Units of work used in ECMAScript's execution model.

These ideas will be developed deeply in later chapters.

------------------------------------------------------------------------

# 36. A Crucial Principle: Do Not Anthropomorphize the Engine

Beginners often say:

> "JavaScript sees that this is an object."

or:

> "JavaScript stores this in the stack."

These statements can be useful informally, but they can also become
misleading.

A stronger engineering vocabulary is:

``` text
The language semantics require...
The host provides...
The engine may represent...
The implementation can optimize...
```

This keeps standardized guarantees separate from implementation
assumptions.

------------------------------------------------------------------------

# 37. Why "JavaScript Is Interpreted" Is an Incomplete Statement

You will hear:

> "JavaScript is an interpreted language."

That is an oversimplification.

Modern JavaScript engines can combine:

-   parsing,
-   interpretation,
-   baseline compilation,
-   optimizing compilation,
-   deoptimization,
-   native runtime calls,
-   other adaptive techniques.

Different engines use different strategies.

Therefore the useful statement is:

> JavaScript is specified as a language, while engines choose
> implementation strategies that may include interpretation and
> compilation.

The language specification does not require one universal execution
strategy.

This becomes important in the performance chapters.

------------------------------------------------------------------------

# 38. Why "Node Is Single-Threaded" Is Also Incomplete

Node.js applications often execute JavaScript on a main thread, but the
runtime can involve other threads and concurrent infrastructure.

Later chapters will distinguish:

``` text
JavaScript execution model
        vs
runtime I/O implementation
        vs
worker threads
        vs
processes
        vs
OS-level concurrency
```

This distinction prevents another common myth.

The correct mental model will be developed in the asynchronous and
Node.js sections.

------------------------------------------------------------------------

# 39. Running the Same Code in Different Hosts

Consider:

``` js
const value = 42;

console.log(value);
```

The core language semantics for `const`, numeric values, and function
invocation are ECMAScript territory.

But `console.log` availability and behavior depend on the host/runtime.

Now consider:

``` js
const button = document.querySelector("button");
```

The JavaScript syntax is language-level.

`document` and `querySelector` are host/platform functionality.

Now consider:

``` js
import fs from "node:fs";
```

The module syntax belongs to ECMAScript and the broader module
ecosystem, while the `node:` namespace and filesystem API are
Node-specific.

Same language.

Different environment.

------------------------------------------------------------------------

# 40. A Practical Investigation Method

When JavaScript behaves unexpectedly, use this sequence.

### Step 1 --- Is this language behavior?

Examples:

-   coercion,
-   scope,
-   function calls,
-   property lookup,
-   equality,
-   promise semantics.

Check ECMAScript semantics.

### Step 2 --- Is this engine behavior?

Examples:

-   optimization,
-   deoptimization,
-   memory usage,
-   GC characteristics,
-   performance cliffs.

Check engine-specific material.

### Step 3 --- Is this host behavior?

Examples:

-   DOM,
-   file access,
-   timers,
-   networking,
-   process APIs.

Check host/runtime documentation.

### Step 4 --- Is this framework/library behavior?

Examples:

-   React scheduling,
-   Express middleware,
-   NestJS lifecycle,
-   ORM behavior.

Check the relevant project documentation/source.

### Step 5 --- Is this application behavior?

Examples:

-   race conditions,
-   state bugs,
-   incorrect error handling,
-   cache invalidation.

Inspect your own code.

This process dramatically improves debugging accuracy.

------------------------------------------------------------------------

# 41. Common Misconception: ECMAScript = Browser JavaScript

False.

A browser is a host environment containing a JavaScript engine and a
large platform.

ECMAScript is the language standard.

A browser adds APIs and behavior outside the core language.

Therefore:

``` text
Browser JavaScript
=
ECMAScript
+
Web Platform
+
Browser-specific implementation details
```

The exact composition varies by browser and specification.

------------------------------------------------------------------------

# 42. Common Misconception: Node.js Is a JavaScript Engine

False.

Node.js is a runtime environment.

V8 is the engine used by Node.js.

Conceptually:

``` text
Node.js
  └── JavaScript engine
       └── ECMAScript implementation
```

Plus Node-specific runtime components.

------------------------------------------------------------------------

# 43. Common Misconception: If Chrome Supports a Feature, JavaScript Supports It Everywhere

False.

Chrome may support:

-   a standardized feature,
-   a Stage 3 proposal,
-   an experimental feature,
-   a Chrome-specific API,
-   a web platform feature not available elsewhere.

Always identify the standardization layer and compatibility scope.

------------------------------------------------------------------------

# 44. Common Misconception: Stage 3 Means Standard

False.

Stage 3 means the proposal is close to completion but is not yet Stage
4.

Production systems must consider actual runtime support and
compatibility regardless of proposal stage.

------------------------------------------------------------------------

# 45. Common Misconception: Stage 4 Means Every Runtime Has It

False.

Stage 4 means standardized.

Runtime support can still depend on:

-   engine version,
-   runtime version,
-   browser version,
-   configuration,
-   deployment environment.

Therefore:

``` text
Standardized
   ≠
universally deployed
```

------------------------------------------------------------------------

# 46. Common Misconception: The ECMAScript Specification Tells Engines Exactly How to Implement JavaScript

False.

The specification defines language semantics.

Implementations have freedom in internal architecture as long as
externally observable behavior conforms to the relevant requirements.

This separation is what makes implementation innovation possible.

------------------------------------------------------------------------

# 47. Common Mistakes

### Mistake 1 --- Treating host APIs as language primitives

Wrong mental model:

``` text
fetch = JavaScript keyword
```

Better:

``` text
fetch = host/platform API
```

### Mistake 2 --- Treating V8 behavior as universal JavaScript behavior

Observation:

``` text
V8 did X
```

does not automatically establish:

``` text
ECMAScript requires X
```

### Mistake 3 --- Reading old tutorials as current standard truth

JavaScript evolves.

Old explanations may describe:

-   old proposal stages,
-   old browser support,
-   obsolete engine behavior,
-   historical workarounds.

For modern language status, consult current standards sources.

### Mistake 4 --- Mixing runtime documentation with language semantics

Node documentation can tell you how Node exposes an API.

It is not the authority for the ECMAScript definition of `ToPrimitive`,
`Promise`, or `Array.prototype.map`.

------------------------------------------------------------------------

# 48. Performance Perspective

Performance questions must also be layered.

Suppose this code is slow:

``` js
function add(a, b) {
  return a + b;
}
```

Potential explanations can exist at different levels:

### Language semantics

The operator has specified semantic requirements.

### Engine implementation

The engine may optimize repeated numeric operations.

### Application behavior

Perhaps the function is called millions of times inside a larger
algorithm.

### System behavior

Perhaps serialization, networking, database latency, or memory pressure
dominates the actual workload.

Therefore:

> Never optimize a language-level micro-detail before measuring the
> actual system cost.

This principle will return repeatedly in the performance and
architecture chapters.

------------------------------------------------------------------------

# 49. Security Perspective

The same layering applies to security.

Example:

``` js
const code = userInput;
eval(code);
```

The language exposes `eval`.

But a real security analysis also involves:

-   application trust boundaries,
-   input origin,
-   sandboxing,
-   host capabilities,
-   process isolation,
-   permissions,
-   dependency behavior.

Security is therefore not solved by memorizing JavaScript syntax.

You must understand the runtime boundary.

------------------------------------------------------------------------

# 50. Production Engineering Perspective

Suppose a backend is deployed on Node.js.

You need to know at least:

``` text
JavaScript language semantics
        ↓
V8 implementation characteristics
        ↓
Node runtime APIs
        ↓
Operating system behavior
        ↓
Database/network dependencies
        ↓
Your application architecture
```

A senior engineer asks:

> "Which layer owns this behavior?"

A principal engineer additionally asks:

> "Which layer's guarantee am I depending on, and what happens if that
> layer changes?"

That is the beginning of engineering judgment.

------------------------------------------------------------------------

# 51. Implementation Exercise --- Build the Layer Map

Create a document or whiteboard with these rows:

  --------------------------------------------------------------------------
  Feature         ECMAScript   Engine           Host/Runtime   Application
  --------------- ------------ ---------------- -------------- -------------
  `const`         ✓            implementation   ---            uses it

  `===`           ✓            implementation   ---            uses it

  `Promise`       ✓            implementation   may integrate  uses it
                                                scheduling     

  `document`      ---          ---              browser        uses it

  `fetch`         not core     implementation   web/runtime    uses it
                  language     participates     API            
                  syntax                                       

  `process`       ---          ---              Node.js        uses it

  `fs.readFile`   ---          ---              Node.js        uses it

  JIT             ---          ✓                runtime may    indirectly
  optimization                                  expose         affected
                                                diagnostics    
  --------------------------------------------------------------------------

Then add ten more examples from your own development work.

The goal is not memorization.

The goal is learning to classify behaviors correctly.

------------------------------------------------------------------------

# 52. Prediction Exercise

Before looking anything up, classify each item as primarily
language-level, host/runtime-level, engine-level, or application-level.

``` js
const x = 10;
```

``` js
x === 10;
```

``` js
fetch("/users");
```

``` js
process.memoryUsage();
```

``` js
document.body;
```

``` js
someFunction();
```

``` js
new Map();
```

``` js
v8.getHeapStatistics();
```

The important part is not only the classification.

The important part is explaining **why**.

------------------------------------------------------------------------

# 53. Code Review Exercise

A developer writes:

> "Node's `Array.prototype.map()` is faster because V8 uses optimized
> hidden classes."

Review this statement.

A strong review should challenge at least three assumptions:

1.  `Array.prototype.map` is an ECMAScript-defined method; Node is not
    the source of its core language semantics.
2.  Hidden classes are an implementation detail, particularly associated
    with engines such as V8; they are not an ECMAScript guarantee.
3.  Performance cannot be established from one implementation detail
    without measuring the actual workload.

A better engineering statement is:

> "The semantics of `Array.prototype.map` come from ECMAScript. V8 may
> apply implementation-specific optimizations that affect its observed
> performance. Benchmark the relevant workload before drawing
> conclusions."

That distinction is a model for technical writing throughout this
curriculum.

------------------------------------------------------------------------

# 54. Interview Questions

## Junior

1.  What is JavaScript?
2.  What is ECMAScript?
3.  Why does ECMAScript exist?
4.  What is a JavaScript engine?
5.  Name three JavaScript engines.

## Mid-Level

6.  Is Node.js a JavaScript engine?
7.  What is the difference between V8 and Node.js?
8.  Why can the same JavaScript language run in browsers and servers?
9.  What is a host environment?
10. Why is `document` not part of ECMAScript?

## Senior

11. How would you distinguish language semantics from engine behavior?
12. Why does specification knowledge matter when debugging?
13. Why is Stage 3 not the same as standardized?
14. What is the difference between a yearly ECMAScript edition and the
    living specification?
15. How do you investigate a behavior that differs across runtimes?

## Staff / Principal

16. How would you design a portability strategy for a JavaScript
    library?
17. How would you decide whether to rely on a runtime-specific
    optimization?
18. How would you document behavior that depends on V8 rather than
    ECMAScript?
19. How do language, host, and engine boundaries affect architecture?
20. What evidence would you require before calling a JavaScript behavior
    "guaranteed"?

------------------------------------------------------------------------

# 55. Predict-the-Output / Predict-the-Environment Exercises

These exercises are intentionally about classification rather than
arithmetic.

For each example, answer:

1.  Does the syntax belong to ECMAScript?
2.  Does the runtime need to provide anything additional?
3.  Would it run in a browser?
4.  Would it run in Node.js?
5.  Is the behavior standardized or runtime-specific?

### Exercise A

``` js
const users = ["A", "B"];
console.log(users.length);
```

### Exercise B

``` js
document.querySelector("#app");
```

### Exercise C

``` js
import fs from "node:fs";
```

### Exercise D

``` js
await Promise.resolve(42);
```

### Exercise E

``` js
process.env.NODE_ENV;
```

### Exercise F

``` js
new Uint8Array([1, 2, 3]);
```

Do not merely answer "browser" or "Node".

Explain each layer involved.

------------------------------------------------------------------------

# 56. Mastery Exercise --- Explain JavaScript in 60 Seconds

Without using the phrase "JavaScript is a language that runs in a
browser," explain JavaScript in approximately sixty seconds.

A strong answer should include:

``` text
JavaScript
→ standardized by ECMAScript
→ implemented by engines
→ executed inside hosts/runtimes
→ extended by host APIs
→ used by application code
```

Then explain one example involving:

``` text
fetch
```

and one involving:

``` text
process
```

------------------------------------------------------------------------

# 57. Mastery Exercise --- Investigate a Cross-Runtime Bug

Suppose:

``` js
const result = someFeature();
```

works in one runtime and fails in another.

Write an investigation plan containing:

``` text
1. Identify exact runtime versions.
2. Identify exact engine versions where possible.
3. Determine whether the feature is ECMAScript-standardized.
4. Check the relevant standard edition/living specification.
5. Check proposal status if not standardized.
6. Check runtime implementation status.
7. Determine whether host APIs are involved.
8. Reduce to a minimal reproducible example.
9. Compare observable behavior.
10. Document the compatibility requirement.
```

This workflow will become useful repeatedly when learning modern
ECMAScript features.

------------------------------------------------------------------------

# 58. Current Standards Snapshot

At the time this handbook chapter was written:

-   Ecma International lists **ECMA-262, 16th edition, June 2025**, as
    the published ECMAScript 2025 language specification.
-   The official TC39 specification site identifies its living
    ECMAScript specification as the most accurate and up-to-date
    specification and notes that it includes the latest yearly snapshot
    plus completed Stage 4 proposals.
-   TC39 currently exposes proposal stages including Stage 0, Stage 1,
    Stage 2, Stage 2.7, Stage 3, and Stage 4.
-   Future-dated specification drafts exist and must not be represented
    as already-published standards.

**Source references used for this status snapshot:**

-   Ecma International, ECMA-262 publication/archive information.
-   Ecma International, ECMA-262 16th edition / ECMAScript 2025.
-   TC39, current ECMAScript specification.
-   TC39, current proposal-stage information.

------------------------------------------------------------------------

# 59. Concept Connections

## Depends On

-   Basic programming concepts.
-   Source code and execution as general ideas.

## Builds Toward

-   Values and types.
-   Execution contexts.
-   Scope and lexical environments.
-   Objects and property semantics.
-   Async execution and jobs.
-   Engine internals.
-   Browser APIs.
-   Node.js architecture.
-   Modules and tooling.
-   Production architecture.

## Related Concepts

-   Standards
-   Compatibility
-   Portability
-   Runtime architecture
-   API boundaries
-   Specification vs implementation
-   Feature detection
-   Versioning

## Concepts Revisited Later

This chapter deliberately introduces concepts that later chapters will
explain deeply:

-   execution contexts,
-   jobs,
-   realms,
-   agents,
-   internal methods,
-   abstract operations,
-   garbage collection,
-   event loop,
-   host scheduling,
-   streams,
-   runtime diagnostics.

## Why This Chapter Matters Later

Without this chapter, later material easily becomes a collection of
isolated facts.

With this chapter, the learner should continually ask:

``` text
Who defines this?
Who implements this?
Who exposes this?
Who depends on this?
What is guaranteed?
What is merely observed?
```

That question is one of the most valuable habits in advanced JavaScript
engineering.

------------------------------------------------------------------------

# 60. Key Takeaways

1.  **JavaScript is the common name for the language; ECMAScript is its
    standardized language specification.**
2.  **ECMA-262 defines ECMAScript language semantics.**
3.  **A JavaScript engine implements ECMAScript.**
4.  **A runtime/host surrounds the engine with additional APIs and
    infrastructure.**
5.  **V8 is an engine; Node.js is a runtime built around an engine plus
    runtime capabilities.**
6.  **Browser APIs such as the DOM are not simply ECMAScript language
    primitives.**
7.  **Specification semantics are not the same thing as engine
    implementation details.**
8.  **TC39 evolves ECMAScript through a proposal process.**
9.  **Stage 3 is not Stage 4; proposal status must be verified.**
10. **Standardization does not guarantee universal runtime support.**
11. **A yearly ECMAScript edition is a published snapshot; the living
    specification can contain newer completed work.**
12. **When behavior is surprising, first identify which layer owns the
    behavior.**

------------------------------------------------------------------------

# 61. Completion Criteria

Chapter 01 is complete when the learner can, without notes:

### Understand

-   Define JavaScript, ECMAScript, engine, runtime, and host.
-   Explain ECMA-262.
-   Explain TC39.

### Explain

-   Explain why ECMAScript and JavaScript are related but not identical
    terms.
-   Explain browser JavaScript vs Node.js.
-   Explain specification vs implementation.

### Predict

-   Classify a behavior as language, engine, runtime, library, or
    application behavior.

### Implement / Apply

-   Build a language/runtime compatibility matrix for a real project.
-   Identify which APIs are portable and which are host-specific.

### Debug

-   Investigate cross-runtime differences using the source hierarchy.

### Compare

-   Compare V8, SpiderMonkey, and JavaScriptCore at an architectural
    level without confusing implementation details with language
    guarantees.

### Defend

-   Defend a technical claim about JavaScript using the correct
    authority:
    -   ECMAScript specification,
    -   engine documentation,
    -   runtime documentation,
    -   or application evidence.

### Mastery gate

``` text
Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend
```

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 02 --- Values, Types, and the
JavaScript Type System