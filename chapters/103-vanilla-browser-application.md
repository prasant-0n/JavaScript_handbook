# Chapter 103 — Production Vanilla Browser App

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build a production-grade browser application using the Web Platform and JavaScript without a UI framework.
>
> **Role perspective:** Principal JavaScript Engineer · Browser-Platform Engineer · Frontend Architect · Performance Engineer · Accessibility Engineer · Security Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **A browser application is not merely a collection of DOM manipulation calls. It is a stateful client system operating inside a constrained event, rendering, networking, security, storage, and accessibility environment.**

---

# 1. Project Mission

Build a real browser application using:

```text
HTML
CSS
modern JavaScript
Web Platform APIs
```

without relying on a UI framework.

The application should demonstrate:

```text
DOM architecture
events
state
rendering
forms
validation
networking
storage
URL state
accessibility
security
performance
offline behavior
testing
observability
```

Recommended project:

```text
FocusBoard
```

A browser task/project management application.

Core workflow:

```text
open app
→ load state
→ view projects
→ create task
→ edit task
→ filter/sort
→ persist
→ sync with API
→ recover from errors
```

The exact domain may evolve.

The browser engineering principles should remain.

---

# 2. Learning Objectives

By completing this project you should be able to:

- Design a browser application's architecture without a framework.
- Separate application state from DOM state.
- Design DOM ownership boundaries.
- Render dynamic content safely.
- Handle browser events correctly.
- Use event delegation deliberately.
- Manage event listener lifetimes.
- Implement forms.
- Validate user input.
- Manage URL state.
- Use browser storage appropriately.
- Implement network requests.
- Handle loading/error/empty states.
- Implement request cancellation.
- Handle race conditions.
- Avoid stale responses overwriting newer state.
- Design optimistic UI carefully.
- Design offline-friendly behavior.
- Understand cache semantics.
- Build accessible interactions.
- Manage focus.
- Manage keyboard navigation.
- Respect reduced-motion preferences.
- Avoid XSS.
- Avoid unsafe URL handling.
- Avoid DOM clobbering assumptions.
- Avoid insecure client-side authorization assumptions.
- Measure browser performance.
- Minimize unnecessary layout work.
- Use `requestAnimationFrame` appropriately.
- Avoid long main-thread tasks.
- Use Web Workers where justified.
- Optimize large lists.
- Test browser behavior.
- Test network failure.
- Test keyboard behavior.
- Test rendering/state synchronization.
- Debug production browser failures.
- Build progressively enhanced features.
- Design a versioned client architecture.
- Defend browser architecture decisions at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 22 — Arrays
Chapter 25 — Iterables / Iterators
Chapter 28 — JSON / Structured Clone
Chapter 29 — Errors
Chapter 31–38 — Async / Promises / Events / Streams / Cancellation
Chapter 49 — DOM Architecture
Chapter 50 — Browser Events
Chapter 51 — Browser APIs
Chapter 52 — Web Workers
Chapter 53 — Web Streams
Chapter 54 — Web Components
Chapter 55 — Fetch / HTTP
Chapter 56 — Browser Security
Chapter 57 — JavaScript Security Engineering
Chapter 71–73 — Data Structures / Complexity / Algorithms
Chapter 78–89 — Production / Testing / Debugging
Chapter 94 — Compatibility
Chapter 98–101 — Judgment
Chapter 102 — Production JavaScript CLI
```

---

# 4. Project Requirements

The application must support:

```text
projects
tasks
task status
filters
search
sort
task creation
task editing
task deletion
persistence
network synchronization
loading state
error state
empty state
keyboard interaction
responsive layout
accessible controls
```

Advanced targets:

```text
offline mode
optimistic updates
undo
background refresh
virtualized large lists
Web Worker analytics
```

---

# 5. Recommended Architecture

Use:

```text
Browser Platform
       ↓
Application Shell
       ↓
State
       ↓
Domain
       ↓
Services
       ↓
Renderers
       ↓
DOM
```

More concretely:

```text
src/
├── main.js
├── app/
│   ├── store.js
│   ├── state.js
│   ├── actions.js
│   └── selectors.js
├── domain/
│   ├── task.js
│   └── project.js
├── services/
│   ├── api.js
│   ├── storage.js
│   └── sync.js
├── ui/
│   ├── shell.js
│   ├── task-list.js
│   ├── task-form.js
│   ├── filters.js
│   └── notifications.js
├── platform/
│   ├── events.js
│   ├── focus.js
│   └── online.js
└── styles/
    └── app.css
