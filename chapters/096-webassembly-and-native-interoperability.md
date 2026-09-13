# Chapter 96 — WebAssembly and Native Interoperability

> **JavaScript Mastery — Part XVIII: Legacy / Interoperability**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-10
>
> **Current-source note:** The official WebAssembly site currently lists the **WebAssembly 3.0** core specification and separate JavaScript API, Web API, and WASI specifications. The WebAssembly site describes Wasm as a portable compilation target designed for efficient loading/execution and for both browser and non-browser embeddings. citeturn259126search0turn259126search10
>
> **2026 ecosystem note:** The WebAssembly Component Model ecosystem is actively evolving. The Bytecode Alliance Component Model documentation currently describes components, WIT interfaces, WASI integration, and a June 11, 2026 WASI 0.3 milestone with native async primitives such as `async func`, `stream<T>`, and `future<T>`. Treat component-model/WASI details as time-sensitive and verify the exact tool/runtime versions before production adoption. citeturn259126search3turn259126search6

---

# 0. Chapter Mission

WebAssembly is not “JavaScript but faster.”

It is a portable low-level execution format and specification stack that lets languages other than JavaScript target a common runtime model.

For JavaScript engineers, the important question is:

> **How do I design a safe, performant, observable boundary between JavaScript and WebAssembly/native code?**

That boundary can involve:

```text
JavaScript
   ↓
WebAssembly JavaScript API
   ↓
Wasm module
   ↓
linear memory / tables / globals / exports
   ↓
compiled native machine instructions
```

Or, in a broader non-browser system:

```text
JavaScript host
   ↓
Wasm runtime
   ↓
Wasm core module
   ↓
WASI / Component Model / host interfaces
   ↓
system capabilities
```

The hard parts are not merely loading `.wasm`.

They are:

```text
ABI design
data representation
ownership
copying
memory lifetime
string encoding
numeric widths
error handling
async boundaries
security
versioning
toolchain integration
performance measurement
observability
```

This chapter develops those concepts from first principles.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- Explain what WebAssembly is and is not.
- Explain the role of the WebAssembly core specification.
- Explain the JavaScript API for WebAssembly.
- Explain the browser Web API additions.
- Explain how a Wasm module is compiled and instantiated.
- Distinguish:
  - module,
  - instance,
  - memory,
  - table,
  - globals,
  - exports,
  - imports.
- Explain linear memory.
- Explain WebAssembly numeric types at the JS boundary.
- Explain integer width conversions.
- Explain `BigInt` interop for Wasm 64-bit integers.
- Explain pointers as offsets into linear memory.
- Explain why strings and objects are not magically shared.
- Explain encoding and memory ownership.
- Explain copying versus shared memory.
- Explain `WebAssembly.Memory`.
- Explain typed-array views over Wasm memory.
- Explain growth and detached/changed views.
- Explain `WebAssembly.Table`.
- Explain imports and exports.
- Explain `WebAssembly.Module`.
- Explain `WebAssembly.Instance`.
- Explain synchronous and asynchronous compilation/instantiation.
- Explain `instantiateStreaming()`.
- Explain MIME-type requirements for streaming.
- Explain WebAssembly compile/link/runtime error classes.
- Explain how JavaScript functions can be imported into Wasm.
- Explain how Wasm functions are exported to JavaScript.
- Explain WASI.
- Explain the WebAssembly Component Model at a high level.
- Explain WIT interfaces.
- Explain canonical ABI concepts.
- Explain FFI and ABI design.
- Explain why naïve JS↔Wasm crossings can dominate performance.
- Explain batching and coarse-grained calls.
- Explain CPU-bound workload selection.
- Explain why moving code to Wasm does not automatically improve performance.
- Explain security boundaries and capability exposure.
- Explain CSP implications.
- Explain source maps/debugging.
- Explain native interoperability in Node.js.
- Explain when to use WebAssembly instead of:
  - JavaScript,
  - Web Workers,
  - native addons,
  - subprocesses,
  - pure libraries.
- Design production JS/Wasm architecture.
- Build a minimal Wasm module.
- Build a JavaScript memory bridge.
- Build an FFI wrapper.
- Debug ABI failures.
- Test memory ownership.
- Benchmark boundary overhead.
- Design versioning.
- Answer senior and principal-level interoperability interview questions.

---

# 2. Prerequisites

Recommended chapters:

```text
Chapter 03 — Numbers / Floating Point / BigInt
Chapter 15 — Objects / Property Semantics
Chapter 19 — Proxy / Reflect / Metaprogramming
Chapter 27 — Typed Arrays / Binary Data
Chapter 28 — JSON / Serialization / Structured Clone
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 45 — Memory / GC
Chapter 47 — JavaScript Engine Architecture
Chapter 48 — V8 Internals / Optimization
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams / Data Flow
Chapter 55 — Fetch / HTTP
Chapter 57 — JavaScript Security Engineering
Chapter 58 — Node.js Architecture
Chapter 61 — Worker Threads / Child Processes
Chapter 63 — Async Context / Diagnostics
Chapter 68 — Transpilation / Compilation
Chapter 69 — Bundlers / Build Systems
Chapter 70 — Source Maps / Production Debugging
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 94 — Compatibility Engineering
Chapter 95 — Legacy JavaScript
```

---

# 3. What Is WebAssembly?

WebAssembly (Wasm) is a portable low-level bytecode/runtime model.

The official WebAssembly specifications define the core semantics independently from a concrete embedding, while separate specifications define JavaScript and Web API integration and WASI. citeturn259126search0

A useful mental model:

```text
source language
(C / C++ / Rust / Go / Zig / etc.)
          ↓
      compiler
          ↓
       Wasm
          ↓
      Wasm engine
          ↓
      target machine
```

The engine may compile Wasm to machine code or otherwise execute it according to the runtime implementation.

---

# 4. What WebAssembly Is Not

Wasm is not:

```text
a replacement for JavaScript
a browser DOM API
a universal operating system
an ABI by itself
a guarantee of faster execution
a garbage collector for arbitrary languages
a magic native-object bridge
```

A Wasm module cannot directly assume:

```js
window
document
fetch
fs
```

unless the embedding environment explicitly provides those capabilities through imports or higher-level interfaces.

---

# 5. WebAssembly Design Goals

The official WebAssembly high-level goals include:

- portable and size/load-time-efficient binaries,
- incremental evolution through proposals,
- backwards compatibility,
- a core sandboxed computation layer,
- integration with the Web,
- support for non-browser embeddings,
- a high level of determinism,
- formal semantics. citeturn259126search10

These goals explain why Wasm is layered.

---

# 6. The WebAssembly Stack

A useful model:

```text
Core Wasm
    ↓
JavaScript API
    ↓
Web API
```

And outside the browser:

```text
Core Wasm
    ↓
WASI
    ↓
host/runtime
```

And at a higher interoperability level:

```text
Core Wasm
    ↓
Component Model
    ↓
WIT / interfaces
    ↓
host + components
```

The layers should not be conflated.

---

# 7. Core Module vs Embedding

A Wasm module is not inherently:

```text
browser code
```

It is an artifact that can be embedded by different hosts.

Potential embeddings:

```text
browser
Node.js
standalone Wasm runtime
server
edge runtime
plugin environment
OS-adjacent system
```

The host decides what imports/capabilities are available.

---

# 8. Module and Instance

The most important runtime distinction:

```text
Module
=
compiled/validated executable representation

Instance
=
stateful executable instantiation of that module
```

MDN describes `WebAssembly.Instance` as a stateful executable instance of a `WebAssembly.Module`. citeturn259126search4

One module can be instantiated multiple times.

---

# 9. `WebAssembly.Module`

Conceptually:

```js
const module =
  await WebAssembly.compile(bytes);
```

The compiled module can then be instantiated:

```js
const instance =
  await WebAssembly.instantiate(module, imports);
```

A module is reusable.

An instance contains runtime state associated with one instantiation.

---

# 10. `WebAssembly.Instance`

Access exports:

```js
const { instance } =
  await WebAssembly.instantiateStreaming(
    fetch("/math.wasm")
  );

console.log(instance.exports);
```

The `exports` object contains the module's exported functions, memories, tables, and globals. citeturn259126search5

---

# 11. Imports

A Wasm module can import host-provided values.

Example import object:

```js
const imports = {
  env: {
    log(value) {
      console.log(value);
    }
  }
};
```

The module must declare a matching import.

Missing/mismatched imports can produce a `WebAssembly.LinkError`. citeturn259126search1

---

# 12. Exports

A module can expose:

```text
functions
memory
table
globals
```

JavaScript accesses them through:

```js
instance.exports.someFunction
```

This is the primary FFI entry point in the JavaScript API.

---

# 13. Numeric Types at the Boundary

Wasm core supports numeric value types including:

```text
i32
i64
f32
f64
```

Newer Wasm features add additional types and capabilities, but classic JS↔Wasm reasoning begins here.

At the JavaScript boundary:

```text
i32
→ Number

f32
→ Number

f64
→ Number

i64
→ BigInt
```

for modern JavaScript API integer semantics.

Always verify feature support in your target runtime when using newer Wasm features.

---

# 14. `i32`

An exported Wasm function:

```text
(i32) → i32
```

can be called from JavaScript using Number values.

Example shape:

```js
const result =
  instance.exports.add(10, 20);

console.log(result);
```

The engine enforces Wasm's numeric type expectations at the boundary.

---

# 15. `i64` and `BigInt`

64-bit integer interoperability is important because JavaScript `Number` cannot exactly represent every 64-bit integer.

Modern WebAssembly JS integration represents Wasm `i64` values through JavaScript `BigInt`.

Example shape:

```js
const result =
  instance.exports.add64(
    10n,
    20n
  );
```

This connects directly to:

```text
Chapter 03 — BigInt
```

---

# 16. `f32` and `f64`

Both map to JavaScript `Number`, but their precision differs.

```text
f32
→ 32-bit IEEE floating semantics

f64
→ 64-bit IEEE floating semantics
```

When values cross:

```text
f32 → JS Number → f32
```

the application must understand the precision boundary.

---

# 17. Pointers

Wasm does not pass ordinary JavaScript objects into Wasm as native object references in the classic linear-memory model.

Instead, a common ABI represents:

```text
pointer
=
integer offset into linear memory
```

For example:

```text
ptr = 1024
len = 100
```

means:

```text
bytes [1024, 1124)
```

inside the module's memory.

The exact ownership/ABI contract is application-defined unless a higher-level interface standard is used.

---

# 18. Linear Memory

Wasm linear memory is a contiguous byte-addressable memory space exposed through a `WebAssembly.Memory` object in JavaScript.

Typical concept:

```text
0
────────────────────────────→
| bytes | bytes | bytes |
────────────────────────────→
```

The module can load/store values at addresses.

---

# 19. `WebAssembly.Memory`

JavaScript can provide or access Wasm memory.

Example:

```js
const memory =
  new WebAssembly.Memory({
    initial: 2
  });
```

Memory sizing is expressed in WebAssembly pages.

A page is:

```text
64 KiB
```

for classic Wasm linear memory semantics.

---

# 20. Typed-Array Views

JavaScript can create a view:

```js
const bytes =
  new Uint8Array(memory.buffer);
```

Then:

```js
bytes[0] = 42;
```

Wasm can observe the underlying bytes according to its own memory operations.

This is one of the most common JS↔Wasm data bridges.

---

# 21. Memory Ownership

A pointer alone is not enough.

You also need:

```text
who allocated?
who owns?
who may mutate?
when can it be freed?
who validates length?
```

Example:

```text
JS allocates
→ Wasm reads
→ JS retains ownership
```

versus:

```text
Wasm allocates
→ JS receives pointer
→ JS reads
→ Wasm later frees
```

These are different contracts.

---

# 22. The ABI Contract

An FFI boundary needs a contract.

Example:

```text
function:
  process(ptr, len) -> status

memory:
  module.exports.memory

encoding:
  UTF-8

ownership:
  caller-owned input
  callee-owned output

error:
  non-zero status
```

Without a defined contract, the integration is fragile.

---

# 23. Strings

Strings usually require encoding.

JavaScript:

```text
Unicode string
```

Wasm linear memory:

```text
bytes
```

A common ABI is:

```text
UTF-8 bytes
+
pointer
+
length
```

Do not assume:

```text
JavaScript string pointer
```

exists automatically.

---

# 24. UTF-8 Conversion

A modern browser/Node bridge can use:

```js
const encoder =
  new TextEncoder();

const bytes =
  encoder.encode("hello");
```

Then copy:

```js
new Uint8Array(
  memory.buffer,
  ptr,
  bytes.length
).set(bytes);
```

The Wasm side reads bytes according to the ABI.

---

# 25. Decoding

After Wasm writes output bytes:

```js
const decoder =
  new TextDecoder();

const text =
  decoder.decode(
    new Uint8Array(
      memory.buffer,
      ptr,
      len
    )
  );
```

The encoding contract must match.

---

# 26. String Ownership

Define:

```text
input buffer
output buffer
allocation owner
free operation
encoding
lifetime
```

Example:

```text
JS
 → alloc(len)
 → write UTF-8
 → call process(ptr,len)
 → read output
 → free(ptr)
```

The exact order must match the module's ABI.

---

# 27. Objects

A plain JS object:

```js
{
  name: "Milan",
  age: 30
}
```

does not automatically become a native Wasm object.

Possible strategies:

```text
JSON
flat binary schema
struct-like memory layout
WIT/interface types
host references using supported Wasm features
```

Choose based on performance and architecture.

---

# 28. JSON at the Boundary

Simple:

```js
const bytes =
  new TextEncoder()
    .encode(JSON.stringify(value));
```

Then pass bytes to Wasm.

Advantages:

```text
simple
debuggable
language-agnostic
```

Costs:

```text
serialization
allocation
parsing
copies
```

For high-frequency calls, JSON can dominate the workload.

---

# 29. Binary ABI

A binary schema can reduce overhead.

Example:

```text
offset 0 → uint32 length
offset 4 → float64 value
offset 12 → flags
```

This requires stricter versioning.

Binary interfaces should document:

```text
alignment
endianness
field sizes
ownership
version
validation
```

---

# 30. Typed Array ABI

For numeric bulk data:

```js
const input =
  new Float64Array([
    1,
    2,
    3,
    4
  ]);
```

Copy or share the bytes:

```text
JS typed array
→ linear memory
→ Wasm loop
→ output bytes
→ JS view
```

This is often much cheaper than per-element JS calls.

---

# 31. Bulk Work Principle

Prefer:

```text
one call
+
large buffer
```

over:

```text
100000 calls
+
one small value each
```

Why?

Each crossing can involve:

```text
argument conversion
boundary bookkeeping
host/runtime transitions
```

The exact cost depends on engine/runtime and workload, so benchmark.

---

# 32. FFI Granularity

Bad design:

```js
for (const value of values) {
  wasm.process(value);
}
```

Potentially better:

```js
wasm.processBatch(ptr, values.length);
```

The boundary becomes:

```text
coarse-grained
```

and Wasm can exploit contiguous memory and internal loops.

---

# 33. When WebAssembly Helps

Strong candidates often include:

```text
CPU-heavy numeric computation
image/video/audio processing
cryptographic algorithms
compression
parsing
simulation
geometry
signal processing
large deterministic transforms
existing native-language codebases
```

Wasm is particularly valuable when moving a computation-heavy kernel reduces total work enough to justify integration overhead.

---

# 34. When WebAssembly Does Not Help

Poor candidates:

```text
DOM manipulation
simple CRUD logic
network orchestration
high-frequency tiny calls
small string transformations
already-fast JavaScript code
workloads dominated by serialization
```

If 95% of the workload is:

```text
JS → Wasm → JS
```

boundary overhead may erase computational benefits.

---

# 35. Benchmark the Whole Pipeline

Do not benchmark:

```text
Wasm kernel
```

alone.

Benchmark:

```text
input acquisition
serialization
copy
Wasm call
computation
copy back
deserialization
```

This is the user-visible cost.

---

# 36. Performance Model

A simplified model:

```text
Total time
=
JS preparation
+
boundary crossing
+
input copy
+
Wasm computation
+
output copy
+
JS interpretation
```

Wasm wins when:

```text
saved computation
>
interop overhead
```

---

# 37. Allocation Strategy

Repeated:

```text
allocate
copy
free
```

can be expensive.

Strategies:

```text
preallocated memory
arena-style allocation
buffer reuse
batch processing
ring buffers
```

But custom allocation creates complexity.

Measure before implementing.

---

# 38. Memory Growth

Wasm memory can grow.

If the underlying `ArrayBuffer` changes as a result of growth, existing JavaScript views may no longer represent the same buffer state.

Therefore code must not blindly assume:

```js
const view = new Uint8Array(memory.buffer);
```

remains valid forever after memory growth.

Refresh/rebuild views according to the memory behavior of the target/runtime.

---

# 39. Shared Memory

WebAssembly can interact with shared memory in supported environments.

This can enable:

```text
Workers
shared buffers
atomic operations
```

But it introduces:

```text
data races
synchronization
memory ordering
```

Use only when the workload genuinely benefits.

---

# 40. Web Workers + Wasm

A common browser architecture:

```text
Main Thread
   ↓
Worker
   ↓
Wasm
```

Benefits:

```text
UI isolation
parallel execution
long-running CPU work away from UI
```

The worker can own the Wasm instance.

---

# 41. Worker Ownership

Prefer clear ownership:

```text
worker
  owns
    Wasm instance
    Wasm memory
    computation state
```

The main thread sends:

```text
structured data / transferable buffers
```

and receives results.

This avoids sharing unnecessary mutable state.

---

# 42. SharedArrayBuffer + Wasm

Where security and environment requirements permit, shared memory can allow:

```text
JS worker
↔
Wasm memory
```

coordination.

But shared memory requires careful:

```text
Atomics
ownership
protocols
termination
liveness
```

Do not use it merely because it exists.

---

# 43. `WebAssembly.instantiate()`

The JavaScript API supports instantiation from:

```text
byte sequence
WebAssembly.Module
```

MDN recommends asynchronous `instantiateStreaming()` when possible rather than synchronous construction for large modules. citeturn259126search8turn259126search4

Typical:

```js
const { instance } =
  await WebAssembly.instantiate(
    bytes,
    imports
  );
```

---

# 44. `instantiateStreaming()`

Modern browser loading:

```js
const { instance } =
  await WebAssembly.instantiateStreaming(
    fetch("/module.wasm"),
    imports
  );
```

MDN describes this as compiling and instantiating directly from a streamed source and calls it the most efficient optimized way to load Wasm in supported browser workflows. citeturn259126search1

---

# 45. Wasm MIME Type

For streaming to work as expected, the server should return:

```text
application/wasm
```

The MDN example explicitly notes the `.wasm` response should use this MIME type. citeturn259126search1

A misconfigured server can therefore turn a working module into a production loading failure.

---

# 46. Compile vs Instantiate

Separate:

```text
compile
=
validate + prepare executable module

instantiate
=
create runtime state with imports
```

This distinction allows:

```text
compile once
instantiate many
```

when the architecture benefits from it.

---

# 47. Sharing Compiled Modules

A compiled `WebAssembly.Module` can be instantiated multiple times and can also be transferred/shared through mechanisms supported by the host.

This is useful for:

```text
workers
multiple isolated instances
server pools
```

but instance state remains distinct.

---

# 48. Multiple Instances

```text
one Module
  ├── Instance A
  │     └── state A
  └── Instance B
        └── state B
```

This is useful for isolation.

Do not accidentally share mutable state between instances unless explicitly designed.

---

# 49. Imports as Capability Injection

A powerful mental model:

```text
Wasm module
    ↓
declares imports
    ↓
host provides capabilities
```

This is dependency injection at a low level.

Example:

```js
const imports = {
  env: {
    log,
    now,
    random
  }
};
```

The host decides what the module can access.

---

# 50. Security Boundary

A Wasm module is not a magical security boundary against all application bugs.

Its safety derives from the Wasm execution model plus the capabilities made available by the embedding.

A module with:

```text
no dangerous imports
```

has a smaller capability surface than one with:

```text
filesystem
network
arbitrary host callbacks
```

Capability minimization matters.

---

# 51. Import Surface Review

Before shipping a module, inventory:

```text
imports
memory
tables
host callbacks
WASI interfaces
filesystem capabilities
network capabilities
randomness
clock access
```

Every import is an architectural dependency.

---

# 52. CSP

MDN currently notes that strict Content Security Policy can block WebAssembly compilation/execution unless the site's CSP allows it. citeturn259126search1

Therefore security review must include:

```text
CSP
Wasm compilation policy
asset integrity
deployment headers
```

Do not diagnose every Wasm failure as a module bug.

---

# 53. Loading Security

A production `.wasm` asset should be protected by:

```text
HTTPS
correct content type
integrity/version strategy
controlled origin
cache policy
deployment verification
```

Do not treat Wasm binaries as inherently trusted simply because they are binaries.

---

# 54. Supply Chain

Wasm introduces another artifact:

```text
module.wasm
```

Track:

```text
source repository
compiler version
compiler flags
dependencies
build reproducibility
hash
release version
```

A compromised build pipeline can produce malicious Wasm just like malicious JavaScript.

---

# 55. Binary Reproducibility

For security-sensitive Wasm, consider:

```text
deterministic builds
artifact hashing
provenance
SBOM
signed releases
```

The exact process depends on organizational security requirements.

---

# 56. Debugging Wasm

A debugging workflow:

```text
JS call site
 ↓
boundary arguments
 ↓
memory layout
 ↓
Wasm export/import
 ↓
Wasm execution
 ↓
memory mutation
 ↓
return value
```

When something fails, capture:

```text
pointer
length
numeric type
memory size
module version
ABI version
```

---

# 57. Wasm Errors

The JS API exposes distinct error classes including:

```text
WebAssembly.CompileError
WebAssembly.LinkError
WebAssembly.RuntimeError
```

The exact failure tells you where to investigate.

Conceptually:

```text
CompileError
→ module invalid/unsupported

LinkError
→ imports/exports mismatch

RuntimeError
→ execution failure
```

---

# 58. Compile Errors

Potential causes:

```text
invalid binary
unsupported feature
corrupted asset
wrong build artifact
```

Confirm:

```text
artifact hash
runtime support
Wasm feature set
content delivery
```

---

# 59. Link Errors

Potential causes:

```text
missing import
wrong import namespace
wrong import type
incorrect function signature
```

The import object is part of the ABI.

Log enough information to diagnose contract mismatch without exposing sensitive data.

---

# 60. Runtime Errors

A Wasm execution can trap.

Examples conceptually:

```text
out-of-bounds memory access
integer divide by zero
unreachable
```

Treat runtime traps as correctness failures.

Do not automatically retry indefinitely.

---

# 61. Memory Safety

Wasm's memory model provides isolation from arbitrary host memory, but your application can still have logical memory bugs:

