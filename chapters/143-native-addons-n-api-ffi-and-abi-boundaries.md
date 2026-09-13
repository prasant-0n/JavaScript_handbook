# Chapter 143 — Native Addons, N-API, FFI & ABI Boundaries

> **JavaScript Mastery — Part XXIV: Node.js Runtime, Networking & Systems Engineering**
>
> **Mission:** Master the boundary between JavaScript and native code in Node.js. Learn Node-API (N-API), direct V8/C++ addons, ABI stability, `node-gyp`, `binding.gyp`, native lifecycle, `napi_env`, handles, references, finalizers, buffers, ArrayBuffers, external memory, exceptions, async work, threads, workers, native callbacks, cleanup hooks, FFI, pointers, calling conventions, dynamic libraries, symbol resolution, ABI compatibility, packaging, prebuilds, cross-platform distribution, security, crash diagnosis, and production governance.
>
> **Role perspective:** Principal Node.js Engineer · Runtime Engineer · Native Systems Engineer · N-API Specialist · Build/Release Engineer · Security Engineer · Performance Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **The native boundary changes the failure model. JavaScript mistakes usually produce managed exceptions; native mistakes can corrupt memory, violate ABI assumptions, crash the process, leak resources, deadlock threads, or invalidate pointers. Cross the boundary only with an explicit ownership, lifetime, error, threading, and compatibility contract.**

---

# 1. Learning Objectives

```text
[ ] explain why native addons exist
[ ] explain the JavaScript/native boundary
[ ] explain Node-API
[ ] explain N-API history
[ ] explain ABI stability
[ ] distinguish ABI from API
[ ] distinguish Node-API from V8 APIs
[ ] distinguish Node-API from node-addon-api
[ ] explain direct V8 addons
[ ] explain NAN
[ ] explain C++ addons
[ ] explain .node binaries
[ ] explain dynamic shared libraries
[ ] explain native symbol resolution
[ ] explain node-gyp
[ ] explain binding.gyp
[ ] explain build prerequisites
[ ] explain prebuilt binaries
[ ] explain source builds
[ ] explain platform/architecture matrices
[ ] explain runtime-version compatibility
[ ] explain ABI compatibility
[ ] explain N-API versioning
[ ] explain process architecture compatibility
[ ] explain OS compatibility
[ ] explain libc compatibility at a high level
[ ] explain compiler/runtime ABI concerns
[ ] explain C++ ABI concerns
[ ] explain stable C ABI concepts
[ ] explain napi_env
[ ] explain napi_value
[ ] explain napi_ref
[ ] explain napi_handle_scope
[ ] explain napi_escapable_handle_scope
[ ] explain napi_callback_info
[ ] explain napi_status
[ ] explain napi_extended_error_info
[ ] explain JavaScript values from native code
[ ] create primitive JS values from native code
[ ] extract primitive values from JS
[ ] create objects
[ ] create arrays
[ ] call JS functions
[ ] define native methods
[ ] define native properties
[ ] define classes
[ ] create instances
[ ] wrap native objects
[ ] unwrap native objects
[ ] use external values carefully
[ ] explain finalizers
[ ] explain references
[ ] explain GC interaction
[ ] explain environment lifecycle
[ ] explain cleanup hooks
[ ] explain instance data
[ ] explain module initialization
[ ] explain addon context
[ ] explain per-environment state
[ ] explain multiple Node environments
[ ] understand context awareness
[ ] explain worker-thread addon behavior
[ ] explain shared addon library vs per-environment state
[ ] explain thread safety
[ ] distinguish JS thread from native thread
[ ] explain libuv interaction
[ ] explain async worker execution
[ ] explain napi_create_async_work
[ ] explain execute callback
[ ] explain complete callback
[ ] explain async resource identity
[ ] avoid blocking the event loop
[ ] design thread-safe native work
[ ] understand data races
[ ] understand atomics at a high level
[ ] understand lock ownership
[ ] understand deadlocks
[ ] understand native memory ownership
[ ] explain Buffer memory
[ ] explain ArrayBuffer backing stores
[ ] explain externalized memory
[ ] explain zero-copy
[ ] explain copy vs view trade-offs
[ ] explain pointer lifetime
[ ] explain use-after-free
[ ] explain double-free
[ ] explain buffer overflow
[ ] explain use-after-detach
[ ] explain invalid pointer
[ ] explain stale callback
[ ] explain stale native handle
[ ] explain finalizer ordering
[ ] explain reference lifetime
[ ] explain native-to-JS ownership
[ ] explain JS-to-native ownership
[ ] build a native object wrapper
[ ] build a native buffer wrapper
[ ] expose synchronous native function
[ ] expose asynchronous native function
[ ] expose native error
[ ] expose native class
[ ] expose native resource lifecycle
[ ] understand FFI in Node
[ ] explain node:ffi
[ ] understand experimental FFI status
[ ] explain dynamic libraries
[ ] use ffi.dlopen conceptually
[ ] explain dlsym
[ ] explain dlclose
[ ] explain foreign function signatures
[ ] explain integer type widths
[ ] explain pointer parameters
[ ] explain bigint for 64-bit integers
[ ] explain callback pointers
[ ] explain native callback lifetime
[ ] explain function-pointer safety
[ ] explain raw memory APIs
[ ] explain ffi.getRawPointer
[ ] explain FFI pointer invalidation
[ ] explain native string ownership
[ ] explain exportString
[ ] explain exportBuffer
[ ] explain exportArrayBuffer
[ ] understand temporary string copies
[ ] distinguish copying from zero-copy
[ ] explain calling conventions
[ ] explain architecture differences
[ ] understand FFI fast-call limitations
[ ] understand generic FFI path
[ ] explain platform-specific ABI
[ ] understand shared library suffixes
[ ] understand loader behavior
[ ] understand symbol visibility
[ ] explain native library lifetime
[ ] safely close dynamic libraries
[ ] avoid closing a library while callbacks can run
[ ] understand callback thread restrictions
[ ] understand FFI exception restrictions
[ ] explain why FFI can crash the process
[ ] explain why FFI is unsafe
[ ] compare N-API with FFI
[ ] compare native addon with child process
[ ] compare native addon with worker
[ ] compare native addon with WASI
[ ] compare native addon with WebAssembly
[ ] understand sandbox limitations
[ ] understand native-code trust boundary
[ ] explain Node Permission Model addon restrictions
[ ] explain --allow-addons
[ ] explain --allow-ffi
[ ] understand Permission Model is not native sandboxing
[ ] understand OS-level native isolation
[ ] explain security implications of native code
[ ] threat model native dependencies
[ ] threat model downloaded binaries
[ ] threat model postinstall builds
[ ] threat model compromised compiler toolchain
[ ] threat model malicious shared library
[ ] validate native binary provenance
[ ] use checksums/signatures conceptually
[ ] understand SBOM implications
[ ] understand reproducible native builds
[ ] explain cross-compilation
[ ] explain prebuild selection
[ ] explain optional native dependencies
[ ] explain npm package install scripts
[ ] explain package manager compatibility
[ ] explain runtime architecture selection
[ ] explain CPU architecture
[ ] explain OS triplets
[ ] explain Node runtime constraints
[ ] explain node_modules native binary placement
[ ] explain package distribution strategy
[ ] understand source fallback
[ ] understand prebuilt fallback
[ ] understand CI native matrix
[ ] test addon loading
[ ] test native methods
[ ] test native errors
[ ] test memory behavior
[ ] test concurrent use
[ ] test worker usage
[ ] test shutdown behavior
[ ] test repeated load/unload
[ ] test package installation
[ ] test multiple Node versions
[ ] test multiple platforms
[ ] debug native crashes
[ ] use core dumps conceptually
[ ] use debugger for C/C++
[ ] use AddressSanitizer conceptually
[ ] use UndefinedBehaviorSanitizer conceptually
[ ] use ThreadSanitizer conceptually
[ ] use Valgrind conceptually
[ ] distinguish JS stack from native stack
[ ] diagnose SIGSEGV
[ ] diagnose SIGABRT
[ ] diagnose illegal instruction
[ ] diagnose symbol mismatch
[ ] diagnose ABI mismatch
[ ] diagnose missing shared library
[ ] diagnose unresolved symbol
[ ] diagnose architecture mismatch
[ ] diagnose native memory leak
[ ] diagnose event-loop blocking
[ ] diagnose native deadlock
[ ] diagnose worker-thread race
[ ] design native resource finalization
[ ] design cancellation
[ ] design timeout
[ ] design shutdown
[ ] design versioning
[ ] design compatibility
[ ] build production-grade native integration


# 2. Prerequisites

You should already understand:

```text
Chapter 52 — Workers / Concurrency
Chapter 53 — Streams
Chapter 63 — Diagnostics
Chapter 70 — Production Debugging
Chapter 71 — Security
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging Methodology
Chapter 127 — Shared Memory / Memory Model
Chapter 140 — Node HTTP/TLS/DNS/TCP
Chapter 141 — Node Diagnostics & Inspector
Chapter 142 — Node Permission Model & Runtime Isolation
```

Supporting systems concepts:

```text
C/C++
pointers
heap/stack
threads
mutexes
atomics
shared libraries
linking
symbol resolution
ABI
OS processes
signals
```

---

# 3. What Is a Native Addon?

A native addon is code written outside JavaScript that is loaded by Node.js as a module.

Node's official documentation describes addons as dynamically linked shared objects loaded through `require()`, and identifies three implementation approaches:

```text
Node-API
nan
direct V8 / libuv / Node APIs
```

with Node-API recommended for most new addons. citeturn212928search2

Conceptually:

```text
JavaScript
    ↓
Native addon boundary
    ↓
C/C++
    ↓