```

---

# 6. Core Architecture Rule

Do not let every function directly mutate arbitrary DOM and global state.

Use an explicit flow:

```text
user interaction
→ action
→ state transition
→ render
```

For external input:

```text
network/storage
→ validate
→ domain state
→ render
```

---

# 7. Application State

Example:

```js
const initialState = {
  projects: [],
  tasks: [],
  selectedProjectId: null,
  filters: {
    status: "all",
    search: ""
  },
  request: {
    status: "idle",
    error: null
  }
};
```

Keep transient UI state distinct from durable domain data.

---

# 8. Durable vs Ephemeral State

Durable:

```text
tasks
projects
preferences
```

Ephemeral:

```text
loading
focused item
open dialog
hover
request status
```

Do not persist every UI detail.

---

# 9. State Ownership

For each field ask:

```text
Who owns it?
Who changes it?
Who reads it?
Who persists it?
```

Ambiguous ownership creates UI bugs.

---

# 10. State Transition

Prefer explicit actions:

```js
{
  type: "TASK_ADDED",
  task
}
```

rather than arbitrary hidden mutation.

---

# 11. Reducer-Style Transition

Example:

```js
function reduce(state, action) {
  switch (action.type) {
    case "TASK_ADDED":
      return {
        ...state,
        tasks: [
          ...state.tasks,
          action.task
        ]
      };

    default:
      return state;
  }
}
```

A reducer is a design choice, not a framework requirement.

---

# 12. Selectors

Keep derived state out of stored state where possible.

Example:

```js
function selectVisibleTasks(state) {
  return state.tasks
    .filter(...)
    .sort(...);
}
```

Be aware of repeated computation.

---

# 13. Derived State

Do not store:

```text
tasks
visibleTasks
taskCount
completedCount
```

unless there is a clear consistency/performance reason.

Derived values can drift.

---

# 14. Immutable State

A shallow immutable update can make transitions easier to reason about.

But do not clone the entire application graph for every keystroke without measuring.

---

# 15. Domain Invariants

Define:

```text
task must have ID
task title must be non-empty
project must exist
status must be valid
```

The UI is not the only place where invariants matter.

---

# 16. Input Validation

For a task title:

```js
function validateTitle(value) {
  const title = value.trim();

  if (title.length === 0) {
    return {
      ok: false,
      message: "Title is required"
    };
  }

  return {
    ok: true,
    value: title
  };
}
```

---

# 17. Rendering Strategy

Choose a clear model:

```text
state
→ render
```

or incremental updates.

For small/medium applications, clarity often wins.

For very large lists, measure the render cost.

---

# 18. Rendering Contract

A renderer should know:

```text
what it owns
what input it receives
what DOM it may change
```

Avoid unrelated DOM mutation.

---

# 19. Safe DOM Construction

Prefer:

```js
element.textContent = userText;
```

over injecting untrusted content with:

```js
element.innerHTML = userText;
```

---

# 20. HTML Templates

Static template markup can be defined in HTML:

```html
<template id="task-template">
  <li class="task">
    <span class="task-title"></span>
  </li>
</template>
```

Clone and populate safely.

---

# 21. DocumentFragment

For batch insertion:

```js
const fragment =
  document.createDocumentFragment();
```

Build nodes first, then append.

Do not assume this is always faster; profile important paths.

---

# 22. Event Ownership

A component that registers a listener should own its cleanup.

Possible lifecycle:

```text
mount
→ add listeners
→ update
→ unmount
→ remove listeners
```

---

# 23. Event Delegation

For a task list:

```js
list.addEventListener(
  "click",
  handleListClick
);
```

One delegated listener can handle task actions.

This can reduce registration overhead.

---

# 24. Event Delegation Boundaries

Do not create one global click handler for the entire application if unrelated components become coupled.

Delegate within meaningful ownership boundaries.

---

# 25. Event Listener Cleanup

Use:

```js
const controller =
  new AbortController();