```text
use-after-free in custom allocator
buffer length mistakes
integer overflow
wrong pointer
wrong alignment assumptions
```

The runtime sandbox does not fix your ABI.

---

# 62. Pointer Validation

When JavaScript passes:

```text
ptr
len
```

validate:

```text
ptr >= 0
len >= 0
ptr + len <= memory size
```

The exact bounds may be checked by the Wasm memory access itself, but application-level validation is useful at host boundaries to produce clearer failures.

---

# 63. Integer Overflow at the Boundary

A pointer/length calculation can overflow in host code.

Example conceptual mistake:

```js
const end = ptr + len;
```

Use safe integer/typed representations and explicit range checks.

Security-sensitive parsers should treat lengths as untrusted input.

---

# 64. Alignment

For binary ABIs, document:

```text
alignment
field offsets
padding
```

A JavaScript typed-array view has its own element-size alignment rules.

Do not invent a struct layout independently in JS and Rust/C/C++ without a shared ABI specification.

---

# 65. Endianness

WebAssembly memory is byte-addressable and multi-byte values have defined representation.

JavaScript `DataView` allows explicit endian handling:

```js
view.getUint32(
  offset,
  true
);
```

A binary protocol should document endianness even if your current platform is little-endian.

---

# 66. ABI Versioning

Define:

```text
ABI v1
ABI v2
```

Then include version information.

Example:

```text
header:
magic
version
flags
payload length
```

This allows controlled evolution.

---

# 67. Function Signature Stability

Changing:

```text
process(i32, i32) -> i32
```

to:

```text
process(i64, i32) -> i32
```

is an ABI change.

Version it or provide a compatibility wrapper.

Do not silently change exported signatures for independently deployed clients.

---

# 68. Wrapper Layer

Do not let the entire application know:

```text
ptr
len
allocation
encoding
```

Create:

```js
class WasmCodec {
  encode(input) {}
  decode(buffer) {}
}
```

The wrapper owns the ABI.

The application sees:

```text
domain values
```

This keeps interoperability complexity localized.

---

# 69. FFI Contract

Document:

```text
Name:
Parameters:
Return:
Memory ownership:
Encoding:
Failure:
Version:
Threading:
Async behavior:
Reentrancy:
```

The more languages consume the component, the more important this becomes.

---

# 70. Reentrancy

Imported host functions can call back into Wasm or other host code depending on the architecture.

Document whether an exported operation permits:

```text
reentrancy
```

If not, enforce it.

Reentrancy bugs can be difficult to diagnose because state can be observed half-updated.

---

# 71. Async Boundaries

Classic Wasm function calls are synchronous.

If the host API needs async work:

```text
network
filesystem
timers
```

the interface must model that explicitly.

Recent Component Model/WASI work adds higher-level async concepts; the exact support depends on runtime/toolchain maturity. citeturn259126search3

---

# 72. Avoid Fake Async

Bad:

```text
Wasm call
→ blocks
→ host starts async work
→ waits somehow
```

This can create:

```text
deadlock
UI blocking
thread starvation
complex state machines
```

Use the host/runtime's supported async integration.

---

# 73. Callback Imports

A Wasm module can import a JS function.

```js
const imports = {
  env: {
    log(value) {
      console.log(value);
    }
  }
};
```

This enables callbacks.

But frequent callbacks across the boundary can become expensive.

Batch when possible.

---

# 74. JS Function Identity

If a Wasm module stores a function reference in an imported table or callback structure, understand:

```text
identity
lifetime
ownership
garbage collection
```

Do not create unbounded callback registrations.

---

# 75. `WebAssembly.Table`

Tables hold references used by Wasm, historically especially for function references.

JavaScript can access a table through:

```js
instance.exports.table
```

or provide one in imports.

Tables enable indirect calls and reference-based interoperability.

---

# 76. Function Tables and Dynamic Dispatch

A common pattern:

```text
integer/function index
       ↓
Wasm table
       ↓
function reference
```

This can support:

```text
callbacks
dynamic dispatch
language runtimes
function pointers
```

The exact table type and reference features depend on the Wasm capabilities used.

---

# 77. Global Values

Wasm globals can be:

```text
immutable
mutable
```

depending on the declaration.

JS can import/export compatible global values.

Globals are useful for:

```text
configuration
constants
small state
```

but not as a substitute for a clear data model.

---

# 78. Module Metadata

A production Wasm artifact should have discoverable:

```text
version
ABI version
build commit
compiler/runtime assumptions
feature requirements
```

The exact mechanism can include custom sections or external manifests.

Keep operational metadata outside executable semantics where appropriate.

---

# 79. Source Maps and Debugging

Wasm toolchains can emit debugging information.

A production debugging setup should retain:

```text
source maps/debug symbols
build identifiers
artifact hashes
```

while controlling exposure in public deployments.

---

# 80. Native Interoperability in Node.js

Node can run Wasm through the JavaScript WebAssembly API.

This can be preferable to a native addon when:

```text
portable sandboxed computation
cross-platform artifact
language interoperability
```

are priorities.

Native addons can be preferable when:

```text
deep OS integration
native APIs
hardware APIs
very low-level integration
```

are required.

---

# 81. WebAssembly vs Native Addon

| Dimension | WebAssembly | Native addon |
|---|---|---|
| Portability | High | Lower |
| Isolation | Stronger sandbox model | More direct host access |
| OS APIs | Via host interfaces | Direct |
| Deployment | Portable artifact | Platform binaries |
| Startup | Can be efficient | Can be efficient |
| Security surface | Capability-limited possible | Larger host surface |
| Language choice | Broad | Broad |
| Integration complexity | ABI/interface | Native ABI/Node ABI |
| Hardware access | Indirect | Direct |

No option is universally superior.

---

# 82. WebAssembly vs Child Process

Use a child process when you need:

```text
strong process isolation
existing executable
separate failure domain
OS-level capabilities
```

Use Wasm when you need:

```text
embedded computation
low-overhead in-process execution
portable sandboxed module
```

Process boundaries and Wasm boundaries solve different problems.

---

# 83. WebAssembly vs Worker

A Worker gives:

```text
thread/task isolation from main UI thread
```

Wasm gives:

```text
portable computation module
```

They can be combined:

```text
Worker + Wasm
```

This is often strong for CPU-heavy browser workloads.

---

# 84. WebAssembly vs JavaScript

Stay in JS when:

```text
workload already fast
logic needs DOM
logic is orchestration-heavy
interop dominates
team/tooling costs outweigh benefit
```

Move a kernel to Wasm when:

```text
CPU-bound
well-defined boundary
portable algorithm
measurable speedup
existing native code
```

---

# 85. Component Model

The WebAssembly Component Model is a higher-level architecture for interoperable Wasm libraries, applications, and environments. Its documentation describes components, interfaces, worlds, and composition. citeturn259126search6

The key concept:

```text
Core Wasm module
≠
full language-neutral application interface
```

The Component Model aims to standardize richer interfaces between components.

---

# 86. WIT

WIT (WebAssembly Interface Types) defines interfaces and types used by components.

Conceptually:

```text
WIT interface
      ↓
language bindings
      ↓
component
```

This can reduce ad hoc pointer-based FFI.

The Component Model documentation describes WIT as the interface description language used by current component tooling. citeturn259126search3turn259126search6

---

# 87. Canonical ABI

The Component Model defines conventions for representing higher-level interface types across component boundaries.

This moves the abstraction from:

```text
ptr + len
```

toward:

```text
string
record
list
variant
result
resource
```

with a defined canonical representation.

This is a major architectural shift from hand-built ABI glue.

---

# 88. WASI

WASI provides system-style interfaces for non-browser WebAssembly environments.

The official WebAssembly specifications page describes WASI as a modular system interface with capabilities including files, network connections, clocks, and randomness. citeturn259126search0

The key model:

```text
Wasm
 ↓
declared capability interface
 ↓
host provides capability
```

This avoids giving a module unrestricted access to the operating system.

---

# 89. WASI and Security

A capability-oriented interface lets the host decide:

```text
which files?
which network?
which environment variables?
which clocks?
which random source?
```

The host can provide only what the component needs.

This is the principle of:

```text
least authority
```

---

# 90. WASI Versioning

WASI is evolving.

Current component-model documentation describes WASI 0.3 as a June 11, 2026 milestone that adds native async primitives to the Component Model. citeturn259126search3

Because this ecosystem is actively evolving, pin:

```text
WASI version
toolchain version
runtime version
interface definitions
```

before deploying.

---

# 91. Component Composition

Components can be composed:

```text
component A
     ↓
component B
     ↓
component C
```

Each can expose/consume typed interfaces.

This supports reusable cross-language modules.

---

# 92. JS as a Component Host

A JavaScript application may eventually sit at a higher-level interface rather than manually managing:

```text
malloc
pointer
length
UTF-8
free
```

This is one reason the Component Model matters.

But browser/native support is not identical to standalone component runtimes.

Check exact host/toolchain support.

---

# 93. WebAssembly and the Browser

Browser Wasm is constrained by web security and host policies.

The host controls:

```text
network
DOM
storage
workers
fetch
timers
```

through JavaScript/Web APIs and related interfaces.

Wasm does not bypass same-origin or browser security policies. The official WebAssembly goals explicitly call out integration with existing Web security policies and Web APIs. citeturn259126search10

---

# 94. WebAssembly and the DOM

Wasm does not directly manipulate the DOM as if it were native browser script.

Common architecture:

```text
Wasm
  ↓
compute
  ↓
return data
  ↓
JavaScript
  ↓
DOM
```

Avoid moving DOM orchestration into the Wasm layer just because the algorithm is written in Rust/C++.

---

# 95. WebAssembly and Fetch

Typical loading:

```js
const response =
  await fetch("/module.wasm");

const { instance } =
  await WebAssembly.instantiateStreaming(
    response,
    imports
  );
```

Streaming ties:

```text
network
+
compile
+
instantiate
```

into one pipeline.

Ensure correct HTTP handling and MIME type.

---

# 96. Caching

Compiled modules can be expensive to recreate repeatedly.

Use:

```text
HTTP caching
module reuse
worker initialization strategy
application-level caching
```

where appropriate.

The exact browser cache/engine compilation behavior is implementation-dependent.

Do not promise zero recompilation without evidence.

---

# 97. Versioned Wasm Assets

Prefer content-addressed or versioned artifacts:

```text
module.ab12cd34.wasm
```

rather than mutable:

```text
module.wasm
```

This helps:

```text
CDN caching
rollback
debugging
artifact identification
```

---

# 98. Integrity

For security-sensitive deployments, pair:

```text
versioned artifact
+
integrity/provenance
+
controlled deployment
```

A module binary is executable code and should receive the same supply-chain scrutiny as JavaScript.

---

# 99. Compression

Wasm binaries often benefit from HTTP compression.

Measure:

```text
raw size
compressed size
decompression
compile time
network time
```

Do not optimize only for file size.

---

# 100. Startup Cost

A Wasm module can have:

```text
download
decompress
compile
instantiate
initialize memory
```

costs.

For short-lived operations, startup can dominate execution.

Use long-lived instances or lazy loading when appropriate.

---

# 101. Lazy Loading

For route-specific functionality:

```js
async function loadCodec() {
  return import("./codec.js");
}
```

or an equivalent Wasm-loading layer.

Benefits:

```text
initial load reduction
feature-specific cost
```

Costs:

```text
first-use latency
cache behavior
complexity
```

---

# 102. Warmup

For repeated heavy computation:

```text
startup
→ instantiate
→ warm
→ serve many calls
```

Measure real workload.

Do not create/destroy a Wasm instance per tiny request unless the workload explicitly benefits from that isolation.

