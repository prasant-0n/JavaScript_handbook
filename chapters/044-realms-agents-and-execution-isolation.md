# Chapter 44 — Realms, Agents, and Execution Isolation

## Chapter Metadata

```text
Chapter: 44
Title: Realms, Agents, and Execution Isolation
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what an ECMAScript Realm is.
- Explain what an ECMAScript Agent is.
- Distinguish a Realm from an Agent.
- Distinguish a Realm from an OS process.
- Distinguish an Agent from a thread in implementation terminology.
- Explain the relationship among:
  - Realm;
  - Agent;
  - Agent Cluster;
  - execution context;
  - global environment;
  - global object;
  - intrinsics.
- Explain why JavaScript objects from different Realms are not simply interchangeable in every identity check.
- Understand that each Realm has its own set of intrinsic objects.
- Explain why:

```js
value instanceof Array
```

can behave unexpectedly across Realm boundaries.
- Explain how constructors and prototypes differ across Realms.
- Understand the purpose of `globalThis`.
- Explain why `globalThis` can differ by Realm.
- Explain the purpose of `CreateRealm`.
- Explain Realm initialization conceptually.
- Understand Realm execution context association.
- Explain what an Agent represents at the ECMAScript specification level.
- Understand the concept of an execution agent that can execute jobs/code.
- Understand how multiple Agents can support concurrent/parallel execution.
- Understand that Realms and Agents solve different problems.
- Explain how Web Workers relate conceptually to Agents and host-defined execution environments.
- Explain how Node.js worker threads relate conceptually to Agents.
- Explain what shared memory changes when multiple Agents interact through SharedArrayBuffer/Atomics.
- Understand:
  - data cloning;
  - transferable data;
  - shared memory;
  - message passing;
  - identity boundaries.
- Explain why ordinary object identity does not automatically cross execution boundaries.
- Understand the isolation properties of separate Realms.
- Understand the security significance of Realm boundaries.
- Understand prototype and intrinsic separation.
- Explain why built-ins from one Realm are not necessarily identical to built-ins from another Realm.
- Understand cross-Realm brand checks conceptually.
- Understand why methods such as `Array.isArray` can be more robust across Realms than `instanceof`.
- Explain `instanceof` prototype-chain dependence.
- Explain constructor identity across Realms.
- Understand why `Object.prototype` differs by Realm.
- Understand how intrinsics are initialized.
- Explain the relationship between a Realm's global object and global environment.
- Explain host-defined Realm creation.
- Understand that ECMAScript does not specify every browser/Node process isolation detail.
- Implement a simplified Realm model.
- Implement a teaching Agent model.
- Simulate cross-Realm object boundaries.
- Simulate isolated intrinsics.
- Build cross-Realm tests.
- Diagnose identity/brand/prototype issues across execution contexts.
- Reason about shared memory races at the Agent boundary.
- Evaluate data-transfer strategies using correctness, memory, performance, security, and ownership criteria.
- Explain the architectural trade-offs between:
  - same Realm;
  - separate Realms;
  - separate Agents;
  - worker threads;
  - child processes;
  - external processes/services.
- Defend Realm and Agent decisions at principal-engineer depth.

### Mastery Gate

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

Required:

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations
- Chapter 43 — Ordinary Object Internal Methods

Strongly related:

- Chapter 25 — Iterables / Iterators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 37 — Cancellation / Abort
- Chapter 39 — Concurrency / Parallelism

Builds toward:

- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 95 — Legacy JavaScript
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 97 — Edge / Serverless JavaScript
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

---

## 3. What Is It?

### Realm

A **Realm** is a specification-level execution environment containing a set of intrinsic objects, a global object, a global environment, and associated execution state.

A useful mental model:

```text
Realm
├── Intrinsics
├── Global Object
├── Global Environment
├── GlobalThis
└── Realm-specific execution configuration/state
```

Another Realm can have its own:

```text
Object
Array
Function
Promise
Map
Set
Error
Object.prototype
Array.prototype
...
```

These are distinct object identities from those of another Realm.

### Agent

An **Agent** is a specification-level execution entity capable of evaluating ECMAScript code and processing jobs.

A simplified model:

```text
Agent
├── execution state
├── job processing
├── execution contexts
└── associated Realms
```

The important distinction is:

```text
Realm → environment of intrinsics/global state
Agent → execution entity that runs code/jobs
```

One Agent can be associated with multiple Realms.

Multiple Agents can support concurrent execution.

---

## 4. Why Does It Exist?

JavaScript needs to model both:

```text
which global environment?
```

and:

```text
which independent execution entity?
```

These are not the same question.

A browser can contain:

```text
main page Realm
iframe Realm
worker Agent / Realm
other execution contexts
```

The host needs to reason about:

- global objects;
- constructors;
- prototypes;
- execution isolation;
- concurrency;
- shared memory;
- message passing.

Realms provide isolation of language-level intrinsic state.

Agents provide a model for execution and concurrency.

The distinction becomes critical when reasoning about:

```js
Object
Array
Function
Promise
instanceof
Array.isArray
globalThis
SharedArrayBuffer
Atomics
Worker
```

---

## 5. Mental Model

Think of the relationship as:

```text
                    Host
                     │
          ┌──────────┴──────────┐
          │                     │
       Agent A               Agent B
          │                     │
      ┌───┴───┐               Realm C
      │       │
   Realm A  Realm B
```

A more concrete model:

```text
Agent
  │
  ├── Realm 1
  │     ├── global object
  │     ├── Object
  │     ├── Array
  │     └── Promise
  │
  └── Realm 2
        ├── global object
        ├── Object
        ├── Array
        └── Promise
```

The two Realms can have distinct intrinsic constructors.

Then:

```text
Agent 1
  │
  └── Realm A

Agent 2
  │
  └── Realm B