element.addEventListener(
  "click",
  handler,
  { signal: controller.signal }
);
```

Then:

```js
controller.abort();
```

when the component is destroyed.

---

# 26. Event Object Ownership

Do not pass the full DOM Event deep into domain logic.

Convert it to application data:

```js
dispatch({
  type: "TASK_COMPLETED",
  taskId
});
```

---

# 27. Click vs Keyboard

Interactive elements should use semantic controls:

```html
<button>
```

rather than clickable arbitrary `div` elements when possible.

---

# 28. Forms

Use:

```html
<form>
```

and handle:

```js
form.addEventListener(
  "submit",
  onSubmit
);
```

This supports keyboard interaction and semantic browser behavior.

---

# 29. Prevent Default Deliberately

Use:

```js
event.preventDefault();
```

only when replacing a browser action intentionally.

---

# 30. Form Validation

Combine:

```text
HTML constraints
+
JavaScript validation
```

where appropriate.

Client validation improves UX.

Server validation remains necessary for security.

---

# 31. Focus Management

When opening a dialog:

```text
move focus into dialog
```

when appropriate.

When closing:

```text
restore focus to the invoking control
```

---

# 32. Dialog Design

Use the native `<dialog>` element where its support and behavior fit the target.

Define:

```text
open
close
Escape
focus
background interaction
```

---

# 33. Focus Trap

A modal dialog needs appropriate keyboard focus management.

Do not implement a complex focus trap blindly; understand the chosen dialog primitive and accessibility behavior.

---

# 34. Keyboard Navigation

Support:

```text
Tab
Enter
Space
Escape
Arrow keys
```

only where the control semantics require them.

---

# 35. ARIA Principle

Prefer native semantic HTML before adding ARIA.

Do not use ARIA to recreate:

```text
button
checkbox
dialog
```

when native elements already express the correct semantics.

---

# 36. Accessible Names

Every control needs an understandable accessible name.

Test:

```text
visual label
accessible name
keyboard reachability
screen reader semantics
```

---

# 37. Live Regions

Use `aria-live` carefully for:

```text
important asynchronous status
```

Do not announce every UI update.

---

# 38. Reduced Motion

Respect:

```css
@media (prefers-reduced-motion: reduce) {
  ...
}
```

Animation is a user preference, not merely a design choice.

---

# 39. Responsive Layout

Design around:

```text
content
viewport
input mode
```

rather than device names alone.

---

# 40. URL State

Search/filter state may be represented in the URL:

```text
/tasks?status=done&search=api
```

Benefits:

```text
shareability
refresh persistence
navigation
history
```

---

# 41. URL Encoding

Use platform URL APIs:

```js
const url =
  new URL(location.href);

url.searchParams.set(
  "search",
  search
);
```

Avoid manual string concatenation.

---

# 42. History API

Decide when to use:

```js
history.pushState(...)
```

versus:

```js
history.replaceState(...)
```

Push when a meaningful navigation state should create history.

Replace for incidental changes.

---

# 43. Back/Forward

Handle:

```js
window.addEventListener(
  "popstate",
  restoreFromUrl
);
```

The URL becomes part of the application state model.

---

# 44. Local Storage

Useful for:

```text
small preferences
small durable client state
```

It is synchronous and has limitations.

Do not treat it as a general database.

---

# 45. Storage Schema

Version local data:

```js
{
  version: 2,
  tasks: [...]
}
```

Future migrations become possible.

---

# 46. Storage Corruption

Treat storage as untrusted input.

Handle:

```text
invalid JSON
wrong schema
partial data
old version
```

---

# 47. IndexedDB

Use for larger structured client data where its transactional/storage model fits.

Do not add it merely because:

```text
“localStorage is old.”
```

Choose based on requirements.

---

# 48. Cache Storage

Cache Storage can support controlled HTTP-response caching patterns.

It is not automatically a replacement for application state storage.

---

# 49. Network Service

Create a boundary:

```js
const api = {
  listTasks,
  createTask,
  updateTask,
  deleteTask
};
```

UI should not know raw HTTP details.

---

# 50. Fetch Wrapper

Centralize concerns:

```text
base URL
headers
error mapping
authentication
timeouts/cancellation
JSON parsing
```

---

# 51. HTTP Error Handling

Do not treat all fetch fulfillment as business success.

Check:

```js
if (!response.ok) {
  throw ...
}
```

or handle status codes explicitly according to the API contract.

---

# 52. Network Error Categories

Distinguish:

```text
network failure
HTTP failure
application/domain failure
validation failure
cancellation
```

---

# 53. Cancellation

Use:

```js
const controller =
  new AbortController();

fetch(url, {
  signal: controller.signal
});
```

Cancel when:

```text
user changes search
component unmounts
request becomes irrelevant
```

---

# 54. Search Race Condition

User types:

```text
a
ap
api
```

Requests overlap.

Response order might be:

```text
api
→ a
→ ap
```

The stale response must not overwrite newer state.

---

# 55. Race Solution — Sequence ID

Track:

```js
const requestId =
  ++currentRequestId;
