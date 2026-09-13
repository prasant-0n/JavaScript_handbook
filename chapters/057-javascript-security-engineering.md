# Chapter 57 — JavaScript Security Engineering

> **Curriculum Position:** Part X — Networking / Security  
> **Prerequisites:** Chapters 42–48, 54–56  
> **Primary Focus:** JavaScript-specific security engineering across language semantics, object integrity, dynamic code execution, prototype pollution, unsafe deserialization, dependency/supply-chain risk, capability design, sandboxing, data validation, secure APIs, browser integration, Node interoperability, and production threat modeling  
> **Status:** `[ ] Not Started`  
> **Depth Target:** JavaScript semantics → attacker-controlled data → object/code capabilities → runtime abuse → defensive engineering → secure library/application architecture  
> **Scope Rule:** This chapter is about securing JavaScript systems themselves. Browser security controls from Chapter 56 remain important, but this chapter goes deeper into JavaScript execution, object semantics, dynamic behavior, and application/library design.

---

# Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What JavaScript Security Engineering Means](#3-what-javascript-security-engineering-means)
- [4. Threat Model](#4-threat-model)
- [5. Mental Model](#5-mental-model)
- [6. JavaScript as Executable Authority](#6-javascript-as-executable-authority)
- [7. Code vs Data](#7-code-vs-data)
- [8. Dynamic Code Execution](#8-dynamic-code-execution)
- [9. `eval`](#9-eval)
- [10. `Function`](#10-function)
- [11. String-Based Timers](#11-string-based-timers)
- [12. Dynamic Module and Script Loading](#12-dynamic-module-and-script-loading)
- [13. JavaScript Object Integrity](#13-javascript-object-integrity)
- [14. Prototype Chains as an Attack Surface](#14-prototype-chains-as-an-attack-surface)
- [15. Prototype Pollution](#15-prototype-pollution)
- [16. Pollution Sources](#16-pollution-sources)
- [17. Pollution Gadgets](#17-pollution-gadgets)
- [18. Defending Against Prototype Pollution](#18-defending-against-prototype-pollution)
- [19. `Object.create(null)` and Data Dictionaries](#19-objectcreatenull-and-data-dictionaries)
- [20. Map and Set as Security Design Choices](#20-map-and-set-as-security-design-choices)
- [21. Property Access and Own-Property Checks](#21-property-access-and-own-property-checks)
- [22. Destructuring and Default-Value Hazards](#22-destructuring-and-default-value-hazards)
- [23. Object Freezing, Sealing, and Integrity](#23-object-freezing-sealing-and-integrity)
- [24. Deep Freezing and Transitive Trust](#24-deep-freezing-and-transitive-trust)
- [25. Getters, Setters, and Accessor Hazards](#25-getters-setters-and-accessor-hazards)
- [26. Proxies and Security](#26-proxies-and-security)
- [27. Reflect and Safe Meta-Operations](#27-reflect-and-safe-meta-operations)
- [28. Symbols and Capability-Oriented APIs](#28-symbols-and-capability-oriented-apis)
- [29. Closures as Encapsulation](#29-closures-as-encapsulation)
- [30. Private Class Fields](#30-private-class-fields)
- [31. Capability Security](#31-capability-security)
- [32. Least Authority](#32-least-authority)
- [33. Confused Deputy](#33-confused-deputy)
- [34. Object-Capability Thinking](#34-object-capability-thinking)
- [35. Input Validation](#35-input-validation)
- [36. Schema Validation](#36-schema-validation)
- [37. Parsing and Deserialization](#37-parsing-and-deserialization)
- [38. JSON Safety and Limits](#38-json-safety-and-limits)
- [39. `structuredClone`](#39-structuredclone)
- [40. Unsafe Merging and Deep Assignment](#40-unsafe-merging-and-deep-assignment)
- [41. URL and Protocol Validation](#41-url-and-protocol-validation)
- [42. Regular Expressions and ReDoS](#42-regular-expressions-and-redos)
- [43. Resource Exhaustion](#43-resource-exhaustion)
- [44. Algorithmic Complexity Attacks](#44-algorithmic-complexity-attacks)
- [45. Denial of Service in JavaScript](#45-denial-of-service-in-javascript)
- [46. Error Handling and Information Disclosure](#46-error-handling-and-information-disclosure)
- [47. Logging Safety](#47-logging-safety)
- [48. Serialization and Prototype Semantics](#48-serialization-and-prototype-semantics)
- [49. Prototype-Safe API Design](#49-prototype-safe-api-design)
- [50. Dependency and Supply-Chain Security](#50-dependency-and-supply-chain-security)
- [51. Package Lifecycle Threats](#51-package-lifecycle-threats)
- [52. Dependency Confusion and Name Resolution](#52-dependency-confusion-and-name-resolution)
- [53. Lockfiles and Reproducibility](#53-lockfiles-and-reproducibility)
- [54. Postinstall and Arbitrary Build-Time Code](#54-postinstall-and-arbitrary-build-time-code)
- [55. Third-Party Packages as Trusted Code](#55-third-party-packages-as-trusted-code)
- [56. Secrets and Configuration](#56-secrets-and-configuration)
- [57. Environment Boundaries](#57-environment-boundaries)
- [58. JavaScript in the Browser vs Node.js](#58-javascript-in-the-browser-vs-nodejs)
- [59. Sandboxing JavaScript](#59-sandboxing-javascript)
- [60. `vm`-Style Isolation and Its Limits](#60-vm-style-isolation-and-its-limits)
- [61. Realms, Compartments, and SES Concepts](#61-realms-compartments-and-ses-concepts)
- [62. Dynamic Plugin Systems](#62-dynamic-plugin-systems)
- [63. Untrusted Code Execution](#63-untrusted-code-execution)
- [64. Tainted Data and Dataflow](#64-tainted-data-and-dataflow)
- [65. Secure API Boundaries](#65-secure-api-boundaries)
- [66. Security-Sensitive Defaults](#66-security-sensitive-defaults)
- [67. Secure Library Design](#67-secure-library-design)
- [68. Browser Integration](#68-browser-integration)
- [69. Web Workers and Security](#69-web-workers-and-security)
- [70. Web Components and Security](#70-web-components-and-security)
- [71. Node Integration Preview](#71-node-integration-preview)
- [72. Observability and Incident Response](#72-observability-and-incident-response)
- [73. Static Analysis and Secure Tooling](#73-static-analysis-and-secure-tooling)
- [74. Security Testing Strategy](#74-security-testing-strategy)
- [75. Performance/Security Trade-offs](#75-performancesecurity-trade-offs)
- [76. Memory/Security Trade-offs](#76-memorysecurity-trade-offs)
- [77. Production Security Architecture](#77-production-security-architecture)
- [78. Anti-Patterns](#78-anti-patterns)
- [79. Debugging Methodology](#79-debugging-methodology)
- [80. Implementation From Scratch](#80-implementation-from-scratch)
- [81. Debugging Exercises](#81-debugging-exercises)
- [82. Code Review Exercise](#82-code-review-exercise)
- [83. Interview Questions](#83-interview-questions)
- [84. Predict-the-Outcome Exercises](#84-predict-the-outcome-exercises)
- [85. Mastery Exercises](#85-mastery-exercises)
- [86. Key Takeaways](#86-key-takeaways)
- [87. Concept Connections](#87-concept-connections)
- [88. Completion Criteria](#88-completion-criteria)
- [89. Revision / Retrieval Record](#89-revision--retrieval-record)
- [90. Canonical References and Source Discipline](#90-canonical-references-and-source-discipline)
- [91. Completion Snapshot](#91-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain JavaScript-specific security threats.
2. Distinguish code from data.
3. Explain why dynamic code execution is dangerous.
4. Analyze `eval`, `Function`, and string-based timer hazards.
5. Explain prototype pollution.
6. Identify prototype-pollution sources and gadgets.
7. Defend object-merging and dynamic-property code.
8. Explain when `Object.create(null)`, `Map`, and `Set` are safer choices.
9. Use own-property checks correctly.
10. Explain why default property lookups can interact with polluted prototypes.
11. Explain `Object.freeze`, `seal`, and integrity levels.
12. Understand accessors/getters as executable behavior.
13. Analyze Proxies as security-sensitive meta-programming.
14. Explain capability-based security.
15. Explain least authority and confused-deputy problems.
16. Treat external input as hostile until validated.
17. Design runtime schema validation.
18. Evaluate parsing and deserialization risks.
19. Understand prototype-related concerns around merges and object construction.
20. Validate URLs and protocols safely.
21. Understand ReDoS and algorithmic complexity attacks.
22. Identify JavaScript resource-exhaustion paths.
23. Build safe error and logging policies.
24. Threat-model dependencies and third-party code.
25. Understand package lifecycle/build-script risks.
26. Explain dependency confusion.
27. Explain lockfiles and reproducibility.
28. Protect secrets and configuration.
29. Distinguish browser and Node security boundaries.
30. Explain why ordinary JavaScript execution contexts are not automatically sandboxes.
31. Understand the limits of VM-like isolation.
32. Explain Realms/Compartments/SES concepts at a high level.
33. Design safer plugin architectures.
34. Apply tainted-data/dataflow reasoning.
35. Build secure API boundaries.
36. Design security-sensitive defaults.
37. Author safer JavaScript libraries.
38. Integrate browser security controls with JavaScript security.
39. Design security testing and static-analysis strategies.
40. Build a production JavaScript security architecture.
41. Defend decisions across correctness, performance, memory, security, reliability, maintainability, and operational complexity.

### Mastery target

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

---

# 2. Prerequisites

## Core Language

Required:

```text
Chapter 15 — Objects
Chapter 17 — Prototypes
Chapter 19 — Proxy / Reflect
Chapter 20 — Symbols
Chapter 22 — Arrays
Chapter 24 — Map / Set
Chapter 28 — JSON / Serialization
Chapter 29 — Errors
Chapter 42 — Abstract Operations
Chapter 43 — Internal Methods
```

## Runtime

Required:

```text
Chapter 47 — JavaScript Engine Architecture
Chapter 48 — V8 Internals
```

## Browser

Required:

```text
Chapter 54 — Web Components
Chapter 55 — Fetch
Chapter 56 — Browser Security
```

---

# 3. What JavaScript Security Engineering Means

JavaScript security engineering is the discipline of designing JavaScript systems so that:

```text
untrusted input
+
dynamic execution
+
object semantics
+
dependencies
+
runtime capabilities
```

cannot be combined into unintended authority.

The core problem is not simply:

```text
"avoid XSS"
```

It is:

```text
control who can cause what code to execute
control what data becomes behavior
control what objects can be modified
control what authority code receives
control what dependencies are trusted
control what resources untrusted input can consume
```

---

# 4. Threat Model

Consider a JavaScript application.

Potential attackers include:

```text
anonymous user
authenticated attacker
malicious web page
malicious iframe
compromised dependency
malicious plugin
compromised CDN
insider
supply-chain attacker
```

Potential assets:

```text
credentials
PII
tenant data
authorization state
browser session
server secrets
financial operations
source code
build pipeline
```

Potential entry points:

```text
URL
HTML
form
JSON
postMessage
API response
file upload
dependency
configuration
plugin
database
environment variable
```

Potential dangerous sinks:

```text
innerHTML
eval
Function
dynamic import
script loading
object property assignment
prototype mutation
command execution bridge
filesystem APIs
network APIs
```

---

# 5. Mental Model

Use:

```text
                    UNTRUSTED DATA
                         │
                         ↓
                  parse / validate
                         │
                         ↓
                  trusted structure
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
          objects      strings     URLs
              │          │          │
              ↓          ↓          ↓
         safe access   safe sink   allowlist
              │
              ↓
          capability
              │
              ↓
          controlled action
```

Security failure occurs when:

```text
data
 ↓
unexpected interpretation
 ↓
unexpected authority
```

---

# 6. JavaScript as Executable Authority

JavaScript is not merely data transformation.

A JavaScript object can contain:

```text
functions
getters
setters
proxies
symbols
prototype references
```

and therefore property access itself can execute code.

---

## Example

```js
const object = {
  get secret() {
    console.log("executed");
    return 42;
  }
};

object.secret;
```

The apparently harmless expression:

```js
object.secret
```

can invoke executable behavior.

---

# 7. Code vs Data

Security improves when data remains data.

Prefer:

```js
const operation = {
  type: "delete",
  id: "42"
};
```

over:

```js
const operation = `
  deleteRecord("42")
`;
```

Then:

```text
validate operation
→ dispatch allowed operation
```

instead of:

```text
execute string
```

---

# 8. Dynamic Code Execution

Dynamic execution includes:

```text
eval
Function constructor
string timers
dynamic script insertion
unsafe module loading
```

The fundamental danger:

```text
attacker-controlled string
→ JavaScript parser
→ executable code
```

---

# 9. `eval`

Avoid direct:

```js
eval(userInput);
```

MDN explicitly warns that direct `eval()` executes code with the privileges of its caller and can expose or alter the caller's local scope; it also creates optimization and security problems. citeturn219912search3

---

## Why It Is Dangerous

If:

```js
userInput = "stealSecrets()";
```

then:

```js
eval(userInput);
```

turns data into behavior.

---

## Performance

Modern JavaScript engines optimize statically analyzable code aggressively.

Direct `eval` complicates those assumptions and may inhibit optimizations. MDN documents both the security and optimization concerns. citeturn219912search3

---

# 10. `Function`

Example:

```js
const fn = new Function(
  "input",
  "return " + input
);
```

This is still dynamic code generation.

---

## Principle

```text
Function constructor
≈
runtime code generation
```

Treat it with the same security suspicion as `eval`.

---

# 11. String-Based Timers

Avoid:

```js
setTimeout("doSomething()", 1000);
```

Prefer:

```js
setTimeout(doSomething, 1000);
```

String timers create a code/data boundary that should not exist in normal application code.

---

# 12. Dynamic Module and Script Loading

Dynamic loading is not inherently insecure:

```js
await import("./feature.js");
```

The danger is allowing attacker-controlled module/script destinations.

Bad conceptual pattern:

```js
await import(userControlledUrl);
```

unless the security model explicitly permits that capability.

---

## Validate

Use:

```text
trusted module map
```

rather than arbitrary URL interpretation.

---

# 13. JavaScript Object Integrity

Objects can be:

```text
extensible
non-extensible
sealed
frozen
```

Security may require controlling mutation.

---

## Example

```js
const config = Object.freeze({
  secure: true
});
```

Now:

```js
config.secure = false;
```

cannot change the property in ordinary strict/suitable semantics.

---

# 14. Prototype Chains as an Attack Surface

Property lookup:

```js
object.flag
```

can traverse:

```text
object
 ↓
prototype
 ↓
prototype
 ↓
Object.prototype
```

MDN documents this prototype lookup model and explains how polluted inherited properties can unexpectedly affect application behavior. citeturn219912search0

---

# 15. Prototype Pollution

Prototype pollution occurs when attacker-controlled input causes properties to be added or modified on object prototypes.

MDN defines prototype pollution as an attack where an attacker can add or modify properties on an object's prototype, potentially causing logic errors or enabling attacks such as XSS. citeturn219912search0

---

## Two-Phase Model

```text
pollution
   ↓
prototype modified

exploitation
   ↓
application reads polluted property
```

MDN explicitly describes these two phases. citeturn219912search0

---

# 16. Pollution Sources

Common dangerous patterns include:

```js
object[key] = value;
```

when:

```js
key
```

is attacker-controlled.

Especially dangerous historical/property paths include:

```text
__proto__
constructor
prototype
```

MDN identifies dynamic property modification and these property paths as important prototype-pollution attack surfaces. citeturn219912search0

---

## Example

```js
function setValue(target, path, value) {
  target[path] = value;
}
```

Do not assume:

```text
path = harmless
```

when it comes from untrusted input.

---

# 17. Pollution Gadgets

Pollution alone may not immediately produce code execution.

An application becomes exploitable when its normal logic reads polluted properties in security-sensitive decisions.

Example conceptual gadget:

```js
const options = {};

if (options.enableAdmin) {
  grantAdmin();
}
```

If:

```text
Object.prototype.enableAdmin = true
```

then:

```js
options.enableAdmin
```

can unexpectedly become truthy.

The attack is:

```text
pollution
→ inherited value
→ vulnerable gadget
→ security impact
```

---

# 18. Defending Against Prototype Pollution

## Defense 1 — Validate Keys

Reject dangerous keys where dynamic assignment is necessary:

```text
__proto__
constructor
prototype
```

---

## Defense 2 — Validate Structure

Use schemas with:

```text
known fields
correct types
rejected additional properties
```

MDN specifically recommends schema validation, rejecting unneeded properties, and considering `additionalProperties: false` when appropriate. citeturn219912search0

---

## Defense 3 — Use Null-Prototype Objects

```js
const dict = Object.create(null);
```

---

## Defense 4 — Use Map

```js
const map = new Map();
```

---

## Defense 5 — Use Own-Property Checks

```js
Object.hasOwn(options, "enableAdmin");
```

---

## Defense 6 — Avoid Unbounded Deep Merge

Prefer explicit mapping:

```js
const config = {
  timeout: input.timeout,
  retries: input.retries
};
```

over blindly copying every attacker-controlled key.

---

# 19. `Object.create(null)` and Data Dictionaries

A null-prototype object has no ordinary prototype:

```js
const dictionary = Object.create(null);
```

Therefore:

```js
dictionary.toString
```

is not inherited from `Object.prototype`.

MDN notes that null-prototype objects can simultaneously avoid prototype-pollution paths and avoid prototype lookups. citeturn219912search0

---

## Example

```js
const counts = Object.create(null);

counts.apple = 1;
```

---

## Trade-off

You lose inherited convenience methods.

Use:

```js
Object.keys(counts)
Object.hasOwn(counts, "apple")
```

rather than assuming:

```js
counts.hasOwnProperty(...)
```

---

# 20. Map and Set as Security Design Choices

If the requirement is:

```text
key → value
```

consider:

```js
const map = new Map();
```

rather than a general-purpose object dictionary.

OWASP explicitly recommends considering `Map` and `Set` instead of plain object literals where appropriate for prototype-pollution defense. citeturn219912search2

---

# 21. Property Access and Own-Property Checks

Dangerous assumption:

```js
if (options.admin) {
  ...
}
```

Security-sensitive code may need:

```js
if (Object.hasOwn(options, "admin") &&
    options.admin === true) {
  ...
}
```

---

## Why

Inherited properties may not represent:

```text
explicit user input
```

or:

```text
explicit application state
```

---

# 22. Destructuring and Default-Value Hazards

Consider:

```js
function configure(options = {}) {
  if (options.enableDangerousAction) {
    // ...
  }
}
```

A polluted prototype can affect lookup.

A safer approach may define explicit defaults:

```js
function configure(
  options = {
    enableDangerousAction: false
  }
) {
  ...
}
```

MDN demonstrates this pattern as a defense against prototype-derived unexpected defaults. citeturn219912search0

---

# 23. Object Freezing, Sealing, and Integrity

## `Object.preventExtensions`

Stops adding new properties.

## `Object.seal`

Stops adding/removing properties and makes existing properties non-configurable.

## `Object.freeze`

Also makes data properties non-writable.

---

## Integrity Levels

Conceptual:

```text
ordinary
 ↓
non-extensible
 ↓
sealed
 ↓
frozen
```

---

## Important

These are shallow operations.

```js
const config = Object.freeze({
  nested: {
    value: 1
  }
});
```

The nested object can still be mutable unless separately frozen.

---

# 24. Deep Freezing and Transitive Trust

A recursive freeze can enforce stronger integrity:

```js
function deepFreeze(value, seen = new WeakSet()) {
  if (
    value === null ||
    (typeof value !== "object" &&
     typeof value !== "function") ||
    seen.has(value)
  ) {
    return value;
  }

  seen.add(value);

  for (const key of Reflect.ownKeys(value)) {
    deepFreeze(value[key], seen);
  }

  return Object.freeze(value);
}
```

---

## Caveats

Deep freezing can:

```text
cost CPU
break libraries
break expected mutation
interact with proxies/accessors
```

Use it selectively.

---

# 25. Getters, Setters, and Accessor Hazards

This:

```js
object.value
```

may execute:

```js
get value() {
  // code
}
```

Therefore object data received from another trust boundary should not automatically be assumed to be inert.

---

## Security Question

When passing an object across a capability boundary ask:

```text
Can property access execute code?
```

---

# 26. Proxies and Security

Proxy can intercept:

```text
get
set
has
ownKeys
defineProperty
getPrototypeOf
setPrototypeOf
deleteProperty
apply
construct
```

---

## Example

```js
const proxy = new Proxy(target, {
  get(target, prop) {
    audit(prop);
    return Reflect.get(target, prop);
  }
});
```

---

## Security Risks

A proxy can:

```text
observe access
change values
throw unexpectedly
trigger side effects
hide properties
```

Do not treat arbitrary objects as inert records.

---

# 27. Reflect and Safe Meta-Operations

When writing proxy handlers:

```js
return Reflect.get(target, property, receiver);
```

is often safer and more semantically complete than manually reproducing internal semantics.

---

## Principle

Understand:

```text
ordinary behavior
```

before modifying it with:

```text
Proxy
```

---

# 28. Symbols and Capability-Oriented APIs

Symbols can create hard-to-collide property keys:

```js
const secret = Symbol("secret");

object[secret] = value;
```

This is useful for avoiding accidental string-key collisions.

---

## Important

A Symbol is not a cryptographic secret.

Code with object access can discover symbols through:

```js
Object.getOwnPropertySymbols(object);
```

---

# 29. Closures as Encapsulation

A closure can hide state:

```js
function createCounter() {
  let value = 0;

  return {
    increment() {
      value++;
    },

    get() {
      return value;
    }
  };
}
```

The state is not directly exposed as an ordinary object property.

---

# 30. Private Class Fields

Modern JavaScript offers:

```js
class Vault {
  #token;

  constructor(token) {
    this.#token = token;
  }

  getToken() {
    return this.#token;
  }
}
```

---

## Benefit

Private fields provide language-level access restrictions for code that lacks the class's private-name authority.

---

## Caveat

This is encapsulation, not encryption.

---

# 31. Capability Security

A **capability** is an unforgeable authority reference that grants permission to perform some operation.

Conceptually:

```text
object reference
→ authority
```

If you give a function only:

```js
sendEmail
```

rather than:

```js
global
```

it has less authority.

---

# 32. Least Authority

Give code the minimum authority it needs.

Bad:

```js
plugin(globalEnvironment);
```

Better:

```js
plugin({
  log,
  readConfig,
  emitEvent
});
```

---

## Security Benefit

If plugin code is compromised:

```text
available capabilities
```

limit the damage.

---

# 33. Confused Deputy

A confused deputy has authority but is tricked into using it on behalf of an attacker.

Example:

```text
trusted service
+
attacker-controlled target
→
trusted service uses privileged capability
```

---

## JavaScript Example

```js
async function proxyFetch(url) {
  return fetch(url, {
    headers: {
      Authorization: adminToken
    }
  });
}
```

If the caller controls `url`, you may have created a privileged proxy.

The service becomes the deputy.

---

# 34. Object-Capability Thinking

Ask:

```text
What reference does this function receive?
What can that reference do?
Can the reference be passed elsewhere?
Can it be used to reach more authority?
```

---

## Secure API

Instead of:

```js
runTask(task, globalState)
```

prefer:

```js
runTask(task, {
  report,
  readOnlyConfig
});
```

---

# 35. Input Validation

Treat external values as untrusted:

```text
URL
JSON
query
headers
postMessage
database record
plugin configuration
environment variable
```

---

## Validation Stages

```text
receive
 ↓
parse
 ↓
validate shape
 ↓
validate semantics
 ↓
normalize
 ↓
authorize
 ↓
use
```

Do not combine all six steps into one giant helper.

---

# 36. Schema Validation

A schema can specify:

```text
required fields
types
allowed values
nested structure
additional properties
```

---

## Example

```js
const userSchema = {
  type: "object",
  additionalProperties: false,
  required: ["id", "name"],
  properties: {
    id: {
      type: "string"
    },
    name: {
      type: "string"
    }
  }
};
```

---

## Why Runtime Validation

TypeScript types disappear at runtime.

A network response remains untrusted at runtime.

Therefore:

```text
compile-time type
≠
runtime validation
```

---

# 37. Parsing and Deserialization

Parsing turns:

```text
bytes/string
```

into:

```text
structured value
```

Security risk depends on parser behavior.

Potential issues include:

```text
prototype manipulation
resource exhaustion
unexpected types
large nesting
large strings
ambiguous formats
```

---

# 38. JSON Safety and Limits

JSON is generally data-oriented, but application logic can still make unsafe assumptions.

Bad:

```js
const config = JSON.parse(input);

Object.assign(defaultConfig, config);
```

Better:

```text
parse
→ validate
→ explicitly map fields
```

---

## Limits

Consider limits for:

```text
body size
nesting
array lengths
string lengths
request rate
```

---

# 39. `structuredClone`

`structuredClone()` creates a structured clone of supported values.

Example:

```js
const copy = structuredClone(input);
```

---

## Security Value

Cloning can help prevent consumers from sharing mutable object identity.

But:

```text
clone ≠ validate
```

An attacker can still provide an object graph with dangerous size/shape.

---

# 40. Unsafe Merging and Deep Assignment

Dangerous abstraction:

```js
merge(defaults, untrustedInput);
```

without a security model.

Questions:

```text
Can special keys mutate prototypes?
Can getters execute?
Can arrays expand huge?
Can objects be deeply nested?
Can functions cross the boundary?
```

---

# 41. URL and Protocol Validation

Never assume:

```js
new URL(userInput).href
```

is automatically safe.

Validate:

```text
protocol
origin
hostname
port
path
```

---

## Example

```js
function isSafeHttpUrl(value) {
  const url = new URL(value);

  return (
    url.protocol === "https:" &&
    url.hostname === "api.example.com"
  );
}
```

Use an explicit allowlist where the application knows the intended destination.

---

# 42. Regular Expressions and ReDoS

A regex can become a denial-of-service vector when crafted input causes catastrophic backtracking.

Example concept:

```js
/(a+)+$/
```

with specially chosen input can cause pathological runtime in engines supporting the relevant backtracking semantics.

---

## Defense

Prefer:

```text
bounded input
simple regex
linear-time parser
safe regex review
timeouts/limits where available
```

Do not use regex as a universal parser.

---

# 43. Resource Exhaustion

Attackers can exploit:

```text
huge arrays
deep objects
large strings
large JSON
recursive traversal
expensive regex
CPU-heavy transformations
```

---

## Example

```js
while (input.length--) {
  expensiveOperation();
}
```

The security issue may be:

```text
attacker controls input size
```

---

# 44. Algorithmic Complexity Attacks

Suppose:

```text
input size = n
algorithm = O(n²)
```

At:

```text
n = 100
```

fine.

At:

```text
n = 1,000,000
```

potentially catastrophic.

---

## Security Model

Complexity becomes a security issue when:

```text
attacker controls n
```

and:

```text
cost grows rapidly
```

---

# 45. Denial of Service in JavaScript

Browser/client:

```text
main-thread freeze
memory pressure
render jank
watchdog termination
```

Server:

```text
CPU saturation
event-loop blocking
memory exhaustion
connection exhaustion
```

---

## Important

JavaScript's asynchronous APIs do not automatically prevent CPU denial of service.

A synchronous loop can still block everything.

---

# 46. Error Handling and Information Disclosure

Do not expose:

```text
stack traces
filesystem paths
internal URLs
database details
tokens
environment values
dependency versions
```

to untrusted users.

---

## Internal vs External Error

Internal:

```js
logger.error(error);
```

External:

```json
{
  "code": "INTERNAL_ERROR",
  "message": "An unexpected error occurred."
}
```

---

# 47. Logging Safety

Never casually log:

```text
Authorization
Cookie
session
password
API key
private token
raw personal data
```

---

## Structured Logging

Prefer:

```js
logger.info({
  event: "user.login.failed",
  userId,
  requestId,
  reason: "invalid_credentials"
});
```

over:

```js
console.log("request", request);
```

---

# 48. Serialization and Prototype Semantics

When serializing objects:

```js
JSON.stringify(value)
```

does not preserve every JavaScript semantic.

It does not serialize:

```text
functions
symbols
undefined object properties
prototypes
accessor behavior as behavior
```

---

## Security Principle

Never assume:

```text
serialize → deserialize
```

is identity-preserving.

---

# 49. Prototype-Safe API Design

Prefer explicit structures.

Bad:

```js
function configure(input) {
  return {
    ...defaults,
    ...input
  };
}
```

The spread operation has important semantics, but a production security review still needs to ask:

```text
Which properties are accepted?
Which values are expected?
Could unexpected properties alter downstream behavior?
```

Better:

```js
function configure(input) {
  return {
    timeout: Number.isInteger(input.timeout)
      ? input.timeout
      : 5000,

    retries: Number.isInteger(input.retries)
      ? input.retries
      : 3
  };
}
```

---

# 50. Dependency and Supply-Chain Security

Your application may execute:

```text
direct dependencies
transitive dependencies
build tools
bundler plugins
test tools
code generators
postinstall scripts
```

All can become security-relevant.

MDN categorizes supply-chain attacks as compromises of parts of the software supply chain used by a site. citeturn219912search4

---

# 51. Package Lifecycle Threats

A package can execute code during:

```text
install
build
test
runtime
```

---

## Build-Time Risk

A compromised build dependency may steal:

```text
CI secrets
cloud credentials
signing keys
environment variables
source code
```

even if the final browser bundle looks harmless.

---

# 52. Dependency Confusion and Name Resolution

Dependency confusion occurs when tooling resolves a malicious public package instead of an intended private/internal package due to naming/resolution configuration.

---

## Defense

Use:

```text
private registries
scoped package names
explicit registry configuration
lockfiles
dependency review
provenance
```

---

# 53. Lockfiles and Reproducibility

A lockfile captures a dependency graph/version resolution more precisely than a broad semver range.

Benefits:

```text
repeatability
reviewability
incident investigation
controlled updates
```

---

## But

A lockfile does not prove:

```text
package is safe
```

It proves:

```text
this build resolves to these artifacts
```

subject to registry/provenance integrity.

---

# 54. Postinstall and Arbitrary Build-Time Code

Package installation/build systems can execute scripts.

Therefore:

```text
npm install
```

is not merely:

```text
download files
```

It can become:

```text
execute third-party code
```

---

## Security Controls

Consider:

```text
trusted dependencies
script policy
CI isolation
secret minimization
sandboxed builds where appropriate
dependency allowlists
artifact review
```

---

# 55. Third-Party Packages as Trusted Code

If a package executes in the same process/origin with broad capability:

```text
package
≈
your code
```

from a trust perspective.

A small UI utility may technically gain access to:

```text
DOM
storage
network
application state
tokens
```

depending on the runtime and application.

---

# 56. Secrets and Configuration

Never embed true secrets in frontend bundles.

Examples:

```text
database password
private signing key
admin API secret
cloud root credential
```

---

## Configuration Classification

### Public configuration

```text
API origin
feature flag
public client ID
build version
```

### Secret configuration

```text
private key
server credential
database password
```

The browser should receive the first category only when intentionally public.

---

# 57. Environment Boundaries

A variable such as:

```text
process.env.SECRET
```

may be transformed into a browser bundle depending on tooling.

Therefore:

```text
environment variable
≠
secret
```

by itself.

The build system determines whether it is exposed.

---

# 58. JavaScript in the Browser vs Node.js

Same language:

```text
JavaScript
```

Different security environments:

### Browser

```text
SOP
CSP
DOM
cookies
Fetch
sandboxed origin model
```

### Node.js

```text
filesystem
process
network sockets
child processes
environment variables
native bindings
```

Node code can hold far more authority.

---

# 59. Sandboxing JavaScript

A critical question:

> Can I safely execute arbitrary attacker-controlled JavaScript inside my process?

In general, do not assume an ordinary JavaScript API boundary is a secure sandbox for hostile code.

---

## Stronger Boundary

When truly untrusted code must execute, consider:

```text
separate process
container
isolated service
restricted runtime
network isolation
OS-level policy
```

rather than trusting language-level cleverness alone.

---

# 60. `vm`-Style Isolation and Its Limits

Node.js provides VM-related APIs, but an ordinary VM context should not automatically be interpreted as a complete security boundary against malicious code.

Security depends on:

```text
runtime version
available APIs
escape paths
host integration
native addons
resource controls
network/filesystem exposure
```

---

## Principal Rule

For hostile code:

```text
language sandbox
< 
OS/process isolation
```

as a general security design principle.

---

# 61. Realms, Compartments, and SES Concepts

A **Realm** conceptually provides a distinct JavaScript global environment.

**Compartments** are a higher-level security architecture for controlling module execution and capabilities.

**SES (Secure ECMAScript)** is an object-capability-oriented hardening approach built around concepts such as:

```text
lockdown
Compartments
taming
least authority
```

MDN's prototype-pollution guidance notes “realm lockdown” and SES-style hardening as a high-sensitivity defense that can prevent modification of built-in objects. citeturn219912search0

---

## Important

These concepts do not imply:

```text
ordinary iframe/Realm/VM = perfect sandbox
```

Security depends on the actual host boundary and capabilities exposed.

---

# 62. Dynamic Plugin Systems

Suppose an application loads:

```text
plugin.js
```

Ask:

```text
What can it read?
What can it call?
What can it import?
Can it access DOM?
Can it access storage?
Can it send network requests?
Can it modify global state?
```

---

## Better

Give plugins:

```js
createPluginContext({
  log,
  ui,
  storage
});
```

with carefully constrained interfaces.

---

# 63. Untrusted Code Execution

For truly hostile user-supplied code:

```text
do not execute in the main trusted process
```

unless you have a rigorously reviewed isolation architecture.

Safer design:

```text
untrusted code
 ↓
isolated execution environment
 ↓
resource limits
 ↓
explicit message protocol
 ↓
trusted host
```

---

# 64. Tainted Data and Dataflow

Think in terms of:

```text
source
→ transformation
→ sink
```

Example:

```text
location.search
→ parse
→ string
→ innerHTML
```

Danger.

Another:

```text
request body
→ JSON.parse
→ dynamic object assignment
→ prototype pollution
```

Danger.

Another:

```text
user input
→ URL
→ allowlist
→ fetch
```

Potentially safe if validation is correct.

---

# 65. Secure API Boundaries

A secure function should define:

```text
accepted inputs
accepted types
accepted authority
outputs
errors
side effects
resource limits
```

---

## Example

```js
async function fetchUser(
  userId,
  {
    signal
  } = {}
) {
  if (!/^[a-zA-Z0-9_-]{1,64}$/.test(userId)) {
    throw new TypeError("Invalid user ID");
  }

  return api.get(`/users/${encodeURIComponent(userId)}`, {
    signal
  });
}
```

---

# 66. Security-Sensitive Defaults

Prefer secure-by-default behavior.

Examples:

```text
HTTPS over HTTP
no arbitrary eval
bounded input
no broad object merging
explicit origins
least privilege
minimal dependencies
safe serialization
```

---

# 67. Secure Library Design

A secure library should:

```text
minimize authority
avoid surprising side effects
avoid dynamic code execution
validate external input
document trust assumptions
produce structured errors
avoid logging secrets
support cancellation
bound expensive operations
```

---

## Public API Review

Ask:

```text
Can callers accidentally create an unsafe state?
Can malicious input reach a code sink?
Can a dependency escape its intended scope?
Can a failure expose secrets?
```

---

# 68. Browser Integration

Browser security controls from Chapter 56 remain necessary:

```text
SOP
CORS
CSP
Trusted Types
cookies
CSRF
COOP/COEP/CORP
```

JavaScript-level controls complement them.

---

# 69. Web Workers and Security

Workers isolate DOM access but still execute application code under the relevant origin/security model.

They can reduce UI-thread exposure to CPU-heavy processing.

They do not automatically mean:

```text
untrusted worker = secure sandbox
```

---

# 70. Web Components and Security

From Chapter 54:

```text
Shadow DOM
```

provides encapsulation.

It does not provide:

```text
trusted execution boundary
```

A component that executes with page authority remains part of the page's trusted code.

---

# 71. Node Integration Preview

Node-specific security concerns will become central in Chapters 58–63:

```text
filesystem
process
child_process
worker_threads
network
DNS
environment variables
native addons
diagnostics
```

The key transition:

```text
browser JS
→ limited host authority

Node JS
→ potentially powerful host authority
```

---

# 72. Observability and Incident Response

Security observability should record enough to answer:

```text
what happened?
when?
where?
who?
which build?
which dependency?
which request?
which user/tenant?
```

---

## Avoid

```text
raw tokens
full cookies
secrets
sensitive payloads
```

---

## Useful Fields

```text
requestId
traceId
buildId
dependency graph version
user/tenant identifier where appropriate
security event type
decision outcome
```

---

# 73. Static Analysis and Secure Tooling

Use:

```text
linters
SAST
dependency scanners
secret scanners
lockfile verification
Semgrep
CodeQL
npm audit / ecosystem scanners
SBOM
```

where appropriate.

---

## But

Static analysis is:

```text
evidence
```

not:

```text
proof of security
```

---

# 74. Security Testing Strategy

Use layers:

```text
unit tests
property tests
fuzzing
integration tests
security regression tests
dependency checks
dynamic testing
manual review
penetration testing
```

---

## Property-Based Example

For a parser:

```text
never throw catastrophic error
never allocate unboundedly
reject malformed input
preserve invariants
```

---

# 75. Performance/Security Trade-offs

Security controls can cost:

```text
CPU
latency
memory
developer complexity
compatibility
```

Examples:

```text
schema validation
deep freezing
sanitization
cryptographic verification
resource limits
```

---

## Principal Decision

Measure:

```text
security benefit
+
threat likelihood
+
impact
+
runtime cost
+
operational cost
```

---

# 76. Memory/Security Trade-offs

Security bugs can become memory bugs.

Examples:

```text
unbounded JSON
large arrays
attacker-controlled cache
detached sensitive objects
long-lived secrets
```

---

## Principle

Use:

```text
size limits
TTL
bounded queues
streaming
cleanup
minimum retention
```

---

# 77. Production Security Architecture

A mature JavaScript security architecture:

```text
                       External Input
                            │
                            ↓
                    Parse / Normalize
                            │
                            ↓
                     Schema Validation
                            │
                            ↓
                    Authorization Check
                            │
            ┌───────────────┼───────────────┐
            ↓               ↓               ↓
        Data APIs       Capability APIs   UI APIs
            │               │               │
            ↓               ↓               ↓
       safe objects     least authority   safe sinks
            │
            ↓
     bounded operations
            │
            ↓
      observable actions
            │
            ↓
       secure output
```

---

## Principal Security Framework

For every feature ask:

```text
1. What is the trust boundary?
2. What is attacker-controlled?
3. What is the dangerous sink?
4. What authority is available?
5. Can data become code?
6. Can a prototype be modified?
7. Can resource use become unbounded?
8. Can dependencies alter the behavior?
9. Can secrets escape?
10. What is the server-side enforcement?
11. How is abuse detected?
12. How is the feature disabled during incident response?
```

---

# 78. Anti-Patterns

## Anti-Pattern 1

```js
eval(userInput);
```

---

## Anti-Pattern 2

```js
new Function(userInput);
```

---

## Anti-Pattern 3

```js
setTimeout(userInput, 1000);
```

---

## Anti-Pattern 4

```js
target[key] = value;
```

where `key` is uncontrolled.

---

## Anti-Pattern 5

```js
merge(defaults, userInput);
```

without schema/key validation.

---

## Anti-Pattern 6

```js
if (options.isAdmin) {
  ...
}
```

without an own-property/security-aware model.

---

## Anti-Pattern 7

```js
Object.freeze(config);
```

and assuming nested values are frozen.

---

## Anti-Pattern 8

Treating:

```text
Symbol
private field
closed shadow root
```

as cryptographic secrets.

---

## Anti-Pattern 9

Executing arbitrary plugins in the primary process.

---

## Anti-Pattern 10

Logging all requests and responses.

---

## Anti-Pattern 11

Assuming TypeScript types validate network data.

---

## Anti-Pattern 12

Assuming a lockfile proves dependency security.

---

## Anti-Pattern 13

Putting secrets into frontend configuration.

---

## Anti-Pattern 14

Using a browser security control as a substitute for server authorization.

---

# 79. Debugging Methodology

## Step 1 — Identify the Source

Examples:

```text
URL
user input
network response
database
dependency
plugin
configuration
```

---

## Step 2 — Track Transformations

```text
parse
decode
deserialize
merge
normalize
map
```

---

## Step 3 — Identify the Sink

```text
HTML
eval
Function
dynamic import
URL
filesystem
network
prototype assignment
command execution
```

---

## Step 4 — Identify Authority

```text
DOM
storage
filesystem
network
credentials
tenant data
```

---

## Step 5 — Determine Attacker Control

Ask exactly:

```text
Can the attacker control the source?
Fully?
Partially?
Indirectly?
```

---

## Step 6 — Reproduce with Minimal Input

Security debugging improves when you minimize:

```text
one source
one transformation
one sink
one payload
```

---

# 80. Implementation From Scratch

Build a **Toy JavaScript Security Boundary Framework**.

## Stage 1 — Trust-Labeled Data

Create:

```js
class TaintedValue {
  constructor(value) {
    this.value = value;
    this.tainted = true;
  }
}
```

Represent:

```text
tainted
validated
trusted
```

as conceptual states.

---

## Stage 2 — Validator

Implement:

```js
function validateUserId(value) {
  if (!/^[A-Za-z0-9_-]{1,64}$/.test(value)) {
    throw new Error("Invalid ID");
  }

  return value;
}
```

---

## Stage 3 — Safe Object Builder

Accept only:

```text
known keys
known types
known ranges
```

---

## Stage 4 — Prototype-Safe Dictionary

Implement:

```js
Object.create(null)
```

and compare with ordinary objects.

---

## Stage 5 — Safe Merger

Reject:

```text
__proto__
constructor
prototype
```

and unknown fields.

---

## Stage 6 — Capability Object

Implement:

```js
function createCapability({
  send
}) {
  return Object.freeze({
    send
  });
}
```

Give plugins only this object.

---

## Stage 7 — Plugin Host

Create:

```text
trusted host
+
plugin
+
restricted capability object
```

Demonstrate how limiting references reduces authority.

---

## Stage 8 — Resource Budget

Implement:

```text
max input size
max operations
max execution time
max queue length
```

---

## Stage 9 — Safe Logger

Redact:

```text
Authorization
Cookie
token
password
secret
```

before output.

---

## Stage 10 — Security Event Model

Emit:

```js
{
  type: "security.validation_failed",
  requestId,
  source,
  rule
}
```

without including secrets.

---

## Stage 11 — Dependency Policy

Create a build manifest:

```text
package
version
source
integrity
reviewedAt
owner
risk
```

---

## Stage 12 — Principal Security Harness

Given:

```text
input
parser
transformers
sinks
capabilities
resources
dependencies
```

produce:

```text
trust graph
threats
controls
residual risk
```

---

# 81. Debugging Exercises

## Exercise 1 — `eval`

```js
const input = location.hash.slice(1);

eval(input);
```

Identify:

```text
source
sink
authority
impact
remediation
```

---

## Exercise 2 — Dynamic Property

```js
function set(target, key, value) {
  target[key] = value;
}
```

Assume `key` is supplied from JSON.

What security issue should be investigated?

---

## Exercise 3 — Prototype Pollution

Given:

```js
const input = JSON.parse(payload);

for (const key of Object.keys(input)) {
  config[key] = input[key];
}
```

What must be checked?

---

## Exercise 4 — Inherited Property

```js
const options = {};

if (options.debug) {
  enableSensitiveLogging();
}
```

Explain why a polluted prototype could change behavior.

---

## Exercise 5 — Null Prototype

```js
const dict = Object.create(null);

dict.foo = "bar";
```

Which inherited methods are absent?

---

## Exercise 6 — Getter

```js
const input = {
  get value() {
    throw new Error("executed");
  }
};

console.log(input.value);
```

Explain why “object” does not necessarily mean inert data.

---

## Exercise 7 — Proxy

Create a Proxy that changes:

```js
config.timeout
```

unexpectedly.

Then identify how the behavior would complicate security review.

---

## Exercise 8 — `structuredClone`

Clone a complex object and explain why cloning does not replace validation.

---

## Exercise 9 — Regex DoS

Create a local test harness for a pathological regex and measure runtime as input length grows.

Do not run untrusted payloads against production systems.

---

## Exercise 10 — Capability Leak

```js
plugin({
  api,
  storage,
  logger,
  window
});
```

What authority is unnecessarily exposed?

---

## Exercise 11 — Confused Deputy

Build a helper that uses an admin credential to call a URL supplied by a low-privilege caller.

Explain the deputy problem and redesign it.

---

## Exercise 12 — Dependency Risk

Choose one third-party package.

Map:

```text
who publishes it
what it executes
what dependencies it has
when scripts run
what secrets the build can access
```

---

# 82. Code Review Exercise

Review:

```js
export function applyConfig(config, input) {
  const parsed =
    typeof input === "string"
      ? JSON.parse(input)
      : input;

  Object.assign(config, parsed);

  if (config.debug) {
    console.log("Config:", config);
  }

  if (config.template) {
    document.body.innerHTML = config.template;
  }

  if (config.plugin) {
    const Plugin = new Function(
      "return " + config.plugin
    )();

    return new Plugin();
  }

  return config;
}
```

The developer says:

> “It accepts flexible configuration and plugins.”

It is a critical security hazard.

## Problems

1. Unvalidated parsing.
2. Unbounded object assignment.
3. Potential prototype pollution interactions.
4. Inherited properties can influence security-sensitive checks.
5. Sensitive configuration may be logged.
6. `innerHTML` is an XSS sink.
7. Arbitrary runtime code generation through `Function`.
8. No capability boundary for plugins.
9. Plugin code receives full process/page authority.
10. No resource limits.
11. No schema.
12. No allowlist.
13. No authentication/authorization model.
14. No safe error policy.
15. No audit trail.
16. No dependency/plugin provenance strategy.

A stronger architecture is:

```text
untrusted config
 ↓
parse
 ↓
schema validation
 ↓
explicit mapping
 ↓
safe state
 ↓
capability-limited plugin
 ↓
approved operations
 ↓
safe rendering
```

---

# 83. Interview Questions

## Foundations

1. What does JavaScript security engineering mean?
2. Why is “code vs data” a security boundary?
3. Why is JavaScript object access not always inert?
4. What are getters?
5. What are proxies?
6. Why can property lookup itself execute code?

## Dynamic Code

7. Why is eval dangerous?
8. Why is Function dangerous?
9. Are string timers dangerous?
10. Why is dynamic import not automatically safe?
11. When might dynamic code execution ever be justified?

## Prototype Pollution

12. What is prototype pollution?
13. What are common pollution sources?
14. What is a pollution gadget?
15. Why are `__proto__`, `constructor`, and `prototype` important?
16. How do null-prototype objects help?
17. Map vs object for dictionaries?
18. Why do own-property checks matter?

## Integrity

19. `preventExtensions` vs `seal` vs `freeze`?
20. Is Object.freeze deep?
21. When would you deep-freeze?
22. What are the costs?

## Capability Security

23. What is least authority?
24. What is a confused deputy?
25. What is an object capability?
26. How do closures provide encapsulation?
27. How do private fields help?

## Input / Parsing

28. Why don't TypeScript types validate network data?
29. Why use schema validation?
30. What is unsafe object merging?
31. Why is structuredClone not validation?
32. How do you safely validate URLs?

## DoS

33. What is ReDoS?
34. What is algorithmic complexity attack?
35. How can JavaScript block an entire server?
36. How do you bound attacker-controlled resource use?

## Supply Chain

37. What is a supply-chain attack?
38. What is dependency confusion?
39. Why is a lockfile not proof of trust?
40. What can a postinstall script do?
41. What is build-time supply-chain risk?

## Runtime Isolation

42. Browser JS vs Node JS security?
43. Is Node VM a sandbox?
44. When should untrusted code use process/container isolation?
45. What are Realms and Compartments conceptually?
46. What is SES-style hardening?

## Principal-Level

47. Design a plugin system for untrusted extensions.
48. Design a secure configuration loader.
49. Design a prototype-pollution-resistant merge system.
50. Threat-model a JavaScript build pipeline.
51. Secure a multi-tenant plugin architecture.
52. How would you prove your component has least authority?
53. How would you respond to a compromised npm dependency?
54. How would you bound CPU/memory use for hostile input?
55. How would you distinguish a browser security control from a JavaScript-level security control?

---

# 84. Predict-the-Outcome Exercises

## Exercise 1

```js
const object = {};

console.log("toString" in object);
console.log(Object.hasOwn(object, "toString"));
```

Predict the conceptual difference.

---

## Exercise 2

```js
const dict = Object.create(null);

console.log(dict.toString);
```

Predict.

---

## Exercise 3

```js
Object.prototype.debug = true;

const config = {};

console.log(config.debug);
```

Predict.

Then explain why this can be a security problem.

---

## Exercise 4

```js
const config = Object.freeze({
  nested: {
    enabled: true
  }
});

config.nested.enabled = false;
```

Is nested state necessarily frozen?

---

## Exercise 5

```js
const input = {
  get value() {
    console.log("getter");
    return 42;
  }
};

console.log(input.value);
```

Predict.

---

## Exercise 6

```js
const secret = Symbol("secret");

const object = {
  [secret]: 123
};

console.log(Object.getOwnPropertySymbols(object).length);
```

Predict.

---

## Exercise 7

```js
const input = {
  __proto__: {
    isAdmin: true
  }
};

console.log(input.isAdmin);
```

Carefully distinguish object-literal prototype syntax from generic attacker-controlled property assignment.

---

## Exercise 8

```js
const input = JSON.parse(
  '{"constructor":{"prototype":{"polluted":true}}}'
);

console.log(input.constructor);
```

Explain why parsing alone is not the same thing as prototype pollution; identify where the later unsafe assignment/merge would matter.

---

## Exercise 9

```js
const plugin = Object.freeze({
  read: () => "ok"
});
```

Does freezing automatically make the entire authority graph safe?

---

## Exercise 10

```js
const data = structuredClone({
  nested: {
    value: 1
  }
});
```

Does structured cloning validate that `nested.value` is safe for your business logic?

---

# 85. Mastery Exercises

## Level 1 — Safe Dynamic Data

Rewrite unsafe configuration handling so only approved properties are accepted.

---

## Level 2 — Prototype Pollution

Build a vulnerable merge function in a local lab.

Then harden it with:

```text
schema
key allowlist
null-prototype objects
explicit mapping
```

---

## Level 3 — Capability Object

Build a plugin that receives only:

```text
log
readConfig
emit
```

and prove it cannot call unrelated operations through the exposed references.

---

## Level 4 — Confused Deputy

Build:

```text
low-privilege caller
high-privilege helper
```

that becomes vulnerable.

Then enforce:

```text
destination allowlist
capability scoping
authorization
```

---

## Level 5 — Secure Parser

Create a parser that enforces:

```text
maximum input size
maximum array size
maximum nesting
known keys
known types
```

---

## Level 6 — ReDoS Lab

Benchmark:

```text
safe linear regex
vs
pathological backtracking regex
```

against increasing input lengths.

---

## Level 7 — Dependency Audit

Produce an SBOM-style table:

| Package | Direct/Transitive | Runtime/Build | Maintainer/Source | Script Hooks | Risk | Owner |
|---|---|---|---|---|---|---|

---

## Level 8 — Secure Plugin Platform

Design:

```text
plugin manifest
capabilities
permissions
version policy
resource quotas
message protocol
audit logs
revocation
```

---

## Level 9 — Secure Configuration System

Design:

```text
remote config
schema validation
defaults
versioning
signature/integrity
rollback
auditability
```

---

## Level 10 — Browser + Node Threat Model

Map the exact differences between:

```text
browser plugin
Node plugin
```

with respect to:

```text
DOM
network
filesystem
process
storage
secrets
```

---

## Level 11 — Supply-Chain Incident

Simulate:

```text
dependency compromise
→ malicious release
→ CI execution
→ secret exposure
→ release artifact
```

Develop:

```text
detection
containment
revocation
rollback
credential rotation
forensics
communication
```

---

## Level 12 — Principal JavaScript Security Architecture

Design a secure architecture for a:

```text
multi-tenant enterprise SaaS
```

with:

```text
browser SPA
Node backend
custom plugins
customer configuration
third-party packages
SSR
Web Workers
Web Components
background jobs
CI/CD
```

Deliver:

```text
threat model
trust boundaries
capability map
input validation architecture
prototype-pollution defense
XSS/CSP strategy
dependency policy
plugin isolation
resource limits
secret management
observability
incident response
```

---

# 86. Key Takeaways

1. JavaScript security is fundamentally about controlling execution, authority, mutation, data interpretation, and resource use.
2. Treat attacker-controlled strings as data, not code.
3. Avoid `eval`, `Function`, and string-based timers.
4. Dynamic module/script loading needs an explicit trust model.
5. JavaScript property access can execute getters or proxy traps.
6. Prototype chains are a real attack surface.
7. Prototype pollution has a pollution phase and an exploitation/gadget phase.
8. `__proto__`, `constructor`, and `prototype` deserve special handling in dynamic-key code.
9. Validate input shape and reject unnecessary properties.
10. Explicit mapping is often safer than broad object merging.
11. `Object.create(null)` can be appropriate for dictionaries.
12. `Map` and `Set` can avoid object-prototype pitfalls for suitable data models.
13. Use `Object.hasOwn()` when you need own-property semantics.
14. Default property lookups can be influenced by polluted prototypes.
15. `Object.freeze()` is shallow.
16. Deep freezing can improve integrity but costs CPU and compatibility.
17. Getters and setters are executable behavior.
18. Proxies can alter fundamental object operations.
19. Symbols are collision-resistant keys, not secrets.
20. Closures and private fields can reduce mutable public surface.
21. Capability security means limiting references and authority.
22. Least authority reduces blast radius.
23. Confused deputies occur when privileged helpers act on attacker-selected targets.
24. Runtime validation is required for untrusted external data even in TypeScript applications.
25. Parsing and validation are separate operations.
26. `structuredClone()` is not validation.
27. URL validation requires protocol/origin/destination constraints.
28. Regular expressions can create denial-of-service risk.
29. Attacker-controlled sizes turn algorithmic complexity into a security issue.
30. JavaScript can suffer CPU, memory, queue, and event-loop denial of service.
31. Errors and logs can become information leaks.
32. Third-party dependencies are trusted executable code.
33. Build-time dependencies can compromise CI before browser runtime.
34. Dependency confusion exploits package-resolution ambiguity.
35. Lockfiles improve reproducibility but do not establish trust.
36. Package lifecycle scripts can execute arbitrary code during installation/build.
37. Frontend configuration is not a secret container.
38. Browser and Node JavaScript have radically different host authority.
39. Ordinary Node VM contexts should not be assumed to be security sandboxes for hostile code.
40. Strong untrusted-code isolation generally requires stronger process/OS/runtime boundaries.
41. Realms/Compartments/SES are security architecture concepts, not magical isolation switches.
42. A secure plugin system should expose narrow capabilities rather than global authority.
43. Security should be analyzed as a source → transformation → sink dataflow.
44. Security-sensitive APIs need explicit trust assumptions and resource limits.
45. Static analysis is valuable evidence, not a proof of security.
46. JavaScript security must be integrated with browser security and server authorization.
47. Principal-level security engineering requires reasoning about exploitability, blast radius, controls, residual risk, and operations.

---

# 87. Concept Connections

## Depends On

```text
Chapter 15 — Objects
       ↓
Chapter 17 — Prototypes
       ↓
Chapter 19 — Proxy / Reflect
       ↓
Chapter 24 — Map / Set
       ↓
Chapter 28 — JSON / Serialization
       ↓
Chapter 42 — Abstract Operations
       ↓
Chapter 43 — Internal Methods
       ↓
Chapter 45 — Memory / GC
       ↓
Chapter 47 — Engine Architecture
       ↓
Chapter 48 — V8 Optimization
       ↓
Chapter 54 — Web Components
       ↓
Chapter 55 — Fetch
       ↓
Chapter 56 — Browser Security
       ↓
Chapter 57 — JavaScript Security Engineering
```

## Builds Toward

```text
Chapter 58 — Node Architecture
Chapter 59 — Node Core APIs
Chapter 60 — Node Streams
Chapter 61 — Worker Threads / Processes
Chapter 62 — Process Lifecycle
Chapter 63 — Async Context / Diagnostics
Chapter 67 — Dependency Management / Supply Chain
Chapter 78 — Production Architecture
Chapter 79 — API Design
Chapter 80 — Library Authoring
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

```text
XSS
CSP
Trusted Types
prototype pollution
ReDoS
supply chain
capability security
least authority
confused deputy
sandboxing
compartments
SES
tainted data
schema validation
resource limits
secure defaults
```

## Concepts Revisited

### Chapter 17 — Prototypes

Prototype inheritance becomes a concrete security boundary.

### Chapter 19 — Proxy / Reflect

Meta-programming can create hidden executable behavior.

### Chapter 24 — Map / Set

Data-structure choice can become a security control.

### Chapter 28 — Serialization

Serialization changes object semantics and must not be confused with validation.

### Chapter 45 — Memory

Unbounded attacker-controlled structures create both reliability and security risks.

### Chapter 55 — Fetch

Network data must be treated as untrusted until validated.

### Chapter 56 — Browser Security

Browser-level isolation and JavaScript-level controls work together.

---

## Why This Chapter Matters Later

This chapter changes how you review JavaScript.

Instead of:

```text
Does the code work?
```

ask:

```text
What can the attacker control?
What code can execute?
What objects can mutate?
What authority is reachable?
What resource cost can be forced?
What secrets are exposed?
What dependency can change behavior?
```

This becomes the foundation for secure Node.js architecture in the next part of the curriculum.

---

# 88. Completion Criteria

## Fundamentals

- [ ] Explain JavaScript security engineering.
- [ ] Explain code vs data.
- [ ] Identify trust boundaries.
- [ ] Identify sources and sinks.

## Dynamic Execution

- [ ] Explain eval.
- [ ] Explain Function.
- [ ] Explain string timers.
- [ ] Explain dynamic module loading.
- [ ] Identify arbitrary-code risks.

## Objects

- [ ] Explain prototype security.
- [ ] Explain prototype pollution.
- [ ] Identify dangerous dynamic keys.
- [ ] Explain pollution gadgets.
- [ ] Use null-prototype objects.
- [ ] Evaluate Map/Set.
- [ ] Use own-property checks.
- [ ] Explain freeze/seal.
- [ ] Explain deep freeze trade-offs.
- [ ] Identify getter/proxy hazards.

## Capability Security

- [ ] Explain capabilities.
- [ ] Explain least authority.
- [ ] Explain confused deputy.
- [ ] Design narrow capability objects.
- [ ] Use closures/private fields for encapsulation.

## Input / Data

- [ ] Validate external data.
- [ ] Use runtime schemas.
- [ ] Separate parsing and validation.
- [ ] Understand structuredClone limits.
- [ ] Defend object merging.
- [ ] Validate URLs/protocols.

## Resource Security

- [ ] Explain ReDoS.
- [ ] Identify algorithmic complexity attacks.
- [ ] Bound input sizes.
- [ ] Bound array/nesting sizes.
- [ ] Prevent event-loop blocking where feasible.

## Supply Chain

- [ ] Explain supply-chain attacks.
- [ ] Explain dependency confusion.
- [ ] Understand lockfiles.
- [ ] Review lifecycle scripts.
- [ ] Audit build dependencies.
- [ ] Treat third-party code as trusted code.

## Runtime Isolation

- [ ] Browser vs Node security model.
- [ ] Explain VM isolation limits.
- [ ] Explain process/container isolation.
- [ ] Explain Realm/Compartment/SES concepts.
- [ ] Design a plugin boundary.

## Secure APIs

- [ ] Define accepted input.
- [ ] Define authority.
- [ ] Define resource limits.
- [ ] Define error contract.
- [ ] Define auditability.

## Testing / Tooling

- [ ] Use SAST.
- [ ] Use dependency scanning.
- [ ] Use secret scanning.
- [ ] Use fuzz/property testing.
- [ ] Write security regression tests.

## Principal Judgment

- [ ] Threat-model a JS platform.
- [ ] Design least-authority plugins.
- [ ] Defend prototype-pollution strategy.
- [ ] Defend sandboxing strategy.
- [ ] Defend dependency policy.
- [ ] Defend resource limits.
- [ ] Defend secret handling.
- [ ] Defend browser/Node boundaries.
- [ ] Defend residual risk.

---

# 89. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What is JavaScript security engineering?
2. Why is code/data separation important?
3. Why is eval dangerous?
4. What is dynamic code generation?
5. What is prototype pollution?
6. What is a pollution source?
7. What is a pollution gadget?
8. Why are `__proto__`, `constructor`, and `prototype` dangerous?
9. When should you use `Object.create(null)`?
10. Map vs object dictionary?
11. Why use Object.hasOwn?
12. Is Object.freeze deep?
13. Why can getters be dangerous?
14. Why can Proxies be dangerous?
15. Are Symbols secrets?
16. What is least authority?
17. What is confused deputy?
18. Why are TypeScript types insufficient for runtime validation?
19. What is unsafe object merging?
20. Is structuredClone validation?
21. What is ReDoS?
22. What is algorithmic complexity attack?
23. What is supply-chain compromise?
24. What is dependency confusion?
25. Why is a lockfile not proof of trust?
26. What can postinstall do?
27. Why are browser and Node authority different?
28. Is Node VM a secure sandbox?
29. What are Compartments conceptually?
30. How do you design a secure plugin system?
31. What is a source-to-sink dataflow model?
32. What should a secure API contract specify?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Code/data boundary | [ ] | [ ] | [ ] | [ ] |
| eval | [ ] | [ ] | [ ] | [ ] |
| Function | [ ] | [ ] | [ ] | [ ] |
| dynamic loading | [ ] | [ ] | [ ] | [ ] |
| prototypes | [ ] | [ ] | [ ] | [ ] |
| prototype pollution | [ ] | [ ] | [ ] | [ ] |
| pollution gadgets | [ ] | [ ] | [ ] | [ ] |
| null-prototype objects | [ ] | [ ] | [ ] | [ ] |
| Map/Set security | [ ] | [ ] | [ ] | [ ] |
| own-property checks | [ ] | [ ] | [ ] | [ ] |
| freeze/seal | [ ] | [ ] | [ ] | [ ] |
| getters | [ ] | [ ] | [ ] | [ ] |
| Proxies | [ ] | [ ] | [ ] | [ ] |
| capability security | [ ] | [ ] | [ ] | [ ] |
| least authority | [ ] | [ ] | [ ] | [ ] |
| confused deputy | [ ] | [ ] | [ ] | [ ] |
| validation | [ ] | [ ] | [ ] | [ ] |
| schema | [ ] | [ ] | [ ] | [ ] |
| serialization | [ ] | [ ] | [ ] | [ ] |
| URL validation | [ ] | [ ] | [ ] | [ ] |
| ReDoS | [ ] | [ ] | [ ] | [ ] |
| resource exhaustion | [ ] | [ ] | [ ] | [ ] |
| errors | [ ] | [ ] | [ ] | [ ] |
| logging | [ ] | [ ] | [ ] | [ ] |
| supply chain | [ ] | [ ] | [ ] | [ ] |
| dependency confusion | [ ] | [ ] | [ ] | [ ] |
| lockfiles | [ ] | [ ] | [ ] | [ ] |
| lifecycle scripts | [ ] | [ ] | [ ] | [ ] |
| secrets | [ ] | [ ] | [ ] | [ ] |
| browser vs Node | [ ] | [ ] | [ ] | [ ] |
| sandboxing | [ ] | [ ] | [ ] | [ ] |
| VM limitations | [ ] | [ ] | [ ] | [ ] |
| Compartments | [ ] | [ ] | [ ] | [ ] |
| plugin isolation | [ ] | [ ] | [ ] | [ ] |
| tainted data | [ ] | [ ] | [ ] | [ ] |
| secure API design | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw:

```text
untrusted input
 ↓
parse
 ↓
validate
 ↓
safe object
 ↓
least authority
 ↓
controlled sink
```

### Day 2

Explain:

```text
prototype pollution
vs
XSS
vs
CSRF
vs
supply-chain compromise
```

### Day 7

Build a vulnerable merge function and harden it.

### Day 14

Design a capability-limited plugin system.

### Day 30

Threat-model a production JavaScript platform from dependency installation through runtime.

---

# 90. Canonical References and Source Discipline

## MDN — JavaScript Security

Use MDN for practical browser/JavaScript security guidance and current API semantics.

### `eval()`

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval

MDN currently warns that direct eval executes input with the caller's privileges and creates security/performance problems. citeturn219912search3

---

## MDN — Prototype Pollution

https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Prototype_pollution

Use for:

```text
prototype lookup
pollution phases
sources
defenses
null-prototype objects
Node --disable-proto
realm lockdown
```

MDN describes prototype pollution as modification of object prototypes and recommends input validation, rejecting unexpected properties, own-property checks, null-prototype objects, and other mitigations. citeturn219912search0

---

## OWASP — Prototype Pollution Prevention

https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html

Use for practical application-defense guidance, including Map/Set and object-integrity strategies. citeturn219912search2

---

## OWASP — DOM XSS Prevention

https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html

Use for:

```text
sources
sinks
context-sensitive encoding
safe DOM APIs
dataflow
```

OWASP recommends safe DOM construction APIs and specifically recommends `textContent` for rendering untrusted text rather than unsafe HTML sinks. citeturn219912search1

---

## OWASP — Cross-Site Scripting Prevention

https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html

Use for defense-in-depth around:

```text
encoding
sanitization
framework defenses
CSP
Trusted Types
```

OWASP explicitly emphasizes that no single mechanism solves XSS and recommends layered defenses. citeturn219912search5

---

## WHATWG ECMAScript / JavaScript Standards

Use the ECMAScript specification for:

```text
objects
property access
prototype semantics
Proxy
Reflect
private fields
descriptors
integrity
```

Use host-specific standards separately for browser/Node security claims.

---

## TC39 / SES Boundary

When discussing:

```text
Realms
Compartments
SES
```

distinguish:

```text
standardized ECMAScript feature
proposal
host API
third-party security framework
```

Do not present experimental isolation architectures as universal JavaScript guarantees.

---

## Source Classification

Classify claims as:

```text
[ECMAScript Standard]
[Web Platform Standard]
[Node Runtime]
[OWASP Guidance]
[MDN Practical Guidance]
[Browser-specific]
[Tool-specific]
[Measured]
[Historical]
```

---

## Security Discipline

Never state:

```text
freeze = deep immutable
Symbol = secret
closed = secure
TypeScript = runtime validation
lockfile = package trust
CSP = complete XSS defense
CORS = authorization
VM = hostile-code sandbox
```

without qualification.

---

## Compatibility Discipline

Security APIs and runtime behaviors evolve.

Verify current compatibility for:

```text
Trusted Types
structuredClone
security headers
Node runtime flags
runtime isolation APIs
dependency tooling
```

when making deployment decisions.

---

# 91. Completion Snapshot

## Chapter Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current status:

```text
[ ] Not Started
```

## Knowledge Snapshot

### I can explain

- [ ] code vs data
- [ ] dynamic code execution
- [ ] eval
- [ ] Function
- [ ] string timers
- [ ] dynamic loading
- [ ] object integrity
- [ ] prototype chains
- [ ] prototype pollution
- [ ] pollution sources
- [ ] pollution gadgets
- [ ] null-prototype objects
- [ ] Map/Set
- [ ] own-property checks
- [ ] default lookup hazards
- [ ] freeze/seal
- [ ] deep-freeze trade-offs
- [ ] getters/setters
- [ ] Proxy security
- [ ] Reflect
- [ ] Symbols
- [ ] closures
- [ ] private fields
- [ ] capability security
- [ ] least authority
- [ ] confused deputy
- [ ] object capabilities
- [ ] validation
- [ ] schema validation
- [ ] parsing/deserialization
- [ ] structuredClone
- [ ] safe merging
- [ ] URL validation
- [ ] ReDoS
- [ ] resource exhaustion
- [ ] algorithmic complexity
- [ ] error leakage
- [ ] logging security
- [ ] dependency security
- [ ] lifecycle scripts
- [ ] dependency confusion
- [ ] lockfiles
- [ ] secrets
- [ ] browser vs Node
- [ ] sandboxing
- [ ] VM limitations
- [ ] Compartments
- [ ] plugin isolation
- [ ] tainted data
- [ ] secure API boundaries
- [ ] secure defaults
- [ ] library security
- [ ] security testing
- [ ] incident response

### I can predict

- [ ] inherited-property behavior
- [ ] prototype-pollution gadgets
- [ ] freeze shallow/deep behavior
- [ ] getter execution
- [ ] Proxy behavior
- [ ] dynamic-code risks
- [ ] object-merge behavior
- [ ] validation failures
- [ ] ReDoS growth
- [ ] resource exhaustion
- [ ] capability leaks
- [ ] confused-deputy failures
- [ ] dependency attack paths

### I can implement

- [ ] safe config parser
- [ ] schema validator
- [ ] prototype-safe dictionary
- [ ] safe merge
- [ ] capability object
- [ ] plugin permission model
- [ ] resource limits
- [ ] secure logger
- [ ] URL allowlist
- [ ] dependency policy
- [ ] threat-model harness
- [ ] security regression tests

### I can debug

- [ ] dynamic-code execution
- [ ] prototype pollution
- [ ] inherited-property bugs
- [ ] getter/proxy behavior
- [ ] unsafe merge
- [ ] validation bypass
- [ ] ReDoS
- [ ] event-loop blocking
- [ ] supply-chain compromise
- [ ] secret leakage
- [ ] plugin privilege escalation
- [ ] confused deputy

### I can defend

- [ ] eval policy
- [ ] object dictionary strategy
- [ ] prototype-pollution defense
- [ ] integrity strategy
- [ ] capability architecture
- [ ] plugin isolation
- [ ] sandbox strategy
- [ ] schema validation
- [ ] resource limits
- [ ] dependency policy
- [ ] browser/Node boundary
- [ ] supply-chain controls
- [ ] incident-response strategy

---

## Final Principal-Level Test

Explain this system without notes:

```text
External Input
      ↓
Parser
      ↓
Validation
      ↓
Normalization
      ↓
Safe Data Model
      ↓
Capability Boundary
      ↓
Application Logic
      ↓
Controlled Sink
      ↓
Observable Action
```

Then answer:

```text
Where can data become code?
Where can a prototype be modified?
Where can inherited state become authoritative?
Where can a getter/proxy execute unexpectedly?
Where can authority escape?
Where can a confused deputy appear?
Where can CPU/memory become attacker-controlled?
Where can a dependency execute?
Where can secrets leak?
Where can browser policy help?
Where must server policy enforce the rule?
Where can an incident be detected?
Where can the feature be revoked?
```

Your mastery is complete only when you can inspect unfamiliar JavaScript and reason from:

```text
source
→ trust
→ transformation
→ authority
→ sink
→ impact
→ controls
→ residual risk
```

The central Chapter 57 lesson is:

> **Secure JavaScript is not merely JavaScript without obvious XSS. It is JavaScript designed so untrusted data cannot unexpectedly become code, object authority cannot be expanded implicitly, runtime capabilities are limited, resource use is bounded, dependencies are treated as executable trust, and every security boundary is explicit.**