```

can execute independently.

The key principle:

> Realm answers “which language environment and intrinsic set?” Agent answers “which execution entity is running the code?”

---

## 6. Core Rules

### Rule 1 — Realm and Agent are different abstractions

Do not collapse them into one concept.

### Rule 2 — A Realm has its own intrinsics

Its `Object`, `Array`, `Function`, etc. are Realm-specific objects.

### Rule 3 — `globalThis` is Realm-specific

It identifies the global object/value exposed through the active Realm's global environment.

### Rule 4 — Constructor identity is Realm-specific

Two Realms can have distinct:

```js
Array
```

constructors.

### Rule 5 — Prototype identity is Realm-specific

Each Realm has its own intrinsic prototype objects.

### Rule 6 — `instanceof` depends on constructor/prototype relationships

Cross-Realm objects can therefore produce surprising results.

### Rule 7 — `Array.isArray` is based on Array semantics rather than a simple constructor identity comparison

This makes it more reliable for cross-Realm arrays.

### Rule 8 — One Agent can be associated with multiple Realms

Realm and execution entity are orthogonal concepts.

### Rule 9 — Multiple Agents can execute independently

This is the basis for concurrent execution at the ECMAScript semantic level.

### Rule 10 — Separate Agents do not automatically share normal object state

Shared memory requires explicit mechanisms.

### Rule 11 — Message passing and shared memory have different semantics

Cloning, transfer, and sharing are separate architectural choices.

### Rule 12 — Shared memory introduces data races

When multiple Agents access shared mutable memory, synchronization is required.

### Rule 13 — Realm separation is not equivalent to process isolation

A Realm is a language-level environment.

### Rule 14 — Worker isolation is host/runtime behavior built around language execution

Browser and Node worker mechanisms require host/runtime semantics beyond ECMAScript alone.

### Rule 15 — Intrinsic separation affects security and correctness

Do not assume:

```text
same constructor name
→ same constructor identity
```

### Rule 16 — Cross-Realm identity is not ordinary same-Realm identity

An object from one Realm is still an object, but its intrinsic/prototype relationships may belong to that Realm.

---

## 7. Syntax

Realm creation is generally host-invoked rather than ordinary source syntax.

The specification models operations conceptually such as:

```text
CreateRealm()
SetRealmGlobalObject(...)
InitializeHostDefinedRealm()
```

Source-level mechanisms that can expose Realm-like boundaries include host constructs such as:

```js
iframe.contentWindow
iframe.contentDocument
Worker
```

or runtime APIs that create separate execution environments.

Cross-Realm examples:

```js
const iframe = document.createElement("iframe");
document.body.appendChild(iframe);

const otherArray = new iframe.contentWindow.Array(1, 2, 3);
```

Worker-style isolation:

```js
const worker = new Worker("worker.js");
```

The exact isolation and lifetime semantics depend on the host.

---

## 8. Basic Examples

### Example 1 — Same Realm constructor identity

```js
const a = [];
const b = new Array();

console.log(a instanceof Array);
console.log(b instanceof Array);
```

Both use the active Realm's `Array` constructor/prototype relationship.

### Example 2 — Cross-Realm constructor identity

Conceptually:

```js
const otherArray = new iframe.contentWindow.Array(1, 2, 3);

console.log(otherArray instanceof Array);
```

This can be false because:

```text
otherArray
→ prototype from Realm B

Array
→ constructor from Realm A
```

### Example 3 — Cross-Realm array detection

```js
Array.isArray(otherArray);
```

This can correctly identify the array despite constructor identity differences.

### Example 4 — Realm-specific Object constructor

Conceptually:

```js
const OtherObject = iframe.contentWindow.Object;

console.log(Object === OtherObject);
```

These constructors are distinct across Realms.

### Example 5 — Prototype identity

```js
console.log(
  Object.getPrototypeOf(otherArray) === Array.prototype
);
```

The result can be false because the prototype belongs to another Realm.

### Example 6 — Global object

```js
console.log(globalThis);
```

`globalThis` reflects the active global environment.

---

## 9. Execution Walkthrough

Consider a main page Realm and an iframe Realm.

```text
Realm A
  Object_A
  Array_A
  Array.prototype_A

Realm B
  Object_B
  Array_B
  Array.prototype_B
```

Now:

```js
const foreignArray = new Array_B(1, 2, 3);
```

### Step 1

The iframe Realm provides its own `Array_B`.

### Step 2

Construction creates an object whose semantic relationships are based on Realm B's intrinsic objects.

### Step 3

The resulting object is passed to Realm A.

### Step 4

Realm A evaluates:

```js
foreignArray instanceof Array_A
```

### Step 5

The `instanceof` algorithm uses the prototype relationship associated with `Array_A`.

### Step 6

The foreign object's prototype chain is based on Realm B:

```text
foreignArray
→ Array.prototype_B
→ Object.prototype_B
→ null
```

### Step 7

Realm A's:

```text
Array.prototype_A
```

is a different object.

### Step 8

The identity/prototype test therefore does not establish the relationship:

```text
foreignArray → Array.prototype_A
```

### Result

The `instanceof` outcome can be false even though the object is semantically an Array.

This is one of the canonical cross-Realm pitfalls.

---

## 10. Internal Mechanics

### 10.1 Realm record

Conceptually:

```text
Realm Record
├── [[Intrinsics]]
├── [[GlobalObject]]
├── [[GlobalEnv]]
└── other Realm-specific state
```

The exact specification record structure evolves and must be read from the relevant ECMAScript edition.

### 10.2 Intrinsics

A Realm has a set of intrinsic objects.

Conceptual examples:

```text
%Object%
%Array%
%Function%
%Promise%
%Map%
%Set%
%Error%
%Object.prototype%
%Array.prototype%
```

Specification notation may use `%...%` for intrinsic references.

### 10.3 Intrinsic identity

Each Realm initializes its own intrinsic object identities.

Therefore:

```text
Realm A Array !== Realm B Array
```

and generally:

```text
Realm A Array.prototype !== Realm B Array.prototype
```

### 10.4 Global object

A Realm has a global object associated with its global environment.

### 10.5 Global environment

The global environment mediates declarations and global name resolution for the Realm.

### 10.6 GlobalThis

The `globalThis` value provides a standardized way to obtain the active global object/global context in environments where it is exposed.

### 10.7 `CreateRealm`

Conceptually creates a new Realm record and initializes its semantic components.

### 10.8 Realm initialization

Initialization includes setup of:

```text
intrinsics
global environment
global object
global bindings
```

with additional host coordination where required.

### 10.9 Agent

An Agent represents an ECMAScript execution entity.

Conceptually:

```text
Agent
├── execution state
├── Job Queue
├── execution contexts
├── associated Realm state
└── agent-specific semantic state
```

The exact record/state model is specification-defined and can evolve.

### 10.10 Jobs

Jobs execute within Agents.

This connects directly to Chapter 32.

### 10.11 Agent concurrency

Multiple Agents can execute work independently.

The host determines how those Agents map to physical resources.

### 10.12 Agent cluster

Shared-memory semantics involve an Agent Cluster concept in the ECMAScript model.

The purpose is to group Agents that can participate in a shared-memory relationship.

### 10.13 SharedArrayBuffer

A SharedArrayBuffer can provide shared memory between appropriate Agents where the host permits it.

### 10.14 Atomics

`Atomics` provides synchronization primitives for shared-memory operations.

### 10.15 Data ownership

Normal ECMAScript objects are not automatically shared between independent Agents.

### 10.16 Structured cloning

Hosts can clone structured values across execution boundaries using host-defined/standardized mechanisms such as structured clone.

### 10.17 Transfer

Transferable objects can move ownership/underlying resources across an appropriate boundary rather than cloning all data.

### 10.18 Shared memory

SharedArrayBuffer enables a different model:

```text
Agent A ─────┐
             ├── shared memory