```

Accept a response only if it is still current.

---

# 56. Race Solution — Cancellation

Abort previous request when the new query supersedes it.

Prefer both:

```text
cancellation
+
stale-response guard
```

when the API requires strong correctness.

---

# 57. Debounce Search

Debouncing can reduce request volume.

Trade-off:

```text
fewer calls
+
more waiting
```

Tune to UX.

---

# 58. Retry Policy

Do not blindly retry every browser request.

Classify:

```text
safe to retry
not safe to retry
retry only on selected status/errors
```

---

# 59. Idempotency

For writes that may be retried, use a server-supported idempotency strategy where appropriate.

The browser cannot create true end-to-end idempotency alone.

---

# 60. Loading State

Represent:

```text
idle
loading
success
error
```

Do not infer loading from:

```text
tasks.length === 0
```

because empty and loading are different states.

---

# 61. Empty State

Distinguish:

```text
loading
empty
error
loaded with results
```

---

# 62. Error State

Give users:

```text
what happened
what they can do
retry
```

Avoid exposing internal stack traces.

---

# 63. Optimistic UI

Example:

```text
check task
→ UI updates immediately
→ request sent
```

Benefits:

```text
responsiveness
```

Costs:

```text
rollback
conflict
failure handling
```

---

# 64. Optimistic Update With Rollback

Keep enough information to reverse:

```text
previous state
```

or use an explicit operation model.

---

# 65. Undo

For destructive actions:

```text
delete
→ optimistic removal
→ undo window
```

Make the user-visible contract explicit.

---

# 66. Offline State

Monitor:

```js
window.addEventListener(
  "online",
  sync
);
```

and:

```js
window.addEventListener(
  "offline",
  showOffline
);
```

But browser connectivity signals are not proof that an arbitrary API is reachable.

---

# 67. Offline-First Boundary

Separate:

```text
local state
```

from:

```text
server synchronization state
```

This makes offline behavior easier to reason about.

---

# 68. Sync Queue

Potential model:

```js
{
  id,
  operation,
  payload,
  createdAt,
  attempts,
  status
}
```

Persist carefully.

---

# 69. Conflict Resolution

Two devices edit the same task.

Possible strategies:

```text
last-write-wins
server-authoritative
version check
merge
manual conflict
```

Domain decides.

---

# 70. Versioned Writes

A client can send:

```text
expectedVersion
```

The server rejects stale writes.

This is safer than silently overwriting.

---

# 71. Stale While Revalidate

A UX strategy:

```text
show cached data
→ fetch fresh data
→ replace when validated
```

Useful when stale data is acceptable.

---

# 72. Service Worker

Optional advanced feature:

```text
network interception
offline shell
cache strategy
```

Treat the service worker as its own lifecycle-managed program.

---

# 73. Service Worker Update Complexity

You must reason about:

```text
old worker
new worker
old clients
new assets
cache version
activation
```

Do not assume update is instantaneous.

---

# 74. Security — Never Trust the DOM

The DOM can contain:

```text
user-generated content
third-party content
server content
```

Treat data flow carefully.

---

# 75. Security — XSS

Unsafe:

```js
element.innerHTML = task.title;
```

Safer:

```js
element.textContent = task.title;
```

---

# 76. Security — URL Construction

Avoid:

```js
location.href =
  userControlledValue;