---

# 103. Memory Pools

For high-throughput integration:

```text
preallocate
reuse
```

rather than:

```text
allocate
copy
free
```

every call.

But memory pools require:

```text
ownership
fragmentation policy
lifetime
concurrency
```

---

# 104. Zero-Copy Thinking

“Zero copy” should mean something precise.

Possible design:

```text
host buffer
↔
shared/accessible memory
```

But JS and Wasm memory ownership rules, typed-array views, host APIs, and transfer semantics can still introduce copies.

Measure with allocation and memory instrumentation.

---

# 105. Transferables vs Wasm Memory

For workers, a typed array's underlying `ArrayBuffer` can sometimes be transferred.

This can avoid copying between threads, but the sender loses access to a transferred buffer.

The exact architecture can be:

```text
main thread
  ↓ transfer
worker
  ↓
Wasm
```

This is different from shared memory.

---

# 106. Shared vs Transfer vs Copy

Three patterns:

```text
copy
→ independent data, simplest

transfer
→ ownership moved, low copy cost

share
→ simultaneous access, synchronization complexity
```

Choose deliberately.

---

# 107. Backpressure

For streaming data into Wasm:

```text
producer
 ↓
buffer
 ↓
Wasm consumer
```

If the consumer is slower, buffering can grow.

Use:

```text
chunking
flow control
backpressure
```

rather than unbounded accumulation.

---

# 108. Streaming Binary Processing

For large data:

```text
fetch stream
 ↓
chunks
 ↓
Wasm processing
 ↓
output chunks
```

This can reduce peak memory compared with:

```text
download entire file
→ copy entire file
→ process
```

Combine with Chapter 53.

---

# 109. Parsing Workloads

Wasm can be useful for:

```text
binary formats
compression
image codecs
custom parsers
```

But if JS repeatedly converts:

```text
bytes → object → bytes
```

the conversion can dominate.

Keep data compact across the boundary.

---

# 110. Cryptography

Wasm can host cryptographic code, but security requires:

```text
audited algorithm
constant-time expectations where relevant
side-channel analysis
secure randomness
key management
memory zeroization assumptions
host integration
```

Do not implement cryptography from scratch simply because Wasm can run code.

Prefer well-reviewed implementations and platform primitives when appropriate.

---

# 111. Side Channels

WebAssembly's deterministic low-level execution can still exist within environments with:

```text
timing
cache behavior
shared resources
speculative execution concerns
```

Security-sensitive code needs threat-model-specific analysis.

Wasm does not automatically eliminate side channels.

---

# 112. Native Library Reuse

One major advantage of Wasm is reusing an existing library:

```text
C/C++/Rust library
 ↓
Wasm
 ↓
JavaScript application
```

But assess:

```text
licensing
binary size
ABI stability
threading
memory model
security
build maintenance
```

A “free” port creates a second build ecosystem to maintain.

---

# 113. Compiler Toolchain

Common source languages can target Wasm through their own toolchains.

The pipeline is:

```text
source
 ↓
front end
 ↓
optimizer
 ↓
Wasm codegen
 ↓
Wasm binary
```

Toolchain choices affect:

```text
binary size
startup
debugging
feature usage
runtime dependencies
```

Do not attribute all performance to “Wasm.”

Compiler quality matters.

---

# 114. Binary Size

Track:

```text
.wasm raw
gzip
brotli
source maps/debug info
dependencies
```

Larger modules can increase:

```text
network latency
compile time
memory
cache pressure
```

---

# 115. Dead Code Elimination

Use linker/build optimization where appropriate.

A native library can contain large unused portions.

Trim:

```text
unused functions
unused data
unused language runtime
```

But ensure the resulting artifact still exposes all required interfaces.

---

# 116. Exceptions and Errors

Different source languages model errors differently.

Possible Wasm boundary strategies:

```text
status code
result struct
error enum
string error
host exception
Component Model result/error
```

Choose one.

Do not make JavaScript parse arbitrary panic strings for control flow.

---

# 117. Error Mapping

Create a stable JavaScript-facing error model:

```js
class WasmOperationError extends Error {
  constructor(code, message, options) {
    super(message, options);
    this.code = code;
  }
}
```

The FFI wrapper maps:

```text
Wasm status
→
domain error
```

---

# 118. Panic / Trap Policy

For Rust/C/C++ libraries, define:

```text
panic
assert
abort
trap
```

behavior.

A crash-like failure inside an embedded component may become:

```text
WebAssembly.RuntimeError
```

or a higher-level mapped error depending on the interface.

Do not hide catastrophic failures as ordinary validation errors.

---

# 119. Resource Lifetime

Wasm has linear memory but source-language runtimes may implement:

```text
heap
allocator
GC
reference counting
destructors
```

Those models are not identical to JavaScript GC.

The host must understand ownership.

---

# 120. Garbage Collection Boundary

A JavaScript object can be collected when unreachable.

A pointer into Wasm memory may remain “numerically valid” while the underlying allocation has been freed or reused by the Wasm allocator.

Therefore:

```text
JavaScript reachability
≠
Wasm allocation lifetime
```

This is a critical interoperability concept.

---

# 121. Resource Wrapper

A safe JS wrapper can expose:

```js
class WasmBuffer {
  constructor(pointer, length, free) {
    this.pointer = pointer;
    this.length = length;
    this.free = free;
  }

  dispose() {
    if (this.pointer !== null) {
      this.free(this.pointer);
      this.pointer = null;
    }
  }
}
```

A production design should integrate explicit resource-management strategy with finalization only as a secondary safety net, not as the primary lifecycle mechanism.

---

# 122. Finalization Is Not Deterministic

Do not expect garbage collection to immediately free Wasm resources.

If the underlying module exposes:

```text
malloc/free
```

use deterministic ownership.

This connects to Chapter 46.

---

# 123. Reentrancy and Resource Ownership

If JS calls Wasm:

```text
acquire
→ call
→ callback to JS
→ callback calls Wasm
```

the resource state may be temporarily inconsistent.

Document reentrancy rules.

---

# 124. Threading

Wasm can be used in worker-style parallel architectures where supported.

The right question is:

```text
Do we need parallel computation?
```

rather than:

```text
Can Wasm spawn threads?
```

Threading introduces:

```text
synchronization
shared memory
startup
scheduling
debugging
```

costs.

---

# 125. SIMD

WebAssembly SIMD can accelerate data-parallel kernels.

Candidates:

```text
image processing
audio
numeric vectors
compression
```

But hardware and runtime support must be verified.

Benchmark scalar vs SIMD implementations.

---

# 126. Bulk Memory

Modern Wasm includes bulk-memory capabilities useful for:

```text
copy
fill
memory initialization
```

These can improve low-level data movement.

But again:

```text
Wasm feature support
=
runtime-dependent
```

for newer capabilities.

---

# 127. Reference Types

Reference-oriented Wasm features reduce some limitations of purely integer-pointer ABIs.

They are useful for advanced interop architectures, but support and semantics should be verified against the target engine/runtime.

Do not build a production contract on a feature your target fleet does not support.

---

# 128. JS Builtins

Current browser WebAssembly APIs can expose compile options for specific JavaScript builtins in environments that support them. MDN documents a `builtins` option including `"js-string"` in current API documentation. citeturn259126search1turn259126search2

Treat this as a concrete runtime feature that must be tested rather than a universal Wasm assumption.

---

# 129. Compilation Caching Strategy

A mature application may cache:

```text
downloaded bytes
compiled module
initialized instance
```

at different layers.

Choose based on:

```text
memory
startup latency
instance state
security
tenant isolation
```

---

# 130. Multi-Tenant Isolation

For multi-tenant services, decide:

```text
one instance per tenant
one instance shared
pool of instances
separate process/runtime
```

Evaluate:

```text
state isolation
memory
startup
security
concurrency
```

Do not share a stateful Wasm instance across tenants without an explicit isolation design.

---

# 131. Module Purity

A highly reusable Wasm module is easier to operate when:

```text
initialization is explicit
state is bounded
imports are minimal
ABI is stable
```

Avoid hidden global mutable state where possible.

---

# 132. Determinism

Wasm's design emphasizes determinism, but application-level determinism still depends on imports and host behavior.

If the module imports:

```text
clock
random
network
filesystem
```

results may no longer be deterministic.

Testing needs controlled imports.

---

# 133. Testing Imports

Use test doubles:

```js
const imports = {
  env: {
    now: () => 123,
    random: () => 0.5,
    log: () => {}
  }
};
```

Then deterministic Wasm tests can run.

---

# 134. Golden Binary Tests

For a stable ABI, test:

```text
input bytes
→ expected output bytes
```

This can catch:

```text
layout changes
encoding bugs
version mismatches
```

---

# 135. Contract Testing

Test the JS wrapper separately:

```text
JS domain object
→ wrapper
→ Wasm
→ wrapper
→ domain object
```

Then test Wasm independently.

This localizes failures.

---

# 136. Fuzzing

Wasm parsers and binary interfaces are good fuzzing targets.

Fuzz:

```text
pointer
length
payload
UTF-8
binary schema
malformed messages
```

Validate:

```text
no memory corruption
no unexpected traps
no infinite loops
```

---

# 137. Differential Testing

Compare:

```text
JavaScript implementation
vs
Wasm implementation
```

for the same inputs.

Useful for:

```text
codec
parser
math
compression
protocol implementation
```

The test oracle can expose semantic divergences.

---

# 138. Numeric Edge Cases

Test:

```text
0
-0
NaN
Infinity
- Infinity
maximum integer
minimum integer
integer overflow
f32 rounding
f64 precision
BigInt boundaries
```

JS and Wasm numeric semantics are not identical in every detail.

---

# 139. String Edge Cases

Test:

```text
ASCII
emoji
surrogate pairs
combining marks
NFC/NFD
invalid UTF-8
embedded null byte
very large strings
empty strings
```

Especially when the ABI is UTF-8 bytes.

---

# 140. Memory Edge Cases

Test:

```text
zero-length
last byte
page boundary
maximum supported length
growth
stale view
freed pointer
double free
```

These are classic interop failure modes.

---

# 141. Boundary Benchmark

Benchmark:

```text
1 call / 1 byte
1 call / 1 KB
1 call / 1 MB
100 calls / 1 MB total
1000 calls / 1 MB total
```

This exposes boundary amortization.

---

# 142. Throughput vs Latency

A Wasm pipeline may improve:

```text
throughput
```

while hurting:

```text
single-request latency
```

or vice versa.

Benchmark both.

---

# 143. Cold vs Warm Performance

Measure:

```text
cold start
warm instance
steady-state
reinitialization
```

Wasm startup can materially affect serverless/edge architectures.

---

# 144. Memory Benchmark

Track:

```text
Wasm memory size
JS heap
number of views
allocation rate
GC
resident set size
```

A fast module that consumes excessive memory can still be a poor production choice.

---

# 145. Observability

Expose metrics such as:

```text
wasm_module_load_total
wasm_compile_duration
wasm_instantiate_duration
wasm_call_duration
wasm_bytes_copied
wasm_trap_total
wasm_memory_bytes
wasm_fallback_total
```

Do not expose user-sensitive payloads.

---

# 146. Tracing

Create spans:

```text
JS request
  └── wasm.call
       ├── encode
       ├── copy-in
       ├── execution
       └── copy-out
```

This makes it possible to locate bottlenecks.

---

# 147. Logging

Log:

```text
module version
ABI version
runtime version
feature set
error category
```

Do not log raw secrets from memory buffers.

---

# 148. Health Checks

A service that relies on Wasm should validate:

```text
module loads
ABI matches
critical exports exist
required features available
```

at startup or deployment validation.

---

# 149. Startup Validation

Example:

```js
function validateWasm(instance) {
  const required = [
    "process",
    "malloc",
    "free"
  ];

  for (const name of required) {
    if (typeof instance.exports[name] !== "function") {
      throw new Error(
        `Missing Wasm export: ${name}`
      );
    }
  }
}
```

Do not wait for the first production request to discover an incompatible module.

---

# 150. Graceful Fallback

If the Wasm path is optional:

```text
Wasm available
→ use Wasm

Wasm unavailable
→ compatible JS implementation
```

But only where semantic equivalence and performance requirements allow it.

For security-sensitive or contract-critical capabilities, fail closed when required.

---

# 151. Compatibility Policy

Record:

```text
minimum Wasm features
minimum browser/runtime
minimum Node
component/WASI version
compiler version
ABI version
```

Chapter 94 applies directly.

---

# 152. Browser Fallback

A browser application can sometimes ship:

```text
modern Wasm
+
JS fallback
```

This increases:

```text
bundle
testing
maintenance
```

Use real user distribution to justify the fallback.

---

# 153. Native Fallback in Node

A Node package might offer:

```text
Wasm backend
native backend
pure JS backend
```

Do not automatically load the native backend for every environment.

Use explicit capability detection and clear failure semantics.

---

# 154. Package Design

For a JS package with Wasm:

```text
package/
  dist/
    index.js
    module.wasm
    worker.js
```

Document:

```text
Wasm asset loading
browser support
Node support
CSP
bundler configuration
fallback
```

---

# 155. Bundler Integration

Bundlers may treat `.wasm` specially.

Check:

```text
URL resolution
asset copying
code splitting
base paths
SSR
worker builds
```

Do not assume a bundler will handle Wasm exactly like JavaScript.

---

# 156. SSR / Wasm

Server-side rendering may instantiate Wasm on:

```text
server
browser
```

with different URLs/loading mechanisms.

A wrapper should hide host-specific loading.

---

# 157. Workers and Bundlers

Wasm loaded from workers can fail due to:

```text
asset path
MIME
CSP
cross-origin policy
bundler output
```

Test the production artifact rather than only development.

---

# 158. Caching Policy

Use content hashes:

```text
codec-8a31f.wasm
```

and long-lived immutable caching where appropriate.

Tie the asset to:

```text
JS wrapper ABI version
```

so incompatible combinations are not accidentally cached together.

---

# 159. Artifact Compatibility

A JavaScript wrapper and Wasm module form a pair.

Version them:

```text
JS ABI client 3
Wasm ABI 3
```

Startup validation can reject mismatches.

---

# 160. ABI Manifest

Example:

```json
{
  "moduleVersion": "3.2.0",
  "abiVersion": 3,
  "features": [
    "simd"
  ],
  "exports": [
    "malloc",
    "free",
    "process"
  ]
}
```

The manifest is an operational aid.

---

# 161. Security Review Checklist

```text
[ ] artifact provenance
[ ] hash/signature
[ ] imports reviewed
[ ] host capabilities minimized
[ ] CSP reviewed
[ ] memory bounds validated
[ ] input lengths limited
[ ] parser fuzzed
[ ] error handling reviewed
[ ] secrets excluded from logs
[ ] dependency licenses reviewed
[ ] compiler supply chain reviewed
```

---

# 162. Performance Review Checklist

```text
[ ] cold startup
[ ] warm startup
[ ] boundary cost
[ ] copies
[ ] allocations
[ ] Wasm execution
[ ] serialization
[ ] memory growth
[ ] GC impact
[ ] worker overhead
[ ] binary download
[ ] compressed size
```

---

# 163. Principal Decision Matrix

| Requirement | JS | Wasm | Native addon | Worker | Child process |
|---|---|---|---|---|---|
| DOM orchestration | Excellent | Poor | Poor | Indirect | Poor |
| CPU-heavy kernel | Good–Excellent | Excellent candidate | Excellent | Excellent combined | Good |
| Strong process isolation | No | No | No | No | Yes |
| Portable binary | Excellent source | Excellent | Lower | Excellent | Lower |
| OS-native API | Host APIs | Via interface | Excellent | Host APIs | Excellent |
| Low-level C/C++ reuse | No | Excellent | Excellent | Combined | Excellent |
| Simple app logic | Excellent | Overkill | Overkill | Overkill | Overkill |
| Rich language-independent interface | Limited | Component Model path | ABI-dependent | N/A | IPC-dependent |

No cell is absolute.

Use workload evidence.

---

# 164. When to Choose Wasm

Choose Wasm when:

```text
the kernel is computationally expensive
the boundary can be coarse-grained
portable sandboxed execution matters
existing language implementation is valuable
measured performance justifies complexity
```

---

# 165. When to Choose Native

Choose native integration when:

```text
deep OS integration
hardware access
native libraries
system calls
extreme host-specific optimization
```

are required and the operational cost is acceptable.

---

# 166. When to Choose a Child Process

Choose a child process when:

```text
fault isolation
untrusted executable
resource quotas
separate crash domain
independent lifecycle
```

matter more than in-process latency.

---

# 167. When to Choose a Worker

Choose a worker when:

```text
CPU work blocks the main/UI thread
same-language application logic is sufficient
isolation is needed without a process boundary
```

Combine with Wasm for:

```text
heavy CPU kernel + UI isolation
```

---

# 168. Architecture Pattern — Browser Codec

```text
UI
 ↓
Worker
 ↓
JS wrapper
 ↓
Wasm codec
 ↓
linear memory
```

Inputs:

```text
ArrayBuffer / Uint8Array
```

Outputs:

```text
Uint8Array
```

This is a good coarse-grained boundary.

---

# 169. Architecture Pattern — Node Parser

```text
HTTP request
 ↓
Node service
 ↓
Wasm parser
 ↓
typed result
 ↓
domain service
```

Use Wasm only for the deterministic parse/validation kernel.

Keep:

```text
auth
network
DB
logging
```

in the host layer unless there is a strong reason otherwise.

---

# 170. Architecture Pattern — Edge Function

```text
request
 ↓
JS edge runtime
 ↓
Wasm module
 ↓
compute
 ↓
response
```

Key concerns:

```text
startup
module size
runtime feature support
memory limits
```

---

# 171. Architecture Pattern — Multi-Language Component

```text
Rust component
      ↕
WIT interface
      ↕
host/runtime
      ↕
JavaScript application
```

This can be preferable to hand-designed:

```text
ptr + len + malloc + free
```

when Component Model tooling/runtime support is appropriate.

---

# 172. Production Migration

If replacing pure JS with Wasm:

```text
1. benchmark baseline
2. extract kernel
3. define ABI
4. build Wasm
5. wrap
6. differential test
7. canary
8. observe
9. optimize
10. roll out
```

Do not start with a full rewrite.

---

# 173. Differential Migration

Run:

```text
JS result
Wasm result
```

and compare.

For deterministic computation:

```js
const expected =
  jsImplementation(input);

const actual =
  wasmImplementation(input);

assert.deepEqual(actual, expected);
```

Then add edge cases and performance tests.

---

# 174. Fuzz Differential Testing

Generate inputs:

```text
random
boundary
malformed
large
Unicode
negative
overflow
```

Run both implementations.

This is particularly powerful for:

```text
parsers
codecs
math
serialization
compression
```

---

# 175. Golden ABI Tests

Create binary fixtures:

```text
fixture-001.bin
fixture-002.bin
```

Test:

```text
JS wrapper
↔
Wasm
```

across releases.

This catches accidental ABI drift.

---

# 176. Version Negotiation

For independent deployment:

```text
client ABI 3
server/module ABI 4
```

possible strategies:

```text
reject
fallback
negotiate
compatibility adapter
```

Do not silently guess.

---

# 177. Backwards-Compatible Exports

Prefer:

```text
v2 exports
+
v1 adapter
```

during a transition.

Remove only after consumers are migrated.

---

# 178. ABI Deprecation

Document:

```text
deprecated export
replacement
deprecation date
removal release
migration example
```

This is especially important when the Wasm module is consumed by multiple languages.

---

# 179. Language-Neutral Interface

A good cross-language interface avoids language-specific assumptions such as:

```text
Rust Vec
C++ std::string
JS object identity
Go slice internals
```

Instead use:

```text
string
list
record
variant
result
resource
```

This is one reason interface description systems such as WIT exist.

---

# 180. Component Model Trade-Off

The Component Model can simplify high-level interop.

Costs include:

```text
toolchain complexity
runtime availability
versioning
generated bindings
learning curve
deployment compatibility
```

Do not introduce it solely to avoid writing ten lines of pointer glue.

Use it when the interface architecture justifies it.

---

# 181. WASI vs Browser APIs

Do not assume:

```text
WASI
=
browser platform
```

WASI targets non-browser system-style capabilities.

Browsers use:

```text
Web APIs
JavaScript APIs
Web security model
```

The two ecosystems have different host assumptions.

---

# 182. Portability Is Layered

A module can be portable at one layer:

```text
core Wasm
```

but require a host-specific capability:

```text
filesystem
```

which reduces practical portability.

Always state:

```text
core portability
host portability
runtime portability
application portability
```

---

# 183. Deterministic Builds

For production Wasm:

```text
source commit
compiler version
flags
dependency lock
```

should be reproducible.

A change in compiler/version may alter:

```text
binary
performance
feature set
debug info
```

Pin toolchains where appropriate.

---

# 184. License and Compliance

Native source dependencies can bring:

```text
licenses
notices
SBOM requirements
export controls
patent considerations
```

Review before shipping a Wasm build of a third-party native library.

---

# 185. Failure Domain

If Wasm traps:

```text
What fails?
request?
worker?
process?
tenant?
whole service?
```

Choose architecture accordingly.

A Wasm kernel inside a Worker can isolate UI impact differently from Wasm inside a Node service.

---

# 186. Resource Limits

For untrusted modules:

```text
memory limits
execution limits
fuel/time budgets where supported
capability limits
```

may be needed.

The exact mechanism depends on the embedding/runtime.

---

# 187. Untrusted Wasm

Treat untrusted Wasm as code, not data.

Threat model:

```text
malicious computation
resource exhaustion
host capability abuse
side channels
parser/runtime vulnerabilities
```

Provide minimal imports and resource controls.

---

# 188. Host Callback Abuse

An imported function such as:

```js
log(value)
```

can become:

```text
host capability
```

If it exposes:

```text
filesystem
network
process
```

the security stakes become much higher.

Use least authority.

---

# 189. Capability-Based Design

Prefer:

```text
module
  imports only hash function
```

over:

```text
module
  imports universal host executor
```

Small capability surfaces are easier to:

```text
audit
test
reason about
```

---

# 190. Observability by ABI Version

Tag metrics:

```text
wasm_abi_version
module_version
runtime_version
```

Then a regression can be correlated to:

```text
module deployment
```

rather than merely:

```text
JavaScript service
```

---

# 191. Principal-Level Review Questions

Before adopting Wasm, ask:

```text
What is the computational bottleneck?
Why isn't optimized JS sufficient?
What is the measured speedup?
What is boundary cost?
How many copies occur?
Who owns memory?
How are errors mapped?
How is the module loaded?
What are the host capabilities?
How is the artifact secured?
What is the ABI version?
What happens during rollback?
```

If these answers are unclear, the design is not production-ready.

---

# 192. Common Misconceptions

### “Wasm is always faster.”

False.

Interop, serialization, startup, and poor generated code can make it slower.