OS / CPU / external native library
```

---

# 4. Why Native Addons Exist

Native code can be justified for:

```text
existing C/C++ library integration
special hardware APIs
cryptography/native algorithms
compression
image/video processing
database engines
OS-specific functionality
performance-critical native workloads
```

But:

```text
native ≠ automatically faster.
```

---

# 5. Native Boundary Cost

Crossing:

```text
JS
→ native
```

can involve:

```text
argument conversion
allocation
copying
thread synchronization
context switches
lifetime management.
```

A native implementation can therefore be slower for:

```text
tiny operations
called millions of times
```

than optimized JavaScript.

---

# 6. Native Boundary Risk

Native code can cause:

```text
memory corruption
process crash
deadlock
data race
use-after-free
double-free
ABI failure
```

These failure modes exist because:

```text
native memory
```

is not garbage-collected in the same way as:

```text
JavaScript objects.
```

---

# 7. Three Addon Strategies

## Strategy A — Node-API

```text
stable ABI-oriented Node interface
```

Recommended for most new production addons. citeturn212928search0turn212928search2

## Strategy B — node-addon-api / NAN

```text
C++ convenience/abstraction
```

with different stability trade-offs.

## Strategy C — Direct V8

```text
maximum runtime-level control
maximum coupling.
```

---

# 8. Node-API

Node-API is:

```text
a stable API/ABI-oriented native interface
```

maintained as part of Node.js.

Current Node v26 documentation states that Node-API is stable, is independent of the underlying JS runtime such as V8, and is intended to provide ABI stability across Node.js versions. citeturn212928search0

---

# 9. Why ABI Stability Matters

Without ABI stability:

```text
addon compiled for Node X
```

might fail under:

```text
Node X+1
```

and require:

```text
recompilation.
```

Node-API is designed to reduce this coupling.

---

# 10. ABI vs API

### API

```text
source-level interface
```

### ABI

```text
binary-level calling/layout contract
```

An API can remain:

```text
source-compatible
```

while ABI changes break:

```text
already-compiled binaries.
```

---

# 11. Node-API Boundary

```text
Addon
 ↓
Node-API
 ↓
Node runtime
 ↓
V8 / underlying runtime
```

The goal is:

```text
addon does not depend directly on V8 internals.
```

---

# 12. Direct V8 Addon

Direct V8 code looks conceptually like:

```cpp
v8::Isolate*
v8::Local<v8::Value>
v8::FunctionCallbackInfo
```

This can provide:

```text
runtime-specific capabilities
```

but creates:

```text
strong V8 coupling
```

and:

```text
rebuild/compatibility burden.
```

---

# 13. NAN

NAN historically provides:

```text
C++ abstractions around V8 version differences.
```

It reduces some source-level pain but:

```text
does not provide Node-API-style ABI stability.
```

---

# 14. node-addon-api

Node-API can be consumed through:

```text
C API
```

or:

```text
node-addon-api
```

a C++ wrapper.

Important distinction:

```text
Node-API = underlying stable C-oriented API
node-addon-api = C++ convenience layer.
```

---

# 15. `.node` Module

Compiled native addons are usually distributed as:

```text
*.node
```

files.

Conceptually:

```text
require("./build/Release/addon.node")
```

loads:

```text
dynamic native module.
```

---

# 16. Module Loading

Native loading involves:

```text
path resolution
shared-library loading
symbol resolution
addon initialization.
```

Failures can therefore occur before:

```text
your JavaScript function
```

ever executes.

---

# 17. Common Load Failures

Examples:

```text
module not found
wrong architecture
missing shared library
unresolved symbol
wrong ABI
permission denied
loader failure.
```

---

# 18. Build Systems

Common native addon tooling includes:

```text
node-gyp
CMake
Meson
Ninja
Visual Studio
Xcode
Make
```

The exact build system depends on:

```text
project
platform
native dependency.
```

---

# 19. node-gyp

`node-gyp` provides tooling to build native Node addons using:

```text
Python
native compiler
platform build system
binding.gyp
```

Treat:

```text
build prerequisites
```

as part of:

```text
developer experience
```

and:

```text
CI reliability.
```

---

# 20. binding.gyp

A `binding.gyp` file describes:

```text
targets
sources
include paths
compiler flags
libraries
platform-specific settings.
```

Example:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "src/addon.cc"
      ]
    }
  ]
}
```

---

# 21. Native Build Pipeline

```text
source
 ↓
compiler
 ↓
object files
 ↓
linker
 ↓
shared library
 ↓
.node addon
```

---

# 22. Compiler ABI

Native compatibility can depend on:

```text
compiler
standard library
runtime library
architecture
calling convention
compile flags.
```

Avoid exposing:

```text
C++ STL object layouts
```

across ABI boundaries unless:

```text
both sides share a controlled toolchain.
```

---

# 23. Why C ABI Helps

C-style interfaces tend to have:

```text
simpler binary contracts
```

than arbitrary C++ class layouts.

Node-API uses:

```text
a stable C-oriented interface
```

to help provide ABI stability. citeturn212928search0

---

# 24. Architecture Matrix

Common architectures include:

```text
x64
arm64
arm
riscv64
```

not every native dependency supports every:

```text
OS
architecture
```

combination.

---

# 25. OS Matrix

Examples:

```text
Linux
Windows
macOS
```

can differ in:

```text
dynamic library format
system calls
compiler
filesystem
threading
linker.
```

---

# 26. Library Suffix

Typical native library suffixes:

```text
Linux/Unix → .so
macOS → .dylib
Windows → .dll
```

Node's current experimental FFI documentation exposes `ffi.suffix` for these platform-dependent suffixes. citeturn212928search1

---

# 27. Runtime Matrix

Native distribution must account for:

```text
Node version
N-API version
OS
architecture
libc/runtime
native dependencies.
```

Node-API reduces:

```text
Node-version rebuild coupling
```

but does not remove:

```text
OS/architecture/native-library constraints.
```

---

# 28. Prebuilt Binaries

Packages often distribute:

```text
prebuilt native binaries
```

to avoid local compilation.

Benefits:

```text
fast install
simpler developer setup
fewer compiler failures.
```

Costs:

```text
large package matrix
supply-chain risk
download complexity
platform coverage.
```

---

# 29. Source Build Fallback

If no prebuilt binary exists:

```text
download source
→ compile locally
```

This requires:

```text
compiler toolchain
Python/build system
headers
native dependencies.
```

---

# 30. Prebuild Selection

A package may choose binaries based on:

```text
OS
architecture
runtime
N-API support
```

The selection mechanism must be:

```text
deterministic
well-tested.
```

---

# 31. Native Package Layout

Example:

```text
package/
  package.json
  src/
  prebuilds/
    linux-x64/
    linux-arm64/
    darwin-arm64/
    win32-x64/
  build/
  lib/
```

Keep:

```text
runtime selection
```

separate from:

```text
business logic.
```

---

# 32. Install Scripts

Native packages often need:

```text
install
postinstall
```

scripts to:

```text
download/build binaries.
```

This expands:

```text
supply-chain execution surface.
```

---

# 33. Native Dependency Threat

A compromised native package can potentially:

```text
execute arbitrary native code
```

with:

```text
process privileges.
```

This is a deeper trust boundary than:

```text
ordinary typed JavaScript.
```

---

# 34. Supply Chain Security

For native dependencies consider:

```text
source provenance
binary provenance
checksums
signatures
SBOM
reproducible builds
pinned versions
trusted CI.
```

---

# 35. Node-API Versioning

Node-API evolves by:

```text
versioned capabilities.
```

An addon built against an older compatible Node-API version can be designed to continue working across later Node versions.

Do not interpret:

```text
Node-API stability
```

as:

```text
all newer Node APIs are automatically available.
```

---

# 36. Feature Detection

Native code can use:

```text
runtime-supported Node-API features
```

where the API provides version/feature mechanisms.

Build:

```text
capability checks
```

rather than:

```text
assuming newest feature.
```

---

# 37. napi_env

A `napi_env` represents:

```text
a Node-API environment/context
```

in which native code interacts with:

```text
JavaScript values
runtime state
```

Treat it as:

```text
environment-owned state
```

not:

```text
process-global JS runtime pointer.
```

---

# 38. napi_value

A `napi_value` represents:

```text
a JavaScript value.
```

Native code uses it to manipulate:

```text
numbers
strings
objects
arrays
functions
symbols
BigInts
```

through Node-API functions.

---

# 39. Handle Scope Mental Model

Native interaction may create:

```text
temporary handles
```

that should not outlive:

```text
their scope.
```

Handle-scope APIs help:

```text
manage native references to JS values
```

during native execution.

---

# 40. Persistent References

If native code must retain a JS object/function beyond a callback:

```text
napi_ref
```

or equivalent reference management is required.

Do not retain:

```text
temporary local handles
```

for later use.

---

# 41. Reference Counting

References can keep:

```text
JavaScript object
```

alive.

Therefore:

```text
reference retained
→ object retained
→ GC cannot reclaim.
```

This creates:

```text
native-induced memory retention
```

possibilities.

---

# 42. Weak References

A weak reference may allow:

```text
GC
```

to collect the object while native code stops treating it as strongly owned.

But callback/resource semantics must be designed carefully.

---

# 43. Finalizers

Finalizers run when:

```text
associated JS object/resource
```

is being collected according to the API/lifetime model.

Never assume:

```text
finalizer = prompt deterministic destructor.
```

GC timing is not deterministic.

---

# 44. Native Resource Ownership

Suppose:

```text
JS object
   ↕
native handle
```

Define:

```text
who owns native handle?
when created?
who releases?
what if JS is GCed?
what if explicit close occurs?
```

---

# 45. Explicit Close + Finalizer

A robust native resource often supports:

```js
resource.close();
```

plus:

```text
finalizer fallback.
```

This combines:

```text
deterministic cleanup
+
leak protection.
```

---

# 46. Double-Free Risk

Bad:

```text
close()
→ native free

GC
→ finalizer
→ native free again.
```

Prevent with:

```text
closed flag
atomic ownership state
```

or:

```text
single-owner transfer.
```

---

# 47. Native Resource State Machine

```text
NEW
 ↓
ACTIVE
 ↓
CLOSING
 ↓
CLOSED
```

Error:

```text
ACTIVE
 ↓
FAILED
 ↓
CLOSED
```

No transition:

```text
CLOSED → ACTIVE
```

unless the API explicitly creates a new resource.

---

# 48. Instance Data

Native module state should be associated with:

```text
correct environment
```

rather than:

```text
global singleton
```

when the addon may be loaded into:

```text
multiple Node contexts/workers.
```

---

# 49. Context Awareness

An addon may be used:

```text
main thread
worker thread
multiple environments.
```

Avoid:

```text
process-global mutable native state
```

unless:

```text
thread-safety and lifetime
```

are explicitly designed.

---

# 50. Worker Threads + Addons

Workers execute:

```text
separate JS environments.
```

Native addon code must be compatible with:

```text
multi-environment
and/or multi-thread usage
```

as applicable.

Current Node documentation notes that native addons can only be loaded from multiple threads when they fulfill certain conditions. citeturn212928search6

---

# 51. Shared Native Library

The same native shared library may exist in:

```text
multiple workers.
```

This does not mean:

```text
all state is automatically isolated.
```

