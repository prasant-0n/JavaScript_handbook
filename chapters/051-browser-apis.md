# Chapter 51 — Browser Web APIs

> **Curriculum Position:** Part IX — Browser  
> **Prerequisites:** Chapters 41–50  
> **Primary Focus:** Browser APIs as host/platform capabilities; `window`, `navigator`, timers, scheduling, storage, observers, media/device capabilities, permissions, visibility, performance, lifecycle, and production architecture  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Web Platform semantics → host capability model → API lifecycles → scheduling → security/permissions → production engineering  
> **Important Scope Rule:** Browser Web APIs are not part of core ECMAScript. They are supplied by the browser/Web Platform and are specified across standards such as WHATWG, W3C, and related specifications.

---

## Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Is a Browser Web API?](#3-what-is-a-browser-web-api)
- [4. Why Do Browser APIs Exist?](#4-why-do-browser-apis-exist)
- [5. Mental Model](#5-mental-model)
- [6. Core Rules](#6-core-rules)
- [7. Browser Global Environment](#7-browser-global-environment)
- [8. `window`, `globalThis`, `document`, and `navigator`](#8-window-globalthis-document-and-navigator)
- [9. Web IDL and Platform Objects](#9-web-idl-and-platform-objects)
- [10. Timers](#10-timers)
- [11. `requestAnimationFrame`](#11-requestanimationframe)
- [12. Scheduling and Cooperative Work](#12-scheduling-and-cooperative-work)
- [13. Page Visibility and Lifecycle Signals](#13-page-visibility-and-lifecycle-signals)
- [14. Storage APIs](#14-storage-apis)
- [15. Cookies and Client Storage Boundaries](#15-cookies-and-client-storage-boundaries)
- [16. IndexedDB](#16-indexeddb)
- [17. BroadcastChannel](#17-broadcastchannel)
- [18. Web Locks](#18-web-locks)
- [19. Observer APIs](#19-observer-apis)
- [20. IntersectionObserver](#20-intersectionobserver)
- [21. ResizeObserver](#21-resizeobserver)
- [22. Performance APIs](#22-performance-apis)
- [23. Network/Connection-Adjacent APIs](#23-networkconnection-adjacent-apis)
- [24. Media and Device APIs](#24-media-and-device-apis)
- [25. Permissions](#25-permissions)
- [26. Clipboard, Fullscreen, Wake Lock, and User Activation](#26-clipboard-fullscreen-wake-lock-and-user-activation)
- [27. Notifications](#27-notifications)
- [28. Geolocation](#28-geolocation)
- [29. Web Workers and Shared Capabilities](#29-web-workers-and-shared-capabilities)
- [30. API Availability and Feature Detection](#30-api-availability-and-feature-detection)
- [31. Security Contexts and Powerful Features](#31-security-contexts-and-powerful-features)
- [32. Browser Compatibility](#32-browser-compatibility)
- [33. Edge Cases](#33-edge-cases)
- [34. Common Misconceptions](#34-common-misconceptions)
- [35. Common Mistakes](#35-common-mistakes)
- [36. Comparison With Related Concepts](#36-comparison-with-related-concepts)
- [37. Performance Considerations](#37-performance-considerations)
- [38. Memory Considerations](#38-memory-considerations)
- [39. Security Considerations](#39-security-considerations)
- [40. Production Usage](#40-production-usage)
- [41. Implementation From Scratch](#41-implementation-from-scratch)
- [42. Debugging Exercises](#42-debugging-exercises)
- [43. Code Review Exercise](#43-code-review-exercise)
- [44. Interview Questions](#44-interview-questions)
- [45. Predict-the-Output Exercises](#45-predict-the-output-exercises)
- [46. Mastery Exercises](#46-mastery-exercises)
- [47. Key Takeaways](#47-key-takeaways)
- [48. Concept Connections](#48-concept-connections)
- [49. Completion Criteria](#49-completion-criteria)
- [50. Revision / Retrieval Record](#50-revision--retrieval-record)
- [51. Canonical References and Source Discipline](#51-canonical-references-and-source-discipline)
- [52. Completion Snapshot](#52-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define a browser Web API precisely.
2. Explain why browser APIs are separate from ECMAScript.
3. Explain the difference between language features, Web Platform APIs, browser implementation details, and third-party libraries.
4. Explain the browser global environment and the roles of `globalThis`, `window`, `document`, and `navigator`.
5. Explain Web IDL-backed platform interfaces.
6. Explain why browser APIs can look like JavaScript objects while being backed by browser-native systems.
7. Explain timer scheduling conceptually without treating `setTimeout()` as an exact sleep mechanism.
8. Explain `requestAnimationFrame()` and its relationship to rendering.
9. Explain cooperative scheduling and why long synchronous work blocks browser progress.
10. Explain page visibility and lifecycle-related signals.
11. Compare `localStorage`, `sessionStorage`, IndexedDB, cookies, and in-memory state.
12. Explain IndexedDB's transactional/object-store model conceptually.
13. Explain cross-context messaging with `BroadcastChannel`.
14. Explain same-origin coordination using Web Locks.
15. Explain observer APIs and why they exist.
16. Explain IntersectionObserver and ResizeObserver without misrepresenting them as ordinary event loops.
17. Explain the Performance API and user timing.
18. Explain permission-controlled/powerful browser capabilities.
19. Explain secure-context requirements at a conceptual level.
20. Explain feature detection and progressive enhancement.
21. Distinguish API availability from permission to use an API.
22. Distinguish API permission from successful runtime operation.
23. Explain browser compatibility and version sensitivity.
24. Explain how Workers interact with browser APIs.
25. Reason about browser resource lifecycles.
26. Diagnose stale resources, unbounded observers, timers, listener leaks, storage misuse, and permission failures.
27. Design a browser-capability abstraction layer.
28. Defend browser API choices using correctness, accessibility, security, performance, memory, resilience, and maintainability.

### Mastery target

Progress through:

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

Reading the chapter alone does not establish mastery.

---

# 2. Prerequisites

## Chapter 41 — Specification Architecture

Understand:

```text
normative semantics
implementation freedom
host/environment boundaries
```

## Chapter 42 — Abstract Operations

Understand the semantic machinery behind JavaScript-visible operations.

## Chapter 43 — Ordinary Object Methods

Understand that:

```js
navigator.geolocation
```

is accessed through object/property semantics while its underlying behavior is platform-defined.

## Chapter 44 — Realms and Agents

Required for:

```text
Window
Worker
Realm
Agent
```

## Chapter 45 — Memory / GC

Required for:

```text
timers
listeners
observers
storage references
application lifecycle
```

## Chapter 47 — Engine Architecture

Required to separate:

```text
JavaScript engine
```

from:

```text
browser host/platform
```

## Chapter 49 — DOM Architecture

Required for:

```text
document
elements
layout
rendering
```

## Chapter 50 — Browser Events

Required for:

```text
EventTarget
listeners
dispatch
propagation
```

---

# 3. What Is a Browser Web API?

A browser Web API is an interface provided by the Web Platform that allows web applications to interact with browser capabilities, document state, devices, networking, storage, rendering, scheduling, or other platform facilities.

Examples include:

```js
document.querySelector(...)
fetch(...)
localStorage.getItem(...)
navigator.geolocation.getCurrentPosition(...)
requestAnimationFrame(...)
new ResizeObserver(...)
new BroadcastChannel(...)
navigator.locks.request(...)
```

The category is broad.

MDN describes browser Web APIs as capabilities built into browsers on top of the JavaScript language. citeturn422860search0turn422860search2

---

## 3.1 API vs Language

ECMAScript includes:

```js
Array
Promise
Map
Set
Symbol
```

A browser adds:

```js
document
window
fetch
navigator
localStorage
Notification
IntersectionObserver
ResizeObserver
```

The browser APIs are not the same standard layer as the JavaScript language.

---

## 3.2 API vs Library

A browser API is supplied by the platform.

A library is application/developer code you add.

Example:

```text
fetch
→ browser API

axios
→ library
```

---

## 3.3 API vs Framework

A framework provides architectural conventions and abstractions.

A browser API provides platform capability.

Example:

```text
DOM
→ Web Platform

React
→ library/framework ecosystem
```

---

# 4. Why Do Browser APIs Exist?

JavaScript alone cannot provide:

```text
document rendering
camera
microphone
persistent browser storage
clipboard
screen
network stack
geolocation
display scheduling
```

without platform integration.

Browser APIs bridge:

```text
JavaScript
   ↕
browser capabilities
```

---

# 5. Mental Model

Use:

```text
               Web Application
                      |
                  JavaScript
                      |
              Browser Web APIs
                      |
      ┌───────────────┼────────────────┐
      ↓               ↓                ↓
   Document        Storage          Device/OS
      ↓               ↓                ↓
   Rendering      Persistence       Permissions
      ↓               ↓                ↓
                     Browser
                       ↓
                  Operating System
```

---

## 5.1 Four Layers

### Layer 1 — ECMAScript

```text
language
```

### Layer 2 — Web Platform

```text
DOM
Fetch
Storage
Workers
Permissions
Observers
Media
```

### Layer 3 — Browser implementation

```text
Blink
Gecko
WebKit
network stack
graphics stack
storage engine
```

### Layer 4 — OS/hardware

```text
camera
GPU
disk
network
display
```

A production engineer must know which layer owns a behavior.

---

# 6. Core Rules

## Rule 1 — Browser APIs are not core JavaScript

`fetch`, `document`, and `navigator` are not ECMAScript language features.

## Rule 2 — API presence does not guarantee availability

```js
"geolocation" in navigator
```

does not prove:

```text
permission granted
hardware available
request will succeed
```

## Rule 3 — Permission is not the same as feature detection

These are separate questions:

```text
Does API exist?
Can this context use it?
Does user permit it?
Does runtime operation succeed?
```

## Rule 4 — Secure context can matter

Powerful features often require HTTPS or another secure context.

## Rule 5 — APIs may be context-specific

An API can be available in:

```text
Window
Worker
both
neither
```

depending on the standard.

## Rule 6 — Async API does not mean “background thread”

An API can be asynchronous while its callback ultimately executes on the JavaScript event loop.

## Rule 7 — Timer delay is not exact

```js
setTimeout(fn, 1000);
```

means roughly:

```text
do not run earlier than allowed by timer semantics
then wait until the relevant event-loop conditions permit execution
```

not:

```text
CPU sleeps exactly 1000 ms
```

## Rule 8 — `requestAnimationFrame()` is rendering-oriented

It requests callback execution before a browser repaint in a rendering cycle. citeturn422860search8

## Rule 9 — Observers reduce polling

IntersectionObserver and ResizeObserver provide browser-managed observation models intended to avoid repeated application-side polling. citeturn422860search3turn422860search5

## Rule 10 — Browser APIs have lifecycles

Timers, observers, channels, media streams, locks, workers, and event listeners should have explicit ownership.

## Rule 11 — Cross-context coordination is not a shared-memory free-for-all

Use the platform mechanism appropriate to the problem:

```text
postMessage
BroadcastChannel
SharedArrayBuffer/Atomics
Web Locks
storage events
```

## Rule 12 — Feature detection should test capability, not browser name

Prefer:

```js
if ("IntersectionObserver" in globalThis) {
  // ...
}
```

over:

```js
if (isChrome) {
  // ...
}
```

---

# 7. Browser Global Environment

A browser page usually exposes a global object through:

```js
globalThis
```

and, in Window contexts:

```js
window
```

---

## 7.1 `globalThis`

Standard JavaScript access to the global object:

```js
globalThis
```

works across multiple environments where supported.

---

## 7.2 `window`

In a normal Window context:

```js
window === globalThis
```

is typically true.

But this model does not apply to Workers in the same way.

---

## 7.3 `document`

```js
document
```

represents the document associated with the current window context.

---

## 7.4 `navigator`

```js
navigator
```

exposes browser/environment-related APIs and capability information.

Examples include:

```js
navigator.language
navigator.onLine
navigator.geolocation
navigator.clipboard
navigator.permissions
```

Availability depends on browser/context/security/permission.

---

# 8. `window`, `globalThis`, `document`, and `navigator`

## 8.1 `globalThis`

Language-level standard global reference.

---

## 8.2 `window`

Window-specific host object.

Includes or exposes many APIs such as:

```text
document
location
history
requestAnimationFrame
setTimeout
```

---

## 8.3 `document`

DOM document.

---

## 8.4 `navigator`

Browser/user-agent capabilities and metadata.

---

## 8.5 Worker Contrast

Workers have:

```js
self
globalThis
```

but do not expose a normal page `window`/`document`.

This reinforces:

```text
browser API availability depends on execution context
```

---

# 9. Web IDL and Platform Objects

Many Web APIs are specified using **Web IDL**.

A conceptual interface:

```webidl
interface Example {
  readonly attribute DOMString name;
  undefined start();
};
```

becomes JavaScript-visible behavior such as:

```js
example.name;
example.start();
```

Web IDL describes interface shape and binding behavior; it does not implement the underlying feature.

---

## 9.1 Why Web IDL Matters

It explains why Web APIs often expose:

```text
interfaces
attributes
methods
inheritance
exceptions
conversions
```

that feel like JavaScript objects.

---

## 9.2 Platform Object ≠ Plain Object

A browser may expose:

```js
navigator.geolocation
```

as a JavaScript-visible object while the actual capability is backed by native browser subsystems.

---

## 9.3 Host Operations

A method can conceptually transition:

```text
JavaScript
 ↓
Web IDL/platform binding
 ↓
browser implementation
 ↓
OS/hardware/network
```

---

# 10. Timers

The classic APIs:

```js
setTimeout(fn, delay);
setInterval(fn, delay);
clearTimeout(id);
clearInterval(id);
```

are host APIs.

---

## 10.1 `setTimeout`

```js
setTimeout(() => {
  console.log("later");
}, 1000);
```

does not execute immediately.

The callback becomes eligible according to timer/event-loop semantics.

---

## 10.2 Delay Is Not Deadline

This is incorrect:

> “The callback runs exactly after 1000 ms.”

The delay is a minimum scheduling constraint subject to the browser event loop and other rules.

---

## 10.3 Long Task

```js
setTimeout(() => console.log("timer"), 0);

for (;;) {}
```

The timer cannot interrupt the current synchronous JavaScript execution.

---

## 10.4 Timer Nesting and Clamping

Browsers can apply minimum-delay rules to certain nested timer patterns.

Do not rely on:

```text
0 ms = immediate
```

---

## 10.5 Timer Cancellation

```js
const id = setTimeout(work, 1000);

clearTimeout(id);
```

Cancellation should be part of component lifecycle.

---

## 10.6 Intervals

```js
const id = setInterval(work, 1000);
```

Intervals can overlap logically when `work` takes longer than the interval cadence.

For important jobs, recursive scheduling can provide clearer back-pressure:

```js
async function loop() {
  while (!stopped) {
    await work();
    await delay(1000);
  }
}
```

The appropriate model depends on requirements.

---

# 11. `requestAnimationFrame`

```js
requestAnimationFrame(callback);
```

requests a callback before the next repaint.

MDN describes it as a browser request to perform animation work before the next repaint and notes that callbacks generally follow the display refresh rate; calls are often paused in background tabs/hidden frames. citeturn422860search8

---

## 11.1 One-Shot Nature

```js
requestAnimationFrame(update);
```

runs once.

For continuous animation:

```js
function frame(time) {
  update(time);
  requestAnimationFrame(frame);
}

requestAnimationFrame(frame);
```

---

## 11.2 Timestamp

Use the callback's timestamp or another current-time source to calculate movement.

Do not assume:

```text
60 Hz
```

on every device. High-refresh displays exist. citeturn422860search8

---

## 11.3 Why rAF Helps

For visual updates:

```text
input
→ schedule
→ rAF
→ DOM/style write
→ browser render
```

is often a better model than arbitrary timer intervals.

---

# 12. Scheduling and Cooperative Work

Browser JavaScript is cooperative.

A long synchronous function:

```js
while (true) {
  heavyWork();
}
```

blocks:

```text
other JS
input handling
rendering progress
microtask checkpoints
```

until it yields.

---

## 12.1 Chunking

Instead of:

```js
for (let i = 0; i < 1e9; i++) {
  work(i);
}
```

break work into chunks:

```js
function processChunk(start) {
  const end = Math.min(start + 1000, data.length);

  for (let i = start; i < end; i++) {
    work(data[i]);
  }

  if (end < data.length) {
    setTimeout(() => processChunk(end), 0);
  }
}
```

This provides opportunities for browser progress.

---

## 12.2 Prioritized Task Scheduling

The Web Platform also has newer scheduling capabilities such as the Prioritized Task Scheduling API.

Treat such APIs as compatibility-sensitive and use feature detection.

---

## 12.3 Workers

For CPU-heavy work, a Worker can move JavaScript execution to another agent.

That is a different architecture from simply yielding the main thread.

---

# 13. Page Visibility and Lifecycle Signals

Useful APIs/events include:

```js
document.visibilityState
document.addEventListener("visibilitychange", ...)
```

---

## 13.1 Visibility

A page can move between states such as:

```text
visible
hidden
```

Applications can use this to reduce work.

---

## 13.2 Practical Example

```js
document.addEventListener("visibilitychange", () => {
  if (document.hidden) {
    pausePolling();
  } else {
    resumePolling();
  }
});
```

---

## 13.3 Visibility ≠ Process Termination

A hidden page may still run some work.

Do not assume:

```text
hidden = destroyed
```

---

## 13.4 Page Lifecycle

Browsers can freeze, discard, or otherwise manage background pages under resource pressure.

Production applications must tolerate lifecycle changes.

---

# 14. Storage APIs

Browser storage is not one thing.

Common options:

```text
cookies
localStorage
sessionStorage
IndexedDB
Cache Storage
in-memory state
```

---

## 14.1 localStorage

```js
localStorage.setItem("theme", "dark");
```

Simple key/value storage.

Use for small, simple client-side state.

---

## 14.2 sessionStorage

```js
sessionStorage.setItem("draft", "hello");
```

Associated with a browsing context/session model rather than general durable cross-session application storage.

---

## 14.3 Synchronous API Warning

`localStorage` and `sessionStorage` operations are synchronous JavaScript APIs.

Large or frequent operations should not become a hot-path storage strategy.

---

## 14.4 IndexedDB

Designed for structured client-side storage with asynchronous APIs and transactions.

Good candidates include:

```text
large structured data
offline application state
indexed records
```

---

## 14.5 Cache Storage

The Cache API supports storing `Request`/`Response` pairs and is commonly used with service-worker architectures.

It is distinct from IndexedDB.

---

# 15. Cookies and Client Storage Boundaries

Cookies are sent with applicable HTTP requests according to cookie rules.

They are not equivalent to:

```js
localStorage
```

---

## 15.1 Server Interaction

Conceptually:

```text
Cookie
↕
HTTP request/response
```

while:

```text
localStorage
↕
client JavaScript
```

---

## 15.2 Security Attributes

Cookies can use attributes such as:

```text
Secure
HttpOnly
SameSite
```

These belong to web security design and should not be treated as storage API syntax alone.

---

## 15.3 Storage Selection

Ask:

```text
Does server need this automatically?
Does client JS need direct access?
How much data?
What durability?
What origin/site boundary?
What security properties?
```

---

# 16. IndexedDB

IndexedDB provides a transactional, object-store-oriented browser database.

Conceptual structure:

```text
Database
 ├── object store: users
 ├── object store: orders
 └── object store: settings
```

---

## 16.1 Open

```js
const request = indexedDB.open("app", 1);
```

---

## 16.2 Schema Upgrade

Schema changes happen through versioned upgrade transactions.

---

## 16.3 Object Stores

Think:

```text
table-like logical container
```

but do not reduce IndexedDB to SQL.

---

## 16.4 Indexes

Indexes can accelerate structured lookups.

---

## 16.5 Transactions

Operations occur within transaction semantics.

---

## 16.6 Async Model

IndexedDB uses asynchronous APIs.

Do not assume that:

```text
request issued
→ operation done synchronously
```

---

# 17. BroadcastChannel

`BroadcastChannel` provides same-origin cross-context messaging.

```js
const channel = new BroadcastChannel("app-events");

channel.onmessage = event => {
  console.log(event.data);
};

channel.postMessage({
  type: "logout"
});
```

Useful for communicating among:

```text
tabs
windows
workers
```

where supported by the context.

---

## 17.1 Message Semantics

Messages are structured-cloned rather than sharing arbitrary live object identity.

This connects to Chapter 28.

---

## 17.2 Lifecycle

Close when no longer required:

```js
channel.close();
```

---

# 18. Web Locks

The Web Locks API allows cooperative coordination among scripts in the same origin.

```js
await navigator.locks.request("sync", async lock => {
  await synchronizeData();
});
```

MDN describes Web Locks as allowing tabs/workers from the same origin to coordinate exclusive/shared access to resources. It is available in secure contexts and workers in supporting browsers. citeturn422860search1

---

## 18.1 Why It Exists

Suppose multiple tabs attempt:

```text
refresh token
sync database
migrate cache
write shared resource
```

at the same time.

A named lock can coordinate that work.

---

## 18.2 Origin Scope

Locks are scoped to an origin.

Different origins do not share the same lock namespace. citeturn422860search1

---

## 18.3 Async Ownership

A lock is held while the callback executes and is released according to API semantics when the callback finishes.

---

## 18.4 Not a Shared Memory Primitive

Web Locks coordinate ownership.

They do not expose shared mutable memory.

---

# 19. Observer APIs

Observers are an important Web Platform design pattern:

```text
register interest
→ browser tracks state
→ deliver records/notifications
```

Examples:

```text
MutationObserver
IntersectionObserver
ResizeObserver
PerformanceObserver
```

---

## 19.1 Why Observers Exist

Polling:

```js
setInterval(check, 16);
```

can be wasteful.

The browser already knows when many relevant states change.

Observers let the platform manage detection.

---

## 19.2 Observer ≠ Event

Observers often have:

```text
entries/records
delivery rules
batching
observation lifecycle
```

different from DOM event dispatch.

---

# 20. IntersectionObserver

`IntersectionObserver` asynchronously observes intersection between a target and a root/viewport. MDN describes it as a browser-managed way to detect threshold crossings without constant application-side intersection polling. citeturn422860search3turn422860search4

Example:

```js
const observer = new IntersectionObserver(entries => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      loadMore();
    }
  }
});

observer.observe(document.querySelector("#sentinel"));
```

---

## 20.1 Thresholds

```js
new IntersectionObserver(callback, {
  threshold: [0, 0.5, 1]
});
```

---

## 20.2 Root

Default:

```text
viewport
```

or specify an ancestor:

```js
{
  root: container
}
```

---

## 20.3 Root Margin

```js
{
  rootMargin: "200px"
}
```

can cause pre-triggering before visible intersection.

---

## 20.4 Not Exact Pixel Intersection

IntersectionObserver is intended for threshold-based visibility/intersection use cases rather than arbitrary per-pixel overlap tracking. citeturn422860search4

---

## 20.5 Lifecycle

```js
observer.disconnect();
```

when the observation scope ends.

---

# 21. ResizeObserver

`ResizeObserver` reports element size changes.

Example:

```js
const observer = new ResizeObserver(entries => {
  for (const entry of entries) {
    console.log(entry.contentRect.width);
  }
});

observer.observe(panel);
```

MDN describes it as a mechanism for responding to changes in an element's content or border box without relying on window resize polling. citeturn422860search5turn422860search9

---

## 21.1 Why It Exists

Window resize is not the same as:

```text
element size changed
```

A container can change size due to:

- parent layout,
- content changes,
- fonts,
- grid/flex changes,
- responsive component interactions.

---

## 21.2 Resize Loops

A resize observer callback that changes the observed size can create a feedback cycle.

Browsers have loop-protection behavior, but application code must still avoid unintended resize feedback. MDN documents the possibility of deferred notifications/errors in resize loops. citeturn422860search9

---

# 22. Performance APIs

The Performance API provides browser-side measurement facilities.

Examples:

```js
performance.now();
performance.mark("start");
performance.measure("work", "start", "end");
```

---

## 22.1 High-Resolution Time

```js
performance.now()
```

provides a monotonic time source suitable for measuring elapsed durations.

It is different from wall-clock time.

---

## 22.2 User Timing

```js
performance.mark("fetch-start");

// work

performance.mark("fetch-end");

performance.measure(
  "fetch-duration",
  "fetch-start",
  "fetch-end"
);
```

---

## 22.3 PerformanceObserver

Observe performance entries:

```js
const observer = new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    console.log(entry.entryType, entry.name);
  }
});

observer.observe({
  entryTypes: ["measure"]
});
```

---

## 22.4 Why This Matters

Production performance analysis should create evidence:

```text
mark
→ measure
→ profile
→ compare
```

rather than rely on intuition.

---

# 23. Network/Connection-Adjacent APIs

The browser exposes APIs around networking and connectivity, but these vary significantly in capability and compatibility.

Examples include:

```text
navigator.onLine
Network Information API
Beacon
Fetch
WebSocket
EventSource
```

Detailed HTTP/network semantics belong to Chapter 55.

---

## 23.1 `navigator.onLine`

```js
navigator.onLine
```

is useful as a signal about online/offline state.

Do not treat it as:

```text
"server is reachable"
```

---

## 23.2 Beacon

```js
navigator.sendBeacon(url, data);
```

is designed for sending small amounts of data in situations such as page lifecycle transitions.

The actual networking semantics are browser-controlled.

---

## 23.3 Fetch

`fetch()` is a major browser API and gets its own chapter.

---

# 24. Media and Device APIs

Browser APIs can expose capabilities such as:

```text
camera
microphone
screen capture
audio/video
gamepad
orientation
sensors
```

These are powerful capabilities and commonly involve:

```text
secure context
permission
user activation
device availability
privacy rules
```

---

## 24.1 Media Capture

Conceptual:

```js
const stream = await navigator.mediaDevices.getUserMedia({
  video: true,
  audio: true
});
```

This request can fail for many reasons:

```text
permission denied
device missing
policy restriction
context restriction
```

---

## 24.2 Resource Lifecycle

A media stream should have explicit cleanup:

```js
for (const track of stream.getTracks()) {
  track.stop();
}
```

---

## 24.3 Screen Capture

Screen capture typically requires user-mediated permission and has important security constraints.

---

# 25. Permissions

The Permissions API can query permission state for some browser capabilities.

Example:

```js
const status = await navigator.permissions.query({
  name: "geolocation"
});

console.log(status.state);
```

Conceptual states:

```text
granted
prompt
denied
```

Not every API has identical permission semantics or is represented through this interface.

---

## 25.1 Three Questions

Do not collapse:

```text
API exists?
```

with:

```text
Permission granted?
```

or:

```text
Operation succeeded?
```

---

## 25.2 Permission Can Change

A permission state can change while an application is running.

Applications should not assume it is immutable.

---

# 26. Clipboard, Fullscreen, Wake Lock, and User Activation

Some APIs are especially sensitive to user intent.

Examples:

```js
navigator.clipboard.writeText("hello");
```

```js
element.requestFullscreen();
```

```js
navigator.wakeLock.request("screen");
```

These capabilities can have:

```text
secure-context requirements
permission requirements
user-activation requirements
visibility requirements
```

The exact rule is API-specific.

---

## 26.1 User Activation

A real user gesture can create browser state that allows certain privileged actions.

A synthetic event should not be assumed to create identical privilege.

This connects to Chapter 50.

---

# 27. Notifications

The Notifications API allows web applications to request permission to show notifications.

```js
const permission = await Notification.requestPermission();
```

Then, when permitted:

```js
new Notification("Hello");
```

Availability and permission depend on browser policies and context.

Some notification features require service workers or related architecture.

---

## 27.1 Permission UX

Do not automatically request notification permission on page load.

Permission prompts are user-experience and trust decisions.

---

# 28. Geolocation

Example:

```js
navigator.geolocation.getCurrentPosition(
  position => {
    console.log(position.coords.latitude);
  },
  error => {
    console.error(error);
  }
);
```

Geolocation is permission-controlled and may depend on:

```text
secure context
permission
device capability
browser policy
location availability
```

---

## 28.1 Accuracy vs Cost

Higher location accuracy can involve additional:

```text
battery
sensor
network
latency
```

costs.

Use the least capability necessary.

---

# 29. Web Workers and Shared Capabilities

Workers provide execution contexts separate from the main Window agent.

Examples:

```js
const worker = new Worker("/worker.js");
```

---

## 29.1 Worker API Surface

Workers have access to many APIs but not the DOM document.

---

## 29.2 Messaging

```js
worker.postMessage(data);
```

Messages use structured cloning and can support transferables.

---

## 29.3 SharedArrayBuffer

For shared memory, use:

```text
SharedArrayBuffer
Atomics
```

subject to platform isolation/security requirements.

---

## 29.4 Worker Lifecycle

Terminate when the worker is no longer needed:

```js
worker.terminate();
```

---

## 29.5 Browser API Context Matrix

When designing code, ask:

```text
Window?
DedicatedWorker?
SharedWorker?
ServiceWorker?
Worklet?
```

API availability differs.

---

# 30. API Availability and Feature Detection

## 30.1 Good

```js
if ("IntersectionObserver" in globalThis) {
  startLazyLoading();
}
```

---

## 30.2 Better for Method-Level Checks

```js
if (
  "clipboard" in navigator &&
  typeof navigator.clipboard.writeText === "function"
) {
  // ...
}
```

---

## 30.3 Feature Detection

Test actual capability.

Do not use user-agent strings as the primary compatibility mechanism.

---

## 30.4 Progressive Enhancement

Structure:

```text
core functionality
+
enhanced API when available
```

Example:

```js
if ("requestIdleCallback" in window) {
  requestIdleCallback(work);
} else {
  setTimeout(work, 0);
}
```

---

# 31. Security Contexts and Powerful Features

A secure context generally means:

```text
HTTPS
```

or another browser-recognized secure origin context.

Some powerful APIs are restricted to secure contexts.

Examples can include:

```text
geolocation
clipboard capabilities
camera/microphone
Web Locks
```

Availability is API-specific.

MDN documents Web Locks as a secure-context feature, for example. citeturn422860search1

---

## 31.1 Secure Context ≠ Permission Granted

Even in HTTPS:

```text
API may exist
permission may be denied
operation may still fail
```

---

## 31.2 Permissions Policy

Embedded documents can also be affected by Permissions Policy.

This becomes particularly relevant for:

```text
iframes
camera
microphone
geolocation
fullscreen
```

---

# 32. Browser Compatibility

Browser APIs evolve.

Track:

```text
specification status
browser support
secure-context requirement
permission model
worker support
mobile differences
```

---

## 32.1 Baseline

MDN compatibility information can be useful for high-level support evaluation.

But production decisions should consider:

```text
your browser matrix
your users
your device mix
your feature fallback
```

---

## 32.2 Vendor Prefixes

Legacy APIs may have historical prefixes.

Modern production code should prefer standardized APIs where practical.

---

## 32.3 Experimental APIs

Some browser APIs listed in documentation can have limited or evolving support.

Never promote an experimental API into core architecture without an explicit compatibility strategy.

---

# 33. Edge Cases

## 33.1 Timer After Page Visibility Change

Background pages can experience throttled timer behavior.

Never depend on timer precision for critical correctness.

---

## 33.2 rAF in Background

Browsers commonly pause or reduce animation callbacks for background/hidden contexts. citeturn422860search8

---

## 33.3 `navigator.onLine`

Online does not guarantee server reachability.

---

## 33.4 localStorage Exceptions

Storage can fail due to:

```text
quota
privacy mode/policy
security restrictions
disabled storage
```

Do not assume `setItem()` always succeeds.

---

## 33.5 IndexedDB Version Races

Multiple tabs can interact with database version changes.

Production IndexedDB design must handle upgrade/block/versionchange behavior.

---

## 33.6 BroadcastChannel Lifecycle

Open channels should be closed:

```js
channel.close();
```

when no longer needed.

---

## 33.7 Web Lock Contention

Multiple tabs can wait on the same lock.

Poor lock architecture can create:

```text
delays
starvation-like behavior
deadlock risks through multiple locks
```

Use the smallest critical section.

---

## 33.8 ResizeObserver Feedback

Changing size from within the observer can trigger another notification.

Guard against feedback loops. citeturn422860search9

---

## 33.9 IntersectionObserver Is Asynchronous

Do not use it as a synchronous geometry query.

---

## 33.10 Permission Revocation

A permission previously granted can later become denied.

---

## 33.11 User Activation Expires

Transient user-activation state is not a permanent application privilege.

---

## 33.12 Iframes

APIs can be affected by:

```text
origin
sandbox
Permissions Policy
user activation
top-level vs embedded context
```

---

## 33.13 Worker Restrictions

Code that works in Window may fail in Worker due to:

```text
document unavailable
window unavailable
different API exposure
```

---

## 33.14 Mobile Background Behavior

Mobile browsers can suspend/freeze/discard pages more aggressively than desktop assumptions suggest.

Design for lifecycle interruptions.

---

# 34. Common Misconceptions

## Misconception 1 — “Web APIs are JavaScript.”

No. They are platform APIs exposed to JavaScript.

## Misconception 2 — “If the property exists, the API works.”

No.

## Misconception 3 — “HTTPS guarantees access.”

No.

## Misconception 4 — “Permission means success.”

No. Hardware, policy, visibility, user state, and runtime conditions still matter.

## Misconception 5 — “setTimeout(0) means now.”

No.

## Misconception 6 — “setTimeout(1000) means exactly one second.”

No.

## Misconception 7 — “requestAnimationFrame is a better setInterval.”

Not universally. It serves rendering-oriented scheduling.

## Misconception 8 — “Observer callbacks run immediately when the state changes.”

They have specific asynchronous delivery semantics.

## Misconception 9 — “localStorage is a database.”

It is a small synchronous string key/value store.

## Misconception 10 — “IndexedDB is synchronous.”

Its primary browser API is asynchronous.

## Misconception 11 — “navigator.onLine means internet works.”

No.

## Misconception 12 — “Web Locks are shared memory.”

No.

## Misconception 13 — “BroadcastChannel shares object references.”

No. Messages use structured-clone-style data transfer semantics.

## Misconception 14 — “A Worker is just another callback.”

No. It is a different execution context/agent.

## Misconception 15 — “Synthetic click creates full user privileges.”

No.

## Misconception 16 — “Background pages behave exactly like foreground pages.”

No. Browsers may throttle, freeze, or otherwise manage background work.

## Misconception 17 — “Every Web API works in every browser context.”

No.

## Misconception 18 — “Browser APIs never change.”

They evolve and may gain/remove/deprecate capabilities.

---

# 35. Common Mistakes

## Mistake 1 — Feature detection by browser name

Use capability detection.

## Mistake 2 — No cleanup

Always consider:

```text
timer
observer
channel
worker
listener
media track
lock
```

lifecycle.

## Mistake 3 — Heavy timer loops

Timers are not a substitute for architecture.

## Mistake 4 — Using rAF for non-visual recurring work

Use the scheduler appropriate to the task.

## Mistake 5 — Polling when an observer exists

Polling can waste CPU and complicate correctness.

## Mistake 6 — Storing sensitive information in convenient client storage

Convenience is not a security model.

## Mistake 7 — Assuming permission state is static

It can change.

## Mistake 8 — Assuming every API is secure-context independent

Check the API.

## Mistake 9 — Ignoring iframe policies

Embedded contexts can have additional restrictions.

## Mistake 10 — Treating API errors as exceptional edge cases only

Permission denial, quota, unavailable hardware, unsupported APIs, and lifecycle suspension are normal conditions.

---

# 36. Comparison With Related Concepts

| Concept | Layer | Typical use |
|---|---|---|
| ECMAScript API | Language | `Map`, `Promise`, `Array` |
| DOM API | Web Platform | Document manipulation |
| Browser Web API | Web Platform | Browser/device/network capabilities |
| Library | Application dependency | Reusable abstractions |
| Framework | Application architecture | Rendering/state structure |
| Native OS API | Operating system | Device/system capabilities |
| WebAssembly API | Web/engine | Native-like computation/runtime |
| Worker API | Browser execution model | Background JS execution |

---

## `setTimeout` vs `requestAnimationFrame`

### `setTimeout`

```text
general delayed task
```

### `requestAnimationFrame`

```text
rendering-aligned visual work
```

---

## `localStorage` vs IndexedDB

### localStorage

```text
small
simple
synchronous
string key/value
```

### IndexedDB

```text
structured
larger
transactional
asynchronous
indexed
```

---

## Polling vs Observer

### Polling

```text
check repeatedly
```

### Observer

```text
register
→ browser tracks
→ callback/records
```

---

## BroadcastChannel vs Web Locks

### BroadcastChannel

```text
send messages
```

### Web Locks

```text
coordinate resource ownership
```

---

## Event vs Observer

### Event

```text
dispatch
→ event path
→ listeners
```

### Observer

```text
observe state
→ browser records
→ callback delivery
```

---

# 37. Performance Considerations

## 37.1 Browser APIs Have Different Cost Models

Do not classify an entire API as:

```text
fast
```

or:

```text
slow
```

Measure the specific operation and downstream consequences.

---

## 37.2 Timers

Large numbers of short timers can create scheduling overhead.

Prefer architecture that coalesces work where appropriate.

---

## 37.3 rAF

For visual updates:

```text
collect state
→ one rAF callback
→ one render update
```

can avoid redundant per-event work.

---

## 37.4 Observers

IntersectionObserver and ResizeObserver exist partly to let the browser optimize state detection instead of applications continuously polling. citeturn422860search4turn422860search5

---

## 37.5 Storage

Avoid:

```js
for (...) {
  localStorage.setItem(...);
}
```

on hot paths.

Batch state in memory and persist deliberately.

---

## 37.6 IndexedDB

Transactions should be scoped narrowly enough to avoid unnecessary contention while still maintaining atomicity.

---

## 37.7 BroadcastChannel

High-frequency cross-tab messaging can create unnecessary serialization and event traffic.

---

## 37.8 Web Locks

Keep lock critical sections small:

```text
acquire
→ minimal shared-resource work
→ release
```

---

## 37.9 Media

Camera/microphone streams can consume CPU, memory, bandwidth, and battery.

Stop tracks when no longer required.

---

## 37.10 Workers

Workers remove CPU-heavy JavaScript from the main execution context but introduce:

```text
serialization
messaging
startup
memory
```

costs.

---

## 37.11 API Composition

A production optimization might look like:

```text
pointer events
→ coalesce state
→ rAF
→ batch DOM writes
→ avoid layout reads
```

rather than:

```text
pointermove
→ DOM mutation
→ layout read
```

on every event.

---

# 38. Memory Considerations

## 38.1 Timers Retain Closures

```js
const data = createLargeObject();

setTimeout(() => {
  use(data);
}, 60_000);
```

The timer can keep captured data reachable until it fires or is canceled.

---

## 38.2 Observers Retain Relationships

Observers can keep observation relationships alive.

Disconnect during teardown.

---

## 38.3 BroadcastChannel

Open channels and message handlers participate in object lifecycles.

Close unused channels.

---

## 38.4 Workers

Workers have their own memory/execution resources.

Terminate when unnecessary.

---

## 38.5 Media Streams

Media tracks can represent substantial native resources.

Stop tracks.

---

## 38.6 Storage Memory vs Disk

Persistent storage is not equivalent to:

```text
JavaScript heap
```

and quotas/errors vary by API/browser.

---

## 38.7 Cache Storage

Cached responses consume browser-managed storage rather than ordinary JS heap.

Still requires quota/lifecycle reasoning.

---

## 38.8 DOM Connection

Browser APIs can indirectly retain DOM references:

```text
observer
→ DOM node
→ component state
```

Lifecycle ownership is critical.

---

# 39. Security Considerations

## 39.1 Powerful APIs

Features exposing:

```text
camera
microphone
location
clipboard
screen
notifications
```

need careful permission and user-trust handling.

---

## 39.2 Secure Context

Use HTTPS for production sites that need secure-context APIs.

---

## 39.3 Permissions Policy

Embedded applications should explicitly understand what capabilities are allowed.

---

## 39.4 Storage

Do not treat:

```text
localStorage
IndexedDB
```

as secret vaults.

XSS or same-origin compromise can expose client-side application data.

---

## 39.5 Cookies

Sensitive authentication cookies often rely on:

```text
HttpOnly
Secure
SameSite
```

where appropriate.

---

## 39.6 BroadcastChannel

Same-origin pages can communicate through BroadcastChannel.

Do not broadcast sensitive information unnecessarily.

---

## 39.7 Web Locks

Locks coordinate code within an origin; they do not establish a trust boundary.

---

## 39.8 Notifications

Do not abuse notification permission.

Permission UX affects user trust and browser policy.

---

## 39.9 User Activation

Do not design security around the assumption that a synthetic event is equivalent to a trusted gesture.

---

## 39.10 Feature Fallbacks

Fallback code must not silently weaken security.

Example:

```text
secure clipboard API unavailable
→ do not automatically copy data through an unsafe mechanism
```

---

# 40. Production Usage

## 40.1 Capability Layer

Instead of scattering:

```js
if ("x" in navigator)
```

through the application, create a capability abstraction:

```js
const capabilities = {
  clipboard: Boolean(
    navigator.clipboard &&
    typeof navigator.clipboard.writeText === "function"
  ),
  notifications: "Notification" in globalThis,
  workers: typeof Worker === "function"
};
```

Then centralize behavior and fallbacks.

---

## 40.2 Lifecycle Controller

Use one controller to coordinate teardown:

```js
const controller = new AbortController();

window.addEventListener("resize", onResize, {
  signal: controller.signal
});

button.addEventListener("click", onClick, {
  signal: controller.signal
});

const observer = new ResizeObserver(onResize);
observer.observe(panel);

// teardown
controller.abort();
observer.disconnect();
```

For non-AbortSignal APIs, use their specific cleanup method.

---

## 40.3 Storage Architecture

Choose based on data:

```text
temporary UI state
→ memory

small preferences
→ localStorage

structured offline data
→ IndexedDB

HTTP-managed state
→ cookies

HTTP response caching
→ Cache Storage
```

---

## 40.4 Scheduling Architecture

Choose:

```text
delayed task
→ timer

visual update
→ requestAnimationFrame

CPU-heavy computation
→ Worker

intersection
→ IntersectionObserver

element resize
→ ResizeObserver
```

This is a decision framework, not a rule to use every API mechanically.

---

## 40.5 Failure-First Design

Every powerful API should have an explicit failure model:

```text
unsupported
permission denied
context denied
quota exceeded
device unavailable
lifecycle interruption
runtime failure
```

---

## 40.6 Progressive Enhancement

Core functionality should remain usable where a capability is unavailable whenever product requirements permit.

---

## 40.7 Production Decision Framework

| Dimension | Question |
|---|---|
| Correctness | Does the API behave correctly across lifecycle/context states? |
| Performance | Is it reducing or increasing real work? |
| Memory | What resources remain reachable? |
| Security | What permissions, origins, or sensitive data are involved? |
| Reliability | What happens when API use fails? |
| Accessibility | Does capability use preserve usable interaction? |
| Maintainability | Is API ownership centralized? |
| Scalability | What happens across tabs/components/devices? |
| Observability | Can failures and performance be measured? |
| Operational Complexity | How many lifecycle/fallback states exist? |
| Future Change | What happens when browser support or API behavior evolves? |

---

# 41. Implementation From Scratch

Build a **toy Browser API host** around your Chapter 49 DOM and Chapter 50 event system.

---

## Stage 1 — Global Environment

Implement:

```js
const globalThisMock = {};
```

Expose:

```text
document
navigator
setTimeout
clearTimeout
```

---

## Stage 2 — Timer Scheduler

Build:

```text
schedule(callback, delay)
cancel(id)
```

using an internal priority queue.

Model:

```text
current time
due timers
task queue
```

---

## Stage 3 — Event Loop

Build:

```text
task queue
microtask queue
timer queue
```

and execute:

```text
task
→ microtasks
→ next task
```

---

## Stage 4 — rAF Scheduler

Create:

```text
frame clock
→ callbacks before frame
```

so you can compare:

```text
timer scheduling
vs
frame scheduling
```

---

## Stage 5 — Storage

Implement:

```js
localStorageMock
```

with:

```text
getItem
setItem
removeItem
clear
```

Then enforce an artificial quota.

---

## Stage 6 — IndexedDB-Like Store

Implement:

```text
database
object stores
transactions
indexes
version upgrades
```

---

## Stage 7 — BroadcastChannel

Create:

```text
channel registry
→ postMessage
→ structured clone
→ recipient queues
```

---

## Stage 8 — Web Lock

Implement:

```text
name
queue
owner
grant
release
```

and test competing contexts.

---

## Stage 9 — IntersectionObserver

Use your toy DOM geometry system:

```text
root rectangle
target rectangle
thresholds
```

and generate entries.

---

## Stage 10 — ResizeObserver

Track:

```text
previous size
current size
changed?
```

and queue records.

---

## Stage 11 — Performance API

Implement:

```js
performance.now()
performance.mark()
performance.measure()
```

---

## Stage 12 — Permission Manager

Implement:

```text
feature
→ granted/prompt/denied
```

and require permission for simulated powerful APIs.

---

## Stage 13 — Context Matrix

Create:

```text
WindowContext
WorkerContext
```

and expose different API surfaces.

---

## Stage 14 — Failure Simulation

Make APIs fail due to:

```text
unsupported
denied
quota
timeout
device unavailable
context unavailable
```

This teaches production capability engineering better than implementing only the happy path.

---

# 42. Debugging Exercises

## Exercise 1 — Timer Ordering

```js
console.log("start");

setTimeout(() => {
  console.log("timer");
}, 0);

Promise.resolve().then(() => {
  console.log("microtask");
});

console.log("end");
```

Predict:

```text
start
end
microtask
timer
```

Then explain why the timer callback does not interrupt the current task.

---

## Exercise 2 — Long Task

```js
setTimeout(() => {
  console.log("timer");
}, 0);

const start = performance.now();

while (performance.now() - start < 1000) {
  // block
}
```

What happens to timer responsiveness?

---

## Exercise 3 — rAF

Schedule:

```text
timer
rAF
Promise
```

and investigate their relative timing against a rendering frame using browser tooling.

---

## Exercise 4 — Visibility

Build a polling component that:

```text
polls while visible
pauses while hidden
```

Verify that lifecycle state is handled correctly.

---

## Exercise 5 — localStorage Failure

Simulate:

```text
quota exceeded
storage denied
```

and make the application degrade gracefully.

---

## Exercise 6 — BroadcastChannel

Open two same-origin tabs and synchronize:

```text
theme changes
logout
cache invalidation
```

Then close one channel and verify cleanup.

---

## Exercise 7 — Web Lock

Create:

```text
two tabs
same lock
shared sync task
```

Ensure only one executes the critical section at a time.

---

## Exercise 8 — IntersectionObserver

Build:

```text
infinite scroll
sentinel
```

and compare with timer-based scroll polling.

---

## Exercise 9 — ResizeObserver Loop

Create:

```js
observer = new ResizeObserver(() => {
  element.style.width = `${element.offsetWidth + 1}px`;
});
```

Observe the resulting behavior and explain why this is a feedback loop.

---

## Exercise 10 — Permission Lifecycle

Simulate:

```text
prompt
→ granted
→ revoked
```

and make application state react correctly.

---

# 43. Code Review Exercise

Review:

```js
function startDashboard() {
  setInterval(refreshDashboard, 1000);

  window.addEventListener("resize", refreshDashboard);

  setInterval(checkNotifications, 5000);

  document.addEventListener("visibilitychange", refreshDashboard);
}
```

A developer says:

> “The dashboard keeps itself updated, so this is fine.”

## Problems

There is no:

```text
cleanup
visibility pause
back-pressure
overlap prevention
API capability abstraction
```

It may also perform unnecessary work.

A stronger design could use:

```text
visibility state
AbortController
single scheduler
requestAnimationFrame for visual updates
server push where appropriate
```

and potentially observers instead of polling for browser state.

---

## Review Questions

1. Can multiple refresh operations overlap?
2. What happens after component unmount?
3. Does the hidden page continue polling?
4. Can a resize storm trigger excessive work?
5. Is `refreshDashboard()` visual, network, or CPU work?
6. Should each concern use the same scheduling primitive?
7. Can observer APIs replace some polling?
8. What failure happens when network is unavailable?

---

# 44. Interview Questions

## Foundational

1. What is a Web API?
2. Is `fetch` ECMAScript?
3. What is the difference between a browser API and a library?
4. What is `globalThis`?
5. What is `window`?
6. What is `navigator`?
7. What is Web IDL?

## Scheduling

8. How does `setTimeout` work conceptually?
9. Why is `setTimeout(0)` not immediate?
10. Why is timer delay not exact?
11. What is `requestAnimationFrame`?
12. When should you use rAF instead of a timer?
13. Why do long synchronous tasks block input?

## Storage

14. localStorage vs sessionStorage?
15. localStorage vs IndexedDB?
16. When would you use cookies?
17. What is Cache Storage?

## Coordination

18. What is BroadcastChannel?
19. What is Web Locks?
20. BroadcastChannel vs Web Locks?
21. What is structured cloning?

## Observers

22. Why do IntersectionObserver and ResizeObserver exist?
23. Why is polling often worse?
24. What is a ResizeObserver feedback loop?

## Security/Permissions

25. What is a secure context?
26. Does HTTPS guarantee permission?
27. What is Permissions Policy?
28. What is user activation?
29. Why can synthetic events fail to reproduce privileged browser behavior?

## Advanced

30. Why can API existence differ between Window and Worker?
31. How would you design a browser API compatibility layer?
32. How would you test permission denial?
33. How would you diagnose a tab that remains CPU-active while hidden?
34. How would you diagnose memory growth caused by observers/timers?
35. How would you choose between polling, events, observers, and workers?

---

# 45. Predict-the-Output Exercises

## Exercise 1 — Timer and Microtask

```js
console.log("A");

setTimeout(() => {
  console.log("C");
}, 0);

Promise.resolve().then(() => {
  console.log("B");
});

console.log("D");
```

### Prediction

```text
A
D
B
C
```

### Reasoning

Synchronous code completes first.

Then the Promise reaction runs at the relevant microtask checkpoint.

The timer callback requires later task processing.

---

## Exercise 2 — localStorage

```js
localStorage.setItem("theme", "dark");

console.log(localStorage.getItem("theme"));
```

### Prediction

```text
dark
```

---

## Exercise 3 — BroadcastChannel

```js
const channel = new BroadcastChannel("demo");

channel.onmessage = event => {
  console.log(event.data);
};

channel.postMessage("hello");
```

Do not assume the message callback runs like a direct function call.

The message is delivered through browser messaging/task machinery.

---

## Exercise 4 — Permission

```js
const status = await navigator.permissions.query({
  name: "geolocation"
});

console.log(status.state);
```

### Prediction

One of:

```text
granted
prompt
denied
```

depending on browser/user/context state.

---

## Exercise 5 — Observer Lifecycle

```js
const observer = new ResizeObserver(entries => {
  console.log(entries.length);
});

observer.observe(element);
observer.disconnect();
```

### Prediction

No subsequent size-change notifications should be delivered through that disconnected observer.

---

## Exercise 6 — Worker Context

```js
console.log(typeof document);
```

inside a normal Dedicated Worker.

### Prediction

Typically:

```text
undefined
```

The Worker does not have the Window document object.

---

## Exercise 7 — rAF Scheduling

```js
requestAnimationFrame(() => {
  console.log("frame");
});

console.log("now");
```

### Prediction

```text
now
frame
```

The animation callback is scheduled for a future rendering opportunity rather than executed synchronously.

---

# 46. Mastery Exercises

## Level 1 — Classify APIs

Given:

```text
Array
Promise
document
fetch
localStorage
IndexedDB
setTimeout
requestAnimationFrame
Worker
IntersectionObserver
```

classify:

```text
ECMAScript
Web Platform
host/browser
```

and explain each.

---

## Level 2 — Scheduling Lab

Compare:

```text
setTimeout
requestAnimationFrame
Promise.then
queueMicrotask
Worker
```

for:

```text
delayed work
visual work
microtask continuation
CPU-heavy work
```

---

## Level 3 — Storage Lab

Build the same feature using:

```text
localStorage
IndexedDB
```

and compare:

```text
data model
sync/async
failure modes
capacity
lifecycle
security
```

---

## Level 4 — Observer Lab

Build:

```text
lazy image loader
```

using:

```text
IntersectionObserver
```

Then implement a polling version.

Measure:

```text
CPU
callback count
scroll performance
```

---

## Level 5 — Cross-Tab Coordination

Build:

```text
multi-tab session manager
```

using:

```text
BroadcastChannel
Web Locks
```

Separate:

```text
notification
vs
exclusive ownership
```

---

## Level 6 — Permission Architecture

Design:

```text
camera feature
location feature
notifications feature
clipboard feature
```

with:

```text
unsupported
prompt
granted
denied
revoked
```

states.

---

## Level 7 — API Adapter

Create:

```js
browserCapabilities
```

that normalizes:

```text
availability
permission
request
cleanup
error
```

for each feature.

---

## Level 8 — Lifecycle Lab

Build a component with:

```text
timer
resize observer
event listener
BroadcastChannel
worker
```

and prove every resource is cleaned up on unmount.

---

## Level 9 — Performance Lab

Create:

```text
1000 pointer events
```

and compare:

```text
direct DOM writes
rAF-coalesced updates
worker computation + main-thread rendering
```

Measure:

```text
input latency
CPU
frame rate
memory
```

---

## Level 10 — Principal Browser Architecture

Design an offline-first dashboard with:

```text
multi-tab synchronization
background refresh
large local dataset
visual updates
permission-controlled notifications
network failures
page visibility changes
```

Defend:

```text
IndexedDB
BroadcastChannel
Web Locks
Service Worker
requestAnimationFrame
observers
Workers
permissions
lifecycle
```

and explain why each belongs at its chosen layer.

---

# 47. Key Takeaways

1. Browser Web APIs are Web Platform capabilities exposed to JavaScript.
2. They are separate from ECMAScript language features.
3. `globalThis`, `window`, `document`, and `navigator` represent different host/environment concepts.
4. Web IDL helps define JavaScript-visible platform interfaces.
5. A platform object can look like a JavaScript object while being backed by browser-native systems.
6. Timer APIs schedule callbacks; they are not precise sleep primitives.
7. `requestAnimationFrame()` is oriented around rendering opportunities.
8. Long synchronous JavaScript blocks browser progress.
9. Browser lifecycle and visibility affect scheduling and resource strategy.
10. `localStorage` is synchronous string key/value storage.
11. IndexedDB provides structured, transactional browser storage.
12. Cookies are tied to HTTP/session/security behavior and are not equivalent to localStorage.
13. Cache Storage is distinct from IndexedDB.
14. BroadcastChannel communicates among same-origin contexts.
15. Web Locks coordinate resource ownership among same-origin contexts.
16. Observers provide browser-managed state observation instead of constant polling.
17. IntersectionObserver is suited to threshold-based intersection visibility.
18. ResizeObserver is suited to element-size changes.
19. Performance APIs let applications measure their own work.
20. Browser APIs frequently have distinct availability, context, permission, and runtime-success states.
21. Secure context is one common requirement for powerful APIs.
22. Permission is not authorization to trust the application's own data.
23. User activation can gate privileged actions.
24. Workers provide different execution contexts and can move CPU-heavy work away from the main context.
25. Feature detection is superior to browser-name branching for capability support.
26. Browser API use requires explicit lifecycle cleanup.
27. Production browser architecture is a capability-management problem as much as an API-call problem.
28. Every API choice should consider correctness, performance, memory, security, resilience, accessibility, maintainability, and future compatibility.

---

# 48. Concept Connections

## Depends On

```text
Chapter 41 — Specification Architecture
        ↓
Chapter 42 — Abstract Operations
        ↓
Chapter 43 — Ordinary Object Methods
        ↓
Chapter 44 — Realms / Agents
        ↓
Chapter 45 — Memory / GC
        ↓
Chapter 47 — Engine Architecture
        ↓
Chapter 48 — V8 Internals
        ↓
Chapter 49 — DOM Architecture
        ↓
Chapter 50 — Browser Events
        ↓
Chapter 51 — Browser Web APIs
```

## Builds Toward

```text
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams / Data Flow
Chapter 54 — Web Components
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JS Security Engineering
Chapter 70 — Source Maps / Production Debugging
Chapter 83 — Observability
Chapter 85 — Performance
```

## Related Concepts

- HTML
- CSSOM
- event loop
- tasks
- microtasks
- DOM
- Workers
- storage
- permissions
- browser lifecycle
- rendering
- networking
- security contexts
- structured cloning

## Concepts Revisited

### Chapter 49 — DOM Architecture

Many browser APIs manipulate or observe DOM state:

```text
IntersectionObserver
ResizeObserver
requestAnimationFrame
```

### Chapter 50 — Browser Events

Browser APIs can produce or consume event-driven behavior, but not every callback-based API is an EventTarget event.

### Chapter 28 — Structured Clone

BroadcastChannel and Worker messaging use structured-clone-related data transfer semantics rather than shared object identity.

### Chapter 37 — Cancellation

`AbortSignal` can provide lifecycle cancellation across suitable Web APIs.

### Chapter 44 — Agents

Workers and browser contexts map to distinct execution contexts/agents.

### Chapter 45 — Memory

Timers, observers, channels, workers, and media can all create resource-retention concerns.

---

## Why This Chapter Matters Later

Chapter 52 deepens concurrency.

Chapter 53 deepens browser data flow.

Chapter 55 focuses on networking.

Chapter 56 focuses on browser security.

Chapter 83 and Chapter 85 rely on understanding which browser subsystem is responsible for observed work.

The central lesson is:

```text
Browser API ≠ magic function
```

It is:

```text
JavaScript interface
→ Web Platform semantics
→ browser subsystem
→ OS/device/network
```

---

# 49. Completion Criteria

## Classification

- [ ] Distinguish ECMAScript from Web APIs.
- [ ] Distinguish Web API from library/framework.
- [ ] Explain platform objects.
- [ ] Explain Web IDL.

## Global Environment

- [ ] Explain `globalThis`.
- [ ] Explain `window`.
- [ ] Explain `document`.
- [ ] Explain `navigator`.
- [ ] Explain Window vs Worker context.

## Scheduling

- [ ] Explain timers.
- [ ] Explain timer delay vs exact timing.
- [ ] Explain timer cancellation.
- [ ] Explain requestAnimationFrame.
- [ ] Explain cooperative scheduling.
- [ ] Explain long-task blocking.

## Storage

- [ ] Explain localStorage.
- [ ] Explain sessionStorage.
- [ ] Explain cookies.
- [ ] Explain IndexedDB.
- [ ] Explain Cache Storage.

## Coordination

- [ ] Explain BroadcastChannel.
- [ ] Explain Web Locks.
- [ ] Distinguish messaging from ownership.

## Observers

- [ ] Explain MutationObserver.
- [ ] Explain IntersectionObserver.
- [ ] Explain ResizeObserver.
- [ ] Explain PerformanceObserver.
- [ ] Explain observer lifecycle.

## Capability APIs

- [ ] Explain permissions.
- [ ] Explain secure contexts.
- [ ] Explain user activation.
- [ ] Explain feature detection.
- [ ] Explain progressive enhancement.
- [ ] Explain Worker-specific availability.

## Performance

- [ ] Measure API work.
- [ ] Avoid unnecessary polling.
- [ ] Use rendering-aligned scheduling where appropriate.
- [ ] Reason about Worker trade-offs.

## Memory

- [ ] Diagnose timer retention.
- [ ] Diagnose observer retention.
- [ ] Diagnose channel/worker lifecycle.
- [ ] Manage media resources.

## Security

- [ ] Explain powerful API risks.
- [ ] Explain secure contexts.
- [ ] Explain permissions.
- [ ] Explain Permissions Policy.
- [ ] Explain storage security boundaries.
- [ ] Explain synthetic event limitations.

## Principal Judgment

- [ ] Design a browser capability abstraction layer.
- [ ] Design failure-first API usage.
- [ ] Design lifecycle-safe resources.
- [ ] Defend API choices across multiple browser contexts.

---

# 50. Revision / Retrieval Record

## First-Pass Retrieval

Without opening the chapter:

1. What is a browser Web API?
2. How is it different from ECMAScript?
3. What is a platform object?
4. What is Web IDL?
5. What are `globalThis`, `window`, `document`, and `navigator`?
6. Why is `setTimeout(0)` not immediate?
7. Why is a timer not a deadline?
8. What is requestAnimationFrame?
9. Why can long synchronous code block the browser?
10. What is localStorage?
11. Why is localStorage not a database?
12. When should IndexedDB be used?
13. What is Cache Storage?
14. What is BroadcastChannel?
15. What is Web Locks?
16. What is the difference between messaging and locking?
17. Why do observer APIs exist?
18. What is IntersectionObserver?
19. What is ResizeObserver?
20. What is PerformanceObserver?
21. What is secure context?
22. Does HTTPS guarantee API access?
23. What is the difference between API existence and permission?
24. What is user activation?
25. Why does feature detection beat browser-name detection?
26. What changes in Worker context?
27. Why can background lifecycle affect timers/rAF?
28. How can API resources cause memory retention?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Web API vs ECMAScript | [ ] | [ ] | [ ] | [ ] |
| Platform objects | [ ] | [ ] | [ ] | [ ] |
| Web IDL | [ ] | [ ] | [ ] | [ ] |
| globalThis | [ ] | [ ] | [ ] | [ ] |
| window/document/navigator | [ ] | [ ] | [ ] | [ ] |
| timers | [ ] | [ ] | [ ] | [ ] |
| requestAnimationFrame | [ ] | [ ] | [ ] | [ ] |
| scheduling | [ ] | [ ] | [ ] | [ ] |
| page visibility | [ ] | [ ] | [ ] | [ ] |
| localStorage | [ ] | [ ] | [ ] | [ ] |
| sessionStorage | [ ] | [ ] | [ ] | [ ] |
| cookies | [ ] | [ ] | [ ] | [ ] |
| IndexedDB | [ ] | [ ] | [ ] | [ ] |
| Cache Storage | [ ] | [ ] | [ ] | [ ] |
| BroadcastChannel | [ ] | [ ] | [ ] | [ ] |
| Web Locks | [ ] | [ ] | [ ] | [ ] |
| observers | [ ] | [ ] | [ ] | [ ] |
| IntersectionObserver | [ ] | [ ] | [ ] | [ ] |
| ResizeObserver | [ ] | [ ] | [ ] | [ ] |
| Performance APIs | [ ] | [ ] | [ ] | [ ] |
| permissions | [ ] | [ ] | [ ] | [ ] |
| secure contexts | [ ] | [ ] | [ ] | [ ] |
| user activation | [ ] | [ ] | [ ] | [ ] |
| feature detection | [ ] | [ ] | [ ] | [ ] |
| Workers | [ ] | [ ] | [ ] | [ ] |
| lifecycle cleanup | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Classify twenty browser features as:

```text
ECMAScript
Web Platform
library
```

### Day 2

Explain:

```text
timer
rAF
observer
Worker
```

as four different scheduling/resource patterns.

### Day 7

Design a storage strategy for a browser application.

### Day 14

Design a capability/permission abstraction for:

```text
clipboard
location
camera
notifications
```

### Day 30

Design a multi-tab offline-first application using:

```text
IndexedDB
BroadcastChannel
Web Locks
Workers
observers
visibility
```

and defend the architecture.

---

# 51. Canonical References and Source Discipline

## Primary Web Platform References

### MDN Web APIs

https://developer.mozilla.org/en-US/docs/Web/API

MDN maintains a broad catalog of browser Web APIs and interfaces. citeturn422860search2

### MDN — Introduction to Web APIs

https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Client-side_APIs/Introduction

Useful for the foundational distinction between browser Web APIs, JavaScript, libraries, and frameworks. citeturn422860search0

### WHATWG HTML Standard

https://html.spec.whatwg.org/

Use for:

- event loops
- timers
- page lifecycle
- browser scheduling
- document/Window environment

### WHATWG DOM Standard

https://dom.spec.whatwg.org/

Use for:

- DOM APIs
- EventTarget
- observers
- platform object behavior

### Web IDL

https://webidl.spec.whatwg.org/

Use for:

- platform interface definitions
- attributes
- operations
- Web API bindings

---

## API-Specific References

### requestAnimationFrame

https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame

Use for rendering-aligned scheduling behavior and callback timing considerations. citeturn422860search8

### IntersectionObserver

https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver

Use for asynchronous intersection observation and lifecycle. citeturn422860search3

### ResizeObserver

https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver

Use for element-size observation and feedback-loop considerations. citeturn422860search9

### Web Locks

https://developer.mozilla.org/en-US/docs/Web/API/Web_Locks_API

Use for same-origin cross-context resource coordination. citeturn422860search1

---

## Source Classification

For every claim, classify as:

```text
[ECMAScript]
[DOM Standard]
[HTML Standard]
[Web IDL]
[API-specific specification]
[Browser-specific]
[Measured]
[Historical]
```

Examples:

```text
"globalThis is a JavaScript global reference"
→ [ECMAScript]

"Element implements EventTarget"
→ [DOM/Web Platform]

"Web Locks is secure-context dependent"
→ [API-specific]

"Browser X pauses this API on version Y"
→ [Browser-specific + version-sensitive]

"This API reduced CPU 20% in our workload"
→ [Measured]
```

---

## Capability Matrix

For production support documentation record:

```text
API
Browser support
Version
Window support
Worker support
Secure context
Permission
User activation
Permissions Policy
Fallback
Cleanup
Known lifecycle restrictions
```

---

## Source-Derived Notes

MDN describes browser Web APIs as browser-provided capabilities built on top of the JavaScript language. citeturn422860search0

MDN's Web API catalog shows the breadth of browser interfaces, including storage, observers, media, permissions, scheduling, workers, networking, and newer browser features. citeturn422860search2

MDN documents `requestAnimationFrame()` as requesting work before the next repaint and notes refresh-rate and background-document considerations. citeturn422860search8

MDN documents IntersectionObserver as asynchronous intersection observation and specifically motivates it as an alternative to repeated intersection polling. citeturn422860search3turn422860search4

MDN documents ResizeObserver as an element-size observation mechanism and describes protection/error behavior around resize-feedback loops. citeturn422860search5turn422860search9

MDN documents Web Locks as same-origin coordination available in secure contexts and workers in supporting browsers. citeturn422860search1

---

# 52. Completion Snapshot

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

- [ ] Web APIs
- [ ] ECMAScript vs Web Platform
- [ ] platform objects
- [ ] Web IDL
- [ ] globalThis
- [ ] window
- [ ] document
- [ ] navigator
- [ ] timers
- [ ] requestAnimationFrame
- [ ] cooperative scheduling
- [ ] visibility
- [ ] localStorage
- [ ] sessionStorage
- [ ] cookies
- [ ] IndexedDB
- [ ] Cache Storage
- [ ] BroadcastChannel
- [ ] Web Locks
- [ ] observer APIs
- [ ] IntersectionObserver
- [ ] ResizeObserver
- [ ] Performance API
- [ ] permissions
- [ ] secure contexts
- [ ] user activation
- [ ] Workers
- [ ] feature detection
- [ ] progressive enhancement
- [ ] lifecycle management

### I can predict

- [ ] timer/microtask ordering
- [ ] timer delay limitations
- [ ] rAF asynchronous behavior
- [ ] visibility-related scheduling behavior
- [ ] storage failure cases
- [ ] BroadcastChannel delivery model
- [ ] Web Lock contention
- [ ] observer lifecycle
- [ ] permission states
- [ ] Window vs Worker API availability

### I can implement

- [ ] timer scheduler
- [ ] task/microtask loop
- [ ] rAF scheduler
- [ ] storage mock
- [ ] IndexedDB-like store
- [ ] BroadcastChannel mock
- [ ] Web Lock mock
- [ ] IntersectionObserver mock
- [ ] ResizeObserver mock
- [ ] Performance API subset
- [ ] permission manager
- [ ] Window/Worker capability matrix

### I can debug

- [ ] timer leaks
- [ ] observer leaks
- [ ] worker lifecycle
- [ ] channel lifecycle
- [ ] storage failures
- [ ] permission failures
- [ ] resize loops
- [ ] background throttling effects
- [ ] unsupported API fallback

### I can defend

- [ ] API choice
- [ ] scheduling choice
- [ ] storage architecture
- [ ] observer vs polling
- [ ] Worker usage
- [ ] permission strategy
- [ ] security context assumptions
- [ ] progressive enhancement
- [ ] lifecycle design

---

## Final Principal-Level Test

Explain this statement without notes:

> **A browser Web API is not simply a JavaScript helper function. It is a platform-defined interface that connects JavaScript execution with browser-managed capabilities such as documents, rendering, scheduling, storage, networking, devices, permissions, and other execution contexts. Its correct use requires understanding context, lifecycle, scheduling, failure, security, and compatibility—not merely memorizing method signatures.**

Your explanation is complete only when you can connect:

```text
JavaScript
   ↓
Web IDL / platform interface
   ↓
Web Platform semantics
   ↓
browser subsystem
   ↓
OS / device / network
```

while separately reasoning about:

```text
availability
permissions
user activation
execution context
lifecycle
cleanup
failure
performance
security
compatibility
```

The central mastery target of Chapter 51 is to stop thinking of browser APIs as “extra JavaScript functions” and instead understand them as a capability layer connecting JavaScript to the browser platform.