### “Wasm replaces JavaScript.”

No.

JavaScript remains the primary host/orchestration language on the Web.

### “Wasm can access the DOM directly.”

Not through the classic core model.

### “A pointer is enough to pass a string.”

No.

Encoding, length, lifetime, and ownership matter.

### “GC handles Wasm allocations.”

Not automatically.

Wasm linear-memory allocators are a separate concern.

### “Wasm is automatically secure.”

No.

Security depends on the execution model and exposed capabilities.

---

# 193. Common Mistakes

Do not:

```text
micro-call Wasm in a hot JS loop
copy buffers unnecessarily
use JSON for huge high-frequency payloads
hide ABI details without documenting ownership
ignore memory growth
ignore stale typed-array views
log raw memory
ignore CSP
assume browser = Node
assume Wasm feature support everywhere
ship compiler artifacts without provenance
share stateful instances across tenants accidentally
```

---

# 194. Implementation — Guided

Create a tiny Wasm function using a toolchain of your choice:

```text
add(i32, i32) -> i32
```

Then:

```js
const result =
  instance.exports.add(4, 5);

console.log(result);
```

Document:

```text
source
compiler
Wasm feature requirements
loading method
runtime
```

---

# 195. Implementation — Memory Bridge

Implement:

```text
allocate
write bytes
call function
read bytes
free
```

Use:

```js
TextEncoder
TextDecoder
Uint8Array
```

The wrapper should expose:

```js
processString(value)
```

not:

```js
processPointer(ptr, len)
```

to the rest of the application.

---

# 196. Implementation — Batch Processing

Create:

```text
processBatch(ptr, count)
```

for an array of numeric values.

Benchmark:

```text
one call / item
vs
one call / batch
```

Measure speedup.

---

# 197. Implementation — Differential Testing

Implement:

```text
JS algorithm
Wasm algorithm
```

and compare:

```text
normal
random
boundary
malformed
large
```

inputs.

---

# 198. Implementation — Edge-Case Hardened

Handle:

```text
invalid pointer
invalid length
empty input
memory growth
UTF-8 failure
module mismatch
missing export
trap
double free
```

Produce explicit errors.

---

# 199. Implementation — Production Grade

Build:

```text
wasm/
  loader.js
  abi.js
  memory.js
  errors.js
  health.js
  metrics.js
  version.js
```

The application imports:

```js
import { createProcessor } from "./wasm/loader.js";
```

The application should not know:

```text
ptr
len
malloc
free
```

unless it is itself the ABI layer.

---

# 200. Debugging Exercise — Missing Export

Given:

```js
instance.exports.process(...)
```

and:

```text
TypeError: instance.exports.process is not a function
```

Investigate:

```text
artifact version
export name
build flags
module manifest
deployment cache
```

Do not assume the JS code is the problem.

---

# 201. Debugging Exercise — LinkError

Symptom:

```text
WebAssembly.LinkError
```

Check:

```text
import namespace
import function name
signature
missing import
version mismatch
```

---

# 202. Debugging Exercise — Runtime Trap

Symptom:

```text
WebAssembly.RuntimeError
```

Check:

```text
input values
pointer
length
memory bounds
integer arithmetic
module feature behavior
```

---

# 203. Debugging Exercise — Corrupted String

Input:

```text
hello 🌍
```

Output:

```text
hello ��
```

Likely area:

```text
encoding
length
UTF-8 boundaries
```

Do not assume the Wasm algorithm corrupted the string.

---

# 204. Debugging Exercise — Stale View

Code:

```js
const view =
  new Uint8Array(memory.buffer);

// module grows memory

view[0] = 10;
```

Question:

> Should the code blindly assume this view still represents the current memory state?

No.

Memory growth can invalidate assumptions about existing views.

Refresh views according to the memory behavior of the target runtime.

---

# 205. Debugging Exercise — Slow Wasm

Measured:

```text
JS: 20 ms
Wasm: 5 ms kernel
full pipeline: 80 ms
```

The conclusion is not:

```text
Wasm is slow.
```

The correct hypothesis:

```text
interop/serialization/copy overhead dominates.
```

Optimize the boundary.

---

# 206. Code Review Exercise

Review:

```js
for (const row of rows) {
  wasm.process(
    JSON.stringify(row)
  );
}
```

Problems:

```text
serialization per row
many boundary calls
temporary strings
allocation pressure
potential parse per row
```

A better design may batch rows into a binary representation.

---

# 207. Code Review Exercise — Global Memory

Review:

```js
const memory =
  new WebAssembly.Memory({ initial: 100 });
```

Then every request writes directly into the same memory.

Potential issue:

```text
concurrent requests can overwrite each other
```

Need:

```text
ownership
allocation
locking
instance isolation
worker isolation
```

---

# 208. Code Review Exercise — Unbounded Import

Review:

```js
const imports = {
  env: {
    host: (...args) =>
      eval(args[0])
  }
};
```

Reject.

This turns an imported capability into a dynamic-code execution surface.

The Wasm boundary does not make the host callback safe.

---

# 209. Code Review Exercise — Shared Instance

Review:

```js
const shared = await createWasm();

export async function process(request) {
  return shared.process(request);
}
```

Question:

> Is this safe?

Only if the instance and its memory/state are explicitly designed for concurrent/shared use.

Stateful modules can create cross-request corruption.

---

# 210. Interview Questions — Fundamentals

1. What is WebAssembly?
2. What is a Wasm module?
3. What is an instance?
4. What is linear memory?
5. What is an import?
6. What is an export?
7. What is `WebAssembly.Module`?
8. What is `WebAssembly.Instance`?
9. Why are strings not directly passed like JS objects?
10. What does a pointer usually represent?
11. Why is `i64` connected to BigInt?
12. What is WASI?

---

# 211. Interview Questions — Senior

1. When does Wasm outperform JS?
2. Why can Wasm be slower?
3. What is ABI design?
4. How do you pass strings?
5. Who owns memory?
6. How do you prevent stale typed-array views?
7. How do you load Wasm efficiently in browsers?
8. What is the role of `application/wasm`?
9. What is the difference between compile and instantiate?
10. How do you map Wasm traps to JS errors?

---

# 212. Interview Questions — Principal

1. When would you choose Wasm over native Node addons?
2. When would you choose a child process instead?
3. How would you design a cross-language ABI?
4. How would you version it?
5. How would you secure host imports?
6. How would you handle multi-tenant Wasm execution?
7. How would you benchmark JS vs Wasm honestly?
8. How would you design zero-copy or low-copy data flow?
9. How would you migrate a JS algorithm to Wasm safely?
10. How would you use the Component Model?
11. How would you operationalize Wasm in a large platform?

---

# 213. Predict-the-Result Exercise

Assume a module exports:

```text
add(i32, i32) -> i32
```

JavaScript:

```js
console.log(
  instance.exports.add(2, 3)
);
```

Predict:

```text
5
```

The important issue is the Wasm boundary type contract.

---

# 214. Predict-the-Result Exercise

Assume:

```text
add64(i64, i64) -> i64
```

JavaScript:

```js
instance.exports.add64(2, 3);
```

Question:

> Should the caller use ordinary `Number` values or the JavaScript BigInt representation required for Wasm `i64` interoperability?

Use the exact runtime API contract; for current WebAssembly JS integer interop, `i64` uses `BigInt`.

---

# 215. Predict-the-Result Exercise

Given:

```js
const bytes =
  new TextEncoder().encode("hello");
```

Question:

> Is `bytes` a JavaScript string?

No.

It is a typed byte representation suitable for copying into linear memory.

---

# 216. Mastery Exercise — ABI Design

Design an ABI for:

```text
compress(input)
```

Document:

```text
pointer
length
output ownership
encoding
allocation
error
version
```

Then explain why JSON would or would not be appropriate.

---

# 217. Mastery Exercise — Batch API

Design:

```text
processBatch
```

for:

```text
1 million float64 values
```

Compare:

```text
one-value calls
vs
one batch
```

Estimate:

```text
boundary cost
memory
throughput
```

Then benchmark.

---

# 218. Mastery Exercise — Security Review

Threat-model:

```text
untrusted Wasm module
```

Define:

```text
imports
memory limit
execution limit
logging
artifact trust
rollback
```

Then compare:

```text
Wasm
vs
child process
```

---

# 219. Mastery Exercise — Browser Codec

Build:

```text
UI
→ Worker
→ JS wrapper
→ Wasm codec
→ result
```

Requirements:

```text
transferable buffers
batching
error handling
metrics
fallback
```

---

# 220. Mastery Exercise — Node Service

Build:

```text
HTTP server
→ request validation
→ Wasm parser
→ domain processing
```

Requirements:

```text
ABI version
health check
module cache
memory safety
latency metric
fallback or fail-closed policy
```

---

# 221. Mastery Exercise — Component Model Research

Study one Component Model interface and document:

```text
WIT interface
types
exports
imports
host capabilities
canonical ABI
runtime/toolchain
```

Then explain why it is easier or harder than a hand-built pointer ABI.

Because Component Model tooling is actively evolving, record the exact runtime/toolchain versions you used.

---

# 222. Mastery Exercise — Differential Testing

Implement the same algorithm:

```text
JavaScript
Wasm
```

Run:

```text
100000 random inputs
```

Compare results and classify mismatches.

---

# 223. Mastery Exercise — Production Decision

Write a principal memo:

```text
Problem
JS baseline
Wasm proposal
Expected benefit
Measured benefit
Boundary cost
Operational cost
Security
Maintenance
Fallback
Decision
```

Final categories:

```text
STAY JS
USE WASM
USE WORKER + JS
USE WORKER + WASM
USE NATIVE
USE CHILD PROCESS
```

Defend the decision.

---

# 224. Spaced Retrieval Schedule

### Day 0

Explain:

```text
Module
Instance
Memory
Import
Export
ABI
```

### Day 1

Explain:

```text
i32
i64
f32
f64
BigInt
```

### Day 3

Explain:

```text
pointer
length
ownership
UTF-8
```

### Day 7

Explain:

```text
boundary cost
batching
copying
memory growth
```

### Day 14

Build a JS↔Wasm memory bridge.

### Day 30

Build a differential test.

### Day 60

Design a production Wasm service.

### Day 90

Defend Wasm vs JS/native/process architecture.

---

# 225. Retrieval Prompts

Answer without notes:

```text
What is WebAssembly?
What is a module?
What is an instance?
What is linear memory?
What is an ABI?
How are pointers represented?
How are strings represented?
Why is BigInt relevant to i64?
What is an import?
What is an export?
What is instantiateStreaming?
Why does MIME matter?
What is WASI?
What is WIT?
What is the Component Model?
Why can Wasm be slower?
What makes a good Wasm workload?
How do you secure imports?
How do you handle ownership?
How do you version an ABI?
```

---

# 226. Dependency Graph

```text
Chapter 03 — Numbers / BigInt
        ↓
Chapter 27 — Typed Arrays / Binary Data
        ↓
Chapter 45 — Memory / GC
        ↓
Chapter 47 — Engine Architecture
        ↓
Chapter 52 — Workers
        ↓
Chapter 57 — Security
        ↓
Chapter 61 — Processes / Workers
        ↓
Chapter 68 — Compilation
        ↓
Chapter 85 — Performance
        ↓
Chapter 94 — Compatibility
        ↓
Chapter 95 — Legacy
        ↓
Chapter 96 — WebAssembly / Native Interoperability
        ↓
Chapter 97 — Edge / Serverless JavaScript
```

---

# 227. Concept Connections

## Depends On