Static/global native variables can remain:

```text
process-wide shared state.
```

---

# 52. Thread Safety

Native code must decide:

```text
which state is thread-local
which is shared
which lock protects it
```

.

Do not assume:

```text
“Node uses one JS thread”
```

means:

```text
native code never runs concurrently.
```

---

# 53. JS Thread vs Native Thread

JS callbacks typically run in:

```text
Node's JavaScript execution context.
```

Native work may run on:

```text
other threads
```

through:

```text
libuv
custom thread pool
native worker threads.
```

---

# 54. Event Loop Rule

Never perform long native computation:

```text
inside the JS-facing callback
```

if it blocks:

```text
the event loop.
```

Use:

```text
async native work
```

for:

```text
CPU-heavy operations.
```

---

# 55. `napi_create_async_work`

Node-API provides an async work abstraction where native code can:

```text
execute
```

on worker infrastructure and then:

```text
complete
```

back in JavaScript execution context.

Conceptual lifecycle:

```text
queue
 ↓
execute
 ↓
complete
```

---

# 56. Async Execute Callback

The execute portion should:

```text
avoid JS API calls
```

unless explicitly supported by the API/thread context.

Perform:

```text
native compute
I/O
native library work
```

there.

---

# 57. Async Complete Callback

The complete callback runs where:

```text
Node/JS interaction
```

is valid.

Use it to:

```text
create JS values
resolve/reject
release operation state.
```

---

# 58. Async Cancellation

A good native async operation should support:

```text
cancel
```

when possible.

But cancellation is difficult once:

```text
native operation
```

is already executing.

Design:

```text
cooperative cancellation
```

or:

```text
safe discard of stale result.
```

---

# 59. Stale Native Completion

Race:

```text
request A
→ native operation

user cancels
→ state closed

native A completes
→ callback
```

The completion path must detect:

```text
operation no longer current.
```

---

# 60. Native Callback Lifetime

If native code stores:

```text
JS callback
```

it must manage:

```text
reference lifetime
```

carefully.

Do not call:

```text
stale callback pointer
```

after:

```text
JS object finalized
```

or:

```text
native environment destroyed.
```

---

# 61. Native-to-JS Callback

A native worker can eventually request:

```text
execute JS callback
```

on:

```text
valid JS thread/environment.
```

It must not simply:

```text
call into V8/Node from arbitrary native thread.
```

---

# 62. Thread-Safe Function Concept

Node-API provides abstractions for safely scheduling calls into JavaScript from native threads.

Use them instead of:

```text
direct cross-thread JS invocation.
```

---

# 63. Data Race

Example:

```text
native thread A:
counter++

native thread B:
counter++
```

without synchronization.

Result:

```text
lost updates
undefined behavior
```

depending on implementation.

---

# 64. Mutex

Protect shared native state with:

```text
mutex
```

when needed.

But every mutex introduces:

```text
contention
deadlock risk.
```

---

# 65. Deadlock

Classic:

```text
thread A holds lock 1 → waits lock 2
thread B holds lock 2 → waits lock 1
```

Both stop forever.

Native addons must define:

```text
lock ordering.
```

---

# 66. Atomics

For certain shared counters/state:

```text
atomic operations
```

can avoid:

```text
coarse locks.
```

But atomics are not:

```text
universal replacement for synchronization.
```

---

# 67. Native Memory

Native code may allocate:

```text
malloc
new
aligned allocation
OS resources
library handles.
```

Every allocation needs:

```text
owner
lifetime
release path.
```

---

# 68. Buffer Memory

Node `Buffer` is backed by binary memory.

Native code can often receive:

```text
pointer + length
```

to its storage under controlled APIs.

The critical rule:

```text
pointer lifetime must not exceed backing memory lifetime.
```

---

# 69. Zero-Copy

Zero-copy means:

```text
native code operates on existing memory
```

instead of:

```text
JS → copy
→ native buffer.
```

Benefits:

```text
less copying
lower memory bandwidth
```

Costs:

```text
lifetime complexity
aliasing
mutability
synchronization.
```

---

# 70. Copy vs Zero-Copy

### Copy

```text
safe ownership
simple lifetime
extra memory/time.
```

### Zero-copy

```text
high performance
shared lifetime
higher correctness risk.
```

Use zero-copy only when:

```text
measurement justifies complexity.
```

---

# 71. Detached/Transferred Memory

ArrayBuffer-backed storage can have lifecycle transitions such as:

```text
transfer
detach
resize
```

depending on the API/object type.

A native pointer into such memory can become:

```text
invalid.
```

---

# 72. Use-After-Free

```text
native pointer
→ object freed
→ pointer reused
→ native access
```

This can cause:

```text
crash
corruption
security vulnerability.
```

---

# 73. Double-Free

```text
free(pointer)
free(pointer)
```

is invalid.

Potential results:

```text
heap corruption
crash
arbitrary behavior.
```

---

# 74. Buffer Overflow

Native code must validate:

```text
pointer
length
capacity
```

before:

```text
read
write
```

.

Never trust JavaScript input:

```text
as a correct native length.
```

---

# 75. Integer Overflow

A native allocation like:

```text
count * elementSize
```

can overflow integer types.

Before allocation:

```text
validate bounds
```

and:

```text
check multiplication.
```

---

# 76. Signed/Unsigned Conversion

JS numbers:

```text
double precision
```

are not automatically equivalent to:

```text
uint64_t
int64_t
size_t
```

.

Use:

```text
BigInt
```

where exact 64-bit integer semantics are required.

---

# 77. Native Errors

Map native failures into:

```text
Error
```

objects with:

```text
code
message
cause
```

and:

```text
stable application semantics.
```

Do not expose:

```text
raw native pointer values
```

or:

```text
internal memory addresses
```

to users unless strictly required.

---

# 78. Error Translation

Native:

```text
errno = ENOENT
```

can become:

```js
{
  code: "ENOENT",
  message: "Resource not found"
}
```

Keep:

```text
diagnostic native code
```

and:

```text
user-facing explanation
```

separate.

---

# 79. Exceptions Across Native Boundary

Node-API operations generally report errors through:

```text
napi_status
```

and JS exception state.

Do not:

```text
throw C++ exception
```

across a C ABI boundary unless the API explicitly supports the mechanism.

---

# 80. C++ Exceptions

If native C++ code uses exceptions internally:

```text
catch them
```

and translate into:

```text
Node/JavaScript error.
```

Do not let unmanaged C++ exceptions cross unsupported ABI boundaries.

---

# 81. Native Class Design

A native class wrapper typically needs:

```text
constructor
methods
properties
native state
unwrap
finalizer
close
```

---

# 82. Native Instance Layout

Conceptually:

```text
JS object
  ↓
native wrapper
  ↓
NativeObject*
  ↓
resource/library handle
```

The JS object should not expose:

```text
raw pointer
```

as normal application state.

---

# 83. Wrap / Unwrap

Wrapping associates:

```text
native object
```

with:

```text
JS object.
```

Later native callbacks can:

```text
recover native state
```

from:

```text
JS receiver/object.
```

---

# 84. Ownership Transfer

If:

```text
native function returns handle
```

define:

```text
who owns it?
```

Possible contracts:

```text
borrowed
owned
shared
reference-counted
```

No ownership contract means:

```text
future crash.
```

---

# 85. Native Handle Wrapper

Example conceptual state:

```cpp
struct Handle {
  NativeHandle* ptr;
  bool closed;
};
```

Then:

```text
close()
→ if not closed:
     release(ptr)
     closed = true
```

---

# 86. Finalizer Strategy

Finalizer should:

```text
release native resource
```

only when:

```text
resource still belongs to JS object
```

and:

```text
explicit close has not already transferred ownership.
```

---

# 87. Cleanup Hooks

At environment shutdown:

```text
native resources
```

may need cleanup.

Register:

```text
environment cleanup hook
```

where appropriate.

---

# 88. Shutdown Ordering

A native addon should handle:

```text
JS environment shutting down
```

before:

```text
background native threads
```

attempt:

```text
JS callbacks.
```

---

# 89. Native Thread Shutdown

Correct sequence:

```text
request stop
→ signal thread
→ wait for completion
→ unregister callback
→ free state
→ environment cleanup.
```

Do not:

```text
destroy native object
```

while:

```text
native thread
```

still references it.

---

# 90. Worker Shutdown

Worker termination can stop:

```text
JavaScript execution
```

at arbitrary points according to Node worker semantics.

Native resources associated with:

```text
worker environment
```

must remain safe.

Current Node worker documentation warns that native addons loaded from multiple threads need appropriate conditions and that worker execution can stop through `worker.terminate()`. citeturn212928search6

---

# 91. Multiple Environments

Node can create multiple:

```text
Isolate/Environment
```

contexts.

Native global state should not assume:

```text
one environment forever.
```

---

# 92. Context-Specific State

Prefer:

```text
environment-owned state
```

over:

```text
global singleton
```

for:

```text
JS callback references
per-environment caches
resource registries.
```

---

# 93. Global State Risk

Bad:

```cpp
static NativeManager manager;
```

without:

```text
thread safety
environment ownership
shutdown strategy.
```

Potential problems:

```text
worker cross-talk
shutdown race
memory leak
use-after-free.
```

---

# 94. Async Resource Identity

For native async operations, associate:

```text
request ID
operation ID
trace ID
```

with:

```text
native task.
```

This makes debugging:

```text
native → JS
```

flows much easier.

---

# 95. Native Observability

Expose metrics such as:

```text
native operation count
active operations
duration
queue depth
native errors
native memory
worker count
```

Do not expose:

```text
raw pointers
```

or:

```text
sensitive native data.
```

---

# 96. Native Performance

Benchmark:

```text
boundary crossing
argument conversion
copying
native work
callback completion.
```

A native function with:

```text
5 μs computation
+
10 μs boundary
```

may not be worthwhile.

---

# 97. Batch Native Calls

Instead of:

```text
100,000 JS → native calls
```

consider:

```text
one native call
with 100,000 items
```

if the algorithm allows.

This amortizes:

```text
boundary overhead.
```

---

# 98. Native SIMD

Native code may provide:

```text
SIMD
vectorized CPU
```

but:

```text
memory movement
```

can dominate.

Measure:

```text
end-to-end throughput
```

not:

```text
kernel benchmark alone.
```

---

# 99. Native vs Worker

A native addon can:

```text
execute faster
```

but still:

```text
block the event loop
```

if called synchronously.

A worker can provide:

```text
execution isolation from main JS thread.
```