Agent B ─────┘
```

This requires synchronization discipline.

### 10.19 Prototype boundary

A foreign object can retain its original prototype chain.

Do not silently “reprototype” it into the receiving Realm.

### 10.20 Brand checks

Some built-ins identify objects based on internal state rather than:

```text
constructor name
```

or:

```text
prototype equality
```

This is why some built-in predicates work more robustly across Realms.

### 10.21 `Array.isArray`

`Array.isArray` is based on Array internal semantics rather than a simple constructor identity test.

### 10.22 `instanceof`

`instanceof` depends on constructor/prototype semantics and can therefore expose Realm boundaries.

### 10.23 `Object.prototype.toString`

Historically used for type identification, but modern objects can influence behavior through tags/semantics and should not be treated as a universally reliable cross-Realm brand test.

### 10.24 Realm-specific errors

Errors created in one Realm can have prototypes/constructors from that Realm.

### 10.25 Realm-specific Promise

Promise objects and their intrinsic constructor/prototype relationships belong to the relevant Realm.

### 10.26 Realm-specific collections

A Map or Set created from another Realm has its own Realm-specific prototype chain and constructor identity.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Realm Record

A Realm Record conceptually contains references to:

```text
intrinsics
global object
global environment
```

and related state.

### 11.2 Intrinsic objects

Each Realm gets an independent set of intrinsic objects.

This is fundamental to:

```text
constructor identity
prototype identity
built-in behavior
```

### 11.3 Realm selection

Operations that create built-in objects often determine which Realm's intrinsics should be used.

This means allocation and constructor semantics can depend on Realm context.

### 11.4 Current Realm

When specification algorithms need to create objects or access intrinsics, the relevant Realm can matter.

### 11.5 Function Realm

Functions carry Realm-relevant semantic relationships.

This matters for:

```text
closures
intrinsics
global access
built-ins
function execution
```

### 11.6 Global Environment

A Realm's global environment connects declarations and the global object.

### 11.7 InitializeHostDefinedRealm

Hosts initialize an execution environment and create the relevant Realm according to host/runtime integration semantics.

This is a key bridge between:

```text
host startup
```

and:

```text
ECMAScript Realm
```

### 11.8 Agents and Jobs

Jobs are processed by Agents.

This connects:

```text
Promise reactions
async work
microtask/job processing
```

to an execution entity.

### 11.9 Multiple Realms in one Agent

Multiple language environments can exist without implying multiple Agents.

This is a crucial distinction.

### 11.10 Multiple Agents

Multiple Agents can execute concurrently.

Physical implementation can involve:

```text
threads
processes
or other scheduling mechanisms
```

but that mapping is host/engine-specific.

### 11.11 Agent Cluster

The shared-memory model groups appropriate Agents for shared memory semantics.

### 11.12 Shared memory semantics

When shared memory is involved, ordinary single-threaded assumptions are no longer sufficient.

Atomicity and memory ordering must be reasoned about explicitly.

### 11.13 Host boundaries

Browsers and Node.js decide how host constructs map onto:

```text
Realms
Agents
worker contexts
processes
threads
```

The exact mapping is not a universal ECMAScript-to-OS one-to-one rule.

---

## 12. Advanced Behavior

### 12.1 Cross-Realm `instanceof`

Example:

```text
foreign object
    ↓
foreign prototype
    ↓
foreign built-ins
```

versus:

```text
local constructor
    ↓
local prototype
```

Constructor identity mismatch can produce false results.

### 12.2 Cross-Realm `Array.isArray`

An Array predicate can recognize foreign arrays because Array-ness is not reducible to:

```text
object instanceof local Array
```

### 12.3 Cross-Realm Error handling

Suppose:

```js
try {
  // foreign Realm throws its Error
} catch (error) {
  console.log(error instanceof Error);
}
```

The result can be surprising because:

```text
error prototype
→ foreign Error.prototype

local Error
→ local Error constructor
```

### 12.4 Cross-Realm built-in calls

Calling a method originating in another Realm can execute using that function's semantic Realm relationships.

### 12.5 `eval` and Realm

Evaluation depends on the Realm/environment in which code executes.

### 12.6 Global constructors

Do not assume:

```text
globalThis.Array
```

has the same identity in every execution environment.

### 12.7 Prototype pollution boundaries

A prototype modification in one Realm does not automatically mutate another Realm's intrinsic prototype objects.

This can provide useful isolation.

### 12.8 Shared application objects

Passing a reference within the same Agent/Realm has different semantics from transferring/cloning data across Agents.

### 12.9 Message passing

A common worker architecture is:

```text
main Agent
   │
 message
   ↓
worker Agent
   │
 message
   ↑
result
```

### 12.10 Clone semantics

Cloning usually produces a distinct object graph.

Therefore:

```text
object identity
```

does not survive ordinary cloning.

### 12.11 Transfer semantics

Transferred resources may move ownership.

After transfer, the sender may lose usable access according to the transferable's semantics.

### 12.12 Shared semantics

With shared memory:

```text
both Agents
→ same memory
```

This changes the synchronization model.

### 12.13 Data races

Two Agents may read/write shared memory concurrently.

Without correct synchronization:

```text
race
→ nondeterministic/incorrect results
```

### 12.14 Atomics

Atomic operations provide synchronization and ordering tools for shared-memory coordination.

### 12.15 Lock-free design

Shared memory can enable lock-free data structures, but correctness requires careful memory-order reasoning.

### 12.16 False sharing

Different logical variables located in the same cache line can interfere at the hardware level.

This is an engine/OS/hardware concern, not a core Realm guarantee.

### 12.17 Message-passing versus shared memory

Message passing:

```text
explicit communication
+
copy/transfer cost
+
stronger isolation
```

Shared memory:

```text
less copying
+
shared state
+
synchronization complexity
```

### 12.18 Realm isolation versus Agent isolation

```text
Realm:
isolates language-global/intrinsic state