- numbers and `BigInt`
- typed arrays
- binary memory
- GC
- engine architecture
- workers
- security
- Node architecture
- compilation
- compatibility
- performance

## Builds Toward

- native/edge runtimes
- production interoperability
- sandboxed computation
- multi-language systems
- high-performance platform architecture

## Related Concepts

- FFI
- ABI
- IPC
- sandboxing
- capability security
- zero-copy
- serialization
- SIMD
- multithreading
- component interfaces

## Concepts Revisited

- language vs host
- runtime vs specification
- memory ownership
- typed arrays
- BigInt
- workers
- performance measurement
- security boundaries

## Why This Chapter Matters Later

A principal JavaScript engineer should understand not only JavaScript execution, but also what happens when JavaScript shares a system with other languages and runtimes.

---

# 228. Principal Decision Framework

For every Wasm/native proposal ask:

```text
Correctness
Performance
Memory
Security
Reliability
Maintainability
Scalability
Observability
Developer Experience
Operational Complexity
Future Change
```

Then:

```text
What is the actual bottleneck?
What is the boundary?
What data crosses it?
How often?
Who owns memory?
What host capabilities are exposed?
What is the ABI?
How is it versioned?
What happens on failure?
Can we roll back?
```

---

# 229. Production Checklist

```text
[ ] Workload measured
[ ] JS baseline measured
[ ] Wasm candidate justified
[ ] ABI documented
[ ] Ownership documented
[ ] Encoding documented
[ ] Memory boundaries tested
[ ] Runtime feature support verified
[ ] Browser/Node support verified
[ ] CSP reviewed
[ ] Artifact provenance verified
[ ] Binary versioned
[ ] ABI versioned
[ ] Errors mapped
[ ] Performance benchmarked
[ ] Memory benchmarked
[ ] Observability added
[ ] Canary defined
[ ] Rollback defined
[ ] Fallback defined where appropriate
```

---

# 230. Canonical References and Source Discipline

Primary references:

1. **WebAssembly Specifications**
   https://webassembly.org/specs/
   Current official listing includes the WebAssembly 3.0 core specification plus JavaScript API, Web API, WASI, and tooling-convention specifications. citeturn259126search0

2. **WebAssembly High-Level Goals**
   https://webassembly.org/docs/high-level-goals/
   Official goals covering portability, efficiency, formal semantics, browser integration, and non-browser embeddings. citeturn259126search10

3. **MDN WebAssembly JavaScript API**
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface
   Current API surface includes module compilation, instantiation, validation, and related integration APIs. citeturn259126search2

4. **MDN `instantiateStreaming()`**
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface/instantiateStreaming_static
   Current browser guidance for streamed compile/instantiate and `application/wasm`. citeturn259126search1

5. **MDN `WebAssembly.Instance`**
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface/Instance
   Runtime instance semantics and loading guidance. citeturn259126search4

6. **WebAssembly Component Model**
   https://component-model.bytecodealliance.org/
   Current high-level component architecture, interfaces, composition, and tooling guidance. citeturn259126search6

7. **Component Model FAQ**
   https://component-model.bytecodealliance.org/reference/faq.html
   Current 2026 status information including WASI 0.3 and async component primitives. citeturn259126search3

8. **WASI**
   https://wasi.dev/

9. **WebAssembly GitHub Organization**
   https://github.com/WebAssembly

10. **Wasm Test Suite**
    https://github.com/WebAssembly/spec/tree/main/test

Source discipline:

```text
core semantics
→ WebAssembly specification

JS embedding
→ WebAssembly JS API specification

browser loading
→ Web API + MDN + actual browser tests

Node behavior
→ Node/runtime documentation + actual target tests

Component Model
→ current Component Model/WIT/WASI documentation

production behavior
→ actual target runtime + benchmarks + security tests
```

---

# 231. Source Verification Notes — 2026-09-10

Current verification points:

- The official WebAssembly specifications page lists **WebAssembly 3.0** as the current core specification and separately identifies the JavaScript API, Web API, and WASI API. citeturn259126search0
- The official WebAssembly goals emphasize portability, efficiency, formal semantics, incremental evolution, browser integration, and non-browser embedding. citeturn259126search10
- Current MDN documentation describes `WebAssembly.instantiateStreaming()` as the preferred efficient browser loading path when supported and specifies `application/wasm` for the streamed module response. citeturn259126search1
- Current MDN WebAssembly API documentation includes `WebAssembly.promising()` and other newer interface capabilities; do not assume every API is supported identically by every target runtime. citeturn259126search2
- Current Component Model documentation describes components, WIT interfaces, and a 2026 WASI 0.3 milestone with native async primitives. citeturn259126search3turn259126search6

The Wasm ecosystem is evolving. Record the runtime, compiler, component tooling, and specification versions used by production systems.

---

# Chapter 96 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I explain WebAssembly vs JavaScript? [ ]
- Could I explain module vs instance? [ ]
- Could I explain linear memory? [ ]
- Could I explain ABI design? [ ]
- Could I explain pointers and lengths? [ ]
- Could I explain string encoding? [ ]
- Could I explain memory ownership? [ ]
- Could I explain i64 / BigInt? [ ]
- Could I explain instantiateStreaming? [ ]
- Could I explain Wasm error classes? [ ]
- Could I explain WASI? [ ]
- Could I explain WIT / Component Model? [ ]
- Could I explain boundary performance? [ ]
- Could I design a secure Wasm integration? [ ]
- Could I defend Wasm vs native/process/worker? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 96 — Completion Snapshot

```text
Track A — Core Theory
[ ] WebAssembly fundamentals
[ ] Module / instance
[ ] Imports / exports
[ ] Numeric types
[ ] BigInt / i64
[ ] Linear memory
[ ] Tables
[ ] Globals
[ ] ABI
[ ] Pointers
[ ] Strings / UTF-8
[ ] Ownership
[ ] Error model
[ ] WASI
[ ] WIT
[ ] Component Model
[ ] Browser integration
[ ] Node integration
[ ] Security model

Track B — Implementation
[ ] Load Wasm
[ ] Create JS wrapper
[ ] Memory bridge
[ ] String bridge
[ ] Numeric batch API
[ ] Differential testing
[ ] Fuzz testing
[ ] Version validation
[ ] Observability
[ ] Production loader
[ ] Worker integration
[ ] Native fallback evaluation

Track C — Interview / Reasoning
[ ] Explain when Wasm wins
[ ] Explain when Wasm loses
[ ] Design an ABI
[ ] Design ownership
[ ] Design security boundaries
[ ] Compare Wasm/native
[ ] Compare Wasm/process
[ ] Compare Wasm/worker
[ ] Design multi-tenant isolation
[ ] Defend a principal architecture

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

Do not mark this chapter mastered because you can load a `.wasm` file.

You are ready to move forward when you can independently:

1. Explain the Wasm core model.
2. Explain module vs instance.
3. Explain imports and exports.
4. Explain i32/i64/f32/f64 boundary behavior.
5. Explain why i64 maps to BigInt at the JS boundary.
6. Explain linear memory.
7. Design a pointer/length ABI.
8. Design string encoding.
9. Define ownership and lifetime.
10. Prevent buffer and length mistakes.
11. Explain memory growth and JavaScript views.
12. Explain compile vs instantiate.
13. Use `instantiateStreaming()` correctly.
14. Diagnose MIME/CSP failures.
15. Diagnose CompileError, LinkError, RuntimeError.
16. Explain why boundary overhead matters.
17. Design a batched API.
18. Design a Worker + Wasm architecture.
19. Compare Wasm with native addons and child processes.
20. Explain WASI at a high level.
21. Explain WIT and the Component Model.
22. Version an ABI.
23. Secure host imports.
24. Build differential tests.
25. Produce a production rollout and rollback plan.
26. Defend a Wasm architecture at principal-engineer level.

---

# Principal Challenge

Design a **production-grade cross-language computation platform**.

Requirements:

```text
- browser client
- Node.js backend
- CPU-heavy image/codec workload
- 100 MB inputs
- multiple requests concurrently
- multi-tenant
- optional browser offline support
- strict CSP
- audit requirements
- deterministic tests
- gradual rollout
- runtime heterogeneity
```

Your design must cover:

```text
1. JS/Wasm boundary
2. ABI
3. memory layout
4. encoding
5. ownership
6. worker architecture
7. backend architecture
8. artifact loading
9. caching
10. versioning
11. ABI compatibility
12. security
13. CSP
14. supply chain
15. observability
16. tracing
17. performance
18. memory
19. failure isolation
20. rollback
21. fallback
22. browser compatibility
23. Node compatibility
24. testing
25. fuzzing
26. migration
```

Then defend one decision:

```text
Why Wasm?
Why not optimized JS?
Why not native addon?
Why not child process?
Why not Worker + JS?
Why not all of them?
```

The answer should be evidence-driven.

---

# Final Mental Model

```text
JavaScript
   ↓
host API
   ↓
Wasm module
   ↓
ABI
   ↓
memory / references
   ↓