but has:

```text
message/transfer overhead.
```

---

# 100. Native vs Child Process

### Native addon

```text
same process
low call overhead
high crash blast radius
no process boundary.
```

### Child process

```text
IPC overhead
separate memory
stronger fault boundary
independent crash.
```

Choose based on:

```text
risk
latency
resource isolation.
```

---

# 101. Native vs WASM

### Native

```text
maximum OS/native integration
full process privilege
platform-specific.
```

### WebAssembly

```text
portable execution model
sandboxed memory model
limited direct OS access
```

depending on host/runtime capabilities.

WebAssembly is not automatically:

```text
zero risk
```

but usually has a different isolation model.

---

# 102. Native vs FFI

### N-API addon

```text
typed integration with Node
stable API/ABI boundary
custom compiled module.
```

### FFI

```text
load arbitrary native libraries
declare signatures at runtime
raw pointer access.
```

Current Node v26 `node:ffi` is experimental and explicitly marked unsafe. citeturn212928search1turn212928search5

---

# 103. Node `node:ffi`

Node v26 introduces:

```text
node:ffi
```

as an experimental module for:

```text
loading dynamic libraries
calling native symbols
raw memory access
native callbacks
```

It was added in:

```text
Node v26.1.0
```

and is gated by:

```text
--experimental-ffi
```

with `--allow-ffi` required under the Permission Model. citeturn212928search1turn212928search5

---

# 104. FFI Is Unsafe

Current Node documentation explicitly warns that:

```text
invalid pointers
wrong signatures
use-after-free
```

can:

```text
crash the process
or corrupt memory.
```

citeturn212928search1

This is fundamentally different from:

```text
ordinary JavaScript API misuse.
```

---

# 105. `ffi.dlopen()`

Conceptually:

```js
const lib =
  ffi.dlopen("./libexample.so", {
    add: {
      arguments: ["int32", "int32"],
      return: "int32"
    }
  });
```

The runtime constructs:

```text
JavaScript-callable wrappers
```

around:

```text
native symbols.
```

---

# 106. `ffi.dlsym()`

Dynamic libraries expose symbols such as:

```text
add
malloc
custom_function
```

FFI can resolve:

```text
symbol
→ native address.
```

The address is:

```text
unsafe native capability.
```

---

# 107. `ffi.dlclose()`

When a dynamic library is closed:

```text
its code/memory mapping
```

may become unavailable.

Never call:

```text
function pointer
```

after:

```text
owning library is closed.
```

---

# 108. Callback Pointer Lifetime

If native code stores:

```text
callback pointer
```

then:

```text
library.close()
```

while callback can still run is unsafe.

Current Node FFI documentation explicitly warns that closing/unregistering callback resources while they can execute can crash or corrupt memory. citeturn212928search1

---

# 109. FFI Callback Restrictions

Current Node FFI callback rules include:

```text
same system thread where created
must not throw
must not return Promise
return type must match signature
```

and additional lifetime restrictions. citeturn212928search1

---

# 110. Pointer Types

FFI supports:

```text
pointer
```

and primitive numeric types such as:

```text
int8
uint8
int16
uint16
int32
uint32
float32
float64
int64
uint64
```

Current docs use:

```text
BigInt
```

for 64-bit integer arguments/returns. citeturn212928search1

---

# 111. `pointer` Is Not an Object

A pointer is:

```text
memory address.
```

It does not automatically tell you:

```text
length
ownership
type
lifetime
thread safety.
```

A pointer without a contract is:

```text
unsafe.
```

---

# 112. Raw Pointer Access

Current Node FFI exposes:

```js
ffi.getRawPointer(source)
```

for:

```text
Buffer
ArrayBuffer
SharedArrayBuffer
ArrayBufferView
```

but explicitly warns that returned addresses can become invalid after:

```text
detach
resize
transfer
other invalidation.
```

citeturn212928search1

---

# 113. Pointer Lifetime Rule

```text
pointer valid
while backing allocation remains valid
```

not:

```text
pointer valid forever.
```

---

# 114. Pointer Arithmetic

Raw pointers plus:

```text
offset
length
```

can be powerful.

They are also where:

```text
memory corruption
```

becomes easy.

Validate:

```text
base
offset
end
capacity.
```

---

# 115. FFI String Conversion

Current FFI converts JavaScript strings to:

```text
temporary NUL-terminated UTF-8 strings
```

for the duration of a call when used as pointer-like arguments. citeturn212928search1

Therefore native code must not assume:

```text
pointer remains valid after call.
```

---

# 116. FFI Buffer Arguments

Buffers can be passed as:

```text
pointer-like references
```

to native memory.

This enables:

```text
zero-copy native processing.
```

But the native call must complete before:

```text
buffer lifetime ends.
```

---

# 117. FFI Export Helpers

Current FFI includes:

```text
exportString()
exportBuffer()
exportArrayBuffer()
exportArrayBufferView()
```

to copy JavaScript data into:

```text
native memory addresses.
```

These functions:

```text
do not allocate destination memory.
```

The caller must provide:

```text
valid writable native memory.
```

citeturn212928search1

---

# 118. FFI Fast Calls

Current Node documentation includes optimized FFI call paths with platform-specific limitations around:

```text
argument counts
register limits
floating-point arguments
buffer-shaped arguments.
```

When a signature is outside the optimized constraints, Node falls back to:

```text
generic FFI path.
```

citeturn212928search1

---

# 119. ABI Calling Convention

A native function signature depends on:

```text
argument order
argument width
return type
calling convention
register/stack placement
alignment.
```

If you declare:

```text
wrong signature
```

the result can be:

```text
garbage
crash
memory corruption.
```

---

# 120. FFI Signature Discipline

Before calling:

```text
add(int32, int32) → int32
```

confirm the native function actually has:

```text
the same ABI.
```

Never infer:

```text
from function name alone.
```

---

# 121. FFI and Structs

FFI becomes difficult when native APIs use:

```text
struct
union
bitfield
nested pointers
variable-length arrays
callbacks
```

because JavaScript must model:

```text
layout
alignment
ownership
```

exactly.

For complex APIs:

```text
N-API addon
```

may provide a safer integration boundary.

---

# 122. FFI and C++ APIs

C++ name mangling and object layout make:

```text
direct C++ FFI
```

fragile.

Prefer:

```text
C ABI wrapper
```

around:

```text
C++ implementation.
```

---

# 123. Shared Library Lifetime

Native library lifetime:

```text
open
→ symbols
→ calls
→ callbacks
→ close
```

All outstanding users must complete before:

```text
close.
```

---

# 124. FFI `using`

Current Node FFI exposes dynamic libraries with explicit resource-management behavior compatible with:

```text
using
```

where supported by the runtime's resource-management features. citeturn212928search1

This reinforces:

```text
library handles are resources
```

rather than:

```text
ordinary objects with zero-cost destruction.
```

---

# 125. FFI and Permission Model

When Permission Model is enabled:

```text
--allow-ffi
```

is required.

The `node:ffi` module also requires:

```text
--experimental-ffi
```

and appropriate build support. citeturn212928search1turn212928search4

---

# 126. Addon Permission

When Permission Model is enabled:

```text
--allow-addons
```

is required to load native addons.

Current Node Permission Model documentation lists native addon loading among restricted capabilities. citeturn513348search0turn513348search1

---

# 127. Permission Model Is Not Native Sandboxing

Permission Model can prevent:

```text
ordinary Node API paths
```

from being used.

It does not turn:

```text
native addon
```

into:

```text
safe untrusted code.
```

A native addon executes:

```text
inside the process.
```

---

# 128. Native Code + Permissions

A native addon can potentially use:

```text
OS APIs
system calls
native libraries
```

.

Therefore native code should be treated as:

```text
trusted computing base.
```

---

# 129. FFI + Security

FFI expands the attack surface because it can load:

```text
arbitrary dynamic libraries
```

and access:

```text
raw memory
```

.

Keep:

```text
FFI capability
```

off unless:

```text
explicitly required.
```

---

# 130. Native Addon vs Child Process Security

If the native dependency is:

```text
untrusted
```

consider:

```text
separate process
```

instead of:

```text
same-process addon.
```

This changes:

```text
crash blast radius
memory isolation
privilege boundary
```

.

---

# 131. Native Addon vs Worker Security

A worker isolates:

```text
JavaScript execution context
```

but not necessarily:

```text
native process memory
```

.

Do not use:

```text
worker
```

as a guaranteed native-code sandbox.

---

# 132. Strong Native Isolation

For high-risk native workloads:

```text
separate process
+
non-root user
+
container
+
restricted filesystem
+
restricted network
+
resource limits
```

may be appropriate.

---

# 133. Build-Time Security

A native package can execute code during:

```text
install
build
```

.

Therefore harden:

```text
CI runner
package install environment
compiler
credentials
network.
```

Runtime Permission Model does not automatically secure:

```text
build-time execution.
```

---

# 134. Reproducible Native Builds

Reproducible builds aim to ensure:

```text
same source
+
same toolchain
=
same binary
```

or a verifiable equivalent.

This improves:

```text
supply-chain confidence.
```

---

# 135. Native Binary Provenance

Record:

```text
source commit
compiler
compiler version
build flags
dependency versions
OS
architecture
build environment.
```

This creates:

```text
binary provenance.
```

---

# 136. Binary Verification

At distribution time consider:

```text
checksum
signature
trusted registry
artifact attestation
SBOM.
```

The exact mechanism depends on:

```text
organization/toolchain.
```

---

# 137. ABI Matrix

A native package should define:

```text
Node versions
Node-API versions
OS
architecture
compiler/runtime assumptions
native dependency versions.
```

---

# 138. CI Native Matrix

Example:

```text
Linux x64
Linux arm64
Windows x64
macOS x64
macOS arm64
```

and:

```text
supported Node releases.
```

Run:

```text
build
load
unit
integration
stress
leak
shutdown
```

tests.

---

# 139. Installation Test

For each published artifact:

```text
fresh temp project
→ npm install
→ require/import addon
→ execute smoke test
```

This catches:

```text
package layout
prebuild selection
missing binary
loader path
```

errors.

---

# 140. Runtime Test

Test:

```text
addon loads
API works
error works
multiple calls work
multiple instances work.
```

---

# 141. Worker Test

Load addon from:

```text
main thread
worker
multiple workers
```

where supported.

Test:

```text
concurrent use
resource cleanup
worker termination.
```

---

# 142. Repeated Lifecycle Test

Run:

```text
create
use
close
GC
repeat
```