```

without validation.

Use `URL` and explicit allowlists when navigation targets are constrained.

---

# 77. Security — Open Redirect

Never blindly redirect to arbitrary user-provided URLs.

---

# 78. Security — Token Storage

Do not automatically put sensitive authentication material in:

```text
localStorage
```

Evaluate the threat model and server authentication design.

---

# 79. Security — Client Authorization

A hidden button is not authorization.

The server must enforce protected operations.

---

# 80. Security — Third-Party Scripts

Every script has capabilities granted by its execution context.

Minimize unnecessary third-party JavaScript.

---

# 81. Security — Content Security Policy

Use CSP as defense in depth.

Do not treat it as a replacement for safe DOM construction.

---

# 82. Security — Dependencies

Frontend dependencies affect:

```text
bundle
supply chain
runtime behavior
security
```

Keep the dependency graph intentional.

---

# 83. Performance — Measure First

Collect:

```text
startup
long tasks
interaction latency
network timing
memory
rendering
```

Use browser developer tooling.

---

# 84. Main Thread Budget

JavaScript, layout, style, and other work can compete for responsiveness.

Long synchronous tasks can create jank.

---

# 85. Long Task

A large synchronous computation:

```js
for (...) {
  expensiveWork();
}
```

can block interaction.

---

# 86. Chunking Work

Break large work into smaller units:

```text
process chunk
→ yield
→ process chunk
```

The exact scheduling API depends on the goal.

---

# 87. requestAnimationFrame

Use for work that needs to coordinate with visual frame updates.

Do not use it as a universal async queue.

---

# 88. requestIdleCallback

Useful for opportunistic noncritical work where supported.

Do not rely on idle execution for required persistence or correctness.

---

# 89. Web Worker

Move suitable CPU-heavy work to a Worker.

Trade-offs:

```text
parallel execution
vs
communication/transfer overhead
```

---

# 90. Large List Rendering

Rendering 100,000 DOM nodes is usually expensive.

Options:

```text
pagination
virtualization
windowing
incremental rendering
```

---

# 91. Virtualization

Render only visible rows.

Requires handling:

```text
scroll position
item size
focus
accessibility
measurement
```

---

# 92. Layout Thrashing

Avoid repeatedly interleaving:

```text
write DOM
read layout
write DOM
read layout
```

Batch reads and writes when this matters.

---

# 93. DOM Size

Large DOM trees can increase:

```text
style work
layout
memory
```

Keep the document structure purposeful.

---

# 94. Image Optimization

For image-heavy applications consider:

```text
appropriate dimensions
lazy loading
modern formats
responsive images
```

Do not optimize image bytes while ignoring JavaScript main-thread work.

---

# 95. Network Waterfall

Inspect:

```text
request order
dependencies
blocking
payload
cache
```

Eliminate unnecessary sequential requests.

---

# 96. Preload / Prefetch

Use resource hints intentionally.

Incorrect preloading can increase network contention.

---

# 97. Code Splitting

Dynamic import can defer code.

Trade-off:

```text
smaller initial work
later loading
```

Measure user-visible outcomes.

---

# 98. Accessibility Testing

Test:

```text
keyboard only
screen reader
zoom
high contrast where relevant
reduced motion
```

---

# 99. Browser Compatibility

Define actual targets:

```text
supported browser versions
mobile webviews if relevant
embedded contexts
```

Do not claim support you do not test.

---

# 100. Progressive Enhancement

Start with:

```text
semantic HTML
```

then enhance with JavaScript.

A feature should fail gracefully when optional capabilities are unavailable where practical.

---

# 101. Feature Detection

Prefer capability checks:

```js
if ("serviceWorker" in navigator) {
  ...
}
```

when capability is the actual requirement.

---

# 102. Graceful Degradation

For unavailable APIs:

```text
feature unavailable
→ useful fallback
```

or:

```text
clear unsupported message
```

---

# 103. Error Boundary Equivalent

The browser does not give a framework-level component boundary automatically.

Design subsystem boundaries:

```text
network
storage
rendering
analytics
```

so one failure does not destroy unrelated UI.

---

# 104. Global Error Handling

Use global error handlers as safety nets, not as the main error architecture.

Relevant signals include:

```text
error
unhandledrejection
```

---

# 105. Error Reporting

Send sanitized:

```text
error name
message
stack where appropriate
app version
route
browser info
```

Never send secrets blindly.

---

# 106. Source Maps

Use source maps for debugging.

Control exposure according to deployment/security requirements.

---

# 107. Client Versioning

Include an application version:

```text
1.4.0
```

in logs/telemetry.

This helps correlate incidents with deployments.

---

# 108. Telemetry Sampling

High-volume client telemetry can be expensive.

Use sampling and aggregation.

---

# 109. Privacy

Do not collect more user data than necessary.

Telemetry should be:

```text
purposeful
minimal
redacted
```

---

# 110. Testing Architecture

Test:

```text
domain
state transitions
rendering
events
network
storage
accessibility
```

at appropriate levels.

---

# 111. Unit Tests

Test pure logic:

```text
reducers
selectors
validators
formatters
domain rules
```

---

# 112. DOM Integration Tests

Test:

```text
click
submit
render
focus
keyboard
```

using an appropriate browser-like environment.

---

# 113. Browser E2E Tests

Test critical flows:

```text
create
edit
delete
refresh
offline
retry
```

---

# 114. Network Mocking

Test:

```text
success
404
500
timeout
abort
malformed response
```

Do not test only happy paths.

---

# 115. Race Testing

Create:

```text
slow old request
fast new request
```

Ensure the old response cannot overwrite current state.

---

# 116. Storage Testing

Test:

```text
missing data
corrupt data
old version
migration
quota failure
```

---

# 117. Accessibility Tests

Automate what can be automated.

Then manually verify:

```text
keyboard
focus
screen reader experience
```

---

# 118. Visual Regression

Use targeted screenshots for stable critical surfaces.

Do not snapshot the whole application indiscriminately.

---

# 119. Performance Tests

Measure:

```text
startup
interaction
large-list rendering
search
network synchronization
```

Use realistic data.

---

# 120. Production Failure Scenarios

Simulate:

```text
slow network
offline
server 500
expired auth
large response
slow rendering
storage corruption
service worker mismatch
```

---

# 121. Project Milestones

```text
Milestone 1 — HTML shell
Milestone 2 — state model
Milestone 3 — task rendering
Milestone 4 — events/forms
Milestone 5 — local persistence
Milestone 6 — API integration
Milestone 7 — URL state
Milestone 8 — cancellation/races
Milestone 9 — accessibility
Milestone 10 — security
Milestone 11 — performance
Milestone 12 — offline/sync
Milestone 13 — tests
Milestone 14 — production hardening
```

---

# 122. Implementation Progression

## Stage 1 — Guided

Build:

```text
static HTML
basic render
task add
task complete
```

## Stage 2 — Partially Guided

Add:

```text
state store
filters
search
storage
```

## Stage 3 — No Reference

Add:

```text
API
loading/error
URL state
```

## Stage 4 — Edge-Case Hardened

Add:

```text
races
abort
offline
storage migration
accessibility
security
```

## Stage 5 — Production Grade

Add:

```text
performance budgets
telemetry
tests
compatibility
release process
```

---

# 123. Track A — Core Theory

Study while implementing:

```text
DOM
events
event propagation
rendering
forms
storage
URL/history
Fetch
AbortSignal
service workers
workers
security
accessibility
performance
browser lifecycle
compatibility
```

---

# 124. Track B — Implementation

You must build:

```text
application shell
state store
renderers
event handlers
forms
storage adapter
API adapter
URL state
request manager
sync layer
accessibility behavior
error reporting
test suite
performance tests
```

---

# 125. Track C — Interview / Reasoning

Be able to defend:

```text
Why framework-free?
Why this state model?
Why event delegation?
Why textContent?
Why URL state?
Why localStorage vs IndexedDB?
Why cancellation?
Why optimistic UI?
Why Worker?
Why virtualization?
Why service worker?
Why not cache everything?
Why not render everything?
Why not put everything in global state?
```

---

# 126. Mastery Gate

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

---

# 127. Code Review Exercise

Review:

```js
function render(tasks) {
  container.innerHTML = tasks
    .map(task => `
      <li>
        <button data-id="${task.id}">
          ${task.title}
        </button>
      </li>
    `)
    .join("");
}
```

Identify:

```text
XSS risk
HTML-string complexity
DOM ownership
event listener strategy
escaping requirements
```

Then redesign safely.

---

# 128. Code Review Exercise

Review:

```js
searchInput.addEventListener(
  "input",
  async event => {
    const response =
      await fetch(
        `/search?q=${event.target.value}`
      );

    render(
      await response.json()
    );
  }
);
```

Identify:

```text
race condition
missing URL encoding
missing HTTP error handling
no cancellation
request amplification
```

---

# 129. Code Review Exercise

Review:

```js
localStorage.setItem(
  "tasks",
  JSON.stringify(tasks)
);
```

Questions:

```text
How large can tasks become?
What if storage fails?
What if schema changes?
What if data is corrupt?
What if sensitive data is stored?
```

---

# 130. Code Review Exercise

Review:

```js
button.addEventListener(
  "click",
  handle
);
```

inside a repeatedly executed render function.

Possible problem:

```text
listener multiplication
```

Choose a lifecycle-safe strategy.

---

# 131. Code Review Exercise

Review:

```js
if (!navigator.onLine) {
  return;
}
```

Question:

> Does this prove the API is reachable?

No.

Connectivity state and endpoint availability are different.

---

# 132. Interview Questions — Senior

1. How would you architect a framework-free browser application?
2. How do you prevent DOM/state divergence?
3. How do you handle async races?
4. How do you cancel stale requests?
5. How do you choose localStorage vs IndexedDB?
6. How do you design accessible interactions?
7. How do you avoid XSS?
8. How do you optimize large lists?
9. How do you move CPU work off the main thread?
10. How do you test browser failure modes?

---

# 133. Interview Questions — Principal

1. How would you design a browser application used by millions of users?
2. What state belongs in the URL?
3. What belongs in memory?
4. What belongs in durable client storage?
5. How would you design offline synchronization?
6. How would you handle conflicting edits?
7. How would you design client-side observability?
8. How would you control bundle and startup cost?
9. How would you define browser support policy?
10. How would you migrate a framework application to Web Platform primitives?

---

# 134. Mastery Exercise — Build From Scratch

Starting with:

```text
index.html
styles.css
src/main.js
```

build:

```text
project list
task list
create task
complete task
delete task
filter
search
```

Do not begin with a framework.

---

# 135. Mastery Exercise — State

Add:

```text
explicit state
actions
state transitions
selectors
```

Then create tests for transitions.

---

# 136. Mastery Exercise — Persistence

Add:

```text
save
load
version
migration
corruption handling
```

---

# 137. Mastery Exercise — API

Add:

```text
GET
POST
PATCH
DELETE
```

and model:

```text
loading
success
empty
error
```

---

# 138. Mastery Exercise — Race Conditions

Build a search endpoint simulator with controlled delays.

Prove that stale responses cannot overwrite newer state.

---

# 139. Mastery Exercise — Accessibility

Complete the feature using keyboard only.

Then verify:

```text
focus order
labels
semantics
dialogs
announcements
```

---

# 140. Mastery Exercise — Security

Test:

```text
HTML injection
javascript: URL
open redirect
unsafe storage
third-party script exposure
```

---

# 141. Mastery Exercise — Performance

Create:

```text
1,000
10,000
100,000
```

tasks.

Measure:

```text
render time
interaction latency
memory
```

Then choose a strategy.

---

# 142. Mastery Exercise — Worker

Move a genuinely CPU-heavy analytics operation to a Worker.

Measure:

```text
main-thread responsiveness
worker overhead
data transfer
```

---

# 143. Mastery Exercise — Offline

Disable network.

The application should:

```text
continue useful local work
record synchronization intent
recover later
```

Define conflict behavior.

---

# 144. Mastery Exercise — Service Worker

Implement:

```text
app-shell cache
versioned assets
update strategy
offline fallback
```

Test:

```text
old worker
new worker
cache mismatch
```

---

# 145. Mastery Exercise — Production Diagnostics

Add:

```text
app version
route
sanitized errors
performance timing
```

to telemetry.

---

# 146. Mastery Exercise — Progressive Enhancement

Build a meaningful baseline in HTML.

Then layer:

```text
navigation
dynamic rendering
network sync
```

on top.

---

# 147. Acceptance Criteria

The application is production-grade when:

```text
[ ] application state is explicit
[ ] DOM ownership is clear
[ ] events have lifecycle ownership
[ ] forms are accessible
[ ] keyboard interactions work
[ ] URL state is defined
[ ] storage is versioned
[ ] storage failures are handled
[ ] API errors are classified
[ ] requests can be canceled
[ ] stale responses are prevented
[ ] optimistic updates have recovery
[ ] offline behavior is explicit
[ ] XSS paths are controlled
[ ] authorization is server-side
[ ] main-thread work is measured
[ ] large lists have a strategy
[ ] critical flows are tested
[ ] browser compatibility is defined
[ ] observability is useful
[ ] privacy is considered
[ ] release process is documented
```

---

# 148. Production Checklist

```text
[ ] semantic HTML
[ ] CSS architecture
[ ] state model
[ ] DOM renderer
[ ] event strategy
[ ] form validation
[ ] focus management
[ ] accessibility
[ ] URL state
[ ] storage adapter
[ ] API adapter
[ ] cancellation
[ ] race protection
[ ] caching policy
[ ] offline behavior
[ ] security controls
[ ] performance budget
[ ] browser support matrix
[ ] tests
[ ] telemetry
[ ] release
```

---

# 149. Concept Connections

## Depends On

```text
Chapter 49 — DOM Architecture
Chapter 50 — Browser Events
Chapter 51 — Browser APIs
Chapter 52 — Workers
Chapter 53 — Streams
Chapter 55 — Fetch / HTTP
Chapter 56–57 — Browser / JS Security
Chapter 71–73 — Data Structures / Complexity
Chapter 78–89 — Production / Testing / Debugging
Chapter 94 — Compatibility
Chapter 98–101 — Judgment
```

## Builds Toward

```text
Chapter 104 — Production HTTP Client
Chapter 105 — Node REST API
Chapter 106 — Real-Time WebSocket
```

## Revisited

```text
DOM
events
Promises
AbortSignal
storage
HTTP
security
performance
testing
compatibility
observability
architecture
```

---

# 150. Spaced Retrieval Schedule

### Day 0

```text
DOM
state
events
render
```

### Day 1

```text
forms
storage
URL
Fetch
```

### Day 3

```text
cancellation
races
optimistic UI
```

### Day 7

```text
accessibility
security
performance
```

### Day 14

```text
offline
service workers
workers
compatibility
```

### Day 30

Rebuild the application shell.

### Day 60

Rebuild async/network architecture from scratch.

### Day 90

Defend the browser architecture in a principal review.

---

# 151. Revision / Retrieval Record

```md
# Chapter 103 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- DOM ownership [ ]
- State model [ ]
- Rendering [ ]
- Events [ ]
- Forms [ ]
- Focus [ ]
- Accessibility [ ]
- URL state [ ]
- Storage [ ]
- Fetch/API [ ]
- Cancellation [ ]
- Race conditions [ ]
- Optimistic UI [ ]
- Offline/sync [ ]
- Security [ ]
- Performance [ ]
- Workers [ ]
- Testing [ ]
- Observability [ ]
- Compatibility [ ]