compiled execution
```

For production:

```text
standard
+
runtime
+
toolchain
+
ABI
+
ownership
+
security
+
performance
+
observability
+
versioning
```

The deepest lesson is:

> **WebAssembly is not primarily a “faster JavaScript” feature. It is an interoperability and execution boundary.**

Principal engineers optimize the boundary as carefully as the computation.

---

# File Metadata

```text
Chapter: 96
Title: WebAssembly and Native Interoperability
Part: XVIII — Legacy / Interoperability
Format: Standalone Markdown chapter
Verification date: 2026-09-10
Status: [ ] Not Started
Current core spec referenced: WebAssembly 3.0
```

> **Mastery reminder:** Reading does not mark completion. Mastery requires retrieval, prediction, implementation, debugging, application, comparison, and defense.

---

# 232. Advanced Interoperability Drill — Abi Ownership

## Scenario

A production JavaScript application integrates a Wasm component for **ABI ownership**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 233. Advanced Interoperability Drill — Utf-8 Bridge

## Scenario

A production JavaScript application integrates a Wasm component for **UTF-8 bridge**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 234. Advanced Interoperability Drill — Typed-Array Bridge

## Scenario

A production JavaScript application integrates a Wasm component for **typed-array bridge**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 235. Advanced Interoperability Drill — I64 Bigint Boundary

## Scenario

A production JavaScript application integrates a Wasm component for **i64 BigInt boundary**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 236. Advanced Interoperability Drill — Memory Growth

## Scenario

A production JavaScript application integrates a Wasm component for **memory growth**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 237. Advanced Interoperability Drill — Stale Js Views

## Scenario

A production JavaScript application integrates a Wasm component for **stale JS views**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 238. Advanced Interoperability Drill — Batching

## Scenario

A production JavaScript application integrates a Wasm component for **batching**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 239. Advanced Interoperability Drill — Copy Minimization

## Scenario

A production JavaScript application integrates a Wasm component for **copy minimization**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 240. Advanced Interoperability Drill — Module Caching

## Scenario

A production JavaScript application integrates a Wasm component for **module caching**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 241. Advanced Interoperability Drill — Instance Isolation

## Scenario

A production JavaScript application integrates a Wasm component for **instance isolation**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 242. Advanced Interoperability Drill — Csp

## Scenario

A production JavaScript application integrates a Wasm component for **CSP**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 243. Advanced Interoperability Drill — Mime Handling

## Scenario

A production JavaScript application integrates a Wasm component for **MIME handling**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 244. Advanced Interoperability Drill — Compileerror

## Scenario

A production JavaScript application integrates a Wasm component for **CompileError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 245. Advanced Interoperability Drill — Linkerror

## Scenario

A production JavaScript application integrates a Wasm component for **LinkError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 246. Advanced Interoperability Drill — Runtimeerror

## Scenario

A production JavaScript application integrates a Wasm component for **RuntimeError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 247. Advanced Interoperability Drill — Import Capability

## Scenario

A production JavaScript application integrates a Wasm component for **import capability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 248. Advanced Interoperability Drill — Wasi Capability

## Scenario

A production JavaScript application integrates a Wasm component for **WASI capability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 249. Advanced Interoperability Drill — Wit Interface

## Scenario

A production JavaScript application integrates a Wasm component for **WIT interface**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 250. Advanced Interoperability Drill — Component Composition

## Scenario

A production JavaScript application integrates a Wasm component for **component composition**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 251. Advanced Interoperability Drill — Wasm Vs Native

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs native**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 252. Advanced Interoperability Drill — Wasm Vs Worker

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs worker**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 253. Advanced Interoperability Drill — Wasm Vs Child Process

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs child process**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 254. Advanced Interoperability Drill — Node Wasm Integration

## Scenario

A production JavaScript application integrates a Wasm component for **Node Wasm integration**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 255. Advanced Interoperability Drill — Browser Wasm Integration

## Scenario

A production JavaScript application integrates a Wasm component for **browser Wasm integration**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 256. Advanced Interoperability Drill — Multi-Tenant Isolation

## Scenario

A production JavaScript application integrates a Wasm component for **multi-tenant isolation**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 257. Advanced Interoperability Drill — Artifact Provenance

## Scenario

A production JavaScript application integrates a Wasm component for **artifact provenance**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 258. Advanced Interoperability Drill — Abi Versioning

## Scenario

A production JavaScript application integrates a Wasm component for **ABI versioning**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 259. Advanced Interoperability Drill — Differential Testing

## Scenario

A production JavaScript application integrates a Wasm component for **differential testing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 260. Advanced Interoperability Drill — Fuzzing

## Scenario

A production JavaScript application integrates a Wasm component for **fuzzing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 261. Advanced Interoperability Drill — Numeric Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **numeric edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 262. Advanced Interoperability Drill — String Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **string edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 263. Advanced Interoperability Drill — Memory Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **memory edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 264. Advanced Interoperability Drill — Benchmark Design

## Scenario

A production JavaScript application integrates a Wasm component for **benchmark design**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 265. Advanced Interoperability Drill — Cold-Start Cost

## Scenario

A production JavaScript application integrates a Wasm component for **cold-start cost**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 266. Advanced Interoperability Drill — Warm-Start Cost

## Scenario

A production JavaScript application integrates a Wasm component for **warm-start cost**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 267. Advanced Interoperability Drill — Throughput

## Scenario

A production JavaScript application integrates a Wasm component for **throughput**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 268. Advanced Interoperability Drill — Latency

## Scenario

A production JavaScript application integrates a Wasm component for **latency**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 269. Advanced Interoperability Drill — Observability

## Scenario

A production JavaScript application integrates a Wasm component for **observability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 270. Advanced Interoperability Drill — Tracing

## Scenario

A production JavaScript application integrates a Wasm component for **tracing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 271. Advanced Interoperability Drill — Error Mapping

## Scenario

A production JavaScript application integrates a Wasm component for **error mapping**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 272. Advanced Interoperability Drill — Rollback

## Scenario

A production JavaScript application integrates a Wasm component for **rollback**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 273. Advanced Interoperability Drill — Abi Ownership

## Scenario

A production JavaScript application integrates a Wasm component for **ABI ownership**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 274. Advanced Interoperability Drill — Utf-8 Bridge

## Scenario

A production JavaScript application integrates a Wasm component for **UTF-8 bridge**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 275. Advanced Interoperability Drill — Typed-Array Bridge

## Scenario

A production JavaScript application integrates a Wasm component for **typed-array bridge**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 276. Advanced Interoperability Drill — I64 Bigint Boundary

## Scenario

A production JavaScript application integrates a Wasm component for **i64 BigInt boundary**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 277. Advanced Interoperability Drill — Memory Growth

## Scenario

A production JavaScript application integrates a Wasm component for **memory growth**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 278. Advanced Interoperability Drill — Stale Js Views

## Scenario

A production JavaScript application integrates a Wasm component for **stale JS views**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 279. Advanced Interoperability Drill — Batching

## Scenario

A production JavaScript application integrates a Wasm component for **batching**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 280. Advanced Interoperability Drill — Copy Minimization

## Scenario

A production JavaScript application integrates a Wasm component for **copy minimization**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 281. Advanced Interoperability Drill — Module Caching

## Scenario

A production JavaScript application integrates a Wasm component for **module caching**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 282. Advanced Interoperability Drill — Instance Isolation

## Scenario

A production JavaScript application integrates a Wasm component for **instance isolation**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 283. Advanced Interoperability Drill — Csp

## Scenario

A production JavaScript application integrates a Wasm component for **CSP**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 284. Advanced Interoperability Drill — Mime Handling

## Scenario

A production JavaScript application integrates a Wasm component for **MIME handling**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 285. Advanced Interoperability Drill — Compileerror

## Scenario

A production JavaScript application integrates a Wasm component for **CompileError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 286. Advanced Interoperability Drill — Linkerror

## Scenario

A production JavaScript application integrates a Wasm component for **LinkError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 287. Advanced Interoperability Drill — Runtimeerror

## Scenario

A production JavaScript application integrates a Wasm component for **RuntimeError**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 288. Advanced Interoperability Drill — Import Capability

## Scenario

A production JavaScript application integrates a Wasm component for **import capability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 289. Advanced Interoperability Drill — Wasi Capability

## Scenario

A production JavaScript application integrates a Wasm component for **WASI capability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 290. Advanced Interoperability Drill — Wit Interface

## Scenario

A production JavaScript application integrates a Wasm component for **WIT interface**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 291. Advanced Interoperability Drill — Component Composition

## Scenario

A production JavaScript application integrates a Wasm component for **component composition**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 292. Advanced Interoperability Drill — Wasm Vs Native

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs native**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 293. Advanced Interoperability Drill — Wasm Vs Worker

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs worker**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 294. Advanced Interoperability Drill — Wasm Vs Child Process

## Scenario

A production JavaScript application integrates a Wasm component for **Wasm vs child process**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 295. Advanced Interoperability Drill — Node Wasm Integration

## Scenario

A production JavaScript application integrates a Wasm component for **Node Wasm integration**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 296. Advanced Interoperability Drill — Browser Wasm Integration

## Scenario

A production JavaScript application integrates a Wasm component for **browser Wasm integration**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 297. Advanced Interoperability Drill — Multi-Tenant Isolation

## Scenario

A production JavaScript application integrates a Wasm component for **multi-tenant isolation**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 298. Advanced Interoperability Drill — Artifact Provenance

## Scenario

A production JavaScript application integrates a Wasm component for **artifact provenance**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 299. Advanced Interoperability Drill — Abi Versioning

## Scenario

A production JavaScript application integrates a Wasm component for **ABI versioning**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 300. Advanced Interoperability Drill — Differential Testing

## Scenario

A production JavaScript application integrates a Wasm component for **differential testing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 301. Advanced Interoperability Drill — Fuzzing

## Scenario

A production JavaScript application integrates a Wasm component for **fuzzing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 302. Advanced Interoperability Drill — Numeric Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **numeric edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 303. Advanced Interoperability Drill — String Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **string edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 304. Advanced Interoperability Drill — Memory Edge Cases

## Scenario

A production JavaScript application integrates a Wasm component for **memory edge cases**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 305. Advanced Interoperability Drill — Benchmark Design

## Scenario

A production JavaScript application integrates a Wasm component for **benchmark design**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 306. Advanced Interoperability Drill — Cold-Start Cost

## Scenario

A production JavaScript application integrates a Wasm component for **cold-start cost**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 307. Advanced Interoperability Drill — Warm-Start Cost

## Scenario

A production JavaScript application integrates a Wasm component for **warm-start cost**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 308. Advanced Interoperability Drill — Throughput

## Scenario

A production JavaScript application integrates a Wasm component for **throughput**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 309. Advanced Interoperability Drill — Latency

## Scenario

A production JavaScript application integrates a Wasm component for **latency**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 310. Advanced Interoperability Drill — Observability

## Scenario

A production JavaScript application integrates a Wasm component for **observability**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 311. Advanced Interoperability Drill — Tracing

## Scenario

A production JavaScript application integrates a Wasm component for **tracing**.

Analyze:

```text
1. Semantic contract
2. ABI
3. Numeric representation
4. Memory representation
5. Ownership
6. Lifetime
7. Error handling
8. Concurrency
9. Runtime compatibility
10. Browser/Node differences
11. Security boundary
12. Performance cost
13. Memory cost
14. Testing
15. Observability
16. Versioning
17. Rollback
18. Fallback
19. Operational ownership
20. Removal/change strategy
```

## Required output

```text
Decision:
Why:
Boundary:
Data model:
Ownership:
Risk:
Test:
Metric:
Rollback:
```

## Principal question

What evidence would make you reject the design and keep the implementation in JavaScript, or move it to a native/process boundary instead?


---

# 312. Final Reference Card

```text
Wasm
=
portable low-level execution model

Module
=
compiled/validated executable representation

Instance
=
stateful runtime instantiation

Import
=
host-provided capability/value

Export
=
module-provided capability/value

Linear memory
=
byte-addressable Wasm memory

Pointer
=
usually an integer offset into linear memory

String ABI
=
bytes + encoding + pointer/length + ownership

ABI
=
contract for data + functions + lifetime + errors

JS boundary
=
where interop overhead can dominate

Best Wasm workload
=
CPU-heavy + coarse-grained + measurable

Poor Wasm workload
=
tiny hot calls + serialization-heavy + DOM-heavy

Security
=
least-capability imports + artifact trust + resource limits

WASI
=
system-style capability interfaces for non-browser Wasm

WIT
=
interface definition language for Component Model

Component Model
=
higher-level typed interoperability architecture

Principal rule
=
optimize the boundary, not just the kernel
```

---

# Chapter 96 — Final Principal Rule

Do not ask:

> “Can we compile this to WebAssembly?”

Ask:

> **“Does introducing WebAssembly create a better system boundary than JavaScript, a Worker, a native addon, or a process—and can we prove that through correctness, performance, security, operability, and lifecycle evidence?”**

That is the level expected of a principal JavaScript engineer.


---

# Chapter 96 — Canonical References and Source Discipline

Primary sources for this chapter:

1. WebAssembly Specifications  
   https://webassembly.org/specs/

2. WebAssembly High-Level Goals  
   https://webassembly.org/docs/high-level-goals/

3. MDN WebAssembly JavaScript API  
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface

4. MDN `WebAssembly.instantiateStreaming()`  
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface/instantiateStreaming_static

5. MDN `WebAssembly.instantiate()`  
   https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface/instantiate_static

6. WebAssembly Component Model  
   https://component-model.bytecodealliance.org/

7. Component Model FAQ  
   https://component-model.bytecodealliance.org/reference/faq.html

8. WASI  
   https://wasi.dev/

Source discipline:

```text
Core semantics
→ WebAssembly specification

JavaScript embedding
→ WebAssembly JavaScript API

Browser-specific loading
→ WebAssembly Web API + browser documentation

WASI / Component Model
→ current WASI / Component Model specifications and runtime documentation

Production readiness
→ exact runtime/toolchain + tests + benchmarks + security review
```

Record the exact:

```text
runtime version
browser version
compiler version
ABI version
interface version
observed date
```

because the Wasm and component ecosystem is evolving.