thousands of times.

Look for:

```text
native memory growth
descriptor leaks
thread leaks
callback leaks.
```

---

# 143. Leak Testing

Track:

```text
RSS
native heap
external memory
file descriptors
threads
native resource count
```

across:

```text
repeated workloads.
```

---

# 144. Native Memory Leak

A process can show:

```text
heapUsed stable
RSS rising
```

because:

```text
native memory
```

is leaking.

Correlate with:

```text
native allocation tools
```

and:

```text
OS memory maps.
```

---

# 145. Native Crash Debugging

When process crashes:

```text
JavaScript stack
```

may be insufficient.

Use:

```text
native stack
core dump
debug symbols
```

where available.

---

# 146. SIGSEGV

Often indicates:

```text
invalid memory access
```

such as:

```text
use-after-free
null dereference
out-of-bounds access.
```

---

# 147. SIGABRT

Can indicate:

```text
assertion failure
explicit abort
runtime fatal condition.
```

Inspect:

```text
native stack
stderr
diagnostic report.
```

---

# 148. Illegal Instruction

Can occur when a binary uses:

```text
CPU instructions
```

unsupported by the target machine.

This is why:

```text
native architecture/build flags
```

matter.

---

# 149. Missing Shared Library

Error may indicate:

```text
dynamic loader cannot find dependency.
```

Investigate:

```text
library path
packaging
rpath
install location
system package.
```

---

# 150. Unresolved Symbol

Can indicate:

```text
version mismatch
ABI mismatch
missing library
wrong symbol visibility.
```

---

# 151. Debug Symbols

Keep symbols in:

```text
private artifact store
```

rather than:

```text
production package
```

where possible.

They greatly improve:

```text
native crash analysis.
```

---

# 152. AddressSanitizer

ASan can detect classes of:

```text
memory errors
```

including:

```text
out-of-bounds
use-after-free
```

during testing.

Do not treat:

```text
ASan-clean
```

as:

```text
proof of memory safety.
```

---

# 153. UndefinedBehaviorSanitizer

UBSan can detect certain classes of:

```text
undefined behavior
```

such as:

```text
invalid shifts
some overflow categories
alignment errors
```

depending on instrumentation.

---

# 154. ThreadSanitizer

TSan can detect many:

```text
data races
```

in supported native code/toolchains.

Use in:

```text
concurrency test configurations.
```

---

# 155. Valgrind

Valgrind can provide:

```text
dynamic memory diagnostics
```

and other checks.

It may impose:

```text
large performance overhead.
```

Use for:

```text
focused diagnostic runs.
```

---

# 156. Native Debugger

Use:

```text
gdb
lldb
Visual Studio debugger
```

to inspect:

```text
native stack
registers
memory
threads
symbols.
```

---

# 157. Mixed Stack Debugging

A crash may involve:

```text
JavaScript
 ↓
Node-API
 ↓
C++
 ↓
native dependency
```

A good debugger workflow preserves:

```text
both JS and native context.
```

---

# 158. Production Crash Workflow

```text
crash signal
 ↓
diagnostic report
 ↓
core dump if available
 ↓
native stack
 ↓
symbols
 ↓
addon version
 ↓
dependency version
 ↓
reproduce under sanitizer
 ↓
fix
 ↓
regression test.
```

---

# 159. Native Error Telemetry

Collect:

```text
addon version
Node version
OS
architecture
native dependency version
operation
error code
```

Do not collect:

```text
raw pointers
private memory
sensitive payloads
```

as ordinary metrics.

---

# 160. Native Diagnostics

Include:

```text
active native operation count
native queue length
native thread count
native memory estimates
load/unload count
callback count
```

where measurable.

---

# 161. ABI Failure Telemetry

On load failure record:

```text
expected runtime
actual runtime
architecture
binary filename
N-API level
native dependency versions
loader error.
```

---

# 162. Native Feature Flags

Gate native functionality behind:

```text
feature flag
```

when rollout risk is high.

Example:

```text
native implementation = 10%
JS fallback = 90%
```

This supports:

```text
safe rollout
rollback.
```

---

# 163. Native Fallback

If native addon cannot load:

```text
fallback to JS/WASM/service
```

when correctness allows.

Do not silently:

```text
use insecure fallback.
```

---

# 164. Correctness Parity

A native and JS implementation should share:

```text
same semantics
same edge cases
same error contract
```

when one is a fallback for the other.

---

# 165. Differential Testing

Run:

```text
JS implementation
vs
native implementation
```

on:

```text
same inputs
```

and compare:

```text
outputs
errors
edge cases
```

.

---

# 166. Fuzzing Native Boundaries

Native code should be fuzzed with:

```text
empty
tiny
huge
malformed
random
boundary
Unicode
overflow
```

inputs.

This is especially important for:

```text
binary parsers
protocols
image codecs
compression
```

---

# 167. Property-Based Native Testing

Define invariants such as:

```text
encode(decode(x)) = x
```

for valid data.

Then generate:

```text
many inputs
```

across:

```text
edge cases.
```

---

# 168. Native API Contract

Document:

```text
input type
input bounds
output ownership
error semantics
threading
blocking behavior
lifetime
cleanup
version compatibility.
```

---

# 169. Threading Contract

Every native function should state:

```text
main-thread only
worker-safe
thread-safe
reentrant
not reentrant
```

.

Ambiguity creates:

```text
race conditions.
```

---

# 170. Reentrancy

A native callback can cause:

```text
JavaScript
→ native
→ JS callback
→ native
```

again.

Do not assume:

```text
function cannot be called recursively.
```

Protect:

```text
state invariants.
```

---

# 171. Reentrant Lock Risk

A callback under:

```text
mutex
```

calls JavaScript.

JavaScript re-enters:

```text
same native code
```

and requests:

```text
same mutex.
```

Potential:

```text
deadlock.
```

Avoid:

```text
calling JS while holding locks
```

unless carefully designed.

---

# 172. Native Callback Exceptions

If native code invokes a JS callback:

```text
JS callback can fail.
```

Handle:

```text
pending exception
error state
resource cleanup.
```

Do not let:

```text
native state
```

remain:

```text
half-updated.
```

---

# 173. Callback Backpressure

If native thread produces:

```text
100,000 callbacks/s
```

but JS can process:

```text
10,000/s
```

you need:

```text
queue
coalescing
sampling
drop policy
backpressure.
```

---

# 174. Native Event Queue

Conceptual:

```text
native producer
 ↓
bounded queue
 ↓
JS scheduler
 ↓
callback
```

This prevents:

```text
unbounded callback accumulation.
```

---

# 175. Native Batch API

A better design can expose:

```text
poll(1000 events)
```

instead of:

```text
one callback per event.
```

This reduces:

```text
cross-boundary overhead.
```

---

# 176. Native Stream Adapter

For high-throughput native data:

```text
native producer
→ bounded native queue
→ Node stream
→ backpressure
```

can integrate:

```text
native throughput
```

with:

```text
Node's stream model.
```

---

# 177. Native + Event Loop

A well-designed native addon should:

```text
keep JS callback lightweight
keep native work off JS thread
respect event-loop scheduling.
```

---

# 178. Native + AbortSignal

If native operation maps to:

```text
async operation
```

allow:

```text
AbortSignal
```

or:

```text
cancel().
```

Cancellation should:

```text
stop work
or ignore stale completion
```

safely.

---

# 179. Native + Timeout

Do not rely solely on:

```text
JavaScript timer
```

if native code can remain active forever.

The native operation should know:

```text
deadline/cancel state
```

when possible.

---

# 180. Native + Graceful Shutdown

At process shutdown:

```text
stop accepting work
→ cancel/pause new native jobs
→ drain active work
→ stop threads
→ release libraries
→ environment cleanup.
```

---

# 181. Native + Node Permission Model

When Permission Model is enabled:

```text
native addon loading
```

requires:

```text
--allow-addons
```

.

But:

```text
native addon permission
```

does not automatically restrict:

```text
what the native library itself can do.
```

---

# 182. Native + Inspector

A native crash may need:

```text
Inspector
```

for JS-side state and:

```text
native debugger
```

for native-side state.

These are complementary.

---

# 183. Native + Diagnostic Reports

A diagnostic report can preserve:

```text
JS/native stacks
runtime state
OS details
V8 information
```

and is valuable before:

```text
full native debugging.
```

---

# 184. Native Crash + Heap Snapshot

Do not assume:

```text
heap snapshot
```

will explain:

```text
native memory corruption.
```

Use:

```text
core dump
native debugger
sanitizers.
```

---

# 185. Native ABI Contract

Document:

```text
Node-API version
compiler
OS
architecture
native dependencies
exported symbols
calling convention
memory ownership
threading model.
```

---

# 186. Versioning Strategy

Prefer:

```text
stable semantic JS API
```

while allowing:

```text
native implementation changes
```

behind it.

For ABI-sensitive native libraries:

```text
pin incompatible versions
```

and:

```text
test upgrade before release.
```

---

# 187. Native Library Upgrade

Never upgrade:

```text
OpenSSL
SQLite
FFmpeg
libpng
custom vendor SDK
```

in production purely by:

```text
minor version optimism.
```

Run:

```text
ABI
behavior
security
performance
stress
```

tests.

---

# 188. Binary Compatibility

A binary may fail because:

```text
symbol exists
but ABI changed.
```

Do not test only:

```text
library opens
```

.

Run:

```text
real operations
```

too.

---

# 189. C ABI Wrapper Strategy

For third-party C++ library:

```text
C++ library
 ↓
small C wrapper
 ↓
Node-API addon
```

This isolates:

```text
C++ ABI
```

from:

```text
Node ABI.
```

---

# 190. Thin Native Boundary

Prefer:

```text
small adapter
```

around:

```text
large native system.
```

Keep:

```text
business logic
```

in JavaScript when practical.

This reduces:

```text
native code surface
```

and:

```text
crash/debug complexity.
```

---

# 191. Native Boundary API Design

A good boundary exposes:

```text
simple values
opaque resource handles
bounded buffers
explicit errors
explicit close.
```

Avoid exposing:

```text
complex native structs
raw pointers
implicit lifetime.
```

---

# 192. Data Marshalling

Common crossings:

```text
number
string
Buffer
ArrayBuffer
TypedArray
BigInt
object
```

Measure:

```text
copy cost
conversion cost.
```

---

# 193. Object Conversion Cost

A deeply nested JS object crossing into native code may require:

```text
property traversal
type checks
allocation
conversion.
```

For high-throughput data:

```text
binary format
typed arrays
buffers
```

may be more efficient.

---

# 194. Serialization Boundary

Instead of:

```text
many JS objects
```

consider:

```text
contiguous binary representation
```

for:

```text
native processing.
```

This reduces:

```text
object conversion overhead.
```

---

# 195. Native String Pitfalls

Native APIs may use:

```text
NUL-terminated strings
```

while JavaScript strings can contain:

```text
embedded NUL
Unicode
surrogate pairs.
```

Document:

```text
encoding
length semantics
```

explicitly.

---

# 196. Encoding

Define:

```text
UTF-8
UTF-16
UCS-2
binary
```

contracts explicitly.

Do not assume:

```text
char
```

means:

```text
Unicode character.
```

---

# 197. Endianness

Binary native interfaces can depend on:

```text
byte order.
```

Portable binary protocols should define:

```text
endianness
```

rather than:

```text
host-native layout.
```

---

# 198. Alignment

Native structs can include:

```text
padding
alignment.
```

Never serialize:

```text
raw struct bytes
```

as a portable protocol without defining:

```text
layout.
```

---

# 199. Struct Evolution

If a native struct changes:

```text
field
padding
size
```

then:

```text
ABI may change.
```

Prefer:

```text
versioned functions
```

and:

```text
explicit field access.
```

---

# 200. Native Memory Ownership Matrix

| Resource | Created by | Used by | Freed by | Lifetime |
|---|---|---|---|---|
| Buffer backing | Node | native | Node | Buffer lifetime |
| Native handle | addon | addon/JS | addon | explicit close/finalizer |
| Dynamic library | FFI | JS/native | owner | until close |
| Callback | JS | native | owner/GC semantics | explicit contract |
| Worker thread | native | native | controller | until shutdown |

Every resource needs:

```text
one authoritative ownership rule.
```

---

# 201. Security Checklist

```text
[ ] native dependency trusted
[ ] binary provenance recorded
[ ] build pipeline hardened
[ ] install scripts reviewed
[ ] FFI disabled unless needed
[ ] addon permission minimized
[ ] native inputs bounded
[ ] pointer operations validated
[ ] integer overflow checked
[ ] callbacks lifetime-safe
[ ] shared state synchronized
[ ] secrets not logged
[ ] native crash handling documented
[ ] debugger access restricted
```

---

# 202. Performance Checklist

```text
[ ] boundary overhead measured
[ ] copies minimized
[ ] zero-copy justified
[ ] batching considered
[ ] native thread pool bounded
[ ] callback rate bounded
[ ] JS callback lightweight
[ ] native memory tracked
[ ] event-loop blocking absent
[ ] startup cost measured
[ ] binary loading cost measured.
```

---

# 203. Reliability Checklist

```text
[ ] explicit close
[ ] finalizer fallback
[ ] shutdown hook
[ ] cancellation
[ ] timeout
[ ] stale completion handling
[ ] worker termination safe
[ ] native thread join
[ ] library lifetime controlled
[ ] callback unregistration safe
[ ] error translation
[ ] crash runbook
```

---

# 204. Build/Release Checklist

```text
[ ] source build works
[ ] prebuild works
[ ] install script tested
[ ] binary selection tested
[ ] all architectures tested
[ ] all supported OS targets tested
[ ] supported Node versions tested
[ ] Node-API version documented
[ ] native dependencies pinned
[ ] artifact provenance stored
[ ] checksums/signatures available
[ ] SBOM generated where required
[ ] rollback artifact retained.
```

---

# 205. Implementation From Scratch — Node-API Addon

Build a learning addon:

```text
addon.add(a, b)
addon.version()
addon.createResource()
resource.value()
resource.close()
```

Use:

```text
Node-API
```

rather than:

```text
direct V8.
```

---

# 206. Implementation Milestone 1 — Module Initialization

Create:

```text
addon.cc
binding.gyp
package.json
```

Expose:

```text
hello()
```

and verify:

```text
load
call
result.
```

---

# 207. Implementation Milestone 2 — Primitive Conversion

Implement:

```text
number → native integer
string → native string
```

and:

```text
native integer → JS number/BigInt.
```

Add validation for:

```text
range
type
```

---

# 208. Implementation Milestone 3 — Native Error

Map:

```text
invalid input
```

to:

```text
JavaScript TypeError/RangeError/custom Error.
```

---

# 209. Implementation Milestone 4 — Native Resource Class

Build:

```text
NativeResource
```

with:

```text
open
read
close
```

and:

```text
explicit closed state.
```

---

# 210. Implementation Milestone 5 — Finalizer

Add:

```text
finalizer
```

as:

```text
leak protection.
```

Test:

```text
explicit close
+
GC.
```

---

# 211. Implementation Milestone 6 — Async Work

Build:

```text
native heavy operation
```

with:

```text
napi_create_async_work
```

or equivalent Node-API async abstractions.

Test:

```text
event loop remains responsive.
```

---

# 212. Implementation Milestone 7 — Cancellation

Add:

```text
cancel()
```

and:

```text
stale completion check.
```

---

# 213. Implementation Milestone 8 — Worker Safety

Load addon from:

```text
multiple workers.
```

Verify:

```text
no global-state cross-talk
no race
no use-after-environment.
```

---

# 214. Implementation Milestone 9 — Buffer Processing

Implement:

```text
hash(buffer)
```

or:

```text
transform(buffer)
```

without unnecessary copies.

Measure:

```text
copy
vs
zero-copy.
```

---

# 215. Implementation Milestone 10 — Native Queue

Implement:

```text
bounded queue
```

for:

```text
native events.
```

Add:

```text
drop policy
backpressure
metrics.
```

---

# 216. Implementation Milestone 11 — FFI Prototype

In a controlled Node v26 environment, use:

```text
node:ffi
```

to call a tiny:

```text
C shared library
```

such as:

```c
int add(int a, int b);
```

Compare:

```text
FFI
vs
Node-API addon.
```

Use only for:

```text
learning
controlled experiments.
```

---

# 217. Implementation Milestone 12 — ABI Experiment

Build:

```text
library v1
```

and:

```text
library v2
```

with a changed signature.

Call v2 using:

```text
v1 declaration.
```

Observe:

```text
incorrect result/crash risk.
```

Use:

```text
debug environment only.
```

---

# 218. Debugging Exercises

## Exercise A — Addon Won't Load

Error:

```text
Module did not self-register
```

Investigate:

```text
binary/runtime compatibility
loader initialization
N-API/runtime expectations.
```

---

## Exercise B — Undefined Symbol

Error:

```text
undefined symbol
```

Investigate:

```text
missing dependency
ABI mismatch
linker path
symbol visibility.
```

---

## Exercise C — Native Crash

Process:

```text
SIGSEGV
```

Investigate:

```text
native stack
pointer lifetime
sanitizer
core dump.
```

---

## Exercise D — RSS Growth

```text
heapUsed stable
RSS grows.
```

Investigate:

```text
native allocation
Buffer/ArrayBuffer
external memory
native library.
```

---

## Exercise E — Event-Loop Stall

A native method:

```text
takes 2 seconds
```

synchronously.

Measure:

```text
event-loop delay.
```

Move work to:

```text
async native execution.
```

---

## Exercise F — Double-Free

Sequence:

```text
close()
finalizer()
```

crashes process.

Fix:

```text
single ownership state.
```

---

## Exercise G — Worker Race

Two workers call:

```text
native increment()
```

and final count is wrong.

Investigate:

```text
shared static state
data race
missing synchronization.
```

---

## Exercise H — Stale Callback

Native operation completes after:

```text
JS resource closed.
```

Callback accesses:

```text
freed native object.
```

Fix:

```text
lifetime/token check
```

before:

```text
callback work.
```

---

# 219. Code Review Exercise — Raw Pointer

Review:

```js
const ptr = ffi.getRawPointer(buffer);

setTimeout(() => {
  nativeRead(ptr);
}, 5000);
```

Find:

```text
buffer lifetime not guaranteed
pointer may be invalid
buffer could be detached/transferred/resized
nativeRead may crash/corrupt memory.
```

Current Node FFI documentation explicitly warns about raw pointer invalidation. citeturn212928search1

---

# 220. Code Review Exercise — Wrong FFI Signature

Native:

```c
double multiply(double a, double b);
```

JS declaration:

```js
{
  arguments: ["int32", "int32"],
  return: "int32"
}
```

Find:

```text
ABI signature mismatch
```

and explain:

```text
why this is not a normal JavaScript type error.
```

---

# 221. Code Review Exercise — Callback Lifetime

Review:

```js
const lib =
  ffi.dlopen("./lib.so");

const callback =
  lib.registerCallback(
    { arguments: ["int32"] },
    () => {}
  );

lib.close();
```

Find:

```text
library closed while callback resource may remain active.
```

Current Node FFI documentation warns this can crash/corrupt memory. citeturn212928search1

---

# 222. Code Review Exercise — Native Singleton

Review:

```cpp
static NativeClient client;

Napi::Object Init(...) {
  return CreateExports(env, &client);
}
```

Ask:

```text
What happens with:
multiple workers?
multiple environments?
shutdown?
concurrent requests?
```

Potential risks:

```text
cross-context state
race
lifetime mismatch
```

---

# 223. Code Review Exercise — Blocking Native Call

```js
addon.processHugeFileSync(path);
```

The function:

```text
takes 8 seconds.
```

Find:

```text
event-loop blocking
timeout ignorance
poor cancellation
request starvation.
```

---

# 224. Predict-the-Behavior Exercises

### Exercise 1

A native addon stores:

```text
pointer to JS object
```

without:

```text
persistent reference.
```

Predict:

```text
why object lifetime may end while native code still uses it.
```

---

### Exercise 2

A native finalizer frees:

```text
resource
```

and later:

```js
resource.close();
```

also frees it.

Predict:

```text
double-free risk.
```

---

### Exercise 3

A synchronous native call consumes:

```text
5 seconds.
```

Predict:

```text
what happens to the Node event loop during the call.
```

---

### Exercise 4

Two workers update:

```text
one native global counter
```

without locking.

Predict:

```text
why final results can be incorrect.
```

---

### Exercise 5

A library is closed while native code may still call:

```text
callback pointer.
```

Predict:

```text
why later invocation can crash.
```

---

### Exercise 6

FFI declares:

```text
uint64
```

but receives:

```text
JavaScript number
```

Predict:

```text
why exact integer semantics may be lost.
```

Current FFI documentation specifies BigInt for 64-bit integer types. citeturn212928search1