Agent:
isolates execution
```

### 12.19 Same Agent, separate Realms

Can provide:

```text
different intrinsics/global environments
```

without necessarily providing:

```text
parallel execution
```

### 12.20 Separate Agents, separate Realms

Common in worker architectures.

### 12.21 One Agent, many global environments

A host can create multiple independent language environments associated with execution structures in ways defined by host/specification integration.

### 12.22 Function closures across Realms

A function retains semantic relationships to its originating environment/Realm.

This matters when it accesses:

```text
globalThis
intrinsics
lexical bindings
```

### 12.23 `this` does not identify the Realm by itself

A receiver can be an object from another Realm.

Do not infer Realm from:

```js
this
```

alone.

### 12.24 Proxies across Realm boundaries

A Proxy can expose behavior across a boundary, but trap execution and target/handler identity must still be reasoned about carefully.

### 12.25 Security boundary caution

A Realm is not automatically a complete security sandbox.

Host capabilities and object references determine the real authority boundary.

### 12.26 Capability leakage

If a foreign Realm receives a powerful object reference, the isolation benefit can be reduced.

### 12.27 Frozen intrinsics

Hardening a Realm may involve freezing or replacing accessible capabilities.

This is an advanced security/application technique, not a universal ECMAScript guarantee.

### 12.28 SES-style compartments

Compartment-like security architectures use Realm/identity isolation ideas plus explicit capability control.

Do not equate a compartment abstraction with the exact ECMAScript Realm concept.

### 12.29 Worker termination

Agent lifetime can be ended by host-level worker lifecycle mechanisms.

### 12.30 Process termination

Process termination can destroy many Agents/Realms together depending on the runtime.

This is why:

```text
Realm
Agent
process
```

must remain separate concepts.

### 12.31 SharedArrayBuffer security

Shared memory can create timing and synchronization considerations that hosts may constrain.

### 12.32 Cross-origin iframe distinction

A browser iframe's security boundary involves host/platform origin rules in addition to Realm semantics.

Do not reduce same-origin policy to “different Realm”.

### 12.33 Cross-context object identity

An object created in one context can retain identity relationships specific to its origin.

### 12.34 Constructor patching

If one Realm modifies:

```js
Array.prototype
```

another Realm's intrinsic Array prototype is not automatically modified.

### 12.35 Built-in method origin

A method borrowed from another Realm may carry its own semantic Realm/context relationships.

---

## 13. Edge Cases

- Two Arrays from different Realms can fail `instanceof` against the local Realm's `Array`.
- `Array.isArray` can succeed across Realms.
- Errors from another Realm may fail `instanceof Error` locally.
- `Object.prototype`, `Array.prototype`, and similar objects are Realm-specific.
- `globalThis` differs across global environments.
- Same constructor name does not imply constructor identity.
- Same prototype structure does not imply same Realm.
- Separate Realms do not necessarily imply parallel execution.
- Separate Agents do not necessarily imply OS processes.
- Worker threads and Agents are related concepts but not universally one-to-one.
- Cloning does not preserve normal object identity.
- Transfer can change ownership/access semantics.
- Shared memory enables true cross-Agent shared state.
- Shared state requires synchronization.
- Shared memory does not make ordinary object graphs directly shared.
- A Realm is not automatically a security sandbox.
- A Worker is not merely “another Realm”; it is a host/runtime execution facility involving Agent-like isolation.
- A browser origin boundary is not the same thing as a Realm boundary.
- A foreign object can carry a foreign prototype chain.
- Prototype mutation in one Realm does not automatically mutate another Realm's intrinsics.
- Functions retain origin-related semantic relationships.
- `this` alone does not reveal the Realm.
- A Proxy can cross conceptual boundaries while preserving target/handler-specific semantics.
- Host runtime details can determine worker lifetime and physical execution resources.

---

## 14. Common Misconceptions

### Misconception 1 — “A Realm is a thread.”

No.

### Misconception 2 — “An Agent is exactly a thread.”

Not universally. It is a specification-level execution abstraction.

### Misconception 3 — “A Realm is a process.”

No.

### Misconception 4 — “Every worker creates exactly one Realm and one Agent in every implementation.”

Do not assume a universal host mapping.

### Misconception 5 — “Different Realms share the same Array constructor.”

No.

### Misconception 6 — “`instanceof Array` is reliable across every execution context.”

No.

### Misconception 7 — “`Array.isArray` checks the constructor name.”

No.

### Misconception 8 — “Cloning preserves object identity.”

No.

### Misconception 9 — “Worker messages pass object references directly.”

Normally, structured values are cloned/transferred according to the host's communication mechanism rather than simply sharing ordinary object identity.

### Misconception 10 — “SharedArrayBuffer shares normal JavaScript objects.”

No. It provides shared memory, not ordinary object-graph sharing.

### Misconception 11 — “Different Realms are a security sandbox by themselves.”

No.

### Misconception 12 — “Different Agents always mean different OS processes.”

No.

### Misconception 13 — “Same global constructor name means same object.”

No.

### Misconception 14 — “Object.prototype is universal.”

No. Intrinsic prototypes are Realm-specific.

### Misconception 15 — “Prototype pollution in one Realm automatically affects every Realm.”

No.

### Misconception 16 — “A Realm is only about globals.”

No. Intrinsics and associated semantic state are fundamental.

### Misconception 17 — “An Agent is just the event loop.”

No. It is a broader execution abstraction.

### Misconception 18 — “Multiple Realms always provide parallelism.”

No.

---

## 15. Common Mistakes

1. Treating Realm and Agent as synonyms.
2. Equating Agent with OS thread.
3. Equating Realm with process.
4. Assuming constructor identity crosses Realms.
5. Using `instanceof` for cross-Realm type checks without understanding prototype identity.
6. Assuming `Error` identity crosses Realms.
7. Assuming built-in prototypes are globally shared.
8. Forgetting that `globalThis` is Realm-specific.
9. Assuming worker message passing preserves object identity.
10. Ignoring cloning/transfer costs.
11. Sharing memory without synchronization.
12. Treating SharedArrayBuffer as ordinary object sharing.
13. Assuming a Realm is a complete security sandbox.
14. Ignoring host-origin/security rules around browser contexts.
15. Assuming worker lifecycle equals Realm lifecycle.
16. Assuming every implementation maps semantic Agents to OS threads one-to-one.
17. Forgetting function/closure Realm relationships.
18. Inferring Realm from `this`.

---

## 16. Comparison With Related Concepts

| Concept | Main role | Identity/isolation model |
|---|---|---|
| Realm | Intrinsics/global environment | Separate built-in/global identities |
| Agent | Execution entity | Separate execution state/job processing |
| Execution Context | Running-code state | Temporary/active execution state |
| Global Object | Global object for a Realm | Realm-specific |
| Worker | Host execution facility | Often introduces separate execution context/Agent-like isolation |
| Thread | OS/runtime execution resource | Implementation-level |
| Process | OS isolation boundary | Stronger memory isolation |
| Message passing | Communication | Copy/transfer semantics |
| Shared memory | Communication/state sharing | Explicit synchronization |
| `instanceof` | Prototype/constructor test | Sensitive to Realm identity |
| `Array.isArray` | Array semantic test | More robust across Realm boundaries |

### Realm vs Agent

```text
Realm:
Which intrinsics/global environment?