## Build Evidence
- Repository:
- Commit:
- Browser matrix:
- Performance benchmark:
- Security test:
- Accessibility test:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 103 — Canonical References and Source Discipline

Primary references:

1. MDN Web APIs  
   https://developer.mozilla.org/en-US/docs/Web/API

2. MDN DOM  
   https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model

3. MDN Events  
   https://developer.mozilla.org/en-US/docs/Web/Events

4. MDN Fetch API  
   https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

5. MDN Web Storage API  
   https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API

6. MDN IndexedDB API  
   https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API

7. MDN Service Worker API  
   https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API

8. MDN Web Workers API  
   https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API

9. MDN Accessibility  
   https://developer.mozilla.org/en-US/docs/Web/Accessibility

10. WAI-ARIA Authoring Practices  
    https://www.w3.org/WAI/ARIA/apg/

11. ECMAScript Language Specification  
    https://tc39.es/ecma262/

12. OWASP  
    https://owasp.org/

Source discipline:

```text
language semantics
→ ECMAScript

browser APIs
→ Web Platform specifications + MDN

accessibility
→ WAI-ARIA/WCAG and platform semantics

security
→ OWASP + browser security model

performance
→ browser profiler + realistic workload

compatibility
→ actual support matrix + compatibility data
```

---

# 152. Completion Snapshot

