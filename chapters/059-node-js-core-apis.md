# Chapter 59 — Node.js Core APIs

> **Curriculum Position:** Part XI — Node.js  
> **Prerequisites:** Chapters 31–39, 41–48, 55–58  
> **Primary Focus:** Node.js core modules and platform APIs: `fs`, `path`, `url`, `os`, `events`, `buffer`, `stream`, `timers`, `util`, `assert`, `crypto`, `dns`, `net`, `http`, `https`, `tls`, `zlib`, `readline`, `process`, `console`, `perf_hooks`, `module`, `worker_threads`, `child_process`, permissions, and Web-compatible APIs  
> **Status:** `[ ] Not Started`  
> **Depth Target:** API surface → semantics → runtime/OS interaction → failure modes → security → production composition  
> **Source Discipline:** Node APIs are host/runtime facilities. Distinguish Node-specific behavior from ECMAScript and Web Platform behavior.

---

# Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Node Core APIs Are](#3-what-node-core-apis-are)
- [4. The Node Core Module Model](#4-the-node-core-module-model)
- [5. `node:fs`](#5-nodefs)
- [6. `node:path`](#6-nodepath)
- [7. `node:url`](#7-nodeurl)
- [8. `node:os`](#8-nodeos)
- [9. `node:process`](#9-nodeprocess)
- [10. `node:events`](#10-nodeevents)
- [11. `node:buffer`](#11-nodebuffer)
- [12. `node:string_decoder`](#12-nodestring_decoder)
- [13. `node:stream`](#13-nodestream)
- [14. `node:timers`](#14-nodetimers)
- [15. `node:console`](#15-nodeconsole)
- [16. `node:util`](#16-nodeutil)
- [17. `node:assert`](#17-nodeassert)
- [18. `node:crypto`](#18-nodecrypto)
- [19. `node:dns`](#19-nodedns)
- [20. `node:net`](#20-nodenet)
- [21. `node:http`](#21-nodehttp)
- [22. `node:https`](#22-nodehttps)
- [23. `node:tls`](#23-nodetls)
- [24. `node:zlib`](#24-nodezlib)
- [25. `node:readline`](#25-nodereadline)
- [26. `node:perf_hooks`](#26-nodeperf_hooks)
- [27. `node:module`](#27-nodemodule)
- [28. `node:worker_threads`](#28-nodeworker_threads)
- [29. `node:child_process`](#29-nodechild_process)
- [30. Permissions](#30-permissions)
- [31. Web-Compatible Node APIs](#31-web-compatible-node-apis)
- [32. Common API Composition Patterns](#32-common-api-composition-patterns)
- [33. Error Semantics](#33-error-semantics)
- [34. Resource Ownership](#34-resource-ownership)
- [35. Security](#35-security)
- [36. Performance](#36-performance)
- [37. Memory](#37-memory)
- [38. Debugging](#38-debugging)
- [39. Common Misconceptions](#39-common-misconceptions)
- [40. Common Mistakes](#40-common-mistakes)
- [41. Comparison With Related Concepts](#41-comparison-with-related-concepts)
- [42. Production Architecture](#42-production-architecture)
- [43. Implementation From Scratch](#43-implementation-from-scratch)
- [44. Debugging Exercises](#44-debugging-exercises)
- [45. Code Review Exercise](#45-code-review-exercise)
- [46. Interview Questions](#46-interview-questions)
- [47. Predict-the-Output Exercises](#47-predict-the-output-exercises)
- [48. Mastery Exercises](#48-mastery-exercises)
- [49. Key Takeaways](#49-key-takeaways)
- [50. Concept Connections](#50-concept-connections)
- [51. Completion Criteria](#51-completion-criteria)
- [52. Revision / Retrieval Record](#52-revision--retrieval-record)
- [53. Canonical References and Source Discipline](#53-canonical-references-and-source-discipline)
- [54. Completion Snapshot](#54-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what Node core APIs are.
2. Distinguish ECMAScript APIs from Node APIs.
3. Explain the `node:` built-in module convention.
4. Use Node filesystem APIs safely.
5. Choose between callback, Promise, and synchronous filesystem APIs.
6. Use file descriptors and `FileHandle`s correctly.
7. Explain paths and platform differences.
8. Safely resolve user-controlled paths.
9. Use the WHATWG URL API in Node.
10. Inspect OS and runtime information.
11. Use EventEmitter safely.
12. Work with Buffers and binary data.
13. Decode streamed text safely.
14. Use Node streams.
15. Implement and respect backpressure.
16. Use timers correctly.
17. Understand Console/Util/Assert roles.
18. Use cryptographic primitives responsibly.
19. Distinguish cryptographic hashes from password hashing and encryption.
20. Use DNS APIs and understand their implementation differences.
21. Build TCP servers/clients.
22. Build HTTP/HTTPS servers and clients.
23. Understand TLS at the Node API boundary.
24. Compress/decompress data with zlib.
25. Build CLI interfaces using readline.
26. Measure performance using `perf_hooks`.
27. Understand module/runtime introspection.
28. Use Worker Threads appropriately.
29. Use Child Processes safely.
30. Understand Node permission controls.
31. Compose core APIs into production services.
32. Design ownership and cleanup around sockets/files/streams/workers.
33. Handle errors without losing important context.
34. Identify security boundaries around filesystem, commands, networking, and native APIs.
35. Diagnose performance and memory problems using the right Node primitives.

### Mastery target

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

---

# 2. Prerequisites

Required:

```text
Chapter 58 — Node.js Architecture
Chapter 60 — Node Streams (preview)
Chapter 61 — Worker Threads / Child Processes
Chapter 62 — Process Lifecycle
Chapter 63 — Async Context / Diagnostics
```

Also required:

```text
Promises
async/await
objects
iterables
typed arrays
errors
memory
event loop
HTTP
security
```

---

# 3. What Node Core APIs Are

Node core APIs are modules shipped with Node itself.

Examples:

```js
import fs from "node:fs";
import path from "node:path";
import crypto from "node:crypto";
```

The Node v26 documentation index currently includes core areas such as File System, Path, URL, OS, Events, Buffer, Stream, Timers, Crypto, DNS, HTTP, HTTPS, TLS, Workers, Child Processes, Permissions, V8, and more. citeturn632316search0

---

# 4. The Node Core Module Model

A useful mental model:

```text
node:fs
  ↓
Node JavaScript API
  ↓
native/runtime implementation
  ↓
libuv / OS
```

Not every module follows exactly the same path.

Some APIs are predominantly:

```text
JavaScript
```

while others bridge into:

```text
C++
libuv
OS syscalls
OpenSSL
zlib
V8
```

---

## `node:` Prefix

Prefer:

```js
import fs from "node:fs";
```

rather than ambiguous bare imports in code where explicit built-in intent improves clarity.

---

# 5. `node:fs`

The `fs` module provides filesystem interaction.

Node's current documentation describes `node:fs` as interacting with the filesystem and exposes synchronous, callback, and Promise APIs. The Promise APIs use the underlying Node thread pool for filesystem operations. citeturn632316search1

---

## 5.1 Promise API

```js
import {
  readFile,
  writeFile
} from "node:fs/promises";

const data = await readFile(
  "config.json",
  "utf8"
);
```

---

## 5.2 Callback API

```js
import { readFile } from "node:fs";

readFile("config.json", "utf8", (err, data) => {
  if (err) {
    console.error(err);
    return;
  }

  console.log(data);
});
```

---

## 5.3 Synchronous API

```js
import { readFileSync } from "node:fs";

const data = readFileSync(
  "config.json",
  "utf8"
);
```

Use sync APIs carefully in servers.

---

## 5.4 File Handles

```js
import { open } from "node:fs/promises";

const file = await open(
  "data.bin",
  "r"
);

try {
  const buffer = Buffer.alloc(1024);
  await file.read(buffer, 0, buffer.length, 0);
} finally {
  await file.close();
}
```

Node explicitly recommends closing `FileHandle`s rather than relying on eventual automatic cleanup. citeturn632316search1

---

## 5.5 File Paths

Node `fs` operations can accept:

```text
string
Buffer
URL
```

for many APIs, including `file:` URLs. citeturn632316search1

---

## 5.6 Streams

```js
import { createReadStream } from "node:fs";

const input =
  createReadStream("large.log");
```

Use streams when the file is too large or incremental processing is useful.

---

## 5.7 Security

Never do:

```js
readFile(`/uploads/${userInput}`);
```

without a clear path policy.

Safer pattern:

```js
const root = path.resolve("/uploads");
const candidate =
  path.resolve(root, userInput);

if (!candidate.startsWith(root + path.sep)) {
  throw new Error("Invalid path");
}
```

Even this pattern must be reviewed for symlink/race issues when the threat model requires strong containment.

---

# 6. `node:path`

`path` provides platform-aware path manipulation.

```js
import path from "node:path";
```

---

## 6.1 Join

```js
path.join("a", "b", "c");
```

---

## 6.2 Resolve

```js
path.resolve("a", "b");
```

`resolve()` produces an absolute path based on the current working directory and its arguments.

---

## 6.3 Normalize

```js
path.normalize("../a/../b");
```

---

## 6.4 Basename / Dirname / Extname

```js
path.basename("/tmp/file.txt");
path.dirname("/tmp/file.txt");
path.extname("/tmp/file.txt");
```

---

## 6.5 Platform Difference

Use:

```js
path.win32
path.posix
```

when you specifically need platform-specific semantics.

---

## Security Warning

Path normalization is not equivalent to authorization.

A secure filesystem boundary may also need:

```text
canonicalization
symlink analysis
permission checks
openat-like safe patterns where available
race-condition handling
```

---

# 7. `node:url`

Node provides WHATWG URL APIs and legacy URL APIs.

Node's current documentation describes the WHATWG URL API as the newer standard-oriented interface and the legacy URL API as Node-specific. citeturn632316search5

---

## 7.1 WHATWG URL

```js
const url =
  new URL(
    "https://example.com/path?q=1"
  );

console.log(url.hostname);
console.log(url.pathname);
console.log(url.searchParams.get("q"));
```

---

## 7.2 URL Resolution

```js
new URL(
  "/users",
  "https://example.com"
);
```

---

## 7.3 URL vs Path

Do not confuse:

```text
URL
```

with:

```text
filesystem path
```

---

# 8. `node:os`

Provides operating-system information.

```js
import os from "node:os";

os.platform();
os.arch();
os.cpus();
os.totalmem();
os.freemem();
os.homedir();
os.tmpdir();
```

---

## Production Use

Useful for:

```text
diagnostics
capacity hints
logging
worker sizing
temporary paths
```

Do not use `os.cpus().length` as a simplistic universal workload sizing formula.

---

# 9. `node:process`

The `process` object provides process information/control.

```js
process.pid
process.argv
process.cwd()
process.env
process.exitCode
process.platform
```

Use:

```js
process.cwd()
```

for current working directory.

Use:

```js
import { fileURLToPath } from "node:url";
```

when module-relative filesystem paths are needed under ESM.

---

# 10. `node:events`

The EventEmitter model underpins many Node APIs.

```js
import { EventEmitter } from "node:events";

const emitter =
  new EventEmitter();

emitter.on(
  "ready",
  () => console.log("ready")
);

emitter.emit("ready");
```

---

## 10.1 `on`

```js
emitter.on("event", listener);
```

---

## 10.2 `once`

```js
emitter.once(
  "connection",
  handler
);
```

---

## 10.3 `off`

```js
emitter.off(
  "event",
  listener
);
```

---

## 10.4 Error Event

A crucial Node convention:

```js
emitter.emit(
  "error",
  new Error("failed")
);
```

An EventEmitter that emits `'error'` without an appropriate listener can cause process-level failure behavior.

---

# 11. `node:buffer`

`Buffer` represents a fixed-length byte sequence.

Node's current documentation describes Buffer as a subclass of `Uint8Array` and notes that many APIs accept `Uint8Array` where Buffers are supported. citeturn632316search6

---

## 11.1 Create

```js
import { Buffer } from "node:buffer";

const a =
  Buffer.from("hello");

const b =
  Buffer.alloc(16);
```

---

## 11.2 Never Use Unsafe Allocation Carelessly

Prefer:

```js
Buffer.alloc(size);
```

when zero-initialized memory is required.

---

## 11.3 Encoding

```js
Buffer.from(
  "hello",
  "utf8"
);
```

---

## 11.4 Binary Protocols

Buffers are useful for:

```text
TCP
files
crypto
compression
binary formats
```

---

# 12. `node:string_decoder`

Streaming UTF-8 can split a multibyte character across chunks.

A StringDecoder preserves incomplete characters across chunks.

```js
import {
  StringDecoder
} from "node:string_decoder";

const decoder =
  new StringDecoder("utf8");

for (const chunk of chunks) {
  process.stdout.write(
    decoder.write(chunk)
  );
}

process.stdout.write(
  decoder.end()
);
```

---

## Why This Matters

Naively doing:

```js
chunk.toString("utf8")
```

for every arbitrary byte chunk can mishandle characters split across chunk boundaries.

---

# 13. `node:stream`

Core stream abstractions:

```text
Readable
Writable
Duplex
Transform
```

---

## Readable

```js
import {
  Readable
} from "node:stream";

const readable =
  Readable.from([
    "a",
    "b",
    "c"
  ]);
```

---

## Writable

```js
import {
  Writable
} from "node:stream";
```

---

## Transform

```js
import {
  Transform
} from "node:stream";
```

Used for:

```text
input
→ transform
→ output
```

---

## Pipeline

Prefer:

```js
import {
  pipeline
} from "node:stream/promises";
```

Example:

```js
await pipeline(
  source,
  transform,
  destination
);
```

This provides stronger composition/error propagation than manually wiring many events.

---

# 14. `node:timers`

Node timers include:

```text
setTimeout
setInterval
setImmediate
timers/promises
```

The current Node timers documentation lists these APIs as stable. citeturn632316search0

---

## Promise Timer

```js
import {
  setTimeout
} from "node:timers/promises";

await setTimeout(1000);
```

---

## Abortable Timer

```js
const controller =
  new AbortController();

await setTimeout(
  10_000,
  undefined,
  {
    signal:
      controller.signal
  }
);
```

---

# 15. `node:console`

Node provides:

```js
console.log()
console.error()
console.warn()
console.time()
console.timeEnd()
```

---

## Production Rule

Do not confuse:

```text
console logging
```

with a complete structured logging/observability system.

For production systems consider:

```text
structured logger
log levels
correlation ID
redaction
aggregation
retention
```

---

# 16. `node:util`

Utility APIs include areas such as:

```text
promisify
callbackify
inspect
types
format
parseArgs
```

---

## `promisify`

```js
import {
  promisify
} from "node:util";
```

Useful when integrating callback-style APIs.

---

## Inspect

```js
import {
  inspect
} from "node:util";

console.log(
  inspect(value, {
    depth: 4
  })
);
```

---

# 17. `node:assert`

Use assertions for:

```text
invariants
tests
internal assumptions
```

Example:

```js
import assert from "node:assert/strict";

assert.strictEqual(
  result,
  expected
);
```

---

## Production Rule

Do not confuse:

```text
developer assertion
```

with:

```text
input validation
```

External input still requires explicit validation.

---

# 18. `node:crypto`

Node's crypto module provides cryptographic primitives and protocols.

Typical areas:

```text
hashes
HMAC
randomness
digital signatures
public-key cryptography
symmetric encryption
key derivation
Web Crypto
```

---

## Randomness

Use:

```js
import {
  randomBytes
} from "node:crypto";

const token =
  randomBytes(32);
```

for cryptographically strong random bytes.

---

## Hash

```js
import { createHash } from "node:crypto";

const digest =
  createHash("sha256")
    .update("hello")
    .digest("hex");
```

---

## Important

A cryptographic hash is not:

```text
password hashing
encryption
encoding
```

---

## Passwords

Use dedicated password hashing/KDF algorithms and appropriate parameters rather than:

```js
sha256(password)
```

as a password storage strategy.

---

# 19. `node:dns`

Node DNS APIs include:

```text
lookup
resolve*
reverse
Resolver
```

---

## Important Distinction

`dns.lookup()` is closely integrated with OS name-resolution behavior, while explicit `resolve*()` methods perform DNS protocol-oriented resolution.

This distinction can affect:

```text
latency
IPv4/IPv6 behavior
hosts-file semantics
thread-pool usage
```

---

# 20. `node:net`

Provides TCP networking primitives.

```js
import net from "node:net";

const server =
  net.createServer(socket => {
    socket.write("hello");
    socket.end();
  });

server.listen(3000);
```

---

## TCP Concepts

```text
socket
connect
data
drain
end
close
error
```

---

## Backpressure

```js
const canContinue =
  socket.write(data);

if (!canContinue) {
  socket.once(
    "drain",
    continueWriting
  );
}
```

---

# 21. `node:http`

Build HTTP servers:

```js
import http from "node:http";

const server =
  http.createServer(
    (req, res) => {
      res.writeHead(200, {
        "content-type":
          "application/json"
      });

      res.end(
        JSON.stringify({
          ok: true
        })
      );
    }
  );

server.listen(3000);
```

---

## Request

Important properties:

```js
req.method
req.url
req.headers
req.socket
```

---

## Response

Important operations:

```js
res.statusCode
res.setHeader()
res.write()
res.end()
```

---

## Security

Never trust:

```text
method
URL
headers
body
```

without validation.

---

# 22. `node:https`

HTTPS builds on HTTP plus TLS.

```js
import https from "node:https";
```

Use it for:

```text
TLS server
secure HTTP
```

---

# 23. `node:tls`

Provides TLS-secured streams.

```text
TLS socket
certificate
cipher
secure context
SNI
```

---

## Production Concerns

```text
certificate lifecycle
trusted CA
hostname verification
protocol versions
cipher suites
client authentication
```

Do not casually disable:

```text
certificate verification
```

in production.

---

# 24. `node:zlib`

Compression APIs:

```text
gzip
deflate
brotli
zstd where supported by your target Node/runtime
```

---

## Streaming

```js
import {
  createGzip
} from "node:zlib";

const gzip = createGzip();
```

Combine:

```text
Readable
→ compression
→ Writable
```

---

## Security

Compression can create resource-exhaustion risk.

Be careful with untrusted compressed data and decompression ratios.

---

# 25. `node:readline`

Useful for:

```text
CLI
REPL-style tools
line processing
interactive commands
```

Example:

```js
import readline from "node:readline";

const rl =
  readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });

rl.question(
  "Name? ",
  answer => {
    console.log(answer);
    rl.close();
  }
);
```

---

# 26. `node:perf_hooks`

Performance APIs include:

```text
performance.now()
PerformanceObserver
marks
measures
event-loop delay tools
```

Example:

```js
import {
  performance
} from "node:perf_hooks";

const start =
  performance.now();

doWork();

console.log(
  performance.now() - start
);
```

---

## Rule

Measure before optimizing.

---

# 27. `node:module`

Node's module APIs expose runtime/module-related capabilities.

Areas include:

```text
module customization
resolution
load hooks
registration
source transformations
```

Use advanced module customization only when the architecture requires it.

---

## Risk

Module hooks can affect:

```text
startup
debugging
tool compatibility
security
```

---

# 28. `node:worker_threads`

Worker Threads allow parallel JavaScript.

```js
import {
  Worker
} from "node:worker_threads";

const worker =
  new Worker(
    new URL(
      "./worker.js",
      import.meta.url
    )
  );
```

Node's current documentation describes Worker Threads as stable and primarily useful for CPU-intensive JavaScript, while noting that they provide little advantage for I/O-intensive workloads. citeturn632316search3

---

## Worker Decision

Use Worker Threads for:

```text
CPU-heavy JS
parallel algorithms
image transforms
parsing
computation
```

not merely because an operation is asynchronous.

---

# 29. `node:child_process`

Node supports:

```text
spawn
exec
execFile
fork
```

---

## Prefer `spawn`/`execFile`

When arguments are structured:

```js
spawn(
  "git",
  ["status"],
  {
    shell: false
  }
);
```

---

## Security

Avoid:

```js
exec(
  `tool ${userInput}`
);
```

unless command construction is completely constrained.

Shell metacharacters can turn input into command execution.

---

# 30. Permissions

Node has a permission model in current releases that can restrict selected capabilities.

Capabilities can include areas such as:

```text
filesystem
child processes
workers
FFI
```

The exact flags/scopes are Node-version-specific and must be checked against the deployed runtime. The current Node API documentation includes a Permissions section. citeturn632316search0

---

## Principle

Permissions are:

```text
least-authority defense
```

not a substitute for:

```text
input validation
authorization
process isolation
container security
OS controls
```

---

# 31. Web-Compatible Node APIs

Modern Node includes several Web-compatible APIs such as:

```text
URL
URLSearchParams
AbortController
Fetch
Web Streams
Web Crypto
ReadableStream
```

---

## Important

Similar names do not guarantee identical implementation details.

When writing isomorphic code, define the supported API subset explicitly.

---

# 32. Common API Composition Patterns

## Pattern 1 — File → Transform → Network

```text
fs.createReadStream
        ↓
transform
        ↓
http/https request
```

---

## Pattern 2 — Network → Transform → File

```text
HTTP response
      ↓
decompression
      ↓
transform
      ↓
fs.createWriteStream
```

---

## Pattern 3 — HTTP → Worker

```text
request
 ↓
parse
 ↓
Worker
 ↓
result
 ↓
response
```

---

## Pattern 4 — CLI → Process

```text
readline
 ↓
validation
 ↓
spawn
 ↓
stdout/stderr
```

---

## Pattern 5 — Timer → Background Work

```text
timer
 ↓
bounded queue
 ↓
worker/service
```

---

# 33. Error Semantics

Node core APIs expose different error styles:

```text
throw
callback(err)
Promise rejection
EventEmitter 'error'
error event on stream
process-level error
```

---

## Example

Filesystem Promise:

```js
try {
  await readFile(file);
} catch (error) {
  ...
}
```

Stream:

```js
stream.on("error", handler);
```

Process:

```text
uncaughtException
unhandledRejection
```

---

## Rule

Know the error channel of every API you use.

---

# 34. Resource Ownership

For every resource ask:

```text
Who created it?
Who owns it?
Who closes it?
What happens on failure?
What happens on timeout?
What happens on shutdown?
```

Resources include:

```text
FileHandle
socket
server
stream
worker
child process
timer
TLS context
```

---

## Ownership Pattern

```text
acquire
 ↓
use
 ↓
finally cleanup
```

---

# 35. Security

Core APIs expose powerful capabilities.

## Filesystem

Risk:

```text
path traversal
symlink abuse
TOCTOU
permission bypass
secret disclosure
```

## Child Processes

Risk:

```text
command injection
PATH hijacking
unexpected environment
privilege inheritance
```

## Network

Risk:

```text
SSRF
DNS rebinding
credential leakage
unsafe redirects
resource exhaustion
```

## Crypto

Risk:

```text
weak randomness
bad key handling
wrong primitive
algorithm misuse
```

## Buffers

Risk:

```text
uninitialized memory
large allocations
binary parsing bugs
```

---

# 36. Performance

Core API performance depends on:

```text
event-loop work
I/O scheduling
thread pool
memory copies
serialization
buffering
network latency
OS
```

---

## Avoid Accidental Copies

```js
Buffer.from(buffer)
```

can create copying depending on input/type.

Understand:

```text
view
copy
transfer
share
```

before optimizing.

---

## Streams

Prefer streams for:

```text
large data
long-running transfer
bounded memory
backpressure
```

---

# 37. Memory

Watch:

```text
heapUsed
external
arrayBuffers
RSS
```

Core APIs can allocate outside the ordinary JS heap.

---

## Common Sources

```text
Buffer
stream queues
compression
TLS
HTTP body buffering
worker memory
child processes
native libraries
```

---

# 38. Debugging

## Filesystem

Check:

```text
path
cwd
permissions
symlink
file existence
descriptor lifetime
```

---

## Networking

Check:

```text
host
port
DNS
TCP
TLS
HTTP
timeouts
response body
```

---

## Processes

Check:

```text
argv
env
cwd
PATH
stdio
exit code
signal
stderr
```

---

## Streams

Check:

```text
readable
writable
backpressure
error
close
finish
drain
```

---

# 39. Common Misconceptions

## Misconception 1 — “Node core APIs are JavaScript language features.”

No.

## Misconception 2 — “Promise API means no thread pool.”

Not necessarily.

## Misconception 3 — “Async fs means no blocking.”

The JavaScript callback does not perform the file operation synchronously, but other synchronous fs APIs still block.

## Misconception 4 — “Buffer is just an array.”

It is a byte-oriented typed-array subclass with Node-specific behavior.

## Misconception 5 — “Path.normalize solves path traversal.”

No.

## Misconception 6 — “spawn is always safe.”

Only if command/arguments/environment are controlled appropriately.

## Misconception 7 — “HTTPS automatically verifies everything for my application.”

TLS is not authorization.

## Misconception 8 — “Worker Thread is a process.”

No.

## Misconception 9 — “assert validates users.”

No.

## Misconception 10 — “console.log is observability.”

No.

## Misconception 11 — “more threads always means more throughput.”

No.

## Misconception 12 — “FileHandle auto-close makes manual closing unnecessary.”

Node documentation explicitly advises explicit closure. citeturn632316search1

---

# 40. Common Mistakes

1. Using synchronous filesystem APIs in hot server paths.
2. Reading huge files entirely into memory unnecessarily.
3. Ignoring stream backpressure.
4. Forgetting `'error'` listeners on event-driven resources.
5. Leaving FileHandles open.
6. Constructing shell commands with user input.
7. Using relative paths without knowing `cwd`.
8. Confusing URL and filesystem-path semantics.
9. Disabling TLS verification to “fix” development problems and accidentally shipping it.
10. Logging secrets from `process.env`.
11. Assuming Buffer creation is always zero-copy.
12. Creating too many Worker Threads.
13. Creating too many child processes.
14. Swallowing error codes.
15. Ignoring exit codes.
16. Assuming timers are exact.
17. Treating Node's Web-compatible APIs as identical to every browser.
18. Using crypto primitives without understanding their intended purpose.
19. Measuring only CPU and ignoring event-loop delay.
20. Using `os.cpus()` as a blanket concurrency formula.

---

# 41. Comparison With Related Concepts

| API | Main abstraction | Common use | Main risk |
|---|---|---|---|
| `fs` | filesystem | files/directories | path/descriptor misuse |
| `path` | path manipulation | file paths | traversal assumptions |
| `url` | URL parsing | network/resource URLs | origin confusion |
| `events` | event emitter | event-driven APIs | listener/error handling |
| `buffer` | bytes | binary data | memory/parsing bugs |
| `stream` | incremental dataflow | large data | backpressure |
| `crypto` | cryptography | integrity/auth/randomness | primitive misuse |
| `dns` | name resolution | host lookup | blocking/semantics |
| `net` | TCP | sockets | protocol/security |
| `http` | HTTP | web servers/clients | input/headers |
| `https` | HTTP + TLS | secure HTTP | TLS config |
| `tls` | secure transport | TLS | certificate/config |
| `zlib` | compression | payload/storage | CPU/ratio exhaustion |
| `worker_threads` | parallel JS | CPU work | synchronization |
| `child_process` | process | isolation/commands | command injection |
| `perf_hooks` | measurement | diagnostics | bad measurement |
| `readline` | line/CLI | terminals | untrusted commands |

---

# 42. Production Architecture

A production Node application often composes APIs like this:

```text
                HTTP/HTTPS
                    │
                    ↓
              Router / Handler
                    │
          ┌─────────┼──────────┐
          ↓         ↓          ↓
         URL      Headers     Body
          │         │          │
          └─────────┼──────────┘
                    ↓
                Validation
                    ↓
             Domain Service
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
      fs          net/http      crypto
       │            │            │
       └────────────┼────────────┘
                    ↓
               stream/transform
                    ↓
               response
```

---

## API Boundary Rules

For every core API:

```text
validate inputs
set explicit ownership
handle errors
bound resources
instrument critical work
clean up
```

---

## Example Secure File Serving

```js
import path from "node:path";
import {
  stat
} from "node:fs/promises";
import {
  createReadStream
} from "node:fs";

const root =
  path.resolve("/srv/public");

async function sendFile(
  res,
  userPath
) {
  const candidate =
    path.resolve(root, userPath);

  if (
    !candidate.startsWith(
      root + path.sep
    )
  ) {
    res.statusCode = 400;
    res.end("Invalid path");
    return;
  }

  const info = await stat(candidate);

  if (!info.isFile()) {
    res.statusCode = 404;
    res.end("Not found");
    return;
  }

  createReadStream(candidate)
    .pipe(res);
}
```

This remains a simplified example; strong hostile-filesystem containment may require stronger symlink/race defenses.

---

# 43. Implementation From Scratch

Build a **Toy Node Core Runtime Layer**.

## Stage 1 — Path Utility

Implement:

```text
join
normalize
resolve
relative
```

for POSIX-style paths.

---

## Stage 2 — Virtual Filesystem

Implement:

```text
open
read
write
close
stat
```

with explicit resource ownership.

---

## Stage 3 — Buffer

Create:

```text
byte array
offset
length
encoding
```

and compare copies vs views.

---

## Stage 4 — EventEmitter

Implement:

```js
on()
once()
off()
emit()
```

including synchronous listener execution.

---

## Stage 5 — Readable Stream

Implement:

```text
producer
queue
consumer
highWaterMark
backpressure
```

---

## Stage 6 — HTTP Server Model

Implement:

```text
TCP connection
HTTP parser
request
response
headers
body
```

---

## Stage 7 — TLS Model

Simulate:

```text
handshake
certificate validation
secure channel
```

---

## Stage 8 — Child Process Model

Implement:

```text
spawn
stdin
stdout
stderr
exit
signal
```

---

## Stage 9 — Worker Model

Implement:

```text
worker
message port
separate state
termination
```

---

## Stage 10 — Error Model

Support:

```text
throw
reject
error event
exit
```

---

## Stage 11 — Resource Manager

Track:

```text
open files
sockets
workers
timers
children
```

---

## Stage 12 — Production Core Layer

Build a small server with:

```text
HTTP
validation
filesystem
streams
crypto
worker
graceful shutdown
observability
```

---

# 44. Debugging Exercises

## Exercise 1 — File Path

```js
readFile(
  "./data/" + userInput
);
```

Identify path traversal risks.

---

## Exercise 2 — `cwd`

Your application works from:

```text
/home/app
```

but fails from:

```text
/
```

Explain relative-path resolution.

---

## Exercise 3 — FileHandle

Open a file repeatedly without closing.

Measure:

```text
descriptor usage
memory
process stability
```

---

## Exercise 4 — Buffer

Compare:

```js
Buffer.alloc(10)
```

and:

```js
Buffer.allocUnsafe(10)
```

Explain why initialization semantics matter.

---

## Exercise 5 — Stream

Build:

```text
fast source
→ slow destination
```

and demonstrate backpressure.

---

## Exercise 6 — HTTP

Send a request with:

```text
huge header
huge body
unexpected method
```

and design limits.

---

## Exercise 7 — DNS

Compare:

```text
dns.lookup
dns.resolve4
```

and inspect differences.

---

## Exercise 8 — Crypto

Generate:

```text
random bytes
hash
HMAC
```

and explain why these are different primitives.

---

## Exercise 9 — TLS

Run a local HTTPS server with a development certificate and identify what must change for production.

---

## Exercise 10 — Compression

Compress a large payload.

Measure:

```text
CPU
memory
size
latency
```

---

## Exercise 11 — Worker

Move CPU-heavy transformation to a Worker.

Compare event-loop delay.

---

## Exercise 12 — Child Process

Execute a fixed command safely.

Then deliberately introduce shell interpolation in a local lab and explain the security difference.

---

# 45. Code Review Exercise

Review:

```js
import http from "node:http";
import fs from "node:fs";
import { exec } from "node:child_process";

const server = http.createServer((req, res) => {
  const file =
    "./uploads/" + req.url;

  const data =
    fs.readFileSync(file);

  const command =
    `convert ${req.url}`;

  exec(command);

  res.end(data);
});

server.listen(3000);
```

Developer says:

> “It's simple because Node core APIs are low-level and fast.”

This implementation is unsafe.

## Problems

1. Synchronous filesystem access blocks the event loop.
2. `req.url` contains attacker-controlled data.
3. Filesystem path traversal may occur.
4. `req.url` is being interpreted as a command argument.
5. `exec()` invokes shell processing.
6. Command injection is possible.
7. File access is not authorization-aware.
8. No file-size limit exists.
9. The entire file is buffered into memory.
10. No backpressure strategy exists.
11. No HTTP status handling exists.
12. No content type is set.
13. No error listeners/handlers exist.
14. No request timeout.
15. No observability.
16. The process could become vulnerable to CPU/memory exhaustion.
17. The URL path is treated as both filesystem path and command data.
18. No resource ownership strategy exists.

A stronger architecture:

```text
request
 ↓
URL parsing
 ↓
validation
 ↓
authorization
 ↓
safe path mapping
 ↓
streaming file access
 ↓
bounded processing
 ↓
safe process/worker boundary where required
 ↓
response
```

---

# 46. Interview Questions

## Filesystem

1. `fs` callback vs Promise vs sync?
2. Why do fs Promise APIs still involve runtime worker infrastructure?
3. What is a FileHandle?
4. Why explicitly close FileHandles?
5. How do you prevent path traversal?
6. Why is `path.normalize()` insufficient?

## Path / URL

7. `path.resolve` vs `path.join`?
8. URL vs filesystem path?
9. What does `file:` URL support mean in Node?
10. Why can Windows path semantics differ?

## Buffers

11. What is Buffer?
12. Is Buffer an array?
13. Buffer vs Uint8Array?
14. `alloc` vs `allocUnsafe`?
15. Copy vs view?

## Streams

16. What is a Readable?
17. What is a Transform?
18. What is backpressure?
19. Why use pipeline?
20. What happens when the consumer is slower?

## Events

21. Is EventEmitter asynchronous?
22. Why is the `'error'` event special?
23. `once` vs `on`?

## Crypto

24. Hash vs HMAC?
25. Hash vs encryption?
26. Why isn't SHA-256 suitable as a password hash?
27. What is cryptographically secure randomness?

## Networking

28. TCP vs HTTP?
29. HTTP vs HTTPS?
30. What is TLS?
31. `dns.lookup` vs `dns.resolve`?
32. How does HTTP streaming work?

## Processes / Workers

33. Worker vs Child Process?
34. When should you use `spawn`?
35. Why is `exec` dangerous with user input?
36. How does Worker memory differ from a process?
37. When do you need process isolation?

## Performance

38. Which APIs can block?
39. How do streams reduce memory?
40. How do you measure event-loop delay?

## Security

41. How do you secure file serving?
42. How do you prevent command injection?
43. How do you prevent SSRF?
44. How do you handle untrusted compressed data?
45. How do Node permissions help?

## Principal-Level

46. Design a secure file-processing service.
47. Design a Node image-processing pipeline.
48. Design a secure command-execution worker.
49. Design a high-throughput streaming proxy.
50. Design a production crypto/token service.
51. Design a multi-core Node application using only core APIs.
52. Defend process vs Worker isolation.
53. Explain where each core API crosses into OS/native functionality.

---

# 47. Predict-the-Output Exercises

## Exercise 1 — EventEmitter

```js
import { EventEmitter } from "node:events";

const e = new EventEmitter();

e.on("x", () =>
  console.log("listener")
);

console.log("before");

e.emit("x");

console.log("after");
```

Predict:

```text
before
listener
after
```

---

## Exercise 2 — Path

```js
import path from "node:path";

console.log(
  path.join(
    "/a",
    "b",
    "..",
    "c"
  )
);
```

Predict normalized path behavior for your platform.

---

## Exercise 3 — Buffer

```js
const a =
  Buffer.from("hello");

console.log(
  a instanceof Uint8Array
);
```

Predict.

---

## Exercise 4 — Stream Backpressure

A writable returns:

```js
false
```

from:

```js
write()
```

Should the producer immediately continue writing unlimited additional data?

Explain.

---

## Exercise 5 — Timer

```js
import {
  setTimeout
} from "node:timers/promises";

const value =
  await setTimeout(
    10,
    "done"
  );

console.log(value);
```

Predict.

---

## Exercise 6 — Process

```js
console.log(
  process.cwd()
);
```

Is the value necessarily the directory containing the JavaScript file?

Explain.

---

## Exercise 7 — URL

```js
const url =
  new URL(
    "users",
    "https://example.com/api/"
  );

console.log(url.href);
```

Predict using WHATWG URL resolution semantics.

---

## Exercise 8 — Crypto

```js
const digest =
  createHash("sha256")
    .update("hello")
    .digest("hex");
```

Predict the behavior category, but do not memorize the digest unless you can derive/verify it.

---

# 48. Mastery Exercises

## Level 1 — Files

Build:

```text
safe file reader
safe file writer
directory walker
```

with:

```text
async APIs
error handling
cleanup
```

---

## Level 2 — File Server

Build:

```text
HTTP
→ secure path resolution
→ streaming file response
```

with:

```text
size limits
content types
error handling
```

---

## Level 3 — Stream Pipeline

Build:

```text
large file
→ transform
→ gzip
→ output
```

using `pipeline()`.

---

## Level 4 — TCP

Build:

```text
length-prefixed TCP protocol
```

with:

```text
framing
backpressure
timeouts
validation
```

---

## Level 5 — HTTP

Build:

```text
raw Node HTTP server
```

with:

```text
routing
headers
body limits
timeouts
streaming
```

---

## Level 6 — HTTPS

Build a local HTTPS server and understand:

```text
certificate
private key
TLS configuration
hostname verification
```

---

## Level 7 — Crypto

Build:

```text
secure token generator
HMAC verifier
signature verifier
```

and document which cryptographic primitive solves which problem.

---

## Level 8 — Compression

Build:

```text
HTTP request
→ compression
→ stream
→ response
```

and compare:

```text
CPU
latency
bandwidth
```

---

## Level 9 — Worker

Build a CPU-heavy image/data transformation service using Worker Threads.

---

## Level 10 — Child Process

Build a safe command runner:

```text
allowlisted command
structured args
timeout
resource limits
stdout/stderr capture
exit-code handling
```

---

## Level 11 — Core-Only API Server

Build a production-style service without Express/Fastify:

```text
http
router
validation
streams
crypto
fs
workers
observability
shutdown
```

---

## Level 12 — Principal Node Core Platform

Build:

```text
HTTP gateway
+
streaming
+
filesystem
+
worker pool
+
child-process isolation
+
TLS
+
compression
+
crypto
+
metrics
+
graceful shutdown
```

Defend:

```text
API selection
ownership
backpressure
concurrency
security
failure domains
memory
performance
```

---

# 49. Key Takeaways

1. Node core APIs are runtime capabilities, not ECMAScript language features.
2. The `node:` namespace explicitly identifies built-in modules.
3. `fs` has sync, callback, and Promise forms.
4. Promise filesystem operations use Node runtime thread infrastructure according to the operation/platform.
5. FileHandles must be explicitly closed.
6. Filesystem paths may be strings, Buffers, or file URLs for many APIs.
7. Path manipulation does not itself establish security authorization.
8. `path.resolve()` and `path.join()` solve different problems.
9. WHATWG URL APIs should be preferred for standard URL semantics.
10. URL and filesystem path are different abstractions.
11. `process` describes/control the current process.
12. EventEmitter listeners are synchronously invoked by `emit()`.
13. The EventEmitter `'error'` event deserves special handling.
14. Buffer is a byte-oriented `Uint8Array` subclass.
15. Byte streams can split multibyte text characters.
16. StringDecoder helps preserve streaming text boundaries.
17. Streams provide incremental dataflow.
18. Backpressure is essential for bounded memory.
19. `pipeline()` provides strong stream composition and error propagation.
20. Timers provide scheduling eligibility, not exact execution times.
21. Console is not a complete production observability system.
22. `util` contains interoperability and inspection helpers.
23. Assert checks developer assumptions, not external input validation.
24. Crypto primitives must be selected for their intended security purpose.
25. Cryptographic randomness differs from ordinary pseudo-randomness.
26. DNS APIs can have distinct runtime/OS semantics.
27. `net` exposes TCP-level sockets.
28. `http` exposes HTTP semantics.
29. `https` combines HTTP with TLS.
30. `tls` exposes secure transport primitives.
31. Compression can create CPU/memory security risks.
32. `readline` is useful for CLI applications.
33. `perf_hooks` helps measure runtime behavior.
34. Module APIs can alter resolution/loading and should be used intentionally.
35. Worker Threads provide parallel JavaScript for suitable CPU workloads.
36. Child Processes provide process-level isolation and external-command capability.
37. Shell command construction with attacker input is a major security hazard.
38. Node permissions provide capability reduction but are not a complete sandbox.
39. Web-compatible APIs in Node still belong to the Node runtime.
40. Core APIs become powerful when composed into explicit resource/dataflow pipelines.
41. Every resource needs an owner and cleanup strategy.
42. Every external input needs validation before it reaches filesystem/process/network/native capabilities.
43. Performance must be measured at event-loop, I/O, memory, and system boundaries.
44. Principal-level Node API mastery means knowing not only what an API does, but what runtime/OS capability it exposes and what failure/security model comes with it.

---

# 50. Concept Connections

## Depends On

```text
Chapter 31 — Async Fundamentals
        ↓
Chapter 34 — Node Event Loop
        ↓
Chapter 35 — Promises
        ↓
Chapter 37 — Cancellation
        ↓
Chapter 38 — Async Iteration / Streaming
        ↓
Chapter 45 — Memory
        ↓
Chapter 47 — Engine Architecture
        ↓
Chapter 48 — V8
        ↓
Chapter 55 — Fetch / HTTP
        ↓
Chapter 56 — Browser Security
        ↓
Chapter 57 — JS Security
        ↓
Chapter 58 — Node Architecture
        ↓
Chapter 59 — Node Core APIs
```

## Builds Toward

```text
Chapter 60 — Node Streams
Chapter 61 — Worker Threads / Child Processes / Cluster
Chapter 62 — Process Lifecycle
Chapter 63 — Async Context / Diagnostics
Chapter 64 — ES Modules
Chapter 65 — CommonJS / Interoperability
Chapter 66 — package.json / Resolution
Chapter 67 — Dependency Management
Chapter 68 — Transpilation
Chapter 69 — Bundlers
Chapter 70 — Source Maps
Chapter 78 — Production Architecture
Chapter 79 — API Design
Chapter 80 — Library Authoring
Chapter 81 — Database Integration
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

```text
filesystem
TCP
HTTP
HTTPS
TLS
DNS
streams
buffers
events
timers
processes
workers
IPC
crypto
compression
CLI
performance
permissions
```

## Concepts Revisited

### Chapter 43 — Internal Methods

Core API objects still ultimately participate in JavaScript object semantics.

### Chapter 45 — Memory

Buffers, streams, native allocations, workers, and child processes extend memory analysis beyond the V8 heap.

### Chapter 47 — Engine Architecture

Node core bridges JavaScript execution to native/runtime facilities.

### Chapter 55 — Fetch

Node's modern Fetch APIs coexist with the lower-level HTTP/HTTPS APIs.

### Chapter 57 — Security

Filesystem, processes, network, crypto, and native capabilities greatly expand the security surface.

### Chapter 58 — Node Architecture

This chapter turns the architecture model into concrete API choices.

---

## Why This Chapter Matters Later

The next Node chapters go deeper into:

```text
streams
workers/processes
lifecycle
async context
modules
tooling
```

You should now be able to look at:

```js
import fs from "node:fs";
import { pipeline } from "node:stream/promises";
import { Worker } from "node:worker_threads";
```

and immediately ask:

```text
What capability does this expose?
Where does the work run?
Who owns the resource?
How does it fail?
How does it affect memory?
Can it block?
Can it be abused?
How do we observe it?
How do we shut it down?
```

---

# 51. Completion Criteria

## Core Model

- [ ] Define Node core APIs.
- [ ] Distinguish ECMAScript from Node APIs.
- [ ] Explain `node:` imports.
- [ ] Identify JS/native/OS boundaries.

## Files

- [ ] Use fs Promise APIs.
- [ ] Use callback APIs.
- [ ] Understand sync APIs.
- [ ] Use FileHandle.
- [ ] Close resources explicitly.
- [ ] Use file streams.
- [ ] Secure paths.
- [ ] Handle platform differences.

## Paths / URLs

- [ ] `join`
- [ ] `resolve`
- [ ] `normalize`
- [ ] relative paths
- [ ] WHATWG URL
- [ ] URL vs path
- [ ] file URLs

## Events / Timers

- [ ] EventEmitter
- [ ] on/once/off
- [ ] error event
- [ ] timers
- [ ] timer Promises
- [ ] cancellation

## Bytes / Streams

- [ ] Buffer
- [ ] Uint8Array relation
- [ ] alloc vs allocUnsafe
- [ ] StringDecoder
- [ ] Readable
- [ ] Writable
- [ ] Transform
- [ ] pipeline
- [ ] backpressure

## Crypto

- [ ] secure randomness
- [ ] hash
- [ ] HMAC
- [ ] signatures
- [ ] encryption
- [ ] password hashing distinction

## Network

- [ ] DNS
- [ ] TCP
- [ ] HTTP
- [ ] HTTPS
- [ ] TLS
- [ ] timeouts
- [ ] input validation

## Compression / CLI

- [ ] zlib
- [ ] streaming compression
- [ ] decompression limits
- [ ] readline

## Performance

- [ ] perf_hooks
- [ ] event-loop measurement
- [ ] memory measurement
- [ ] copy/view analysis

## Workers / Processes

- [ ] Worker Threads
- [ ] Child Processes
- [ ] safe spawn
- [ ] command injection defense
- [ ] process isolation

## Security

- [ ] filesystem security
- [ ] process security
- [ ] network security
- [ ] crypto security
- [ ] compression security
- [ ] Node permissions

## Architecture

- [ ] resource ownership
- [ ] cleanup
- [ ] error channels
- [ ] observability
- [ ] graceful integration
- [ ] performance
- [ ] memory

## Principal Judgment

- [ ] Select the correct core API.
- [ ] Explain runtime implications.
- [ ] Defend security model.
- [ ] Defend resource strategy.
- [ ] Defend process/thread choice.
- [ ] Defend streaming architecture.
- [ ] Defend operational complexity.

---

# 52. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What are Node core APIs?
2. Why use `node:`?
3. fs Promise vs callback vs sync?
4. What is FileHandle?
5. Why close FileHandles?
6. How do you prevent path traversal?
7. `path.join` vs `path.resolve`?
8. URL vs filesystem path?
9. EventEmitter sync or async?
10. Why is `'error'` special?
11. What is Buffer?
12. Buffer vs Uint8Array?
13. `alloc` vs `allocUnsafe`?
14. Why use StringDecoder?
15. What is backpressure?
16. Why use pipeline?
17. Timers exact or approximate?
18. What is crypto.randomness for?
19. Hash vs HMAC?
20. Hash vs encryption?
21. How does DNS differ across APIs?
22. TCP vs HTTP?
23. HTTP vs HTTPS?
24. What is TLS?
25. Why can compression be dangerous?
26. What does perf_hooks provide?
27. Worker vs Child Process?
28. Why is exec dangerous?
29. What do Node permissions control?
30. How do you design resource ownership?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| `node:` modules | [ ] | [ ] | [ ] | [ ] |
| fs | [ ] | [ ] | [ ] | [ ] |
| FileHandle | [ ] | [ ] | [ ] | [ ] |
| path | [ ] | [ ] | [ ] | [ ] |
| URL | [ ] | [ ] | [ ] | [ ] |
| os | [ ] | [ ] | [ ] | [ ] |
| process | [ ] | [ ] | [ ] | [ ] |
| events | [ ] | [ ] | [ ] | [ ] |
| Buffer | [ ] | [ ] | [ ] | [ ] |
| StringDecoder | [ ] | [ ] | [ ] | [ ] |
| streams | [ ] | [ ] | [ ] | [ ] |
| backpressure | [ ] | [ ] | [ ] | [ ] |
| timers | [ ] | [ ] | [ ] | [ ] |
| console | [ ] | [ ] | [ ] | [ ] |
| util | [ ] | [ ] | [ ] | [ ] |
| assert | [ ] | [ ] | [ ] | [ ] |
| crypto | [ ] | [ ] | [ ] | [ ] |
| DNS | [ ] | [ ] | [ ] | [ ] |
| net | [ ] | [ ] | [ ] | [ ] |
| HTTP | [ ] | [ ] | [ ] | [ ] |
| HTTPS | [ ] | [ ] | [ ] | [ ] |
| TLS | [ ] | [ ] | [ ] | [ ] |
| zlib | [ ] | [ ] | [ ] | [ ] |
| readline | [ ] | [ ] | [ ] | [ ] |
| perf_hooks | [ ] | [ ] | [ ] | [ ] |
| module | [ ] | [ ] | [ ] | [ ] |
| Worker Threads | [ ] | [ ] | [ ] | [ ] |
| Child Process | [ ] | [ ] | [ ] | [ ] |
| permissions | [ ] | [ ] | [ ] | [ ] |
| API composition | [ ] | [ ] | [ ] | [ ] |
| ownership | [ ] | [ ] | [ ] | [ ] |
| security | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw:

```text
Node core
 ↓
JS API
 ↓
runtime/native
 ↓
libuv/OS
```

### Day 2

Explain:

```text
fs
streams
Buffer
HTTP
Worker
Child Process
```

in terms of runtime/OS behavior.

### Day 7

Build a secure streaming file server using only Node core.

### Day 14

Build a TCP/HTTP pipeline with compression and backpressure.

### Day 30

Build a core-only production API server with worker/process isolation and observability.

---

# 53. Canonical References and Source Discipline

## Official Node.js Documentation

https://nodejs.org/api/

The current Node v26.8.2 API index includes the core modules covered in this chapter, including Buffer, Crypto, DNS, Events, File System, HTTP, HTTPS, Net, OS, Path, Process, Stream, Timers, URL, Worker Threads, Zlib, Permissions, and diagnostics. citeturn632316search0

---

## File System

https://nodejs.org/api/fs.html

Use for:

```text
fs
fs/promises
FileHandle
file descriptors
read/write streams
path handling
```

The current documentation states that filesystem APIs have synchronous, callback, and Promise forms and that Promise filesystem APIs use Node's underlying thread pool. citeturn632316search1

---

## URL

https://nodejs.org/api/url.html

Use for:

```text
WHATWG URL
legacy URL
resolution
file URLs
```

Node currently distinguishes the newer WHATWG URL interface from the legacy URL API. citeturn632316search5

---

## Buffer

https://nodejs.org/api/buffer.html

Use for:

```text
Buffer
Uint8Array
binary data
allocation
encoding
```

Node's current documentation defines Buffer as a fixed-length byte sequence and a subclass of `Uint8Array`. citeturn632316search6

---

## Worker Threads

https://nodejs.org/api/worker_threads.html

Use for:

```text
parallel JavaScript
transfer
shared memory
worker lifecycle
```

The current documentation states that Worker Threads are intended primarily for CPU-intensive JavaScript and have less value for I/O-intensive work. citeturn632316search3

---

## Child Processes

https://nodejs.org/api/child_process.html

Use for:

```text
spawn
exec
execFile
fork
stdio
IPC
```

The current Node documentation states that `spawn()` creates a child process asynchronously without blocking the Node event loop. citeturn632316search4

---

## Source Classification

Classify claims as:

```text
[ECMAScript]
[Node Core]
[libuv]
[OS]
[OpenSSL]
[Node Version-Specific]
[Browser-Compatible API]
[Measured]
[Historical]
```

---

## Critical Discipline

Do not generalize:

```text
Node API
=
OS syscall
```

or:

```text
Promise API
=
thread pool
```

or:

```text
Worker
=
process
```

or:

```text
Buffer
=
ordinary JS array
```

---

## Version Discipline

The current official documentation used for this chapter is Node.js **v26.8.2**. Version-sensitive behavior should be verified against the exact runtime deployed by the project. citeturn632316search0

---

# 54. Completion Snapshot

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

- [ ] Node core
- [ ] node: namespace
- [ ] fs
- [ ] FileHandle
- [ ] path
- [ ] URL
- [ ] OS
- [ ] process
- [ ] EventEmitter
- [ ] Buffer
- [ ] StringDecoder
- [ ] streams
- [ ] backpressure
- [ ] timers
- [ ] console
- [ ] util
- [ ] assert
- [ ] crypto
- [ ] DNS
- [ ] TCP/net
- [ ] HTTP
- [ ] HTTPS
- [ ] TLS
- [ ] zlib
- [ ] readline
- [ ] perf_hooks
- [ ] module
- [ ] Worker Threads
- [ ] Child Processes
- [ ] permissions
- [ ] Web-compatible APIs
- [ ] API composition
- [ ] error channels
- [ ] resource ownership
- [ ] security
- [ ] performance
- [ ] memory

### I can predict

- [ ] EventEmitter ordering
- [ ] path behavior
- [ ] URL resolution
- [ ] Buffer type behavior
- [ ] backpressure
- [ ] timer scheduling
- [ ] process cwd
- [ ] Worker boundary
- [ ] Child Process boundary
- [ ] error channels

### I can implement

- [ ] file service
- [ ] secure path resolver
- [ ] stream pipeline
- [ ] TCP server
- [ ] HTTP server
- [ ] HTTPS server
- [ ] crypto utilities
- [ ] compression pipeline
- [ ] Worker
- [ ] Child Process
- [ ] CLI
- [ ] performance instrumentation

### I can debug

- [ ] filesystem failures
- [ ] descriptor leaks
- [ ] path traversal
- [ ] stream errors
- [ ] backpressure
- [ ] DNS issues
- [ ] TCP issues
- [ ] HTTP issues
- [ ] TLS issues
- [ ] compression issues
- [ ] worker failures
- [ ] child-process failures
- [ ] event-loop performance
- [ ] memory issues

### I can defend

- [ ] Promise vs callback vs sync fs
- [ ] stream vs full buffering
- [ ] Worker vs process
- [ ] HTTP vs net
- [ ] URL vs path
- [ ] crypto primitive selection
- [ ] permission strategy
- [ ] security boundaries
- [ ] resource ownership
- [ ] observability
- [ ] performance strategy

---

## Final Principal-Level Test

Take this application:

```text
Node HTTP API
   ↓
user input
   ↓
filesystem
   ↓
compression
   ↓
crypto
   ↓
Worker
   ↓
Child Process
   ↓
network
```

For every boundary, explain:

```text
What API is used?
Where does execution happen?
What resource is created?
Who owns the resource?
How can it fail?
How can it block?
How much memory can it consume?
What can an attacker control?
What security control applies?
How do we observe it?
How do we cancel it?
How do we clean it up?
What happens during shutdown?
```

Your mastery is complete only when you can use Node core APIs as **systems primitives** rather than isolated utility functions.

The central Chapter 59 lesson is:

> **Node core APIs are the bridge between JavaScript and real operating-system capabilities. Mastery means understanding the abstraction, the underlying runtime behavior, the resource lifecycle, the failure channel, the security boundary, and the performance cost of every primitive you compose.**