Agent:
Which execution entity?
```

### Realm vs process

```text
Realm:
language-level environment

Process:
OS-level isolation
```

### Agent vs thread

```text
Agent:
ECMAScript semantic abstraction

Thread:
physical/runtime execution resource
```

### Cloning vs transfer

```text
clone:
new object/data representation

transfer:
ownership/resource movement
```

### Transfer vs shared memory

```text
transfer:
one side's ownership/access changes

shared:
both sides can access designated memory
```

### `instanceof` vs `Array.isArray`

```text
instanceof:
prototype/constructor relationship

Array.isArray:
Array semantic identity
```

### Realm isolation vs security sandboxing

```text
Realm:
language-global/intrinsic separation

sandbox:
capability/security boundary
```

---

## 17. Performance Considerations

### 17.1 Realm initialization

Creating a new Realm can require initialization of many intrinsic objects.

### 17.2 Multiple intrinsics

Each Realm can have its own intrinsic object graph.

This may increase memory use.

### 17.3 Cross-Realm communication

Passing data across execution boundaries can cost:

```text
serialization
cloning
transfer setup
synchronization
```

### 17.4 Shared memory

Shared memory can reduce copying but introduces:

```text
atomic operations
coordination
cache contention
synchronization overhead
```

### 17.5 Workers

Worker startup can carry runtime initialization cost.

### 17.6 Message frequency

Very high message rates can become a communication bottleneck.

### 17.7 Payload size

Large cloned payloads can dominate CPU and memory cost.

### 17.8 Transferables

Transfer can reduce copying for supported resources but changes ownership semantics.

### 17.9 Shared memory contention

Multiple Agents can contend for:

```text
cache
memory bandwidth
atomic operations
```

### 17.10 False sharing

Poorly arranged shared memory can create hardware-level contention.

### 17.11 Process isolation

Processes generally have higher communication cost than in-process worker mechanisms, but exact trade-offs depend on runtime.

### 17.12 Realm count

Creating many isolated Realms can increase:

```text
memory
initialization
GC
management overhead
```

### 17.13 Performance rule

Choose the boundary based on the workload:

```text
same Realm
→ lowest isolation overhead

separate Realm
→ intrinsic/global isolation

separate Agent
→ execution isolation/concurrency

separate process
→ stronger failure/memory isolation
```

---

## 18. Memory Considerations

### 18.1 Per-Realm intrinsic state

Each Realm can require its own copies/instances of intrinsic objects.

### 18.2 Prototype graphs

Realm-specific prototypes create separate reachable object graphs.

### 18.3 Cross-Realm references

A foreign object can keep its originating Realm-related objects reachable.

### 18.4 Function retention

Functions can retain environments and Realm-related state.

### 18.5 Worker memory

Separate Agents can have substantial runtime state.

### 18.6 Shared memory

SharedArrayBuffer avoids some data duplication but does not eliminate:

```text
worker/runtime memory
```

### 18.7 Cloning

Cloning produces additional object/data graphs.

### 18.8 Transfer

Transfer can reduce duplication for supported resources.

### 18.9 Queued messages

Pending messages can retain payload memory.

### 18.10 Worker shutdown

Failure to terminate workers or release communication resources can cause memory retention.

### 18.11 Memory model

Use:

```text
Realm count
+
Agent count
+
payload size
+
queue depth
+
shared memory
+
runtime overhead
```

when planning architecture.

---

## 19. Security Considerations

### 19.1 Realm isolation

Separate intrinsics can prevent direct mutation of another Realm's built-ins.

### 19.2 Capability references

Passing a powerful object into another Realm can grant authority.

### 19.3 Prototype hardening

Security-sensitive environments may harden intrinsics.

### 19.4 Cross-Realm objects

Foreign objects may carry powerful methods or accessors.

### 19.5 Accessor side effects

Reading a property across a boundary can execute code.

### 19.6 Proxy effects

Foreign Proxies can mediate behavior.

### 19.7 Worker isolation

Workers can reduce accidental shared-state coupling.

They are not automatically security sandboxes against all threats.

### 19.8 Process isolation

Processes can provide stronger memory/failure isolation than same-process Realm separation.

### 19.9 Shared memory

Shared memory increases the complexity of correctness and side-channel reasoning.

### 19.10 Browser origin security

Browser cross-origin rules remain host-layer security mechanisms.

### 19.11 Prototype pollution

Realm separation can reduce the scope of prototype mutations, but application data can still be unsafe.

### 19.12 Capability design

The strongest security architecture is often:

```text
minimal authority
+
explicit capabilities
+
clear ownership
+
isolated execution
```

rather than relying on Realm boundaries alone.

---

## 20. Production Usage

### 20.1 Browser iframes

Use when separate global/intrinsic environments are useful.

But browser origin policies must also be considered.

### 20.2 Web Workers

Use for:

```text
CPU-heavy work
long-running computation
execution isolation
responsive UI
```

The host defines the worker lifecycle and communication model.

### 20.3 Node worker threads

Useful for CPU-heavy work or isolation within one process.

### 20.4 Child processes

Use when stronger isolation/failure boundaries are required.

### 20.5 Plugin systems

Potential architecture:

```text
host
 ↓
isolated execution context
 ↓
explicit capability API
```

### 20.6 Multi-tenant JavaScript

Realm/worker/process boundaries may be combined depending on threat model.

### 20.7 Cross-Realm library interoperability

Prefer semantic checks that survive Realm boundaries.

For arrays:

```js
Array.isArray(value)
```

is generally preferable to:

```js
value instanceof Array
```

when foreign Realms are possible.

### 20.8 Data exchange

Choose intentionally:

```text
clone
transfer
shared memory
```

based on ownership and performance.

### 20.9 Shared-memory systems

Use:

```text
Atomics
clear invariants
explicit ownership/protocols
```

rather than treating shared memory as normal mutable state.

### 20.10 Worker lifecycle

Define:

```text
create
initialize
accept work
cancel
drain
terminate
cleanup
```

### 20.11 Observability

Track:

```text
worker count
worker lifetime
queue depth
message latency
task duration
termination reason
memory
CPU
errors
```

### 20.12 Failure domains

Decide whether worker failure should:

```text
restart
retry
fail request
shed load
terminate process
```

### 20.13 Data validation

Treat data crossing execution boundaries as untrusted input when the architecture requires it.

### 20.14 Security model

Document:

```text
what authority crosses the boundary?
what memory is shared?
what data is cloned?
who owns resources?
```

---

## 21. Implementation From Scratch

### Stage 1 — Realm Model

Implement a teaching model:

```js
class RealmModel {
  constructor() {
    this.intrinsics = {
      Object: {},
      Array: {},
      Function: {},
      Promise: {}
    };

    this.globalObject = {};
  }
}
```

The goal is to make Realm-specific identity explicit.

### Stage 2 — Separate Intrinsics

Create:

```text
Realm A → Array_A
Realm B → Array_B
```

and demonstrate:

```text
Array_A !== Array_B
```

### Stage 3 — Prototype Model

Create:

```text
Array.prototype_A
Array.prototype_B
```

and attach objects to the corresponding prototype chains.

### Stage 4 — Cross-Realm Identity

Build tests showing why:

```text
foreignObject instanceof localConstructor
```

can fail.

### Stage 5 — Agent Model

Implement a teaching abstraction:

```js
class AgentModel {
  constructor() {
    this.realms = [];
    this.queue = [];
  }