---

### Exercise 7

A Node-API addon built against:

```text
older Node
```

runs under:

```text
newer supported Node
```

Predict:

```text
why Node-API is designed to improve this compatibility.
```

---

### Exercise 8

A native addon loads successfully on:

```text
x64
```

but fails on:

```text
arm64
```

Predict:

```text
why Node-API stability does not imply architecture portability.
```

---

# 225. Interview Questions

### Native Fundamentals

```text
1. Why do Node.js native addons exist?
2. What is a .node module?
3. What is Node-API?
4. Why is Node-API preferred for many new addons?
5. What is ABI stability?
6. How is ABI different from API?
```

### V8 / NAN / Node-API

```text
7. What is the difference between Node-API and direct V8 APIs?
8. What is NAN?
9. What is node-addon-api?
10. Why does direct V8 coupling increase maintenance cost?
```

### Memory

```text
11. What is napi_env?
12. What is napi_value?
13. Why do native references matter?
14. What is a finalizer?
15. How can a native addon cause a memory leak?
16. What is use-after-free?
17. What is double-free?
18. What is zero-copy?
```

### Async

```text
19. How do you run CPU-heavy native work without blocking Node?
20. What happens in the async execute callback?
21. What happens in the completion callback?
22. How do you cancel native async work?
23. How do you prevent stale completions?
```

### Threads

```text
24. Can native addons be used from workers?
25. What is the danger of global native state?
26. How do you make native code thread-safe?
27. How can a deadlock happen?
28. Why should native threads not directly call JS?
```

### FFI

```text
29. What is node:ffi?
30. Why is FFI unsafe?
31. What is a pointer?
32. Why is an incorrect FFI signature dangerous?
33. Why use BigInt for int64/uint64?
34. What is dlopen?
35. What is dlsym?
36. Why is dlclose dangerous with live callbacks?
```

### Security

```text
37. Does the Node Permission Model sandbox native addons?
38. When is --allow-addons required?
39. When is --allow-ffi required?
40. How would you isolate an untrusted native dependency?
41. How would you secure native build pipelines?
```

### Principal

```text
42. When would you choose Node-API over a child process?
43. When would you choose FFI instead of writing an addon?
44. How would you design a cross-platform native package?
45. How would you debug a SIGSEGV in production?
46. How would you prevent native ABI regressions?
47. How would you design native resource ownership?
48. How would you roll out a risky native optimization safely?
49. How would you compare native, WASM, worker, and child-process strategies?
```

---

# 226. Mastery Exercises

### Exercise 1 — Node-API Resource

Build:

```text
NativeResource
```

with:

```text
create
read
close
finalizer
```

---

### Exercise 2 — Async Native Compute

Implement:

```text
CPU-heavy native operation
```

with:

```text
async execution
cancellation
completion
```

---

### Exercise 3 — Worker Safety

Run:

```text
4 workers
```

against:

```text
native addon
```

and prove:

```text
correct state isolation.
```

---

### Exercise 4 — Memory Safety

Create test cases for:

```text
double close
stale pointer
oversized length
invalid state
```

and verify:

```text
safe rejection.
```

---

### Exercise 5 — Native/JS Differential Test

Implement:

```text
JS algorithm
native algorithm
```

and compare:

```text
10,000 random cases.
```

---

### Exercise 6 — FFI Controlled Lab

Write:

```c
int add(int a, int b);
uint64_t identity(uint64_t x);
```

Call them with:

```text
correct signature
wrong signature
boundary values.
```

Observe the difference.

Use:

```text
isolated development environment
```

for unsafe experiments.

---

### Exercise 7 — Native Crash Lab

Create a deliberately invalid pointer access in a test-only native program and diagnose it with:

```text
ASan
debugger
core dump.
```

Do not use unsafe crash experiments in production.

---

### Exercise 8 — Packaging Matrix

Publish a native package for:

```text
linux-x64
linux-arm64
darwin-x64
darwin-arm64
win32-x64
```

with:

```text
prebuild selection
source fallback
CI matrix
```

---

### Exercise 9 — ABI Compatibility

Test:

```text
library v1
addon declaration v1
library v2
```

and detect:

```text
incompatible signature/version.
```

---

### Exercise 10 — Native Incident Runbook

Write a runbook for:

```text
SIGSEGV
RSS growth
event-loop stall
worker race
missing shared library
```

---

# 227. Track A — Core Theory

Master:

```text
native addons
Node-API
ABI
API
V8
NAN
node-addon-api
napi_env
napi_value
references
handles
finalizers
cleanup hooks
native memory
buffers
ArrayBuffers
zero-copy
async native work
threading
callbacks
FFI
dynamic libraries
symbols
calling conventions
ABI packaging
native security
native diagnostics.
```

Deliverable:

```text
explain every boundary where JavaScript can become
native code, native memory, native threads, or native
libraries—and the lifetime/security implications.
```

---

# 228. Track B — Implementation

Build:

```text
Node-API addon
native resource wrapper
async native operation
thread-safe native state
native callback path
bounded native queue
FFI experiment
prebuilt package
cross-platform CI
sanitizer test suite
native crash diagnostic workflow.
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

# 229. Track C — Interview / Reasoning

Practice:

```text
“When is native code justified?”

“Why does Node-API improve ABI stability?”

“Why can't a worker be treated as a native security sandbox?”

“When is zero-copy worth the lifetime complexity?”

“How do you safely expose a native resource?”

“Why is FFI dangerous?”

“How would you debug a native crash?”

“How would you package a native module across five platforms?”

“When would you choose child process over native addon?”
```

Deliverable:

```text
requirement
+
boundary
+
ownership
+
threading
+
ABI
+
failure
+
security
+
trade-off.
```

---

# 230. Principal Decision Framework

For every native integration ask:

```text
1. Why is native code necessary?
2. Is JavaScript fast enough?
3. Is WebAssembly sufficient?
4. Is a worker sufficient?
5. Is a child process safer?
6. Is an existing native library the real requirement?
7. Can Node-API provide the boundary?
8. Can a thin C ABI wrapper isolate C++ ABI complexity?
9. What data crosses the boundary?
10. Who owns each allocation?
11. Who releases it?
12. What happens on GC?
13. What happens on explicit close?
14. What happens during shutdown?
15. What thread executes each operation?
16. Can native code call JS?
17. How is callback lifetime tracked?
18. Can operation be cancelled?
19. What happens if completion arrives late?
20. What happens if Node worker terminates?
21. What happens if native code crashes?
22. How is ABI compatibility tested?
23. How are binaries distributed?
24. What platforms are supported?
25. How is provenance verified?
26. Is FFI necessary?
27. Is --allow-addons required?
28. Is --allow-ffi required?
29. What OS/container boundary contains compromise?
30. How is memory observed?
31. How is native performance measured?
32. How is rollback performed?
```

---

# 231. Production Native Architecture

```text
JavaScript API
      ↓
Thin Node-API boundary
      ↓
Validation / ownership
      ↓
Native adapter
      ↓
C ABI wrapper
      ↓
Third-party native library
      ↓
OS / hardware
```

Supporting systems:

```text
Versioning
Testing
Sanitizers
CI matrix
Observability
Crash diagnostics
Security policy
Artifact provenance
```

---

# 232. Production Native Checklist

```text
[ ] native requirement justified
[ ] simpler alternative evaluated
[ ] Node-API preferred where appropriate
[ ] direct V8 coupling minimized
[ ] FFI justified separately
[ ] addon permission reviewed
[ ] FFI permission reviewed
[ ] native dependency reviewed
[ ] ABI documented
[ ] N-API version documented
[ ] supported Node versions documented
[ ] OS/architecture matrix documented
[ ] build reproducibility considered
[ ] binary provenance recorded
[ ] prebuild/source fallback tested
[ ] ownership documented
[ ] explicit close implemented
[ ] finalizer implemented where appropriate
[ ] shutdown hook implemented
[ ] async work implemented for heavy tasks
[ ] event loop not blocked
[ ] callback lifetime controlled
[ ] thread safety reviewed
[ ] cancellation reviewed
[ ] stale completion handled
[ ] native memory tested
[ ] sanitizer tests run
[ ] crash diagnostics available
[ ] rollback path exists
```

---

# 233. Native API Contract Checklist

```text
[ ] input types
[ ] bounds
[ ] output types
[ ] ownership
[ ] lifetime
[ ] thread safety
[ ] reentrancy
[ ] blocking behavior
[ ] error codes
[ ] cancellation
[ ] timeout
[ ] cleanup
[ ] version compatibility
[ ] platform compatibility.
```

---

# 234. Security Boundary Matrix

| Strategy | Process boundary | ABI risk | Memory isolation | Typical use |
|---|---:|---:|---:|---|
| JS | no | low | managed | ordinary logic |
| Node-API addon | no | lower Node ABI risk | none | native libraries |
| FFI | no | high | none | controlled native calls |
| Worker | no | low/moderate | JS isolate | CPU parallelism |
| WASM | no | module-defined | linear memory model | portable compute |
| Child process | yes | native-only inside child | strong | risky/native workload |
| Container | yes + OS boundary | native-only inside | stronger | service isolation |

No strategy is universally best.

---

# 235. Specification / Runtime Source Discipline

Primary sources:

```text
Node.js Node-API documentation
Node.js C++ Addons documentation
Node.js Worker Threads documentation
Node.js FFI documentation
Node.js CLI / Permission Model documentation
Node.js Diagnostics documentation
V8 documentation where direct V8 integration is unavoidable
compiler/linker documentation
native library ABI documentation
```

Current Node v26 documentation states:

```text
Node-API: Stable
C++ addons: available through Node-API, NAN, or direct V8/libuv/Node APIs
FFI: Experimental
```

and explicitly distinguishes Node-API's ABI-oriented stability from direct V8 coupling. citeturn212928search0turn212928search2turn212928search3

Current Node v26 FFI was introduced in Node v26.1.0, remains experimental, and is explicitly documented as unsafe because bad pointer/signature/lifetime operations can crash or corrupt the process. citeturn212928search1turn212928search5

---

# 236. Current Platform Notes

As of September 2026:

```text
Node.js v26.8.2 documentation is the current
reference used in this chapter.

Node-API:
Stable.
Designed to insulate addons from underlying
JavaScript runtime changes and provide ABI stability.

C++ addons:
Supported through Node-API, NAN, or direct
V8/libuv/Node APIs.

