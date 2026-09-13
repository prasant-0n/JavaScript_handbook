# Chapter 50 — Browser Events and Event Propagation

> **Curriculum Position:** Part IX — Browser  
> **Prerequisites:** Chapters 41–49  
> **Primary Focus:** `EventTarget`, event dispatch, capture/target/bubble phases, event paths, cancellation, listener options, delegation, default actions, synthetic events, input events, Shadow DOM, event-loop integration, performance, memory, and security  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Web Platform semantics → dispatch algorithm → propagation → browser scheduling → production event architecture  
> **Scope Rule:** Browser events are Web Platform behavior. Do not confuse DOM event semantics with ECMAScript function calls or with one browser's private implementation.

---

## Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Is a Browser Event?](#3-what-is-a-browser-event)
- [4. Why Does the Event System Exist?](#4-why-does-the-event-system-exist)
- [5. Mental Model](#5-mental-model)
- [6. Core Rules](#6-core-rules)
- [7. Event Vocabulary](#7-event-vocabulary)
- [8. EventTarget and EventListener](#8-eventtarget-and-eventlistener)
- [9. Registering and Removing Listeners](#9-registering-and-removing-listeners)
- [10. Event Dispatch](#10-event-dispatch)
- [11. Capture, Target, and Bubble](#11-capture-target-and-bubble)
- [12. target vs currentTarget](#12-target-vs-currenttarget)
- [13. Event Path and Propagation](#13-event-path-and-propagation)
- [14. stopPropagation vs stopImmediatePropagation](#14-stoppropagation-vs-stopimmediatepropagation)
- [15. preventDefault and Cancelable Events](#15-preventdefault-and-cancelable-events)
- [16. Listener Options](#16-listener-options)
- [17. Event Delegation](#17-event-delegation)
- [18. Dynamic DOM and Delegation](#18-dynamic-dom-and-delegation)
- [19. Event Listener Identity](#19-event-listener-identity)
- [20. Listener Mutation During Dispatch](#20-listener-mutation-during-dispatch)
- [21. Event Constructors and Synthetic Events](#21-event-constructors-and-synthetic-events)
- [22. dispatchEvent](#22-dispatchevent)
- [23. Trusted vs Synthetic Events](#23-trusted-vs-synthetic-events)
- [24. Default Actions and Activation Behavior](#24-default-actions-and-activation-behavior)
- [25. Browser Event Loop Integration](#25-browser-event-loop-integration)
- [26. Microtasks and Event Handlers](#26-microtasks-and-event-handlers)
- [27. Pointer, Mouse, Keyboard, and Input Events](#27-pointer-mouse-keyboard-and-input-events)
- [28. Focus and Form Events](#28-focus-and-form-events)
- [29. Shadow DOM Event Propagation](#29-shadow-dom-event-propagation)
- [30. Composed Path and Retargeting](#30-composed-path-and-retargeting)
- [31. Event Ordering](#31-event-ordering)
- [32. Edge Cases](#32-edge-cases)
- [33. Common Misconceptions](#33-common-misconceptions)
- [34. Common Mistakes](#34-common-mistakes)
- [35. Comparison With Related Concepts](#35-comparison-with-related-concepts)
- [36. Performance Considerations](#36-performance-considerations)
- [37. Memory Considerations](#37-memory-considerations)
- [38. Security Considerations](#38-security-considerations)
- [39. Production Usage](#39-production-usage)
- [40. Implementation From Scratch](#40-implementation-from-scratch)
- [41. Debugging Exercises](#41-debugging-exercises)
- [42. Code Review Exercise](#42-code-review-exercise)
- [43. Interview Questions](#43-interview-questions)
- [44. Predict-the-Output Exercises](#44-predict-the-output-exercises)
- [45. Mastery Exercises](#45-mastery-exercises)
- [46. Key Takeaways](#46-key-takeaways)
- [47. Concept Connections](#47-concept-connections)
- [48. Completion Criteria](#48-completion-criteria)
- [49. Revision / Retrieval Record](#49-revision--retrieval-record)
- [50. Canonical References and Source Discipline](#50-canonical-references-and-source-discipline)
- [51. Completion Snapshot](#51-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define a browser event as a Web Platform object and dispatch operation.
2. Explain `EventTarget` and event listener registration.
3. Explain event dispatch as an algorithm rather than simply “call a callback.”
4. Explain capture, target, and bubble phases.
5. Distinguish `event.target` from `event.currentTarget`.
6. Explain the event path and its relationship to DOM and Shadow DOM trees.
7. Explain `stopPropagation()` and `stopImmediatePropagation()` precisely.
8. Explain `preventDefault()`, `cancelable`, and `defaultPrevented`.
9. Explain `capture`, `once`, `passive`, and `signal` listener options.
10. Explain listener identity and correct removal.
11. Explain listener additions/removals during dispatch.
12. Explain event delegation and its limits.
13. Explain `Event`, `CustomEvent`, and synthetic dispatch.
14. Explain `dispatchEvent()` and its return value.
15. Distinguish trusted browser events from script-created events.
16. Explain default actions and activation behavior.
17. Explain how browser events interact with tasks and the event loop.
18. Explain how event handlers can enqueue microtasks.
19. Compare pointer, mouse, keyboard, input, focus, and form event families.
20. Explain that not every event bubbles and not every event is cancelable.
21. Explain Shadow DOM composedness, retargeting, and `composedPath()`.
22. Diagnose common event bugs.
23. Diagnose listener/lifecycle memory problems.
24. Design event systems from first principles.
25. Make production event decisions using correctness, accessibility, performance, memory, security, reliability, and maintainability.

### Mastery target

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

Reading alone does not establish mastery.

---

# 2. Prerequisites

Required knowledge:

### Chapter 43 — Ordinary Object Internal Methods

- object identity
- properties and methods
- prototype behavior
- accessors

### Chapter 44 — Realms and Agents

- Realm
- Agent
- Window and Worker contexts

### Chapter 45 — Memory / GC

- reachability
- roots
- retention
- lifecycle

### Chapter 49 — DOM Architecture

- Node
- Element
- tree relationships
- Shadow DOM
- detached nodes

### Chapter 32 — Promise Reactions / Jobs

- Promise reactions
- microtasks

### Chapter 33 — Browser Event Loop

- tasks
- microtask checkpoints
- rendering coordination

### Chapter 37 — Cancellation

- AbortController
- AbortSignal

---

# 3. What Is a Browser Event?

A browser event is a Web Platform object representing an occurrence or notification that can be delivered through an `EventTarget`.

Examples:

```text
click
keydown
input
submit
focus
pointerdown
pointermove
load
```

A typical listener is:

```js
button.addEventListener("click", handler);
```

But the event system is larger than the callback:

```text
event source
    ↓
event object
    ↓
dispatch
    ↓
event path
    ↓
listener selection
    ↓
listener invocation
    ↓
default action / later browser work
```

Events can originate from:

- user interaction
- browser lifecycle activity
- application code
- resource activity
- platform state changes

Script can also create events.

---

# 4. Why Does the Event System Exist?

An event system avoids constant polling.

Without events, application code would repeatedly ask:

```text
Did the user click?
Did input change?
Did the window resize?
Did a resource load?
```

Instead:

```text
platform occurrence
      ↓
event
      ↓
dispatch
      ↓
registered behavior
```

The system provides:

- loose coupling
- multiple listeners
- propagation
- cancellation
- delegation
- component communication

---

# 5. Mental Model

Use this simplified model:

```text
                 Event occurs
                      ↓
                  Event object
                      ↓
                 dispatch begins
                      ↓
                  event path
                      ↓
              capture toward target
                      ↓
                   target phase
                      ↓
               bubble from target
                      ↓
              default action if any
```

The actual path can be more complex because of Shadow DOM and other platform concepts.

---

## 5.1 Event System as Routing

Think:

```text
target
  ↓
compute path
  ↓
select listeners
  ↓
invoke by phase
  ↓
apply propagation/cancellation state
```

---

# 6. Core Rules

## Rule 1 — Browser events are Web Platform behavior

`EventTarget` and the DOM event model are not core ECMAScript features.

## Rule 2 — Dispatch is not merely callback invocation

The platform determines the path, phase, listener eligibility, and dispatch state.

## Rule 3 — `target` and `currentTarget` differ

```text
target → event target as seen from the listener's context
currentTarget → current EventTarget whose listener is running
```

## Rule 4 — Capture runs toward the target

`capture: true` listeners participate during the capture phase.

## Rule 5 — Bubbling runs away from the target

For bubbling events, eligible ancestor listeners can run after the target phase.

## Rule 6 — Not every event bubbles

Check event semantics; do not assume delegation always works.

## Rule 7 — Not every event is cancelable

`preventDefault()` only has the intended cancellation effect for cancelable events and applicable default actions.

## Rule 8 — `stopPropagation()` and `preventDefault()` solve different problems

```text
stopPropagation → propagation
preventDefault  → default action
```

## Rule 9 — `stopImmediatePropagation()` is stronger

It also stops later listeners on the current target.

## Rule 10 — Listener identity matters

Equivalent-looking function source is not function identity.

## Rule 11 — `once` provides automatic removal

The listener is removed after invocation.

## Rule 12 — `AbortSignal` provides structured cleanup

Aborting the signal removes associated listeners.

## Rule 13 — `dispatchEvent()` is synchronous

Once dispatch starts, listeners execute during the call.

## Rule 14 — Browser scheduling and dispatch are different layers

A user interaction may be delivered through a task, but dispatch itself is a synchronous operation within the running task/work.

## Rule 15 — Synthetic events are not identical to trusted user interaction

Browser security and activation behavior can distinguish them.

## Rule 16 — Shadow DOM adds retargeting/composedness semantics

Do not model every event path as repeated `parentNode` traversal.

---

# 7. Event Vocabulary

## EventTarget

An object with an event listener list and dispatch API.

## Event

The base event interface.

Useful properties include:

```js
event.type
event.target
event.currentTarget
event.eventPhase
event.bubbles
event.cancelable
event.defaultPrevented
event.composed
```

## EventListener

A function or object with `handleEvent()`.

```js
const listener = {
  handleEvent(event) {
    console.log(event.type);
  }
};
```

## Event Path

The dispatch path through relevant invocation targets. Shadow-tree information can be part of this model.

## Event Phase

```text
NONE
CAPTURING_PHASE
AT_TARGET
BUBBLING_PHASE
```

## Bubbling

Propagation from the target toward ancestors where the event bubbles.

## Capture

Propagation toward the target.

## Cancelable

Whether cancellation through `preventDefault()` can affect the event's applicable default behavior.

## Default Action

Browser/platform behavior associated with an event, such as activation, navigation, submission, text editing, or scrolling.

## Trusted Event

An event whose trust state is established by the user agent rather than by ordinary script dispatch.

---

# 8. EventTarget and EventListener

The DOM Standard defines `EventTarget` with:

```js
addEventListener(type, callback, options)
removeEventListener(type, callback, options)
dispatchEvent(event)
```

It defines listener options including:

```text
capture
passive
once
signal
```

citeturn455992search1

Conceptually:

```text
EventTarget
  └── listener list
       ├── click / handlerA / capture=false
       ├── click / handlerB / capture=true
       └── input / handlerC / ...
```

Multiple listeners can exist for one event type.

---

# 9. Registering and Removing Listeners

## 9.1 Basic Registration

```js
function onClick(event) {
  console.log("clicked");
}

button.addEventListener("click", onClick);
```

## 9.2 Removal

```js
button.removeEventListener("click", onClick);
```

## 9.3 Identity

This does not remove the original listener:

```js
button.addEventListener("click", () => console.log("x"));
button.removeEventListener("click", () => console.log("x"));
```

The function expressions create different function objects.

## 9.4 Capture

Use the same capture semantics when removing:

```js
button.addEventListener("click", handler, { capture: true });
button.removeEventListener("click", handler, { capture: true });
```

## 9.5 Once

```js
button.addEventListener("click", handler, { once: true });
```

## 9.6 Signal

```js
const controller = new AbortController();

button.addEventListener("click", handler, {
  signal: controller.signal
});

controller.abort();
```

---

# 10. Event Dispatch

A conceptual dispatch process is:

```text
validate dispatch state
       ↓
set dispatch state
       ↓
construct event path
       ↓
capture processing
       ↓
target processing
       ↓
bubble processing
       ↓
finish dispatch
```

The specification algorithm is more detailed, especially around:

- shadow trees
- related targets
- slots
- closed roots
- listener mutation
- propagation flags

The DOM Standard defines an event path containing invocation-target and shadow-tree-related information. citeturn455992search1

---

# 11. Capture, Target, and Bubble

Consider:

```html
<div id="outer">
  <button id="inner">Click</button>
</div>
```

with:

```js
outer.addEventListener("click", () => {
  console.log("capture");
}, { capture: true });

inner.addEventListener("click", () => {
  console.log("target");
});

outer.addEventListener("click", () => {
  console.log("bubble");
});
```

For a bubbling click the typical order is:

```text
capture
target
bubble
```

A simplified path is:

```text
window
 ↓
document
 ↓
html
 ↓
body
 ↓
outer
 ↓
inner
```

Capture travels toward `inner`; bubbling travels back outward.

---

## 11.1 Target-Phase Subtlety

A capture listener registered directly on the event target can be involved in target processing because the event has reached the target.

Do not model target handling as simply “all capture listeners first, then all non-capture listeners” across every situation.

---

## 11.2 Non-Bubbling Events

Not every event bubbles. Common focus-related distinctions are important:

```text
focus / blur
focusin / focusout
```

Learn event-specific semantics rather than relying on a universal list.

---

# 12. target vs currentTarget

This is a core interview and debugging distinction.

```js
outer.addEventListener("click", event => {
  console.log(event.target.id);
  console.log(event.currentTarget.id);
});
```

If the button is clicked:

```text
target       → inner
currentTarget → outer
```

`target` represents the event target from the current context; `currentTarget` is the `EventTarget` whose listener is executing.

Delegation depends on this difference.

---

# 13. Event Path and Propagation

For ordinary light DOM, a simplified event path resembles:

```text
window
→ document
→ html
→ body
→ parent
→ target
```

But the DOM Standard's actual path model includes data for shadow-tree participation, adjusted targets, related targets, and closed-tree visibility. citeturn455992search1

Therefore:

```text
event path
≠ simply parentNode repeatedly
```

in the general case.

---

## 13.1 `composedPath()`

```js
event.composedPath();
```

returns the invocation-target path visible from the current context, with special handling for closed shadow trees. citeturn455992search1

---

# 14. stopPropagation vs stopImmediatePropagation

## `stopPropagation()`

```js
event.stopPropagation();
```

prevents the event from propagating onward to other objects in its path.

It does not automatically suppress all later listeners on the same target.

## `stopImmediatePropagation()`

```js
event.stopImmediatePropagation();
```

prevents later listener invocation on the current target and stops further propagation.

The DOM Standard explicitly distinguishes these behaviors. citeturn455992search1

---

## Example

```js
outer.addEventListener("click", first);
outer.addEventListener("click", second);

function first(event) {
  event.stopPropagation();
}
```

`second` can still run on the same `outer` target.

With:

```js
event.stopImmediatePropagation();
```

`second` will not run.

---

# 15. preventDefault and Cancelable Events

`preventDefault()` concerns default behavior, not propagation.

```js
link.addEventListener("click", event => {
  event.preventDefault();
});
```

For a cancelable event:

```js
event.defaultPrevented
```

can reflect that cancellation was requested.

---

## 15.1 dispatchEvent Return Value

The DOM Standard specifies that `dispatchEvent()` returns `false` when a cancelable event was canceled, otherwise `true`. citeturn455992search1

Example:

```js
button.addEventListener("custom", event => {
  event.preventDefault();
});

const event = new Event("custom", {
  cancelable: true
});

console.log(button.dispatchEvent(event));
// false
```

---

# 16. Listener Options

## `capture`

```js
node.addEventListener("click", handler, {
  capture: true
});
```

## `once`

```js
node.addEventListener("click", handler, {
  once: true
});
```

## `passive`

```js
node.addEventListener("wheel", handler, {
  passive: true
});
```

A passive listener promises not to cancel the event using `preventDefault()`. The DOM Standard defines the listener semantics, and MDN explains the practical scrolling-responsiveness rationale. citeturn455992search1turn455992search0

## `signal`

```js
const controller = new AbortController();

node.addEventListener("click", handler, {
  signal: controller.signal
});
```

Aborting the signal removes the associated listener.

## Combined

```js
node.addEventListener("click", handler, {
  capture: true,
  once: true,
  passive: true,
  signal: controller.signal
});
```

Choose options based on semantics, not habit.

---

# 17. Event Delegation

Event delegation uses an ancestor listener to handle descendant interactions.

```js
list.addEventListener("click", event => {
  const button = event.target.closest("button");
  if (!button) return;

  handleButton(button);
});
```

Why it works:

```text
button click
→ event bubbles
→ list receives event
→ target identifies originating descendant
```

---

## 17.1 Advantages

Delegation can reduce:

- listener count
- listener setup/teardown
- repeated registration for dynamic children

---

## 17.2 Trade-offs

It can add:

- selector/target resolution
- centralized logic
- shadow-boundary complexity
- coupling to bubbling behavior

One listener is not automatically the fastest design.

---

# 18. Dynamic DOM and Delegation

Delegation naturally handles dynamic children.

```js
list.addEventListener("click", event => {
  if (event.target.matches(".delete")) {
    removeItem(event.target);
  }
});
```

Later:

```js
const button = document.createElement("button");
button.className = "delete";
list.append(button);
```

No new listener is required on the new button.

---

## When delegation fails

Potential causes:

```text
non-bubbling event
propagation stopped
shadow boundary semantics
wrong delegation ancestor
wrong target matching
```

---

# 19. Event Listener Identity

Functions are objects.

These are distinct:

```js
() => console.log("x")
() => console.log("x")
```

So this does not remove the first listener:

```js
button.addEventListener("click", () => console.log("x"));
button.removeEventListener("click", () => console.log("x"));
```

Use a stable reference:

```js
const handler = () => console.log("x");

button.addEventListener("click", handler);
button.removeEventListener("click", handler);
```

An object listener can also provide stable identity:

```js
const listener = {
  handleEvent(event) {
    console.log(event.type);
  }
};
```

---

# 20. Listener Mutation During Dispatch

Do not model event dispatch as a naive JavaScript array loop.

If a listener is added or removed during dispatch, the DOM Standard's dispatch algorithm governs subsequent eligibility and invocation.

Example:

```js
button.addEventListener("click", first);

function first() {
  button.addEventListener("click", second);
}
```

Do not assume `second` behaves exactly as though it had been registered before dispatch.

Similarly:

```js
function first() {
  button.removeEventListener("click", second);
}
```

can affect later listener invocation during that dispatch according to the platform's processing rules.

For correctness-critical code, reason from the standard rather than intuition.

---

# 21. Event Constructors and Synthetic Events

## Event

```js
const event = new Event("custom", {
  bubbles: true,
  cancelable: true
});
```

## CustomEvent

```js
const event = new CustomEvent("user-selected", {
  bubbles: true,
  detail: { id: 42 }
});
```

Then:

```js
target.dispatchEvent(event);
```

---

## `detail`

Application-defined data can be carried in:

```js
event.detail
```

This is useful for component-level protocols.

---

# 22. dispatchEvent

```js
target.dispatchEvent(event);
```

runs dispatch synchronously.

Example:

```js
console.log("before");
target.dispatchEvent(new Event("x"));
console.log("after");
```

A listener runs between:

```text
before
handler

after
```

The DOM Standard defines `dispatchEvent()` to return whether the event ended up canceled. citeturn455992search1

---

# 23. Trusted vs Synthetic Events

Script-created events are not equivalent to real user interaction.

```js
const event = new Event("click");
button.dispatchEvent(event);
```

This does not automatically reproduce every consequence of an actual trusted click.

`event.isTrusted` reflects the trust state assigned by the user agent.

Important:

```text
synthetic event
≠
trusted user activation
```

Do not build authorization around event trust.

---

# 24. Default Actions and Activation Behavior

An event can be associated with a browser default action.

Examples include:

```text
link navigation
form submission
button activation
text editing
scrolling
```

The conceptual flow is:

```text
event dispatch
    ↓
listener can request cancellation
    ↓
default action may or may not occur
```

---

## 24.1 Link Example

```js
link.addEventListener("click", event => {
  event.preventDefault();
});
```

This can prevent the applicable navigation default action.

---

## 24.2 Form Example

```js
form.addEventListener("submit", event => {
  event.preventDefault();
  validateAndSubmitManually();
});
```

---

## 24.3 Event vs Activation

A click event is not itself the entire activation process.

Browser activation behavior may involve additional element-specific platform algorithms.

---

# 25. Browser Event Loop Integration

The HTML Standard states that browser event loops coordinate events, user interaction, scripts, rendering, networking, and related work, and that each agent has an associated event loop. citeturn455992search2

It also defines task sources, including a user interaction task source for user-input-related tasks. citeturn455992search2

---

## 25.1 Event Task vs Dispatch

Keep these layers separate:

```text
event-loop scheduling
→ decides when work executes

dispatch
→ decides which listeners execute and in what phase
```

A user interaction can result in a task during which event dispatch occurs.

---

## 25.2 Not Every Event Is a Separate Task

The HTML Standard notes that events are often dispatched by tasks, but some are dispatched during other tasks. citeturn455992search2

Therefore:

```text
one event = one separate task
```

is not a universal rule.

---

# 26. Microtasks and Event Handlers

Consider:

```js
button.addEventListener("click", () => {
  console.log("handler");

  Promise.resolve().then(() => {
    console.log("microtask");
  });
});
```

The handler executes synchronously as part of dispatch.

The Promise reaction is queued as a microtask.

Conceptually:

```text
event handler
   ↓
queue Promise reaction
   ↓
handler returns
   ↓
microtask checkpoint
   ↓
Promise reaction runs
```

A long synchronous event handler can delay both subsequent event processing and rendering opportunities.

---

# 27. Pointer, Mouse, Keyboard, and Input Events

## Mouse

Examples:

```text
mousedown
mouseup
mousemove
click
dblclick
```

## Pointer

Pointer Events unify common pointer sources:

```text
mouse
pen
touch
```

Examples:

```text
pointerdown
pointermove
pointerup
```

## Keyboard

```text
keydown
keyup
```

Do not treat raw keyboard events as equivalent to final text input.

## Input

```text
beforeinput
input
```

important for editing/input state.

## Composition

International text input can involve:

```text
compositionstart
compositionupdate
compositionend
```

Therefore:

```text
key press
≠
character input
```

in every language/input method.

---

# 28. Focus and Form Events

Important events include:

```text
focus
blur
focusin
focusout
input
change
submit
reset
invalid
```

---

## 28.1 Focus

`focus` and `blur` have different propagation semantics from bubbling focus-related events such as `focusin` and `focusout`.

---

## 28.2 Input vs Change

```text
input
→ live editing/input changes

change
→ control-specific committed change semantics
```

Do not assume identical timing across every form control.

---

## 28.3 Submit

Form submission has its own event/default-action relationship.

---

# 29. Shadow DOM Event Propagation

With:

```text
Document
  ↓
custom-element host
  ↓
ShadowRoot
  ↓
button
```

an event can cross the shadow boundary according to its composedness and the DOM event-path rules.

---

## 29.1 Composed Events

```js
event.composed
```

is relevant to whether an event crosses shadow boundaries in propagation.

---

## 29.2 Retargeting

An outside listener may observe:

```text
event.target === host
```

although the internal node that originated the event was deeper inside the shadow tree.

This provides encapsulation semantics.

---

# 30. Composed Path and Retargeting

Use:

```js
event.composedPath()
```

when debugging event paths through Shadow DOM.

The DOM Standard defines `composedPath()` with filtering behavior around closed shadow trees. citeturn455992search1

---

## 30.1 Why `target.parentNode` Is Not Enough

This:

```js
event.target.parentNode
```

only describes DOM-tree relationships from the target you can see.

It does not replace the platform's full event-path model.

---

# 31. Event Ordering

Event ordering depends on the event family, input device, focus state, default actions, and platform.

Do not memorize one universal sequence.

A common pointer/mouse interaction can contain sequences resembling:

```text
pointerdown
mousedown
...
pointerup
mouseup
click
```

but correctness-critical application logic should rely on the event semantics it actually needs.

---

## 31.1 Focus + Click

A click interaction can involve:

```text
pointer input
focus changes
mouse compatibility events
click
activation/default behavior
```

The exact sequence depends on the target and browser behavior.

---

## 31.2 Keyboard Activation

A keyboard event can trigger element activation and default behavior in addition to the raw keyboard event itself.

---

## 31.3 Pointer Capture

Pointer capture can change where subsequent pointer events are delivered during drag-style interactions.

This matters for robust gesture implementations.

---

# 32. Edge Cases

## 32.1 Non-Bubbling Events

Delegation can fail.

## 32.2 Non-Cancelable Events

`preventDefault()` may have no effect.

## 32.3 Passive Listeners

`preventDefault()` cannot cancel from a passive listener.

## 32.4 Same-Target Listeners

`stopPropagation()` does not mean “stop every listener here.”

## 32.5 Listener Mutation

Adding/removing during dispatch follows the dispatch algorithm.

## 32.6 `once` + Throw

A once listener does not become persistent merely because the callback throws.

## 32.7 Abort During Dispatch

Changing listener eligibility while dispatch is active requires specification-level reasoning.

## 32.8 Synthetic Click

A script-created click does not automatically reproduce user activation/security semantics.

## 32.9 `return false`

Native DOM listeners do not treat returning `false` as a universal shortcut for canceling and stopping propagation.

## 32.10 Shadow DOM

`target`, path visibility, and composedness can differ inside and outside a shadow boundary.

## 32.11 `currentTarget` Outside Dispatch

Do not use it as persistent state after the listener invocation context is gone.

## 32.12 High-Frequency Events

Pointer/scroll-related event streams can become extremely expensive when handlers perform heavy work.

---

# 33. Common Misconceptions

## Misconception 1 — “An event is a callback.”

No. It is an object plus dispatch/progression semantics.

## Misconception 2 — “Every event bubbles.”

False.

## Misconception 3 — “Every event is cancelable.”

False.

## Misconception 4 — “preventDefault stops propagation.”

False.

## Misconception 5 — “stopPropagation stops all listeners.”

Not on the same target.

## Misconception 6 — “target is always the listener's node.”

False; use `currentTarget` for the executing listener's target.

## Misconception 7 — “dispatchEvent is the same as a user click.”

False.

## Misconception 8 — “Events are always asynchronous.”

False; `dispatchEvent()` is synchronous.

## Misconception 9 — “Every event has its own task.”

False. Events can also be dispatched during other work. citeturn455992search2

## Misconception 10 — “Adding the same callback twice always means two callbacks.”

Listener registration has identity/option rules.

## Misconception 11 — “A passive listener is asynchronous.”

No. Passive is about cancellation behavior.

## Misconception 12 — “Shadow DOM hides all events.”

No. Composedness and retargeting determine what crosses the boundary.

## Misconception 13 — “isTrusted proves authorization.”

No.

## Misconception 14 — “Delegation is always faster.”

No. It is a trade-off.

---

# 34. Common Mistakes

1. Using `target` where `currentTarget` is required.
2. Removing listeners with a new function object.
3. Calling `preventDefault()` in a passive listener.
4. Using `stopPropagation()` when same-target listener suppression is required.
5. Assuming a particular input event order without verification.
6. Delegating a non-bubbling event.
7. Ignoring shadow boundaries.
8. Doing expensive synchronous work on pointer/input events.
9. Forgetting listener teardown.
10. Treating `isTrusted` as authentication.
11. Registering global listeners for behavior owned by a small component.
12. Assuming event dispatch and event-loop scheduling are the same layer.

---

# 35. Comparison With Related Concepts

| Concept | Purpose | Main scheduling/dispatch model |
|---|---|---|
| DOM Event | Platform notification | Dispatch through EventTarget |
| Event Listener | Handle an event | Invoked during dispatch |
| Promise reaction | Async continuation | Microtask/job |
| Timer callback | Delayed work | Task/timer processing |
| MutationObserver | DOM mutation reporting | Observer delivery |
| Event loop task | Unit of browser work | Selected by event loop |
| CustomEvent | Application-defined event | Normal event dispatch |

---

## Event vs Promise

```text
Event
→ EventTarget
→ path/propagation
→ listeners
```

```text
Promise
→ settlement
→ reaction job
→ microtask checkpoint
```

---

## Event vs Callback

```text
callback = executable code

event = data/state participating in a dispatch protocol
```

---

## Event vs Observer

MutationObserver is not bubbling event propagation.

---

# 36. Performance Considerations

## 36.1 Handler Duration

Long handlers can block progress:

```text
event handler
→ JS execution
→ no normal progress until it returns
```

This matters especially for:

```text
pointermove
wheel
touchmove
keydown
input
```

---

## 36.2 Passive Listeners

Where the handler never needs to cancel the default action, passive listeners can improve responsiveness for applicable scrolling/input scenarios. MDN documents the practical benefit. citeturn455992search0

---

## 36.3 Delegation

Potential benefit:

```text
many listeners
→ one ancestor listener
```

Potential cost:

```text
per-event target lookup/matching
```

---

## 36.4 High-Frequency Input

Instead of doing expensive work for every pointer event:

```js
let scheduled = false;

surface.addEventListener("pointermove", () => {
  if (scheduled) return;

  scheduled = true;

  requestAnimationFrame(() => {
    scheduled = false;
    updateUI();
  });
});
```

This is a scheduling strategy, not a language requirement.

---

## 36.5 Event Storms

Watch for:

```text
event
→ state update
→ DOM mutation
→ observer
→ more work
→ more events
```

Measure frequency and handler duration.

---

## 36.6 DOM + Event Interaction

A handler that performs:

```text
DOM write
→ layout read
```

can introduce rendering synchronization. See Chapter 49.

---

# 37. Memory Considerations

## 37.1 Listener Retention

A listener closure can retain application state.

```js
button.addEventListener("click", () => {
  use(largeState);
});
```

If the node/listener lifecycle is wrong, the captured state may remain reachable.

---

## 37.2 Detached DOM + Listener

Removing a node does not necessarily make it collectible:

```text
JS reference
or listener/closure graph
→ keeps it reachable
```

---

## 37.3 AbortController

Structured cleanup:

```js
const controller = new AbortController();

window.addEventListener("resize", onResize, {
  signal: controller.signal
});

button.addEventListener("click", onClick, {
  signal: controller.signal
});

// teardown
controller.abort();
```

---

## 37.4 Delegation

Delegation can lower listener count but does not eliminate the need for lifecycle management of the delegated ancestor.

---

## 37.5 WeakRef Is Not Listener Cleanup

Do not replace explicit lifecycle cleanup with weak-reference folklore.

---

# 38. Security Considerations

## 38.1 Synthetic Events

A malicious script can dispatch events.

Do not treat receipt of an event as proof of user intent.

---

## 38.2 `isTrusted`

`isTrusted` is a browser event-trust concept, not server-side authorization.

---

## 38.3 User Activation

Some browser capabilities require real user activation. Synthetic dispatch does not automatically grant equivalent privileges.

---

## 38.4 Global Keyboard Listeners

Avoid unnecessary capture of sensitive keystrokes.

---

## 38.5 Cross-Origin Frames

Events do not turn unrelated cross-origin documents into one shared DOM. Cross-context communication should use proper platform APIs such as `postMessage()` with origin validation.

---

## 38.6 Handler Injection

Prefer:

```js
element.addEventListener("click", safeHandler);
```

over string-based executable handlers:

```js
element.setAttribute("onclick", untrustedString);
```

---

# 39. Production Usage

## 39.1 Component Lifecycle

Use:

```text
mount
→ register
→ operate
→ unmount
→ cleanup
```

---

## 39.2 Local Ownership

Attach a listener to the smallest stable owner that naturally owns the behavior.

Avoid turning every event into:

```js
document.addEventListener(...)
```

---

## 39.3 Structured Cleanup

Use `AbortController` where multiple listeners share one lifecycle.

---

## 39.4 Delegation

Good candidates include:

```text
large dynamic lists
tables
menus
toolbars
```

when the relevant events bubble.

---

## 39.5 Accessibility

Prefer semantic controls:

```html
<button>Save</button>
```

over:

```html
<div onclick="save()">Save</div>
```

A mouse listener does not automatically provide:

- focusability
- keyboard activation
- accessible semantics

---

## 39.6 High-Frequency Work

For high-frequency input:

```text
capture input
→ coalesce/schedule
→ perform minimal UI work
```

---

## 39.7 Production Decision Framework

| Dimension | Question |
|---|---|
| Correctness | Does the intended event reach the correct owner? |
| Accessibility | Does keyboard/focus behavior remain correct? |
| Performance | Is handler/event work measured? |
| Memory | Are listeners and closures released? |
| Security | Can synthetic input reach sensitive behavior? |
| Reliability | Does behavior survive dynamic DOM changes? |
| Maintainability | Is event ownership clear? |
| Scalability | Does it hold under event frequency and DOM size? |
| Observability | Can latency/frequency be measured? |
| Operational Complexity | Is delegation/scheduling complexity justified? |
| Future Change | Will component/browser changes invalidate assumptions? |

---

# 40. Implementation From Scratch

Build a toy event system using your Chapter 49 DOM.

## Stage 1 — EventTarget

Represent:

```text
listener list
```

with entries like:

```js
{
  type,
  callback,
  capture,
  once,
  passive,
  signal
}
```

## Stage 2 — Event

Implement:

```text
type
bubbles
cancelable
defaultPrevented
target
currentTarget
eventPhase
```

## Stage 3 — Tree

Reuse:

```text
parent
children
```

## Stage 4 — Path

Given a target, produce:

```text
root → target
```

## Stage 5 — Capture

Invoke capture listeners on the path toward the target.

## Stage 6 — Target

Implement target-phase listener semantics.

## Stage 7 — Bubble

For bubbling events:

```text
target → root
```

## Stage 8 — Propagation Flags

Implement:

```text
stopPropagation()
stopImmediatePropagation()
```

## Stage 9 — Cancellation

Implement:

```text
preventDefault()
defaultPrevented
```

## Stage 10 — once

Remove listener after invocation.

## Stage 11 — signal

Abort removes associated listeners.

## Stage 12 — Delegation

Add a toy `closest()` and build a delegated handler.

## Stage 13 — Shadow Boundary

Model:

```text
host
shadow root
internal node
```

and experiment with:

```text
composed
retargeting
path visibility
```

## Stage 14 — Event Loop

Build:

```text
task queue
microtask queue
```

and demonstrate:

```text
event task
→ handler
→ queue microtask
→ microtask
→ next task
```

---

# 41. Debugging Exercises

## Exercise 1 — Capture order

Build nested elements and predict capture/target/bubble before executing.

## Exercise 2 — target/currentTarget

```js
outer.addEventListener("click", event => {
  console.log(event.target.id);
  console.log(event.currentTarget.id);
});
```

Click an inner button. Predict.

## Exercise 3 — stopPropagation

Two listeners on the same target. Have the first call `stopPropagation()`. Does the second run?

## Exercise 4 — stopImmediatePropagation

Repeat using `stopImmediatePropagation()`. Compare.

## Exercise 5 — cancelation

Dispatch a cancelable synthetic event and inspect the return value of `dispatchEvent()`.

## Exercise 6 — listener identity

Compare named/stable callbacks with separate arrow expressions.

## Exercise 7 — delegation

Log `target` and `currentTarget` on a list ancestor.

## Exercise 8 — passive

Try `preventDefault()` from a passive listener and inspect the resulting behavior.

## Exercise 9 — once

Click a `once` listener multiple times.

## Exercise 10 — AbortSignal

Abort a controller and verify the listener is gone.

## Exercise 11 — microtask ordering

Combine:

```text
handler
Promise.then
queueMicrotask
setTimeout
```

and predict ordering.

## Exercise 12 — Shadow DOM

Compare:

```text
target
currentTarget
composed
composedPath()
```

inside and outside a shadow root.

---

# 42. Code Review Exercise

Review:

```js
document.addEventListener("click", event => {
  if (event.target.matches(".delete-user")) {
    deleteUser(event.target.dataset.id);
  }
});
```

Developer claim:

> “Global delegation is always best because one event listener is the most performant solution.”

### Review Questions

```text
1. What is the natural owner of this behavior?
2. Does the event need document-wide scope?
3. How frequently does the event fire?
4. What does target matching cost?
5. Are shadow boundaries involved?
6. Does global scope create hidden coupling?
7. How is teardown handled?
8. Does the event bubble?
9. Is the target guaranteed to support matches()?
10. Is the underlying control accessible?
```

A localized alternative may be:

```js
const list = document.querySelector("#user-list");

list.addEventListener("click", event => {
  const button = event.target.closest(".delete-user");
  if (!button) return;

  deleteUser(button.dataset.id);
});
```

The principal goal is not minimum listener count. It is:

```text
correct ownership
+ lifecycle clarity
+ reasonable performance
```

---

# 43. Interview Questions

## Foundational

1. What is a browser event?
2. What is EventTarget?
3. What is event dispatch?
4. What are capture, target, and bubble phases?
5. What is event propagation?
6. What is the difference between target and currentTarget?
7. What does bubbles mean?
8. What does cancelable mean?

## Intermediate

9. What is stopPropagation?
10. What is stopImmediatePropagation?
11. What does preventDefault do?
12. What does passive mean?
13. What does once mean?
14. What does signal mean on addEventListener?
15. What is listener identity?
16. What is event delegation?
17. What is a synthetic event?
18. What does dispatchEvent return?

## Advanced

19. Why does an event path exist?
20. Why is `target.parentNode` not a complete event-path model?
21. How does Shadow DOM affect event propagation?
22. What is retargeting?
23. What is composedPath?
24. Why are trusted and synthetic events different?
25. How can an event handler interact with the event loop?
26. Why can passive listeners help scrolling responsiveness?
27. Why can long input handlers create jank?

## Principal-level

28. How would you decide between delegation and local listeners?
29. How would you design listener cleanup for a component system?
30. How would you diagnose an input-latency regression?
31. How would you debug a listener memory leak?
32. How would you reason about events crossing nested Shadow DOM boundaries?
33. Why is `isTrusted` not authorization?
34. How would you measure whether delegation improved a real production page?
35. How would you reason about browser-version changes affecting event/input behavior?

---

# 44. Predict-the-Output Exercises

## Exercise 1 — Propagation Order

```js
outer.addEventListener("click", () => console.log("capture"), {
  capture: true
});

inner.addEventListener("click", () => console.log("target"));

outer.addEventListener("click", () => console.log("bubble"));
```

Assume a bubbling click on `inner`.

### Prediction

```text
capture
target
bubble
```

---

## Exercise 2 — target/currentTarget

With:

```text
outer
 └── inner
```

and:

```js
outer.addEventListener("click", event => {
  console.log(event.target.id);
  console.log(event.currentTarget.id);
});
```

### Prediction for a click on inner

```text
inner
outer
```

---

## Exercise 3 — stopPropagation

```js
outer.addEventListener("click", event => {
  console.log("one");
  event.stopPropagation();
});

outer.addEventListener("click", () => {
  console.log("two");
});
```

### Prediction

```text
one
two
```

---

## Exercise 4 — stopImmediatePropagation

Replace the call with:

```js
event.stopImmediatePropagation();
```

### Prediction

```text
one
```

---

## Exercise 5 — once

```js
button.addEventListener("click", () => {
  console.log("clicked");
}, { once: true });
```

Click three times.

### Prediction

```text
clicked
```

---

## Exercise 6 — dispatchEvent

```js
button.addEventListener("custom", () => {
  console.log("handler");
});

console.log("before");
button.dispatchEvent(new Event("custom"));
console.log("after");
```

### Prediction

```text
before
handler
after
```

---

## Exercise 7 — Cancelation

```js
button.addEventListener("custom", event => {
  event.preventDefault();
});

const event = new Event("custom", { cancelable: true });

console.log(button.dispatchEvent(event));
```

### Prediction

```text
false
```

---

## Exercise 8 — Microtask

```js
button.addEventListener("click", () => {
  console.log("handler");
  Promise.resolve().then(() => console.log("microtask"));
});
```

### Prediction

```text
handler
microtask
```

with the microtask executing at the appropriate microtask checkpoint after the handler's synchronous work.

---

## Exercise 9 — Boolean Thinking About Events

Explain why these are separate questions:

```text
does the event bubble?
does it cancel?
is it trusted?
does it have a default action?
```

Do not collapse them into a single “event type” property.

---

# 45. Mastery Exercises

## Level 1 — Propagation Map

Draw:

```text
window
→ document
→ html
→ body
→ parent
→ target
```

and label capture/target/bubble.

## Level 2 — Cancellation

Explain separately:

```text
stopPropagation
stopImmediatePropagation
preventDefault
```

## Level 3 — Delegation

Build a 1,000-row dynamic table with one delegated listener and compare it to per-row listeners.

Measure:

```text
setup time
memory
handler time
teardown
```

## Level 4 — Listener Lifecycle

Build a component with:

```text
click
keydown
resize
pointermove
```

sharing one AbortController. Destroy it and prove listeners are cleaned up.

## Level 5 — Shadow DOM

Create a host, shadow root, and internal button. Record:

```text
target
currentTarget
composed
composedPath()
```

from inside and outside where the event's semantics permit.

## Level 6 — Input Performance

Compare:

```text
heavy work every pointermove
```

against:

```text
coalesced work via requestAnimationFrame
```

and profile responsiveness.

## Level 7 — Default Actions

Test cancelation of relevant:

```text
link click
form submit
input interactions
```

Record:

```text
cancelable
defaultPrevented
actual default behavior
```

## Level 8 — Event Loop

Create a test involving:

```text
click handler
queueMicrotask
Promise.then
setTimeout
```

Predict, then verify ordering.

## Level 9 — Delegation Failure

Create a delegated listener that fails because of:

```text
non-bubbling event
stopPropagation
Shadow DOM
```

Diagnose each case.

## Level 10 — Principal Event Architecture

Design events for:

```text
large dashboard
many widgets
dynamic lists
keyboard navigation
drag interaction
nested components
Shadow DOM
high-frequency pointer input
```

Defend ownership, delegation, cancellation, accessibility, scheduling, security, and cleanup.

---

# 46. Key Takeaways

1. Browser events are Web Platform objects plus a dispatch model.
2. `EventTarget` manages listeners and dispatch.
3. Dispatch computes/selects an event path and processes listeners by phase.
4. Capture, target, and bubble are distinct concepts.
5. `target` and `currentTarget` are not interchangeable.
6. Not every event bubbles.
7. Not every event is cancelable.
8. `stopPropagation()` affects path propagation.
9. `stopImmediatePropagation()` also stops later listeners on the current target.
10. `preventDefault()` concerns default action.
11. `passive` constrains cancellation.
12. `once` provides automatic listener cleanup.
13. `signal` provides structured listener lifecycle.
14. Listener identity matters when removing handlers.
15. Event delegation relies on propagation and target inspection.
16. `dispatchEvent()` is synchronous.
17. Synthetic events are not automatically equivalent to trusted user interaction.
18. Event-loop scheduling and event dispatch are different layers.
19. Handlers can queue microtasks.
20. Long handlers can create input/rendering jank.
21. Pointer, keyboard, input, focus, and form events have different semantics.
22. Shadow DOM adds composedness and retargeting.
23. `composedPath()` is valuable for event-path debugging.
24. Event listeners are also memory/lifecycle resources.
25. `isTrusted` is not authorization.
26. Delegation is a trade-off, not a universal performance law.
27. Production event architecture must include accessibility, security, performance, memory, and teardown.

---

# 47. Concept Connections

## Depends On

```text
Chapter 43 — Ordinary Object Methods
        ↓
Chapter 44 — Realms / Agents
        ↓
Chapter 45 — Memory / GC
        ↓
Chapter 49 — DOM Architecture
        ↓
Chapter 32 — Promise Reactions
        ↓
Chapter 33 — Browser Event Loop
        ↓
Chapter 37 — Cancellation
        ↓
Chapter 50 — Browser Events
```

## Builds Toward

```text
Chapter 51 — Browser APIs
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams / Data Flow
Chapter 54 — Web Components
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JS Security Engineering
Chapter 70 — Production Debugging
Chapter 83 — Observability
Chapter 85 — Performance
```

## Related Concepts

- DOM tree
- EventTarget
- Web IDL
- Shadow DOM
- Pointer Events
- keyboard/input events
- forms
- focus
- accessibility
- AbortController
- MutationObserver
- requestAnimationFrame
- browser event loop

## Concepts Revisited

### Chapter 49 — DOM Architecture

The DOM provides the tree/context through which many events propagate.

### Chapter 32 — Promise Reactions

Event handlers can schedule microtasks:

```text
event
→ handler
→ Promise reaction
```

### Chapter 33 — Browser Event Loop

The event loop determines when browser tasks execute; dispatch determines listener routing.

### Chapter 37 — Cancellation

AbortSignal provides structured cancellation for listener lifecycle.

### Chapter 45 — Memory

Listeners and closures can participate in retention graphs.

## Why This Chapter Matters Later

The next browser chapters depend on understanding events as a platform routing system, not merely as callback syntax. This is foundational for Web APIs, Web Components, networking events, concurrency, security, and production UI architecture.

---

# 48. Completion Criteria

## Theory

- [ ] Define Event.
- [ ] Define EventTarget.
- [ ] Explain listener lists.
- [ ] Explain dispatch.
- [ ] Explain event path.
- [ ] Explain capture.
- [ ] Explain target phase.
- [ ] Explain bubble.
- [ ] Explain target/currentTarget.
- [ ] Explain bubbles.
- [ ] Explain cancelable.
- [ ] Explain default action.

## Propagation

- [ ] Explain stopPropagation.
- [ ] Explain stopImmediatePropagation.
- [ ] Explain propagation order.
- [ ] Explain delegation.
- [ ] Explain listener mutation during dispatch.

## Listener Lifecycle

- [ ] Explain identity.
- [ ] Explain once.
- [ ] Explain passive.
- [ ] Explain signal.
- [ ] Remove listeners correctly.
- [ ] Design teardown.

## Synthetic Events

- [ ] Construct Event.
- [ ] Construct CustomEvent.
- [ ] Dispatch synthetic events.
- [ ] Explain dispatchEvent return value.
- [ ] Explain trusted vs synthetic events.

## Shadow DOM

- [ ] Explain composedness.
- [ ] Explain retargeting.
- [ ] Explain composedPath.
- [ ] Diagnose shadow-boundary delegation.

## Event Loop

- [ ] Explain events vs tasks.
- [ ] Explain handler vs microtask.
- [ ] Explain long-handler jank.
- [ ] Explain high-frequency input scheduling.

## Performance

- [ ] Measure handler duration.
- [ ] Measure event frequency.
- [ ] Evaluate delegation.
- [ ] Use passive appropriately.
- [ ] Coalesce high-frequency work.

## Memory

- [ ] Diagnose listener retention.
- [ ] Diagnose detached DOM retention.
- [ ] Use AbortController for lifecycle cleanup.

## Security

- [ ] Explain trusted vs synthetic events.
- [ ] Explain user activation.
- [ ] Avoid event-handler injection.
- [ ] Do not treat event trust as authorization.

## Principal Judgment

- [ ] Design event ownership.
- [ ] Defend delegation vs local listeners.
- [ ] Defend high-frequency scheduling.
- [ ] Design lifecycle-safe events.
- [ ] Handle Shadow DOM correctly.
- [ ] Include accessibility and security in event architecture.

---

# 49. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What is EventTarget?
2. What happens during dispatch?
3. What are capture/target/bubble phases?
4. What is the difference between target and currentTarget?
5. What does stopPropagation do?
6. What does stopImmediatePropagation do?
7. What does preventDefault do?
8. What is the difference between bubbles and cancelable?
9. What do once/passive/signal mean?
10. Why does listener identity matter?
11. What is delegation?
12. What is a synthetic event?
13. What does dispatchEvent return?
14. Why are trusted and synthetic events different?
15. How do event tasks relate to dispatch?
16. How can a handler enqueue a microtask?
17. Why can long handlers create jank?
18. Why is focus different from ordinary bubbling events?
19. What is composedness?
20. What is retargeting?
21. What does composedPath provide?
22. Why is isTrusted not authorization?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Event | [ ] | [ ] | [ ] | [ ] |
| EventTarget | [ ] | [ ] | [ ] | [ ] |
| Listener identity | [ ] | [ ] | [ ] | [ ] |
| Dispatch | [ ] | [ ] | [ ] | [ ] |
| Event path | [ ] | [ ] | [ ] | [ ] |
| Capture | [ ] | [ ] | [ ] | [ ] |
| Target phase | [ ] | [ ] | [ ] | [ ] |
| Bubble | [ ] | [ ] | [ ] | [ ] |
| target/currentTarget | [ ] | [ ] | [ ] | [ ] |
| stopPropagation | [ ] | [ ] | [ ] | [ ] |
| stopImmediatePropagation | [ ] | [ ] | [ ] | [ ] |
| preventDefault | [ ] | [ ] | [ ] | [ ] |
| cancelable | [ ] | [ ] | [ ] | [ ] |
| passive | [ ] | [ ] | [ ] | [ ] |
| once | [ ] | [ ] | [ ] | [ ] |
| AbortSignal | [ ] | [ ] | [ ] | [ ] |
| delegation | [ ] | [ ] | [ ] | [ ] |
| dispatchEvent | [ ] | [ ] | [ ] | [ ] |
| trusted/synthetic | [ ] | [ ] | [ ] | [ ] |
| default action | [ ] | [ ] | [ ] | [ ] |
| event loop integration | [ ] | [ ] | [ ] | [ ] |
| microtasks | [ ] | [ ] | [ ] | [ ] |
| pointer/input events | [ ] | [ ] | [ ] | [ ] |
| focus/form events | [ ] | [ ] | [ ] | [ ] |
| composed | [ ] | [ ] | [ ] | [ ] |
| retargeting | [ ] | [ ] | [ ] | [ ] |
| composedPath | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw capture → target → bubble.

### Day 2

Explain target vs currentTarget and all propagation-stopping methods.

### Day 7

Explain delegation and its limits.

### Day 14

Explain trusted/synthetic events, default actions, and user activation.

### Day 30

Design a production event architecture for a large dynamic application.

---

# 50. Canonical References and Source Discipline

## Primary DOM Standard

### WHATWG DOM Standard

https://dom.spec.whatwg.org/

Use for:

- Event
- EventTarget
- event listeners
- listener options
- event dispatch
- event phases
- event path
- propagation
- `preventDefault`
- `stopPropagation`
- `stopImmediatePropagation`
- `composedPath`
- synthetic dispatch

The current DOM Standard explicitly defines `Event`, event phases, event paths, `EventTarget`, listener options, and `dispatchEvent()` semantics. citeturn455992search1

---

## Primary HTML Standard

### WHATWG HTML Standard

https://html.spec.whatwg.org/

Use for:

- event loops
- tasks/task sources
- user interaction tasks
- activation behavior
- form behavior
- browser scheduling

The HTML Standard states that event loops coordinate events, user interaction, scripts, rendering, and networking, and defines a user-interaction task source for user input. citeturn455992search2

---

## Practical Reference

### MDN — `addEventListener()`

https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener

Use for practical details on:

```text
capture
once
passive
signal
listener identity
```

MDN also explains the responsiveness rationale behind passive listeners for relevant scrolling/input cases. citeturn455992search0

---

## Related Standards

### Pointer Events

https://w3c.github.io/pointerevents/

Use for pointer input and capture behavior.

### UI Events

https://w3c.github.io/uievents/

Use for event families such as keyboard/mouse behavior.

### HTML

https://html.spec.whatwg.org/

Use for HTML-specific activation, forms, focus, and event-loop integration.

---

## Source Classification

Label statements as:

```text
[DOM Standard]
[HTML Standard]
[Pointer Events]
[UI Events]
[Browser-specific]
[Measured]
[Historical]
```

Examples:

```text
"dispatchEvent is synchronous"
→ [DOM Standard]

"user interaction is integrated with task sources"
→ [HTML Standard]

"this browser performs input handling in subsystem X"
→ [Browser-specific]

"this handler took 8 ms"
→ [Measured]
```

Do not turn implementation observations into language guarantees.

---

## Version-Sensitivity

When investigating input/event performance record:

```text
browser
browser version
OS
input device
framework/library
DOM size
event type
handler workload
profiling method
```

---

## Important Layer Separation

Keep these distinct:

```text
event source
      ↓
event-loop scheduling
      ↓
event object
      ↓
dispatch/path
      ↓
listeners
      ↓
default action
      ↓
later rendering/network/browser work
```

They interact, but they are not one mechanism.

---

# 51. Completion Snapshot

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

- [ ] Event
- [ ] EventTarget
- [ ] listener
- [ ] listener identity
- [ ] dispatch
- [ ] event path
- [ ] capture
- [ ] target phase
- [ ] bubble
- [ ] target/currentTarget
- [ ] bubbles
- [ ] cancelable
- [ ] default action
- [ ] preventDefault
- [ ] stopPropagation
- [ ] stopImmediatePropagation
- [ ] once
- [ ] passive
- [ ] AbortSignal
- [ ] event delegation
- [ ] synthetic events
- [ ] dispatchEvent
- [ ] trusted events
- [ ] task/event-loop integration
- [ ] microtasks
- [ ] pointer events
- [ ] keyboard/input events
- [ ] focus/form events
- [ ] Shadow DOM propagation
- [ ] composed
- [ ] retargeting
- [ ] composedPath

### I can predict

- [ ] capture/target/bubble order
- [ ] target vs currentTarget
- [ ] propagation stopping
- [ ] default prevention
- [ ] once behavior
- [ ] synchronous dispatch
- [ ] microtask placement after handlers
- [ ] delegation success/failure
- [ ] shadow-boundary target behavior

### I can implement

- [ ] EventTarget
- [ ] listener registry
- [ ] add/remove
- [ ] event object
- [ ] path creation
- [ ] capture
- [ ] target
- [ ] bubble
- [ ] propagation flags
- [ ] cancellation
- [ ] once
- [ ] AbortSignal integration
- [ ] delegation
- [ ] toy event loop

### I can debug

- [ ] target/currentTarget errors
- [ ] propagation errors
- [ ] default-action errors
- [ ] listener identity errors
- [ ] delegation errors
- [ ] Shadow DOM event errors
- [ ] long-handler jank
- [ ] listener lifecycle leaks

### I can defend

- [ ] local listeners vs delegation
- [ ] passive-listener decisions
- [ ] listener lifecycle design
- [ ] scheduling strategies
- [ ] trusted/synthetic event security
- [ ] Shadow DOM event architecture
- [ ] accessibility-aware event design

---

## Final Principal-Level Test

Explain this statement without notes:

> **A browser event is not merely a callback notification. It is a Web Platform object processed by a dispatch algorithm that constructs an event path, selects listeners by propagation phase and listener options, tracks propagation/cancellation state, and may interact with a default action. Browser input is additionally integrated with the event loop, while Shadow DOM adds composed paths and retargeting.**

Your explanation is complete only when you can connect:

```text
browser/platform occurrence
        ↓
possible task/event-loop scheduling
        ↓
event object
        ↓
event path
        ↓
capture
        ↓
target
        ↓
bubble
        ↓
listener side effects
        ↓
microtasks when queued
        ↓
default action / later browser work
```

and clearly separate:

```text
event object
```

from:

```text
dispatch
propagation
listener callback
cancellation
default action
event-loop scheduling
```

The central mastery target of Chapter 50 is to stop thinking of browser events as “functions the browser calls” and start thinking of them as a specification-defined routing/lifecycle system integrated with the DOM, event loop, input model, Shadow DOM, accessibility, performance, memory, and security.