  enqueue(job) {}

  runNext() {}
}
```

### Stage 6 — Message Passing

Implement:

```text
Agent A
→ serialize/clone
→ Agent B
```

### Stage 7 — Transfer Model

Simulate ownership transfer:

```text
sender owns resource
→ transfer
→ receiver owns resource
```

### Stage 8 — Shared Memory

Build a simplified shared integer buffer.

Require atomic operations for updates.

### Stage 9 — Race Demonstration

Create two Agents that increment shared state.

First implement an unsafe version.

Then implement an atomic/synchronized version.

### Stage 10 — Boundary Matrix

Create a comparison table for:

```text
same Realm
different Realm
different Agent
worker
child process
external service
```

and record:

```text
identity
memory
communication
failure
security
startup
cost
```

---

## 22. Debugging Exercises

### Exercise 1 — Cross-Realm Array

Create an iframe/foreign Realm and demonstrate:

```js
foreignArray instanceof Array
```

versus:

```js
Array.isArray(foreignArray)
```

### Exercise 2 — Cross-Realm Error

Throw an Error from one Realm and inspect:

```js
error instanceof Error
```

### Exercise 3 — Prototype Identity

Compare:

```js
Object.getPrototypeOf(foreignArray)
Array.prototype
```

### Exercise 4 — Constructor Identity

Compare:

```js
foreignArray.constructor
Array
```

and inspect their Realm origin.

### Exercise 5 — Realm Mutation

Modify one Realm's:

```js
Array.prototype
```

and verify that another Realm's Array prototype is unaffected.

### Exercise 6 — Worker Identity

Send an object to a Worker and test whether the same JavaScript object identity is preserved.

### Exercise 7 — Clone Cost

Send increasingly large objects to a Worker and measure:

```text
serialization/clone time
message latency
memory
```

### Exercise 8 — Shared Memory Race

Create two Workers updating the same SharedArrayBuffer.

Demonstrate a race without atomic coordination.

### Exercise 9 — Worker Termination

Start work, terminate the worker, and determine:

```text
which promises resolve?
which reject?
which resources remain?
```

### Exercise 10 — Capability Leak

Pass an object exposing privileged operations into an isolated context.

Determine what authority crosses the boundary.

---

## 23. Code Review Exercise

Review this statement:

> “We can safely isolate untrusted plugins by creating a new Realm and giving them the plugin object.”

Identify the missing questions:

```text
What capabilities does the plugin object expose?
Can objects reference host objects?
Can functions call back into the host?
Can prototypes be modified?
Are intrinsics hardened?
Can the plugin access network/filesystem through host objects?
Is the Realm backed by separate Agent/process isolation?
What happens on infinite loops?
What happens on memory exhaustion?
How are resources terminated?
```

Then classify the resulting security boundary as:

```text
language isolation
execution isolation
process isolation
capability isolation
```

and explain why those categories are not interchangeable.

---

## 24. Interview Questions

### Foundational

1. What is a Realm?
2. What is an Agent?
3. How are they different?
4. What are intrinsics?
5. What is the global object?
6. What is `globalThis`?
7. Why are constructors Realm-specific?
8. Why are prototypes Realm-specific?
9. Why can cross-Realm `instanceof` fail?
10. Why can `Array.isArray` succeed across Realms?

### Intermediate

11. What is `CreateRealm`?
12. What is Realm initialization?
13. What is an Agent's relationship to Jobs?
14. Can one Agent have multiple Realms?
15. Does one Agent equal one OS thread?
16. Does one Realm equal one process?
17. What is an Agent Cluster?
18. How does SharedArrayBuffer relate to Agents?
19. What is cloning?
20. What is transfer?

### Advanced

21. Explain cross-Realm constructor identity.
22. Explain cross-Realm Error objects.
23. Explain prototype identity across Realms.
24. Compare clone, transfer, and shared memory.
25. Explain Agent concurrency.
26. Explain why worker threads are host/runtime constructs rather than pure ECMAScript abstractions.
27. Explain shared-memory synchronization.
28. Explain why a Realm is not automatically a security sandbox.
29. Explain capability leakage across boundaries.
30. Explain how functions retain Realm-related semantic relationships.

### Principal-Level

31. Design a plugin execution architecture using Realms, Workers, or processes.
32. Decide when a separate Realm is sufficient and when a separate Agent is necessary.
33. Decide when a worker process is required.
34. Design a cross-Realm type-checking strategy.
35. Design a safe data-transfer boundary.
36. Design a SharedArrayBuffer protocol with explicit invariants.
37. Diagnose a cross-context memory/performance regression.
38. Diagnose a security bug caused by capability leakage.
39. Explain the relationship among Realm, Agent, Worker, thread, and process without conflating their layers.
40. Defend:

> “Realm is a language environment boundary; Agent is an execution boundary; process is an OS isolation boundary.”

---

## 25. Predict-the-Output Exercises

For each:

```text
Predict
→ Run
→ Compare
→ Identify Realm/Agent semantics
→ Explain
```

### Exercise A

Conceptually with a foreign Realm:

```js
const foreignArray = new foreignWindow.Array();