node:ffi:
Experimental.
Added in Node v26.1.0.
Requires --experimental-ffi.
Under Permission Model, requires --allow-ffi.
Unsafe pointer/signature/lifetime operations
can crash or corrupt the process.

Native addons under Permission Model:
require --allow-addons.

Worker threads:
Stable.
Native addons have additional conditions for
multi-thread loading/use.

Node-API stability does not imply:
universal OS support
universal architecture support
native-library ABI stability
compiler ABI compatibility
security isolation.
```

These platform details are based on current Node v26.8.2 documentation and the Node v26.1.0 release announcement. citeturn212928search0turn212928search1turn212928search2turn212928search5turn212928search6

---

# 237. Performance Considerations

Native performance should be evaluated as:

```text
total workload
+
boundary cost
+
copy cost
+
synchronization
+
native execution
+
callback cost.
```

Avoid:

```text
native micro-optimization
```

when:

```text
boundary overhead
```

dominates.

Prefer:

```text
batching
typed arrays
buffers
zero-copy
async execution
```

when measurement supports them.

---

# 238. Memory Considerations

Track:

```text
V8 heap
external memory
native allocations
Buffer backing
ArrayBuffer backing
library memory
thread stacks
native queues
```

A native leak can remain invisible to:

```text
heap snapshots alone.
```

---

# 239. Security Considerations

Native code expands:

```text
trusted computing base
```

.

A vulnerability can affect:

```text
entire Node process
```

rather than:

```text
one JS object.
```

Treat native dependencies as:

```text
high-trust components.
```

For hostile code:

```text
prefer process/container isolation.
```

---

# 240. Reliability Considerations

Native integrations should survive:

```text
normal calls
bad input
concurrent calls
cancellation
timeout
worker shutdown
process shutdown
dependency failure
library unload
```

and:

```text
fail safely rather than corrupting memory.
```

---

# 241. Common Misconceptions

### Misconception 1

```text
“Node-API means the binary works on every platform.”
```

Reality:

```text
Node-API reduces Node-version ABI coupling.
OS, architecture, compiler, native dependencies, and packaging still matter.
```

### Misconception 2

```text
“Native is always faster.”
```

Reality:

```text
boundary/copy/synchronization costs can dominate.
```

### Misconception 3

```text
“Worker means native code is sandboxed.”
```

Reality:

```text
worker provides a separate JS execution environment,
not a universal native memory/process security boundary.
```

### Misconception 4

```text
“Permission Model makes addons safe.”
```

Reality:

```text
it controls Node-level capability access; it does not sandbox
arbitrary native code.
```

### Misconception 5

```text
“FFI is just a dynamic import.”
```

Reality:

```text
FFI exposes ABI and raw-memory correctness directly.
```

### Misconception 6

```text
“Finalizer is a destructor.”
```

Reality:

```text
GC-driven finalization is not deterministic resource management.
```

---

# 242. Common Mistakes

```text
[ ] storing raw JS handles without references
[ ] retaining stale pointers
[ ] freeing twice
[ ] using freed memory
[ ] reading beyond bounds
[ ] wrong integer width
[ ] wrong FFI signature
[ ] wrong calling convention
[ ] calling JS from arbitrary native thread
[ ] blocking the event loop
[ ] native global state without locks
[ ] lock inversion
[ ] closing libraries too early
[ ] callbacks after teardown
[ ] relying only on finalizers
[ ] no explicit close
[ ] no cancellation
[ ] no shutdown strategy
[ ] no sanitizer testing
[ ] no platform matrix
[ ] assuming Node-API solves all ABI concerns
[ ] treating native code as trusted because package is popular
```

---

# 243. Final Native Boundary Mental Model

```text
JAVASCRIPT
    ↓
Validation
    ↓
Node-API / FFI boundary
    ↓
Type conversion
    ↓
Ownership contract
    ↓
Native operation
    ↓
Thread / synchronization
    ↓
Native resource
    ↓
OS / external library
```

Every crossing asks:

```text
What is the type?
Who owns it?
How long is it valid?
Which thread may use it?
What happens if it fails?
How is it released?
What if JS goes away?
What if native goes away?
```

---

# 244. Native Lifetime Mental Model

```text
CREATE
  ↓
OWN
  ↓
USE
  ↓
CLOSE
  ↓
RELEASE
```

Alternate:

```text
CREATE
  ↓
OWN
  ↓
GC FINALIZER
  ↓
RELEASE
```

Never allow:

```text
CLOSED
→ USE
```

or:

```text
RELEASED
→ RELEASE.
```

---

# 245. Native Thread Mental Model

```text
JS request
    ↓
queue native work
    ↓
native worker
    ↓
native result
    ↓
safe JS completion
```

Not:

```text
JS request
    ↓
native thread
    ↓
direct arbitrary JS call.
```

---

# 246. ABI Mental Model

```text
SOURCE API
   ↓
ABI CONTRACT
   ↓
BINARY
   ↓
LOADER
   ↓
RUNTIME
```

Compatibility requires:

```text
correct ABI
+
correct architecture
+
correct OS
+
correct dependency
+
correct symbols
```

---

# 247. FFI Mental Model

```text
Library
 ↓
symbol
 ↓
signature
 ↓
call
 ↓
pointer/value conversion
 ↓
return
 ↓
lifetime
```

One incorrect assumption can produce:

```text
wrong result
crash
memory corruption.
```

---

# 248. Native Security Mental Model

```text
native dependency
      ↓
process privilege
      ↓
OS access
      ↓
filesystem/network
      ↓
secret exposure
      ↓
system impact
```

Therefore:

```text
trust
→ minimize
→ isolate
→ observe
→ update.
```

---

# 249. Dependency Graph

```text
Chapter 52
Workers / Concurrency
        ↓
Chapter 53
Streams
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Production Debugging
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
Chapter 127
Memory / Shared Memory
        ↓
Chapter 140
Node Networking
        ↓
Chapter 141
Diagnostics / Inspector
        ↓
Chapter 142
Permission Model / Runtime Isolation
        ↓
Chapter 143
Native Addons / Node-API / FFI / ABI
```

Cross-cutting:

```text
C/C++
V8
libuv
ABI
memory
threads
workers
security
build systems
package distribution
```

---

# 250. Concept Connections

## Depends On

```text
Node runtime
V8
libuv
workers
memory
streams
diagnostics
security
testing
build systems.
```

## Builds Toward

```text
native platform engineering
database bindings
image/video processing
crypto integration
hardware integrations
high-performance compute
runtime engineering
cross-platform package development.
```

## Related Concepts

```text
Node-API
NAN
node-addon-api
V8
FFI
N-API ABI
C ABI
C++ ABI
dynamic libraries
pointers
finalizers
async work
worker threads
sanitizers.
```

## Concepts Revisited

```text
Buffers
ArrayBuffers
Workers
Streams
AbortController
Diagnostics
Inspector
Permission Model
Performance
Security
Packaging
Testing
```

## Why This Chapter Matters

Most JavaScript developers live:

```text
inside the managed runtime.
```

A principal Node engineer must also understand:

```text
where the managed runtime ends.
```

The native boundary introduces:

```text
binary compatibility
pointer lifetime
manual ownership
native threads
ABI
toolchains
platform differences
process-level crashes.
```

That is why:

```text
Node-API
```

exists and why:

```text
FFI
```

must be treated carefully.

The principal skill is not:

```text
“write C++.”
```

It is:

```text
design a safe contract
between managed and unmanaged worlds.
```

---

# 251. Retrieval Record

```md
# Chapter 143 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Native Addons
-

## Node-API
-

## ABI
-

## API vs ABI
-

## V8 / NAN / node-addon-api
-

## napi_env / napi_value
-

## References / Handles
-

## Finalizers
-

## Cleanup Hooks
-

## Native Ownership
-

## Buffers / ArrayBuffers
-

## Zero-Copy
-

## Async Native Work
-

## Threads
-

## Callbacks
-

## Cancellation
-

## FFI
-

## Dynamic Libraries
-

## Pointers
-

## Calling Conventions
-

## Packaging
-

## Prebuilds
-

## Supply Chain
-

## Sanitizers
-

## Native Crashes
-

## Diagnostics
-

## Permission Model
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

# 252. Spaced Retrieval Schedule

### Day 0

Study:

```text
Node-API
ABI
native ownership
memory safety
```

### Day 1

Explain:

```text
Node-API vs V8 vs NAN vs FFI.
```

### Day 3

Build:

```text
native resource wrapper.
```

### Day 7

Build:

```text
async native operation.
```

### Day 14

Run:

```text
native memory/lifetime tests
```

with:

```text
sanitizers.
```

### Day 21

Build:

```text
cross-platform native package.
```

### Day 30

Design:

```text
production native integration architecture
```

without notes.

---

# 253. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
build and load a basic Node-API addon.
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly:

```text
confuse ABI with API
ignore ownership
ignore thread safety
treat FFI as safe
```

Mark:

```text
[+] Completed
```

when you can:

```text
build
test
package
debug
and operate
```

a native addon across:

```text
Node versions
workers
platforms
shutdown
failure.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
choose
design
secure
profile
debug
and defend
```

a managed/native boundary for:

```text
performance
compatibility
security
reliability
maintenance
```

under production constraints.

Reading alone does not mark mastery.

---

# 254. Final Principal Principle

> **Native code is a trust and lifetime boundary, not merely an optimization technique.**

The production sequence is:

```text
justify native requirement
→ choose boundary
→ prefer Node-API where appropriate
→ minimize data crossing
→ define ownership
→ define lifetime
→ define threading
→ define errors
→ define cancellation
→ define shutdown
→ define ABI
→ define packaging
→ test every platform
→ sanitize/fuzz
→ observe native health
→ prepare crash diagnostics
→ isolate when trust is low.
```

Remember:

```text
Node-API ≠ all-platform compatibility

ABI ≠ API

native ≠ automatically faster

worker ≠ native sandbox

finalizer ≠ deterministic destructor

pointer ≠ ownership

Buffer address ≠ forever-valid pointer

FFI ≠ safe dynamic import

Permission Model ≠ native sandbox

prebuilt binary ≠ trusted binary

successful load ≠ ABI correctness

no JS heap leak ≠ no process memory leak.
```

The principal question is:

```text
“What exact native capability do we need,
why must it cross the managed boundary,
what binary contract makes the integration safe,
who owns every byte/resource,
which thread may touch it,
what happens when JS or native state disappears,
how do we contain crashes and compromise,
and how will we prove the integration remains
correct across runtimes, platforms, and releases?”
```

That is native boundary engineering.