```text
Part XX — Projects

Chapter 103 — Production Vanilla Browser App
[ ] Not Started

Track A — Core Theory
[ ] DOM architecture
[ ] state
[ ] rendering
[ ] events
[ ] event delegation
[ ] forms
[ ] validation
[ ] focus
[ ] accessibility
[ ] URL/history
[ ] localStorage
[ ] IndexedDB
[ ] Cache Storage
[ ] Fetch
[ ] errors
[ ] cancellation
[ ] races
[ ] retries
[ ] optimistic UI
[ ] offline
[ ] service workers
[ ] XSS
[ ] URLs
[ ] client/server trust
[ ] performance
[ ] Web Workers
[ ] compatibility
[ ] observability
[ ] privacy
[ ] testing

Track B — Implementation
[ ] app shell
[ ] state store
[ ] selectors
[ ] renderers
[ ] task form
[ ] filters
[ ] search
[ ] persistence
[ ] API layer
[ ] URL state
[ ] cancellation
[ ] race protection
[ ] optimistic updates
[ ] offline sync
[ ] accessibility
[ ] security hardening
[ ] large-list strategy
[ ] Worker
[ ] telemetry
[ ] tests
[ ] release

Track C — Interview / Reasoning
[ ] Defend state architecture
[ ] Defend rendering strategy
[ ] Defend event strategy
[ ] Defend storage choice
[ ] Defend URL state
[ ] Defend cancellation
[ ] Defend optimistic UI
[ ] Defend offline model
[ ] Defend security model
[ ] Defend performance approach
[ ] Defend browser support policy
[ ] Defend observability
[ ] Defend migration strategy

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

# 153. Completion Criteria

Do not mark this project mastered because the UI looks good.

You are ready to move forward when you can independently:

1. Explain the application architecture.
2. Separate domain state from DOM state.
3. Define state ownership.
4. Design predictable state transitions.
5. Render safely.
6. Handle events with lifecycle ownership.
7. Build accessible forms.
8. Manage focus.
9. Represent meaningful state in the URL.
10. Persist data with schema/version handling.
11. Handle storage failure.
12. Build a network boundary.
13. Classify HTTP/network/domain/cancellation failures.
14. Cancel stale requests.
15. Prevent stale responses from corrupting current state.
16. Design optimistic updates with rollback.
17. Define offline semantics.
18. Define conflict resolution.
19. Prevent XSS.
20. Keep authorization server-side.
21. Measure browser performance.
22. Reduce main-thread work.
23. Use Workers when justified.
24. Handle large lists.
25. Test critical flows.
26. Test browser failure modes.
27. Define compatibility targets.
28. Build useful client observability.
29. Protect user privacy.
30. Defend the architecture at principal level.

---

# Final Mental Model

```text
User
 ↓
Browser Event
 ↓
Action
 ↓
State Transition
 ↓
Derived State
 ↓
Render
 ↓
DOM
```

External systems:

```text
Network / Storage / Worker
          ↓
       Adapter
          ↓
      Validation
          ↓
        State
```

Cross-cutting:

```text
Security
Accessibility
Performance
Compatibility
Observability
```

The strongest browser architecture is not the one with the most abstractions.

It is the one where:

```text
state is understandable
DOM ownership is explicit
events have lifecycle ownership
network races are controlled
data is validated
security boundaries are clear
performance is measured
accessibility is intentional
failure is recoverable
```

> **Mastery reminder:** Build the application first as a browser system, not as a collection of UI tricks. Every platform boundary should have an explicit contract.