console.log(foreignArray instanceof Array);
console.log(Array.isArray(foreignArray));
```

### Exercise B

```js
console.log(
  Object.getPrototypeOf(foreignArray) === Array.prototype
);
```

### Exercise C

```js
console.log(
  foreignWindow.Array === Array
);
```

### Exercise D

```js
console.log(
  foreignWindow.Object === Object
);
```

### Exercise E

Throw an error from the foreign Realm and evaluate:

```js
error instanceof Error
```

Explain why the answer can differ from the local Realm.

### Exercise F

Mutate:

```js
foreignWindow.Array.prototype.custom = 1;
```

Then inspect:

```js
Array.prototype.custom
foreignWindow.Array.prototype.custom
```

### Exercise G

Send an object to a Worker and compare object identity on each side.

Explain why:

```js
received === sent
```

does not normally preserve original identity.

### Exercise H

Use a SharedArrayBuffer with two workers and reason about why unsynchronized increments are unsafe.

---

## 26. Mastery Exercises

### Exercise 1 — Multi-Realm Laboratory

Create:

```text
Realm A
Realm B
```

and compare:

```text
constructors
prototypes
errors
collections
globalThis
functions
symbols
```

### Exercise 2 — Cross-Realm Type Matrix

Build a matrix for:

```text
Array
Date
RegExp
Map
Set
Error
Function
Object
```

comparing:

```text
instanceof
constructor identity
prototype identity
built-in predicate where available
```

### Exercise 3 — Agent Model

Implement a teaching Agent with:

```text
job queue
execution state
multiple Realms
```

### Exercise 4 — Worker Message Benchmark

Measure:

```text
small clone
large clone
transfer
shared memory
```

for representative workloads.

### Exercise 5 — Shared Counter

Implement:

```text
two Agents
→ shared counter
→ unsafe increment
→ atomic increment
```

and explain the difference.

### Exercise 6 — Plugin Isolation

Design three architectures:

```text
same Realm
separate Realm
separate process
```

and compare:

```text
security
performance
memory
failure containment
communication
complexity
```

### Exercise 7 — Cross-Realm API Boundary

Design an API that safely accepts foreign objects without assuming:

```text
constructor identity
prototype identity
```

### Exercise 8 — Capability Security

Create an isolated environment with only explicitly provided capabilities.

Document:

```text
allowed authority
forbidden authority
escape paths
lifecycle
```

### Exercise 9 — Agent Failure

Simulate a worker/Agent failure and define:

```text
retry
recovery
cleanup
state reconstruction
```

### Exercise 10 — Principal Architecture Challenge

Design a multi-tenant JavaScript execution platform.

Requirements:

```text
untrusted tenant code
CPU isolation
memory limits
resource limits
network control
termination
observability
tenant fairness
```

Compare:

```text
Realm
+
Agent/Worker
+
Process
+
External sandbox
```

and defend the selected architecture.

---

## 27. Key Takeaways

1. A Realm is a specification-level execution environment containing its own intrinsics and global environment.
2. An Agent is a specification-level execution entity capable of running code and processing Jobs.
3. Realm and Agent are different dimensions.
4. A Realm is not a thread.
5. An Agent is not universally identical to an OS thread.
6. A Realm is not a process.
7. Each Realm has its own intrinsic constructors/prototypes.
8. Cross-Realm constructor identity is therefore different.
9. Cross-Realm `instanceof` can fail because prototype identities differ.
10. `Array.isArray` is more robust for identifying arrays across Realm boundaries.
11. `globalThis` belongs to the active global environment.
12. Functions and built-ins can retain Realm-related semantic relationships.
13. Separate Realms do not automatically provide parallel execution.
14. Separate Agents can support concurrent execution.
15. Host/runtime mechanisms such as Workers provide concrete execution facilities around these abstractions.
16. Normal object identity does not automatically cross Agent communication boundaries.
17. Cloning produces distinct object graphs.
18. Transfer moves ownership/access for supported transferable resources.
19. SharedArrayBuffer provides shared memory rather than shared ordinary object graphs.
20. Shared memory requires synchronization and careful memory-order reasoning.
21. Agent Cluster semantics matter for shared memory.
22. A Realm boundary is not automatically a security sandbox.
23. Security depends on capabilities, host restrictions, object references, resource controls, and execution isolation.
24. The central principle is:

> Realm defines the language-global world in which values and intrinsics live; Agent defines the execution entity that processes code and Jobs; the host then decides how those semantic boundaries map onto workers, threads, processes, and other runtime resources.

---

## 28. Concept Connections

### Depends On

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 37 — Cancellation / Abort
- Chapter 39 — Concurrency / Parallelism
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations
- Chapter 43 — Ordinary Object Internal Methods

### Builds Toward

- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 95 — Legacy JavaScript
- Chapter 96 — WebAssembly / Native Interoperability
- Chapter 97 — Edge / Serverless JavaScript
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

### Related Concepts

- Realm
- Agent
- Agent Cluster
- Realm Record
- Intrinsics
- Global Object
- Global Environment
- `globalThis`
- Execution Context
- Job Queue
- SharedArrayBuffer
- Atomics
- Structured Clone
- Transferable
- Worker
- Worker Thread
- Child Process
- Process
- Message Passing
- Shared Memory
- Prototype Identity
- Constructor Identity
- `instanceof`
- `Array.isArray`
- Capability Security
- Sandboxing

### Concepts Revisited

This chapter revisits:

- execution contexts;
- objects;
- prototypes;
- internal slots;
- jobs;
- concurrency;
- structured clone;
- typed arrays;
- shared memory;
- workers.

### Why This Chapter Matters Later

Chapter 44 completes the core specification architecture by establishing the boundaries around execution environments.

The previous chapters explain:

```text
41 → how the specification is organized
42 → reusable semantic operations
43 → ordinary object internal behavior
44 → where execution environments and intrinsic worlds exist
```

This lets the learner reason about seemingly strange cross-context behavior without falling into implementation-level shortcuts.

The resulting model is:

```text
Realm
→ defines the language-global world

Agent
→ executes code/jobs

Host
→ maps Agents/Realms onto workers/threads/processes/etc.

Communication
→ clone / transfer / shared memory

Security
→ capabilities + host restrictions + isolation
```

---

## 29. Completion Criteria

### Conceptual Understanding

- [ ] Define Realm.
- [ ] Define Agent.
- [ ] Distinguish Realm and Agent.
- [ ] Distinguish Realm and process.
- [ ] Distinguish Agent and thread.
- [ ] Explain intrinsics.
- [ ] Explain Realm-specific constructors.
- [ ] Explain Realm-specific prototypes.
- [ ] Explain global object.
- [ ] Explain global environment.
- [ ] Explain `globalThis`.
- [ ] Explain Agent job processing.
- [ ] Explain Agent Cluster.
- [ ] Explain shared memory.
- [ ] Explain cloning.
- [ ] Explain transfer.
- [ ] Explain host worker boundaries.
- [ ] Explain language isolation vs security sandboxing.

### Predictive Mastery

- [ ] Predict cross-Realm `instanceof`.
- [ ] Predict cross-Realm `Array.isArray`.
- [ ] Predict cross-Realm constructor equality.
- [ ] Predict cross-Realm prototype equality.
- [ ] Predict cross-Realm Error identity checks.
- [ ] Predict Realm-specific prototype mutation.
- [ ] Predict Worker object identity.
- [ ] Predict cloning/transfer differences.
- [ ] Predict shared-memory race conditions.
- [ ] Predict Realm vs Agent isolation trade-offs.

### Implementation

- [ ] Implement a simplified Realm model.
- [ ] Implement Realm-specific intrinsics.
- [ ] Implement Realm-specific prototypes.
- [ ] Implement a teaching Agent.
- [ ] Implement message passing.
- [ ] Implement clone semantics.
- [ ] Implement transfer semantics.
- [ ] Implement a shared-memory counter.
- [ ] Implement synchronization.
- [ ] Build a Realm/Agent comparison matrix.

### Debugging

- [ ] Diagnose cross-Realm `instanceof` failures.
- [ ] Diagnose cross-Realm error checks.
- [ ] Diagnose prototype identity issues.
- [ ] Diagnose Worker identity assumptions.
- [ ] Diagnose message clone overhead.
- [ ] Diagnose shared-memory races.
- [ ] Diagnose Worker lifecycle failures.
- [ ] Diagnose capability leakage.
- [ ] Diagnose incorrect Realm/Agent assumptions.
- [ ] Separate ECMAScript semantics from host worker behavior.

### Production Engineering

- [ ] Choose same Realm vs separate Realm.
- [ ] Choose same Agent vs separate Agent.
- [ ] Choose Worker vs process.
- [ ] Choose clone vs transfer vs shared memory.
- [ ] Define lifecycle.
- [ ] Define capabilities.
- [ ] Define resource limits.
- [ ] Define failure containment.
- [ ] Define observability.
- [ ] Define security boundary.
- [ ] Define data ownership.

### Interview Readiness

- [ ] Explain Realm.
- [ ] Explain Agent.
- [ ] Explain Agent Cluster.
- [ ] Explain intrinsics.
- [ ] Explain cross-Realm `instanceof`.
- [ ] Explain `Array.isArray` across Realms.
- [ ] Explain clone/transfer/shared memory.
- [ ] Explain Worker/Agent relationships.
- [ ] Explain Realm vs process.
- [ ] Defend a production isolation architecture.

### Track A — Core Theory

- [ ] Realm model understood.
- [ ] Agent model understood.
- [ ] Realm vs Agent distinction understood.
- [ ] Intrinsic identity understood.
- [ ] Cross-Realm behavior understood.
- [ ] Shared-memory model understood.
- [ ] Host/runtime boundary understood.

### Track B — Implementation

- [ ] Guided Realm model completed.
- [ ] Partially guided model completed.
- [ ] No-reference model completed.
- [ ] Edge-case hardened model completed.
- [ ] Cross-context test suite reviewed.

### Track C — Interview / Reasoning

- [ ] Cross-Realm prediction completed.
- [ ] Worker boundary debugging completed.
- [ ] Shared-memory reasoning completed.
- [ ] Security-boundary review completed.
- [ ] Isolation architecture comparison completed.
- [ ] Principal-level defense completed.

### Mastery Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Do not mark `[*] Mastered` until the learner can independently:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# Chapter 44 — Revision / Retrieval Record

## Retrieval Prompts

1. What is a Realm?
2. What is an Agent?
3. What is the difference between them?
4. What are intrinsics?
5. Why does every Realm have distinct intrinsic constructors?
6. Why does every Realm have distinct prototype objects?
7. What is `globalThis`?
8. What is the global object?
9. What is the global environment?
10. What does `CreateRealm` conceptually do?
11. What is Realm initialization?
12. Can one Agent have multiple Realms?
13. What does an Agent execute?
14. What is an Agent Cluster?
15. How does SharedArrayBuffer relate to Agents?
16. What is Atomics?
17. What is structured cloning?
18. What is transfer?
19. What is shared memory?
20. Why can cross-Realm `instanceof` fail?
21. Why can `Array.isArray` work across Realms?
22. Why can `error instanceof Error` fail across Realms?
23. Why are constructor identities different across Realms?
24. Why are prototype identities different across Realms?
25. Does a Worker equal one Agent?
26. Does an Agent equal one thread?
27. Does a Realm equal one process?
28. Why is a Realm not automatically a security sandbox?
29. How can capabilities leak across a boundary?
30. When should a system use a Realm?
31. When should it use a Worker/Agent?
32. When should it use a process?
33. When should it clone, transfer, or share memory?
34. What are the main performance costs of execution boundaries?
35. What are the main security benefits and limitations?

## Weak Areas

```text
-
-
-
```

## Revision Queue

```text
- [ ] Revisit Realm definition
- [ ] Revisit Agent definition
- [ ] Revisit intrinsics
- [ ] Revisit Realm-specific constructors
- [ ] Revisit Realm-specific prototypes
- [ ] Revisit globalThis
- [ ] Revisit Realm initialization
- [ ] Revisit Agent Jobs
- [ ] Revisit Agent Cluster
- [ ] Revisit SharedArrayBuffer
- [ ] Revisit Atomics
- [ ] Revisit clone/transfer/shared memory
- [ ] Revisit cross-Realm instanceof
- [ ] Revisit cross-Realm Error checks
- [ ] Revisit Worker boundaries
- [ ] Revisit security boundaries
```

## Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

## Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 44 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — primary source for Realm, Agent, Agent Cluster, intrinsics, execution-context relationships, Jobs, and shared-memory language semantics.
2. WHATWG/browser standards — browser execution contexts, Workers, iframe/global-object behavior, structured clone, origin/security boundaries, and host integration.
3. Node.js official documentation — worker threads, child processes, VM/context mechanisms, process lifecycle, and runtime-specific execution isolation.
4. Engine documentation/source — how semantic Agents/Realms are physically implemented.
5. Security architecture documentation — capability isolation and sandbox designs.

Always distinguish:

```text
Realm
vs
Agent
vs
execution context
vs
Worker
vs
thread
vs
process
vs
security sandbox
```

For cross-context behavior, use:

```text
specification semantics
→ host communication model
→ implementation mapping
→ runtime measurement
```

Do not claim that an ECMAScript Realm is a complete security boundary.

Do not assume a one-to-one mapping between Agents and OS threads/processes.

Do not use constructor identity as a universal cross-Realm type test.

---

# Chapter 44 — Completion Snapshot

```text
Chapter: 44
Title: Realms, Agents, and Execution Isolation
Part: VII — ECMAScript Specification
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```