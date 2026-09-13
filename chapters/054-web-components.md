# Chapter 54 — Web Components and Shadow DOM

> **Curriculum Position:** Part IX — Browser  
> **Prerequisites:** Chapters 49–53  
> **Primary Focus:** Web Components, Custom Elements, Shadow DOM, templates, slots, lifecycle callbacks, encapsulation, styling, events, forms, accessibility, scoped registries, declarative shadow DOM, testing, performance, memory, security, and production component architecture  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Web Platform semantics → component model → DOM integration → lifecycle → composition → style/event boundaries → accessibility → production architecture  
> **Important Scope Rule:** Web Components are a set of Web Platform capabilities. The core technologies include **Custom Elements**, **Shadow DOM**, and **HTML templates/slots**. They are standardized independently from ECMAScript itself.

---

## Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Are Web Components?](#3-what-are-web-components)
- [4. Why Web Components Exist](#4-why-web-components-exist)
- [5. Mental Model](#5-mental-model)
- [6. Core Rules](#6-core-rules)
- [7. The Three Core Technologies](#7-the-three-core-technologies)
- [8. Custom Elements](#8-custom-elements)
- [9. Autonomous Custom Elements](#9-autonomous-custom-elements)
- [10. Customized Built-In Elements](#10-customized-built-in-elements)
- [11. CustomElementRegistry](#11-customelementregistry)
- [12. Element Upgrade and Definition Timing](#12-element-upgrade-and-definition-timing)
- [13. Custom Element Lifecycle](#13-custom-element-lifecycle)
- [14. Constructor Rules](#14-constructor-rules)
- [15. connectedCallback and disconnectedCallback](#15-connectedcallback-and-disconnectedcallback)
- [16. adoptedCallback](#16-adoptedcallback)
- [17. observedAttributes and attributeChangedCallback](#17-observedattributes-and-attributechangedcallback)
- [18. Shadow DOM](#18-shadow-dom)
- [19. Shadow Host and Shadow Root](#19-shadow-host-and-shadow-root)
- [20. Open vs Closed Shadow Roots](#20-open-vs-closed-shadow-roots)
- [21. Shadow Tree Encapsulation](#21-shadow-tree-encapsulation)
- [22. CSS Encapsulation](#22-css-encapsulation)
- [23. Styling a Shadow Tree](#23-styling-a-shadow-tree)
- [24. CSS Custom Properties Across Boundaries](#24-css-custom-properties-across-boundaries)
- [25. `::part` and Exposed Styling Hooks](#25-part-and-exposed-styling-hooks)
- [26. `::slotted()`](#26-slotted)
- [27. Templates](#27-templates)
- [28. `<slot>` and Content Projection](#28-slot-and-content-projection)
- [29. Named Slots](#29-named-slots)
- [30. Default Slots and Fallback Content](#30-default-slots-and-fallback-content)
- [31. Slot Assignment](#31-slot-assignment)
- [32. `slotchange`](#32-slotchange)
- [33. Light DOM vs Shadow DOM](#33-light-dom-vs-shadow-dom)
- [34. Composed Tree](#34-composed-tree)
- [35. Events and Shadow Boundaries](#35-events-and-shadow-boundaries)
- [36. Focus and `delegatesFocus`](#36-focus-and-delegatesfocus)
- [37. Forms and Custom Elements](#37-forms-and-custom-elements)
- [38. Form-Associated Custom Elements](#38-form-associated-custom-elements)
- [39. ElementInternals](#39-elementinternals)
- [40. Accessibility Architecture](#40-accessibility-architecture)
- [41. Declarative Shadow DOM](#41-declarative-shadow-dom)
- [42. Scoped Custom Element Registries](#42-scoped-custom-element-registries)
- [43. Web Components and Frameworks](#43-web-components-and-frameworks)
- [44. Web Components and SSR/Hydration](#44-web-components-and-ssrhydration)
- [45. Component Contracts](#45-component-contracts)
- [46. Lifecycle and Resource Management](#46-lifecycle-and-resource-management)
- [47. Performance Considerations](#47-performance-considerations)
- [48. Memory Considerations](#48-memory-considerations)
- [49. Security Considerations](#49-security-considerations)
- [50. Testing and Debugging](#50-testing-and-debugging)
- [51. Edge Cases](#51-edge-cases)
- [52. Common Misconceptions](#52-common-misconceptions)
- [53. Common Mistakes](#53-common-mistakes)
- [54. Comparison With Related Concepts](#54-comparison-with-related-concepts)
- [55. Production Usage](#55-production-usage)
- [56. Implementation From Scratch](#56-implementation-from-scratch)
- [57. Debugging Exercises](#57-debugging-exercises)
- [58. Code Review Exercise](#58-code-review-exercise)
- [59. Interview Questions](#59-interview-questions)
- [60. Predict-the-Output Exercises](#60-predict-the-output-exercises)
- [61. Mastery Exercises](#61-mastery-exercises)
- [62. Key Takeaways](#62-key-takeaways)
- [63. Concept Connections](#63-concept-connections)
- [64. Completion Criteria](#64-completion-criteria)
- [65. Revision / Retrieval Record](#65-revision--retrieval-record)
- [66. Canonical References and Source Discipline](#66-canonical-references-and-source-discipline)
- [67. Completion Snapshot](#67-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define Web Components precisely.
2. Explain the roles of Custom Elements, Shadow DOM, templates, and slots.
3. Distinguish Web Components from frameworks such as React, Vue, and Angular.
4. Define autonomous custom elements.
5. Explain customized built-in elements and their browser-support considerations.
6. Explain `customElements.define()`.
7. Explain the Custom Element registry and upgrade process.
8. Explain what happens when custom markup is parsed before its definition is registered.
9. Explain lifecycle callbacks:
   - constructor
   - `connectedCallback`
   - `disconnectedCallback`
   - `adoptedCallback`
   - `attributeChangedCallback`
10. Explain constructor restrictions for custom elements.
11. Explain `observedAttributes`.
12. Explain Shadow DOM.
13. Explain shadow hosts and shadow roots.
14. Explain open vs closed roots.
15. Explain style encapsulation.
16. Explain CSS inheritance across a shadow boundary.
17. Explain CSS custom properties as component theming channels.
18. Explain `::part`.
19. Explain `::slotted()`.
20. Explain `<template>`.
21. Explain `<slot>`.
22. Explain named slots and default slot behavior.
23. Explain slot assignment and fallback content.
24. Explain `slotchange`.
25. Distinguish light DOM, shadow DOM, and composed tree.
26. Explain event retargeting and composed events across shadow boundaries.
27. Explain `delegatesFocus`.
28. Explain form-associated custom elements.
29. Explain `ElementInternals`.
30. Explain accessibility responsibilities for custom components.
31. Explain Declarative Shadow DOM.
32. Explain scoped custom element registries conceptually.
33. Explain interaction with SSR and hydration.
34. Define good component contracts.
35. Design lifecycle-safe custom elements.
36. Diagnose style leakage, event boundary bugs, slot bugs, registry collisions, and teardown leaks.
37. Compare Web Components with framework components.
38. Implement a component model from first principles.
39. Defend a production Web Components architecture across correctness, accessibility, performance, memory, security, compatibility, and maintainability.

### Mastery target

Progress through:

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

Reading alone does not establish mastery.

---

# 2. Prerequisites

## Chapter 49 — DOM Architecture

Required:

```text
Node
Element
Document
DocumentFragment
tree
shadow tree
attributes
properties
```

## Chapter 50 — Browser Events

Required:

```text
EventTarget
event path
propagation
retargeting
composedPath
```

## Chapter 51 — Browser Web APIs

Required for:

```text
browser lifecycle
AbortController
observers
templates
platform objects
```

## Chapter 52 — Web Workers / Concurrency

Useful for component architectures that offload expensive computation.

## Chapter 53 — Web Streams

Useful for components that consume incremental data.

## Chapter 43 — Ordinary Objects

Needed because custom elements are JavaScript-visible objects with platform-defined lifecycle behavior.

---

# 3. What Are Web Components?

**Web Components** are a family of standardized Web Platform capabilities for building reusable custom HTML elements with encapsulated structure and behavior.

The core technologies are:

```text
Custom Elements
Shadow DOM
HTML templates / slots
```

MDN describes custom elements as JavaScript APIs for defining custom elements and their behavior, Shadow DOM as an encapsulated DOM tree, and `<template>`/`<slot>` as reusable markup/composition primitives. citeturn327281search0turn327281search1turn327281search2

---

## 3.1 Minimal Example

```js
class UserCard extends HTMLElement {
  connectedCallback() {
    this.textContent = "User";
  }
}

customElements.define("user-card", UserCard);
```

HTML:

```html
<user-card></user-card>
```

---

## 3.2 With Shadow DOM

```js
class UserCard extends HTMLElement {
  constructor() {
    super();

    const shadow = this.attachShadow({
      mode: "open"
    });

    shadow.innerHTML = `
      <style>
        :host {
          display: block;
        }
      </style>

      <strong>Prasanta</strong>
    `;
  }
}

customElements.define("user-card", UserCard);
```

---

# 4. Why Web Components Exist

Web Components address problems such as:

```text
reusable native elements
isolation
composition
framework-independent UI components
DOM-integrated lifecycle
```

They let component authors define:

```text
public HTML interface
+
behavior
+
internal structure
+
styling
+
lifecycle
```

without requiring an external framework.

---

## 4.1 Native-Looking API

A component can be used as:

```html
<user-card
  user-id="42"
  theme="compact">
</user-card>
```

The consuming document interacts with it through:

```text
DOM
attributes
properties
events
slots
methods
```

---

## 4.2 Framework Independence

A Web Component can be consumed by:

```text
plain JavaScript
React
Vue
Angular
Svelte
server-rendered HTML
```

provided the integration layer handles the relevant framework/component semantics.

---

# 5. Mental Model

Use:

```text
                  <user-card>
                       │
                Custom Element
                       │
                ┌──────┴──────┐
                ↓             ↓
            light DOM     shadow root
                               │
                          shadow tree
                          ├── style
                          ├── header
                          ├── slot
                          └── button
```

---

## 5.1 Component Boundary

A component exposes:

```text
public contract
```

and encapsulates:

```text
implementation
```

---

## 5.2 Public Surface

Think:

```text
attributes
properties
methods
events
slots
CSS custom properties
parts
```

---

## 5.3 Internal Surface

Think:

```text
shadow elements
internal listeners
internal state
internal DOM structure
private rendering logic
```

---

# 6. Core Rules

## Rule 1 — Web Components are Web Platform technology

They are not synonymous with JavaScript frameworks.

## Rule 2 — Custom Element names must follow custom-element naming rules

Autonomous custom element names contain a hyphen, such as:

```text
user-card
date-picker
```

## Rule 3 — Register once per registry/name

A registry cannot redefine an already-registered name in the same registry.

## Rule 4 — Definitions can occur after elements are parsed

The platform can represent an undefined custom element and later upgrade it when a definition becomes available.

## Rule 5 — Custom element lifecycle is platform-managed

Do not treat callbacks as ordinary arbitrary function calls.

## Rule 6 — Constructors have restrictions

Do not perform arbitrary DOM lifecycle assumptions inside the constructor.

## Rule 7 — `connectedCallback` can run multiple times

A component can be removed and reinserted.

## Rule 8 — `disconnectedCallback` is not necessarily permanent destruction

The element may be reconnected.

## Rule 9 — Attribute changes are observable only for attributes listed in `observedAttributes`

Do not expect every attribute mutation to call `attributeChangedCallback`.

## Rule 10 — Shadow DOM creates a distinct tree

Light DOM descendants are not simply the same tree as shadow descendants.

## Rule 11 — Closed Shadow DOM is not a security boundary

It hides the `shadowRoot` property from ordinary page code, but it is not a secret vault.

## Rule 12 — Shadow CSS is encapsulated but not absolutely isolated

Inheritance, custom properties, slots, and explicit styling mechanisms cross boundaries in defined ways.

## Rule 13 — Slots compose trees; they do not copy nodes

A slotted node keeps its identity.

## Rule 14 — `::slotted()` styles direct slotted children, not arbitrary deep descendants

Understand the exact selector boundary.

## Rule 15 — `::part()` is an explicit styling API

Use it to expose selected internal styling hooks.

## Rule 16 — Component APIs should be explicit

Prefer:

```text
properties
attributes
methods
events
slots
```

over hidden DOM assumptions.

## Rule 17 — Accessibility is the component author's responsibility

A custom element does not automatically inherit the complete semantics of the native element it resembles.

## Rule 18 — Custom elements should preserve semantic HTML where possible

A custom `<user-card>` is not automatically a button.

## Rule 19 — Lifecycle cleanup must be explicit

Remove/disconnect:

```text
listeners
observers
timers
channels
workers
```

when ownership ends.

## Rule 20 — Browser support must be verified for advanced features

Scoped registries, declarative Shadow DOM options, and other newer features can have evolving support.

---

# 7. The Three Core Technologies

## 7.1 Custom Elements

Define new HTML element behavior.

```js
customElements.define("user-card", UserCard);
```

---

## 7.2 Shadow DOM

Attach an encapsulated tree:

```js
this.attachShadow({ mode: "open" });
```

---

## 7.3 HTML Templates / Slots

Templates:

```html
<template>
  ...
</template>
```

provide reusable markup.

Slots:

```html
<slot></slot>
```

allow light-DOM content to be projected into a shadow tree.

MDN describes these three technologies as the central pieces of Web Components and specifically identifies `<template>` and `<slot>` as mechanisms for reusable and composable component markup. citeturn327281search0turn327281search1

---

# 8. Custom Elements

A custom element is an element whose behavior is defined by the application.

Example:

```js
class StatusBadge extends HTMLElement {
  connectedCallback() {
    this.textContent = "Ready";
  }
}

customElements.define("status-badge", StatusBadge);
```

---

## 8.1 Why Extend `HTMLElement`

For autonomous custom elements:

```js
class StatusBadge extends HTMLElement
```

provides the normal HTML-element base interface.

---

## 8.2 Custom Element Name

```text
status-badge
user-card
product-table
```

The hyphen helps distinguish custom names from standard HTML elements.

---

## 8.3 HTML Usage

```html
<status-badge></status-badge>
```

---

## 8.4 JavaScript Creation

```js
const badge = document.createElement("status-badge");
```

---

# 9. Autonomous Custom Elements

An autonomous custom element has its own custom tag name.

Example:

```js
class UserCard extends HTMLElement {}

customElements.define("user-card", UserCard);
```

Use:

```html
<user-card></user-card>
```

---

## 9.1 Advantages

The component owns its semantics and implementation.

---

## 9.2 Responsibility

Because the custom element is not automatically a semantic replacement for:

```text
button
input
dialog
```

the component author must provide proper semantics/accessibility behavior.

---

# 10. Customized Built-In Elements

The platform also supports extending certain built-in elements using:

```js
class FancyButton extends HTMLButtonElement {
  connectedCallback() {
    // ...
  }
}

customElements.define(
  "fancy-button",
  FancyButton,
  {
    extends: "button"
  }
);
```

HTML:

```html
<button is="fancy-button"></button>
```

---

## 10.1 Support Caveat

Customized built-in elements have historically had more limited browser support than autonomous custom elements.

Verify your actual browser matrix before making them foundational to production architecture.

---

## 10.2 Architectural Recommendation

Autonomous custom elements are often easier to integrate consistently:

```html
<fancy-button></fancy-button>
```

but they require explicit semantic/accessibility design.

---

# 11. CustomElementRegistry

The registry maps names to custom element definitions.

```js
customElements.define("user-card", UserCard);
```

---

## 11.1 Lookup

```js
customElements.get("user-card");
```

---

## 11.2 Wait for Definition

```js
await customElements.whenDefined("user-card");
```

Useful for applications that must wait until the platform has upgraded a component type.

---

## 11.3 Upgrade

The registry participates in upgrading custom-element instances after definitions become available.

---

## 11.4 Registration Is Global by Default

The global registry is associated with the Window environment.

This can create naming coordination requirements in large applications.

---

# 12. Element Upgrade and Definition Timing

Consider HTML loaded before JavaScript:

```html
<user-card></user-card>
```

At that moment:

```text
"ready" definition may not exist
```

Later:

```js
customElements.define("user-card", UserCard);
```

The browser can upgrade matching elements.

---

## 12.1 Why This Matters

Your component must work correctly whether:

```text
HTML first
JS first
```

or:

```text
element created before registration
```

and:

```text
element created after registration
```

---

## 12.2 `customElements.whenDefined`

Useful:

```js
await customElements.whenDefined("user-card");

const cards = document.querySelectorAll("user-card");
```

---

## 12.3 Upgrade Is Not Reparse

The browser does not need to throw away the element and recreate a new tag.

The existing element can be upgraded into the defined custom-element behavior.

---

# 13. Custom Element Lifecycle

The most important callbacks are:

```text
constructor
connectedCallback
disconnectedCallback
adoptedCallback
attributeChangedCallback
```

---

## 13.1 Lifecycle Mental Model

```text
create/upgrade
     ↓
constructor
     ↓
connect
     ↓
connectedCallback
     ↓
attributes change
     ↓
attributeChangedCallback
     ↓
disconnect
     ↓
disconnectedCallback
```

The actual lifecycle can involve repeated connect/disconnect cycles.

---

# 14. Constructor Rules

Example:

```js
class UserCard extends HTMLElement {
  constructor() {
    super();

    // initialize internal state
  }
}
```

---

## 14.1 Call `super()`

Required before using `this`.

---

## 14.2 Keep Construction Lightweight

Do not assume:

```text
connected to document
```

during construction.

Avoid using:

```js
this.parentElement
```

as though the component is already mounted.

---

## 14.3 Avoid Heavy Side Effects

Prefer:

```text
construct object state
→ attach internal structures
→ connectedCallback handles connected lifecycle
```

---

## 14.4 Shadow Root in Constructor

It is common to attach a shadow root during construction when the component's internal structure is intrinsic:

```js
constructor() {
  super();

  this.attachShadow({
    mode: "open"
  });
}
```

The exact timing should be chosen intentionally.

---

# 15. connectedCallback and disconnectedCallback

## 15.1 Connect

```js
connectedCallback() {
  this.render();
}
```

Can run when the element becomes connected.

---

## 15.2 Disconnect

```js
disconnectedCallback() {
  this.cleanup();
}
```

Can run when it becomes disconnected.

---

## 15.3 Reconnection

```text
connect
→ disconnect
→ connect
```

can happen.

Therefore:

```text
connectedCallback
```

must not assume:

```text
first and only mount
```

---

## 15.4 Idempotent Setup

A robust component can guard repeated registration:

```js
connectedCallback() {
  if (this._connected) return;

  this._connected = true;
  this._setup();
}
```

Then:

```js
disconnectedCallback() {
  if (!this._connected) return;

  this._connected = false;
  this._cleanup();
}
```

---

# 16. adoptedCallback

When a custom element moves between documents, `adoptedCallback` can run.

Conceptually:

```text
Document A
   ↓
move/adopt
   ↓
Document B
```

---

## 16.1 Why It Matters

For most ordinary applications it is rare.

For advanced multi-document/iframe/component-library systems, ownership may matter.

---

# 17. observedAttributes and attributeChangedCallback

Define:

```js
static observedAttributes = [
  "user-id",
  "theme"
];
```

Then:

```js
attributeChangedCallback(name, oldValue, newValue) {
  // respond
}
```

---

## 17.1 Example

```js
class UserCard extends HTMLElement {
  static observedAttributes = ["user-id"];

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === "user-id") {
      this.render();
    }
  }
}
```

---

## 17.2 Attribute State

Remember:

```text
attribute = string-oriented markup state
```

while:

```text
property = JavaScript-facing runtime state
```

---

## 17.3 Prevent Redundant Rendering

Avoid:

```js
attributeChangedCallback() {
  this.render();
}
```

for every tiny internal transition if multiple changes can be batched.

---

# 18. Shadow DOM

Shadow DOM lets a component attach an encapsulated DOM tree.

```js
const shadow = this.attachShadow({
  mode: "open"
});
```

MDN describes Shadow DOM as a mechanism for attaching an encapsulated “shadow” tree to an element and identifies the attached element as the shadow host. citeturn327281search2

---

## 18.1 Shadow Host

```text
normal DOM element
+
shadow root
```

The outer element is the:

```text
shadow host
```

---

## 18.2 Shadow Tree

The nodes inside the shadow root form the:

```text
shadow tree
```

---

## 18.3 Shadow Boundary

The boundary separates:

```text
document/light DOM
```

from:

```text
shadow tree
```

and affects:

```text
CSS
DOM access
event propagation
selectors
```

---

# 19. Shadow Host and Shadow Root

Example:

```js
const host = document.querySelector("user-card");

const shadow = host.attachShadow({
  mode: "open"
});
```

Conceptual:

```text
Document
  ↓
user-card  ← shadow host
  ↓
#shadow-root
  ↓
internal nodes
```

---

## 19.1 `shadowRoot`

For an open root:

```js
host.shadowRoot
```

returns the ShadowRoot.

---

## 19.2 Internal DOM

```js
shadow.querySelector(".button");
```

queries inside the shadow tree.

---

# 20. Open vs Closed Shadow Roots

## Open

```js
this.attachShadow({
  mode: "open"
});
```

Then:

```js
element.shadowRoot
```

can expose the root.

---

## Closed

```js
this.attachShadow({
  mode: "closed"
});
```

Then:

```js
element.shadowRoot
```

returns `null` to ordinary page access.

---

## 20.1 Closed Is Encapsulation, Not Security

Do not store secrets in a closed shadow root.

Browser DevTools, privileged tooling, or application-level access can still expose implementation details.

---

## 20.2 Which Should You Choose?

### Open

Better for:

```text
debuggability
integration
testing
controlled external tooling
```

### Closed

Can strengthen implementation encapsulation.

But it may complicate:

```text
testing
debugging
integration
```

Choose based on component contract, not “closed is more secure.”

---

# 21. Shadow Tree Encapsulation

A page-level selector:

```js
document.querySelector(".internal-button");
```

does not normally cross into a component's shadow tree.

Inside:

```js
shadowRoot.querySelector(".internal-button");
```

can find it.

---

## 21.1 Why This Matters

A component can use common internal names:

```text
.button
.header
.input
```

without colliding with page CSS selectors.

MDN highlights this encapsulation benefit of Shadow DOM. citeturn327281search2

---

# 22. CSS Encapsulation

A shadow tree creates a style boundary.

Inside:

```html
<style>
  button {
    color: red;
  }
</style>
```

the rule targets appropriate shadow-tree elements rather than arbitrary page buttons.

---

## 22.1 Outside CSS

A page rule:

```css
button {
  color: blue;
}
```

does not simply override every internal shadow button.

---

## 22.2 But Encapsulation Is Not Absolute Isolation

Some information/values cross boundaries:

```text
inherited properties
CSS custom properties
slotted content
host selectors
parts
```

Therefore:

```text
shadow CSS = scoped
```

not:

```text
shadow CSS = physically isolated universe
```

---

# 23. Styling a Shadow Tree

## 23.1 Internal `<style>`

```js
shadow.innerHTML = `
  <style>
    :host {
      display: block;
    }

    button {
      border-radius: 8px;
    }
  </style>

  <button>Save</button>
`;
```

---

## 23.2 `:host`

Target the host:

```css
:host {
  display: block;
}
```

---

## 23.3 `:host(...)`

Conditional host styling:

```css
:host([disabled]) {
  opacity: 0.5;
}
```

---

## 23.4 `:host-context(...)`

Where supported, this can style based on ancestor context.

Use sparingly because component behavior should not become overly coupled to outer page structure.

---

## 23.5 Constructable Stylesheets

Modern browsers can support:

```js
const sheet = new CSSStyleSheet();

sheet.replaceSync(`
  :host {
    display: block;
  }
`);

shadow.adoptedStyleSheets = [sheet];
```

This enables stylesheet reuse among compatible shadow roots.

Check browser support before relying on it universally.

---

# 24. CSS Custom Properties Across Boundaries

Custom properties can act as component theming inputs:

```css
user-card {
  --card-accent: blue;
}
```

Inside shadow CSS:

```css
button {
  background: var(--card-accent);
}
```

---

## 24.1 Why This Is Powerful

The component can expose:

```text
small, intentional theme surface
```

without exposing every internal selector.

---

## 24.2 Contract Design

Prefer:

```text
--component-accent
--component-gap
--component-radius
```

over a huge set of undocumented variables.

---

# 25. `::part` and Exposed Styling Hooks

A shadow element can declare:

```html
<button part="button">Save</button>
```

The outside document can style:

```css
user-card::part(button) {
  font-weight: bold;
}
```

---

## 25.1 Why `::part()` Exists

It creates an explicit styling API:

```text
internal node
→ public part name
→ external styling
```

---

## 25.2 Good Design

Expose:

```text
part="header"
part="button"
part="footer"
```

only where external customization is intentionally supported.

---

# 26. `::slotted()`

Shadow CSS can target assigned slotted elements:

```css
::slotted(p) {
  margin: 0;
}
```

---

## 26.1 Boundary

`::slotted()` applies to slotted elements, generally direct assigned nodes, rather than arbitrarily traversing deep descendants.

---

## 26.2 Ownership

Slotted content remains owned by the light DOM, even while being rendered into a slot.

---

# 27. Templates

A `<template>` stores markup that is not rendered immediately when defined.

```html
<template id="card-template">
  <article class="card">
    <h2></h2>
  </article>
</template>
```

MDN describes `<template>` as holding HTML that is not rendered immediately and can later be instantiated using JavaScript. citeturn327281search1turn327281search4

---

## 27.1 Template Content

```js
const template = document.querySelector("#card-template");

const fragment = template.content.cloneNode(true);
```

---

## 27.2 Why Templates?

They support:

```text
reusable DOM structure
component internals
declarative markup
```

---

## 27.3 Template Is Not Rendered DOM

The template element itself is a container for inert template contents until instantiated.

---

# 28. `<slot>` and Content Projection

A slot is a placeholder in a shadow tree:

```html
<slot></slot>
```

Usage:

```html
<user-card>
  <p>Hello</p>
</user-card>
```

The light-DOM content can be assigned to the slot.

---

## 28.1 Conceptual

```text
Host light DOM
   ↓
slotted node
   ↓
shadow slot
```

---

## 28.2 No Copy

The slotted node remains the same DOM node.

---

## 28.3 Why Slots Exist

They let a component define:

```text
internal structure
```

while allowing consumers to supply:

```text
selected content
```

---

# 29. Named Slots

Shadow:

```html
<header>
  <slot name="title"></slot>
</header>

<main>
  <slot></slot>
</main>
```

Consumer:

```html
<user-card>
  <h2 slot="title">Profile</h2>
  <p>Content</p>
</user-card>
```

---

## 29.1 Assignment

The light-DOM element:

```html
<h2 slot="title">
```

maps to:

```html
<slot name="title">
```

---

## 29.2 Public Composition Contract

Slots create a component API:

```text
title
content
footer
actions
```

This is more stable than asking consumers to manipulate internal shadow DOM.

---

# 30. Default Slots and Fallback Content

Default slot:

```html
<slot></slot>
```

can provide fallback content:

```html
<slot>Nothing supplied</slot>
```

If no compatible slotted content exists, fallback content can appear.

---

## 30.1 Why Fallback Is Useful

A component can remain meaningful when the consumer provides incomplete content.

---

# 31. Slot Assignment

Named slot assignment is the common default.

Conceptually:

```text
light child slot="title"
        ↓
shadow slot name="title"
```

---

## 31.1 Manual Slot Assignment

Modern APIs can support explicit manual assignment for selected shadow roots.

Example:

```js
const shadow = host.attachShadow({
  mode: "open",
  slotAssignment: "manual"
});
```

Then:

```js
slot.assign(node);
```

Support is version-sensitive; verify current browser compatibility before using manual assignment as a baseline dependency.

---

## 31.2 Why Manual Assignment Exists

It gives components more direct control over composition.

---

# 32. `slotchange`

Listen:

```js
slot.addEventListener("slotchange", () => {
  // assignments changed
});
```

---

## 32.1 Why It Matters

Component internals can react when consumer-provided content changes its slot assignment.

---

## 32.2 Important Distinction

`slotchange` concerns slot assignment changes.

It is not a general event for every descendant mutation inside slotted content.

---

# 33. Light DOM vs Shadow DOM

## Light DOM

Normal children of the host:

```html
<user-card>
  <span>consumer content</span>
</user-card>
```

---

## Shadow DOM

Internal:

```text
#shadow-root
```

---

## 33.1 Ownership

The light DOM belongs to the document tree.

The shadow DOM belongs to the shadow root.

---

## 33.2 Composition

The browser composes them for rendering/interaction according to shadow/slot rules.

---

# 34. Composed Tree

The DOM tree and shadow tree are not enough to reason about all rendering/composition behavior.

A useful conceptual model:

```text
light DOM
+
shadow tree
+
slot assignments
↓
composed view/tree
```

---

## 34.1 Why It Matters

This affects:

```text
events
focus
rendering
accessibility
selectors
```

---

# 35. Events and Shadow Boundaries

Chapter 50 established:

```text
event path
retargeting
composed
composedPath
```

Web Components make these concrete.

Suppose:

```text
host
 ↓
shadow root
 ↓
button
```

Click inside the button.

An outside listener may see:

```js
event.target === host
```

rather than the internal button, depending on event/retargeting semantics.

---

## 35.1 Composed Events

Events can be:

```js
event.composed
```

which determines whether they can cross shadow boundaries under the relevant dispatch rules.

---

## 35.2 `composedPath()`

```js
event.composedPath();
```

can reveal the visible dispatch path.

---

## 35.3 Component API

For component-to-consumer communication, custom events are often a clearer contract:

```js
this.dispatchEvent(
  new CustomEvent("user-selected", {
    bubbles: true,
    composed: true,
    detail: {
      userId: this.userId
    }
  })
);
```

Use `composed: true` only when cross-boundary propagation is intentionally part of the public API.

---

# 36. Focus and `delegatesFocus`

Attach:

```js
this.attachShadow({
  mode: "open",
  delegatesFocus: true
});
```

This can influence focus behavior when interacting with the host.

---

## 36.1 Why It Exists

A component can expose one public host while directing focus into an internal focusable control.

---

## 36.2 Accessibility Implication

Use focus delegation intentionally.

A component's:

```text
focus
keyboard behavior
focus ring
accessible name
```

must remain coherent.

---

# 37. Forms and Custom Elements

A custom element is not automatically a form control.

Example:

```html
<form>
  <user-picker></user-picker>
</form>
```

does not by itself make `user-picker` participate like an `<input>`.

---

# 38. Form-Associated Custom Elements

Custom elements can opt into form association.

Example:

```js
class UserPicker extends HTMLElement {
  static formAssociated = true;

  constructor() {
    super();
    this._internals = this.attachInternals();
  }
}

customElements.define("user-picker", UserPicker);
```

---

## 38.1 Why It Exists

A custom control can integrate with:

```text
form submission
validation
labels
disabled state
form ownership
```

in a platform-aligned way.

---

## 38.2 Complexity

Form-associated components require careful handling of:

```text
value
validity
disabled
reset
form association
accessibility
```

---

# 39. ElementInternals

`ElementInternals` allows custom elements to integrate more deeply with platform behavior.

Potential responsibilities include:

```text
form association
constraint validation
custom states
accessibility-related semantics
```

---

## 39.1 Custom States

Modern custom elements can expose custom states that can participate in CSS selectors such as:

```css
:state(...)
```

where supported.

MDN's Web Components documentation includes `ElementInternals`, custom states, and related custom-element features. citeturn327281search0

---

## 39.2 Why ElementInternals Matters

It moves a component beyond:

```text
"div with JavaScript"
```

toward:

```text
platform-integrated custom control
```

---

# 40. Accessibility Architecture

Web Components do not automatically solve accessibility.

A component must define:

```text
role
name
description
state
focus
keyboard behavior
```

as appropriate.

---

## 40.1 Prefer Native Semantics

If a component needs button behavior, consider:

```html
<button>
```

inside the shadow tree instead of:

```html
<div role="button">
```

when that better preserves native interaction and semantics.

---

## 40.2 Custom Element Semantics

The tag:

```html
<user-card>
```

does not automatically become:

```text
region
article
button
```

Choose semantics deliberately.

---

## 40.3 Labels

Form-associated custom elements should support appropriate labeling semantics.

---

## 40.4 Keyboard

If the component behaves interactively:

```text
Tab
Enter
Space
Escape
Arrow keys
```

may matter.

Do not implement only mouse interaction.

---

# 41. Declarative Shadow DOM

Declarative Shadow DOM allows a shadow tree to be represented in HTML:

```html
<user-card>
  <template shadowrootmode="open">
    <style>
      :host {
        display: block;
      }
    </style>

    <slot></slot>
  </template>
</user-card>
```

MDN documents `shadowrootmode` as a way for a `<template>` to become a shadow root during HTML parsing in supporting browsers. citeturn327281search2turn327281search4

---

## 41.1 Why Declarative Shadow DOM?

Benefits can include:

```text
SSR compatibility
HTML-first component structure
less imperative bootstrapping
progressive rendering opportunities
```

---

## 41.2 Hydration

A client-side custom element can later attach behavior to the existing server-rendered structure.

This can reduce the need to reconstruct everything from scratch.

---

## 41.3 Compatibility

Verify browser support for:

```text
shadowrootmode
delegatesFocus
clonable
serializable
slotAssignment
```

before using advanced declarative features as unconditional requirements.

---

# 42. Scoped Custom Element Registries

A global registry can create name conflicts:

```text
<menu-item>
```

defined by two libraries.

Modern Web Platform work supports scoped registries in supporting browsers.

MDN documents `CustomElementRegistry()` and scoped-registry association with shadow roots. citeturn327281search0turn327281search5

---

## 42.1 Conceptual Model

```text
Registry A
  <user-card> → Definition A

Registry B
  <user-card> → Definition B
```

within separate scopes.

---

## 42.2 Why This Matters

Large design systems and independently developed component libraries can otherwise collide in the global custom-element namespace.

---

## 42.3 Support Caveat

Scoped registries are newer than the core custom-element API.

Verify browser support before making them a required baseline.

---

# 43. Web Components and Frameworks

## Web Components

```text
platform-native component model
```

## Framework Components

```text
framework-managed abstraction
```

---

## 43.1 Framework Can Consume Web Components

Example conceptual:

```text
React
 ↓
<user-card>
 ↓
Custom Element
```

The framework owns:

```text
rendering/state lifecycle
```

while the Web Component owns:

```text
custom-element lifecycle
shadow tree
public component contract
```

These lifecycles need careful integration.

---

## 43.2 Web Components Can Consume Framework Code

A custom element can internally use a framework.

Then:

```text
Custom Element
    ↓
framework runtime
    ↓
shadow/light DOM
```

This can be useful for gradual migration but increases runtime complexity.

---

## 43.3 Native vs Framework

Web Components do not automatically replace all frameworks.

Frameworks provide:

```text
state management
reconciliation
routing
data fetching
testing
developer tooling
```

Web Components provide:

```text
browser-native component boundaries
```

---

# 44. Web Components and SSR/Hydration

Server-rendered HTML:

```text
server
 ↓
HTML
 ↓
browser parser
 ↓
DOM / shadow DOM
```

Then client JavaScript can:

```text
define custom elements
→ upgrade elements
→ attach behavior
```

---

## 44.1 Declarative Shadow DOM

Can make server-rendered shadow trees part of initial HTML.

---

## 44.2 Hydration Strategy

A strong component can separate:

```text
markup/state already present
```

from:

```text
behavior activation
```

---

## 44.3 Avoid Double Rendering

Do not automatically reconstruct the same DOM from scratch if server markup is already authoritative.

---

# 45. Component Contracts

A production component should define a public API.

## Attributes

```html
<user-card user-id="42"></user-card>
```

---

## Properties

```js
card.user = user;
```

---

## Methods

```js
card.refresh();
```

---

## Events

```js
card.addEventListener("user-selected", handler);
```

---

## Slots

```html
<user-card>
  <span slot="title">Prasanta</span>
</user-card>
```

---

## CSS Custom Properties

```css
user-card {
  --card-accent: blue;
}
```

---

## Parts

```css
user-card::part(action) {
  ...
}
```

---

## 45.1 Contract Stability

A component should avoid requiring consumers to know:

```text
.shadowRoot.querySelector(".private-button")
```

as a normal integration path.

That exposes implementation rather than contract.

---

# 46. Lifecycle and Resource Management

A production Web Component should own all resources it creates.

Example:

```js
class SearchBox extends HTMLElement {
  #controller;

  connectedCallback() {
    this.#controller = new AbortController();

    this.addEventListener("input", this.#onInput, {
      signal: this.#controller.signal
    });
  }

  disconnectedCallback() {
    this.#controller?.abort();
  }

  #onInput = () => {
    // ...
  };
}
```

---

## 46.1 Other Resources

Clean up:

```text
ResizeObserver
IntersectionObserver
MutationObserver
BroadcastChannel
Worker
timer
media stream
subscriptions
```

---

## 46.2 Reconnection

If a component can reconnect, cleanup/setup must be repeatable.

---

# 47. Performance Considerations

## 47.1 Component Count

Thousands of custom elements are not automatically a problem.

Measure:

```text
creation
upgrade
connection
rendering
style
layout
memory
```

---

## 47.2 Upgrade Cost

Defining a custom element can trigger upgrades for matching elements.

A large page with many instances can therefore experience a burst of initialization work.

---

## 47.3 Shadow DOM Cost

Shadow roots introduce additional browser-managed structures.

Do not assume:

```text
shadow DOM = free
```

---

## 47.4 Deep Component Trees

Large nested component hierarchies can increase:

```text
style calculation
layout
event/path reasoning
memory
```

depending on structure.

---

## 47.5 Attribute Rendering

Avoid:

```js
attributeChangedCallback() {
  this.renderEntireComponent();
}
```

for every attribute change when updates can be batched.

---

## 47.6 Slot Distribution

Complex slot structures can create composition work.

Keep slot contracts intentional.

---

## 47.7 CSS

A highly generic styling surface can be expensive.

Prefer:

```text
small public API
stable internal selectors
```

rather than exposing every internal node as a part.

---

## 47.8 Constructable Stylesheets

Shared style sheets can reduce repeated stylesheet text/management in suitable architectures.

---

## 47.9 Event Delegation

Delegation can reduce listeners inside repeated component collections.

But component-local listeners can be simpler and fast enough.

Measure.

---

## 47.10 Lazy Definition

Large applications can define custom elements lazily:

```text
route loads
→ define component
→ upgrade only relevant instances
```

but delayed definitions can complicate UX/loading states.

---

# 48. Memory Considerations

## 48.1 Shadow Roots

Each shadow root adds browser-managed structure.

---

## 48.2 Component State

Avoid storing huge object graphs directly on long-lived components without need.

---

## 48.3 Listeners

Component closures can retain application state.

---

## 48.4 Observers

Observers can retain component-related objects.

---

## 48.5 Detached Components

A detached custom element can remain alive if:

```text
external reference
listener
observer
timer
worker
```

retains it.

---

## 48.6 Internal DOM

Large shadow trees consume memory just like large light-DOM trees.

Shadow DOM is not a memory optimization.

---

## 48.7 Closed Root

Closed roots do not guarantee garbage-collection semantics different from open roots.

They are an API visibility choice, not a GC optimization.

---

# 49. Security Considerations

## 49.1 Shadow DOM Is Not a Security Boundary

Do not store:

```text
tokens
passwords
secrets
```

inside closed shadow trees as a security strategy.

---

## 49.2 `innerHTML`

Component templates using:

```js
shadow.innerHTML = ...
```

must distinguish trusted template markup from untrusted input.

---

## 49.3 Attribute Injection

Do not inject untrusted strings into:

```text
HTML
style
URL
script
event-handler
```

contexts without appropriate handling.

---

## 49.4 Custom Element Supply Chain

Third-party components are executable code.

A component library becomes part of the application's trusted computing base.

---

## 49.5 Event Contracts

Custom events can expose sensitive data through:

```js
detail
```

Do not publish confidential data merely because it is convenient.

---

## 49.6 `::part`

Exposing a part creates a public styling hook.

Do not expose internal DOM purely for convenience if it creates a fragile API.

---

## 49.7 Scoped Registries

Scoped registries can reduce naming collisions but do not turn third-party component code into a security sandbox.

---

# 50. Testing and Debugging

## 50.1 Unit Test Public Contract

Test:

```text
attributes
properties
methods
events
slots
lifecycle
```

---

## 50.2 Lifecycle Test

Verify:

```text
connect
disconnect
reconnect
```

does not duplicate listeners.

---

## 50.3 Shadow Test

Verify:

```text
internal selector behavior
external CSS isolation
parts
slots
```

---

## 50.4 Event Test

Verify:

```text
bubbles
composed
retargeting
detail
```

---

## 50.5 Attribute Test

```js
element.setAttribute("theme", "dark");
```

verify:

```text
attributeChangedCallback
render/update
```

---

## 50.6 Form Test

For form-associated custom elements, test:

```text
value
validation
reset
disabled
labeling
submission
```

---

## 50.7 Upgrade Test

Test both:

```text
element before define
```

and:

```text
element after define
```

---

## 50.8 SSR Test

Test:

```text
server markup
→ parse
→ component definition
→ activation
```

without unnecessarily reconstructing DOM.

---

## 50.9 Browser Matrix

Test:

```text
Chrome/Chromium
Firefox
Safari/WebKit
mobile browsers
```

especially for advanced Web Component features.

---

# 51. Edge Cases

## 51.1 Definition After Parse

Custom element can exist before definition.

---

## 51.2 Reconnection

`connectedCallback()` can run repeatedly.

---

## 51.3 Attribute Changes Before Connect

Attributes can change while an element is not connected.

---

## 51.4 Constructor Before Attributes Are Final

Do not assume every final attribute is available in a finished application state during construction.

---

## 51.5 Shadow Root Once

An element cannot generally have multiple independent shadow roots attached in the same ordinary way.

---

## 51.6 Declarative + Imperative Shadow DOM

If a shadow root already exists, code that blindly calls:

```js
attachShadow()
```

can fail.

Design hydration/upgrade logic to detect existing shadow state and use appropriate APIs.

---

## 51.7 Closed Shadow Root

```js
host.shadowRoot === null
```

does not mean:

```text
no shadow tree exists
```

---

## 51.8 Slot Fallback

Fallback content appears when no assigned nodes satisfy the slot.

---

## 51.9 Multiple Same-Name Slots

Slot naming contracts should be unambiguous.

MDN notes that if multiple slots share a name, matching children are assigned according to the platform's slot-assignment behavior, including the first matching slot behavior for named slots. citeturn327281search3

---

## 51.10 Nested Slots

Nested component slotting creates more complex composed-tree behavior.

---

## 51.11 Slotted Descendants

`::slotted()` does not give arbitrary deep style access to every descendant inside supplied content.

---

## 51.12 Events

Not every event crosses shadow boundaries.

---

## 51.13 `composedPath()`

Closed shadow trees can affect which internal path items are visible externally. citeturn327281search0

---

## 51.14 Form Association

A custom element does not automatically become form-associated unless it opts into the appropriate platform model.

---

## 51.15 Custom Element Registry Collision

Two libraries defining the same global name can fail registration.

---

## 51.16 Cross-Document Adoption

Custom elements can move across documents and may run adoption-related lifecycle behavior.

---

## 51.17 Framework Double Lifecycle

A framework can mount/unmount a custom element in ways that differ from the component author's assumptions.

---

# 52. Common Misconceptions

## Misconception 1 — “Web Components are a framework.”

No. They are platform technologies.

## Misconception 2 — “Custom Elements automatically have semantic meaning.”

No.

## Misconception 3 — “Shadow DOM is just a hidden div.”

No. It has distinct tree, CSS, event, and lifecycle semantics.

## Misconception 4 — “Closed Shadow DOM is secure.”

No.

## Misconception 5 — “Slots copy child nodes.”

No. Slotting composes existing nodes.

## Misconception 6 — “A slotted node becomes part of the shadow DOM tree.”

It remains a light-DOM node assigned to a shadow slot.

## Misconception 7 — “`connectedCallback()` runs only once.”

No.

## Misconception 8 — “`disconnectedCallback()` means permanently destroyed.”

No.

## Misconception 9 — “Attribute changes always trigger `attributeChangedCallback()`.”

Only observed attributes are handled that way.

## Misconception 10 — “Custom elements automatically work as native controls.”

No.

## Misconception 11 — “Shadow CSS cannot inherit anything from outside.”

Some properties/custom properties and host-related behavior cross the boundary.

## Misconception 12 — “The outside page can style any internal shadow node.”

Not normally; explicit mechanisms such as parts are required.

## Misconception 13 — “A Framework component and a Web Component are the same thing.”

No.

## Misconception 14 — “A Worker can directly update a Web Component's shadow tree.”

No. Worker and DOM execution contexts remain separate.

## Misconception 15 — “Declarative Shadow DOM removes the need for JavaScript.”

It can provide structure, but behavior still requires component logic where appropriate.

---

# 53. Common Mistakes

## Mistake 1 — Rendering only in the constructor

Connection-dependent behavior belongs in lifecycle-aware logic.

## Mistake 2 — Registering listeners on every reconnect

Creates duplicate behavior.

## Mistake 3 — Forgetting cleanup

Leaks:

```text
observers
timers
workers
channels
listeners
```

## Mistake 4 — Exposing internal DOM as the component API

Prefer attributes/properties/methods/events/slots/parts.

## Mistake 5 — Using global selectors to control internal DOM

Breaks encapsulation.

## Mistake 6 — Forgetting accessibility

Custom tag ≠ semantic widget.

## Mistake 7 — Treating shadow DOM as security

It is not.

## Mistake 8 — Overusing slots

Highly complex slot APIs can make components difficult to understand.

## Mistake 9 — Overexposing `::part`

A component with dozens of public parts has a fragile styling contract.

## Mistake 10 — Using `innerHTML` with user input

Injection risk remains.

## Mistake 11 — Assuming one custom-element registry is enough for every architecture

Large multi-library systems may need namespace/scoped-registry strategies.

## Mistake 12 — Assuming browser support from one successful local test

Check the target browser matrix.

---

# 54. Comparison With Related Concepts

| Concept | Main role | Browser-native? | Encapsulation |
|---|---|---:|---:|
| Custom Element | Define reusable element behavior | Yes | Optional Shadow DOM |
| Shadow DOM | Encapsulate DOM/style tree | Yes | Yes |
| `<template>` | Reusable inert markup | Yes | No by itself |
| `<slot>` | Content projection | Yes | Part of Shadow DOM |
| React component | Framework/library abstraction | No | Framework-defined |
| Vue component | Framework/library abstraction | No | Framework-defined |
| Virtual DOM | Framework rendering model | No | Depends |
| iframe | Document/environment isolation | Yes | Much stronger isolation boundary |
| CSS module | Build/tooling style organization | No | Tooling-dependent |

---

## Web Component vs React Component

### Web Component

```text
DOM-native lifecycle
custom element
shadow tree
slots
platform events
```

### React Component

```text
React runtime
state/reconciliation
React lifecycle
framework-specific rendering
```

Both can be excellent.

---

## Shadow DOM vs iframe

### Shadow DOM

```text
same document
same broader page context
component encapsulation
```

### iframe

```text
nested document
stronger document/process/origin boundaries
independent browsing context
```

Do not use Shadow DOM where security isolation requires iframe-style boundaries.

---

## Slots vs Props

A rough comparison:

```text
slot
→ structured DOM content projection

property/attribute
→ typed/configuration state
```

Slots and properties can coexist.

---

# 55. Production Usage

## 55.1 Component Architecture

A strong Web Component often looks like:

```text
             public contract
                   │
      ┌────────────┼────────────┐
      ↓            ↓            ↓
 attributes     properties    events
      │                         │
      └──────────┬──────────────┘
                 ↓
             component
                 │
        ┌────────┴────────┐
        ↓                 ↓
    shadow tree         slots
        │                 │
        ↓                 ↓
 internal state       consumer content
```

---

## 55.2 Example Production Contract

```html
<date-picker
  value="2026-09-10"
  locale="en-IN">
</date-picker>
```

Public API:

```text
attribute:
  value
  locale

property:
  value
  minDate
  maxDate

method:
  open()
  close()

events:
  change
  cancel

CSS:
  --date-picker-accent

parts:
  input
  calendar
  day
```

This is much more stable than:

```js
datePicker.shadowRoot.querySelector(...)
```

---

## 55.3 Internal Implementation

Consumers should not need to know:

```text
div.calendar-grid
button.day
span.month-label
```

unless those are intentionally public parts.

---

## 55.4 Lifecycle Architecture

Use:

```text
constructor
→ static initialization

connectedCallback
→ attach resources

disconnectedCallback
→ release resources

attributeChangedCallback
→ synchronize public state
```

---

## 55.5 Component State

Separate:

```text
public state
```

from:

```text
internal derived/rendering state
```

---

## 55.6 Public Events

Use domain-oriented events:

```text
date-change
user-selected
menu-opened
value-committed
```

rather than exposing internal implementation events.

---

## 55.7 Accessibility Contract

Every interactive component should document:

```text
role
keyboard controls
focus behavior
accessible name
states
labels
```

---

## 55.8 SSR

For server-rendered applications:

```text
HTML
→ declarative shadow root where appropriate
→ custom element registration
→ behavior activation
```

---

## 55.9 Error Handling

A component should fail predictably:

```text
invalid attribute
missing required property
unsupported browser feature
failed resource
```

---

## 55.10 Observability

Instrument component lifecycle when necessary:

```text
upgrade duration
connect duration
render duration
update count
listener count
memory pressure
error count
```

---

## 55.11 Production Decision Framework

| Dimension | Question |
|---|---|
| Correctness | Does the component remain correct through upgrade/reconnect/adoption? |
| Accessibility | Is semantic/focus/keyboard behavior correct? |
| Performance | Is creation/render/update cost acceptable at scale? |
| Memory | Are resources cleaned up on disconnect? |
| Security | Are HTML/style/URL boundaries safe? |
| Reliability | What happens when dependencies fail? |
| Maintainability | Is the public contract smaller than the implementation? |
| Scalability | Can thousands of instances coexist? |
| Observability | Can lifecycle/performance failures be diagnosed? |
| Operational Complexity | Is Shadow DOM/slot architecture justified? |
| Future Change | Can internal structure evolve without breaking consumers? |

---

# 56. Implementation From Scratch

Build a **toy Web Component framework** using your Chapter 49 DOM model.

## Stage 1 — Custom Element Registry

Implement:

```js
class Registry {
  definitions = new Map();

  define(name, constructor) {
    // register
  }

  get(name) {
    return this.definitions.get(name);
  }
}
```

---

## Stage 2 — Upgrade Candidates

When:

```html
<user-card>
```

appears before registration, store it as:

```text
undefined custom element
```

Then:

```js
registry.define("user-card", UserCard);
```

find matching instances and upgrade them.

---

## Stage 3 — Lifecycle

Implement:

```text
constructor
connected
disconnected
attributeChanged
adopted
```

---

## Stage 4 — Shadow Root

Add:

```text
host
shadow root
shadow tree
```

and prevent ordinary light-DOM traversal from entering the shadow tree.

---

## Stage 5 — Slot

Implement:

```text
slot name
assigned nodes
fallback
slotchange
```

---

## Stage 6 — Event Retargeting

Build a simplified:

```text
internal target
→ shadow boundary
→ external target
```

model.

---

## Stage 7 — CSS Scoping Simulation

Create a toy style engine with:

```text
light selector scope
shadow selector scope
part exposure
slotted selector
```

---

## Stage 8 — Public Component Contract

Implement:

```text
attributes
properties
methods
custom events
slots
```

---

## Stage 9 — Lifecycle Resource Management

Add:

```text
listeners
observer
timer
worker
```

and verify cleanup on:

```text
disconnect
```

---

## Stage 10 — Form-Associated Component Model

Simulate:

```text
form
custom control
value
validity
disabled
reset
```

---

## Stage 11 — Declarative Shadow DOM

Parse:

```html
<template shadowrootmode="open">
```

and convert it into:

```text
host
→ shadow root
→ parsed child tree
```

---

## Stage 12 — Scoped Registry

Allow:

```text
component tree A
→ registry A

component tree B
→ registry B
```

with same tag name mapping to different definitions.

---

# 57. Debugging Exercises

## Exercise 1 — Upgrade Timing

HTML:

```html
<user-card></user-card>
```

JavaScript:

```js
class UserCard extends HTMLElement {
  connectedCallback() {
    console.log("connected");
  }
}

customElements.define("user-card", UserCard);
```

Determine when the existing element becomes upgraded and when its lifecycle callback executes.

---

## Exercise 2 — Reconnection

```js
document.body.append(card);
card.remove();
document.body.append(card);
```

How many times can:

```text
connectedCallback
```

execute?

---

## Exercise 3 — Attribute Observation

```js
class UserCard extends HTMLElement {
  static observedAttributes = ["name"];

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(name, oldValue, newValue);
  }
}
```

Now change:

```js
card.setAttribute("theme", "dark");
```

Does the callback execute?

---

## Exercise 4 — Shadow Isolation

Create:

```text
shadow button class="save"
```

Then from the page:

```js
document.querySelector(".save");
```

Predict whether it finds the internal button.

---

## Exercise 5 — Open Shadow Root

```js
const root = host.attachShadow({
  mode: "open"
});

console.log(host.shadowRoot === root);
```

Predict.

---

## Exercise 6 — Closed Shadow Root

```js
const root = host.attachShadow({
  mode: "closed"
});

console.log(host.shadowRoot);
```

Predict.

---

## Exercise 7 — Slot Assignment

Shadow:

```html
<slot name="title"></slot>
```

Light DOM:

```html
<h2 slot="title">Hello</h2>
```

Which node is displayed in the named slot?

---

## Exercise 8 — Slot Fallback

Shadow:

```html
<slot>Default title</slot>
```

Consumer provides no children.

What appears?

---

## Exercise 9 — Slotted Identity

```js
const title = document.createElement("h2");
title.slot = "title";

host.append(title);
```

Is the displayed slotted node a copy?

---

## Exercise 10 — `slotchange`

Add/remove slotted nodes.

Determine which changes trigger:

```text
slotchange
```

and which changes merely mutate descendants.

---

## Exercise 11 — Event Retargeting

Create:

```text
host
→ shadow
→ button
```

Add listeners:

```text
button
host
document
```

Compare:

```js
event.target
event.currentTarget
event.composedPath()
```

---

## Exercise 12 — Form Association

Build a form-associated custom element and test:

```text
submit
reset
disabled
validation
```

---

## Exercise 13 — Reconnection Leak

```js
connectedCallback() {
  window.addEventListener("resize", this.#resize);
}
```

What happens after repeated:

```text
connect
disconnect
connect
disconnect
```

if cleanup is missing?

---

## Exercise 14 — Public Contract

Find all external code that does:

```js
component.shadowRoot.querySelector(...)
```

Refactor the integration to:

```text
property
method
event
slot
part
```

---

## Exercise 15 — SSR Upgrade

Start with:

```html
<user-card>
  <template shadowrootmode="open">
    ...
  </template>
</user-card>
```

Then define the element later.

Determine how to avoid duplicating the existing shadow tree.

---

# 58. Code Review Exercise

Review:

```js
class ProductCard extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <style>
        .title { color: red; }
      </style>

      <div class="title">${this.getAttribute("title")}</div>
    `;

    this.querySelector(".title").addEventListener(
      "click",
      () => {
        this.dispatchEvent(new Event("select"));
      }
    );
  }
}

customElements.define("product-card", ProductCard);
```

A developer says:

> “This is a Web Component, so it is encapsulated and production-ready.”

It is not.

## Problems

1. It does not use Shadow DOM, so internal selectors/styles are not encapsulated.
2. It re-renders on every reconnect without cleanup strategy.
3. It registers a new listener each connection.
4. It interpolates an attribute into HTML.
5. The custom event is not designed as a clear public contract.
6. No accessibility semantics exist.
7. The component may not preserve consumer-provided content.
8. No explicit state/property model exists.
9. It assumes the attribute is always safe as markup.
10. It has no SSR/declarative-shadow strategy.

A stronger architecture could use:

```js
class ProductCard extends HTMLElement {
  static observedAttributes = ["title"];

  #controller;

  constructor() {
    super();

    const shadow = this.attachShadow({ mode: "open" });

    const title = document.createElement("button");
    title.part = "title";
    title.type = "button";

    shadow.append(title);

    this.#title = title;
  }

  connectedCallback() {
    this.#controller = new AbortController();

    this.#title.addEventListener("click", () => {
      this.dispatchEvent(
        new CustomEvent("select", {
          bubbles: true,
          composed: true
        })
      );
    }, {
      signal: this.#controller.signal
    });

    this.#render();
  }

  disconnectedCallback() {
    this.#controller?.abort();
  }

  attributeChangedCallback() {
    if (this.isConnected) {
      this.#render();
    }
  }

  #render() {
    this.#title.textContent = this.getAttribute("title") ?? "Untitled";
  }
}

customElements.define("product-card", ProductCard);
```

This is still only a starting point.

Production readiness also requires:

```text
accessibility
focus
keyboard
testing
browser support
public contract
visual design
error handling
SSR strategy
```

---

# 59. Interview Questions

## Foundational

1. What are Web Components?
2. What are the core Web Component technologies?
3. What is a Custom Element?
4. What is Shadow DOM?
5. What is a slot?
6. What is a template?
7. What is a custom-element registry?

## Lifecycle

8. What is `connectedCallback()`?
9. Can it run more than once?
10. What is `disconnectedCallback()`?
11. What is `adoptedCallback()`?
12. What is `attributeChangedCallback()`?
13. What is `observedAttributes`?
14. Why should constructors remain lightweight?

## Shadow DOM

15. What is a shadow host?
16. What is a shadow root?
17. Open vs closed shadow root?
18. Is closed Shadow DOM a security boundary?
19. How does CSS encapsulation work?
20. What is `:host`?
21. What is `::part()`?
22. What is `::slotted()`?

## Slots

23. What is a named slot?
24. What is the default slot?
25. What is fallback slot content?
26. Does slotting clone nodes?
27. What does `slotchange` report?
28. What is manual slot assignment?

## Events

29. What does `composed` mean?
30. What is retargeting?
31. Why can `event.target` be the host outside a shadow tree?
32. What is `composedPath()`?
33. How should a component expose custom events?

## Forms

34. What is a form-associated custom element?
35. What is `ElementInternals`?
36. Why doesn't a custom element automatically behave like an `<input>`?

## Advanced

37. What is Declarative Shadow DOM?
38. What are scoped custom element registries?
39. How would Web Components integrate with SSR?
40. How would Web Components integrate with React?
41. How would you design a stable component contract?
42. How would you prevent lifecycle leaks?
43. How would you debug a component that becomes slow after repeated reconnection?
44. How would you scale a component library across multiple teams?

## Principal-level

45. When should you choose Web Components over a framework abstraction?
46. When should you not use Shadow DOM?
47. How would you version a Web Component's public API?
48. How would you manage custom-element naming across a large organization?
49. How would you design accessibility for a complex custom widget?
50. How would you support server rendering and browser upgrade timing?
51. How would you expose styling hooks without making internals permanently public?
52. How would you measure the cost of thousands of custom-element instances?

---

# 60. Predict-the-Output Exercises

## Exercise 1 — Definition

```js
class UserCard extends HTMLElement {
  connectedCallback() {
    console.log("connected");
  }
}

customElements.define("user-card", UserCard);

document.body.append(
  document.createElement("user-card")
);
```

### Prediction

```text
connected
```

---

## Exercise 2 — Reconnection

```js
document.body.append(card);
card.remove();
document.body.append(card);
```

### Prediction

A connected custom element can run `connectedCallback()` each time it becomes connected.

---

## Exercise 3 — Attribute Observation

```js
class UserCard extends HTMLElement {
  static observedAttributes = ["name"];

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(name, oldValue, newValue);
  }
}

customElements.define("user-card", UserCard);

const card = document.createElement("user-card");

card.setAttribute("theme", "dark");
card.setAttribute("name", "A");
```

### Prediction

The callback is associated with the observed `name` attribute, not arbitrary unobserved `theme` mutations.

---

## Exercise 4 — Open Root

```js
const root = element.attachShadow({
  mode: "open"
});

console.log(element.shadowRoot === root);
```

### Prediction

```text
true
```

---

## Exercise 5 — Closed Root

```js
const root = element.attachShadow({
  mode: "closed"
});

console.log(element.shadowRoot);
```

### Prediction

```text
null
```

The internal root still exists.

---

## Exercise 6 — Slotted Identity

```js
const child = document.createElement("span");
host.append(child);
```

If the component slots that child:

### Prediction

The slot displays the same node identity; it does not clone it.

---

## Exercise 7 — Property vs Attribute

```js
card.title = "Hello";
```

Does this automatically imply:

```js
card.getAttribute("title")
```

has changed?

### Prediction

Only if the platform/component implementation defines the property to reflect to that attribute.

For a custom property you define yourself, no automatic reflection occurs unless you implement it.

---

## Exercise 8 — `slotchange`

A slotted `<span>` keeps the same node but changes its text:

```js
span.textContent = "new";
```

Does that alone mean the slot assignment changed?

### Prediction

No. Slot assignment and descendant content mutation are different concepts.

---

# 61. Mastery Exercises

## Level 1 — Component Basics

Build:

```html
<user-card></user-card>
```

with:

```text
attribute
property
event
shadow DOM
```

---

## Level 2 — Lifecycle

Implement:

```text
connect
disconnect
reconnect
attribute update
```

and prove there are no duplicate listeners.

---

## Level 3 — Slots

Build:

```text
card
title slot
content slot
actions slot
fallback
```

---

## Level 4 — Styling API

Expose:

```text
CSS custom properties
::part
```

and deliberately keep internal selectors private.

---

## Level 5 — Event Contract

Emit:

```text
component-ready
value-change
submit
cancel
```

with documented:

```text
bubbles
composed
cancelable
detail
```

behavior.

---

## Level 6 — Form Component

Build:

```html
<date-input></date-input>
```

with:

```text
form association
value
validation
disabled
reset
label
```

---

## Level 7 — Declarative Shadow DOM

Build a server-compatible component that starts with:

```html
<template shadowrootmode="open">
```

and activates behavior after JavaScript loads.

---

## Level 8 — Scoped Registry Lab

Build two component families where:

```text
<user-card>
```

has two different implementations in two separate scopes.

---

## Level 9 — Framework Integration

Consume the component from:

```text
plain HTML
React
Vue
```

and identify:

```text
property passing
event handling
custom-element lifecycle
SSR
```

differences.

---

## Level 10 — Performance Lab

Render:

```text
10
100
1,000
10,000
```

component instances.

Measure:

```text
upgrade time
connect time
render time
memory
style
layout
event overhead
```

---

## Level 11 — Lifecycle Leak Lab

Create a component that leaks:

```text
listener
timer
observer
worker
```

on disconnect.

Then fix every leak.

---

## Level 12 — Principal Component Design

Design a reusable:

```text
data-grid
```

component with:

```text
100,000 rows
virtualized rendering
keyboard navigation
sorting
filtering
selection
custom cells
slots/parts
events
form integration where appropriate
```

Defend:

```text
Shadow DOM choice
slot design
public contract
styling API
event API
accessibility
performance
memory
SSR
framework integration
```

---

# 62. Key Takeaways

1. Web Components are Web Platform technologies, not frameworks.
2. The core technologies are Custom Elements, Shadow DOM, and templates/slots.
3. Custom Elements define reusable DOM-integrated behavior.
4. Autonomous custom elements use their own custom tag names.
5. Customized built-in elements use the `is` mechanism and have broader support considerations.
6. `CustomElementRegistry` controls definitions and upgrades.
7. Elements can exist before their definitions are registered.
8. Custom elements can be upgraded when a definition becomes available.
9. `connectedCallback()` can run repeatedly.
10. `disconnectedCallback()` is a lifecycle transition, not guaranteed permanent destruction.
11. `attributeChangedCallback()` applies to observed attributes.
12. Constructors should establish object/internal state without assuming connectedness.
13. Shadow DOM creates an encapsulated tree.
14. The host is the outer element; the shadow root owns the shadow tree.
15. Open roots expose `shadowRoot`; closed roots do not.
16. Closed Shadow DOM is not security isolation.
17. Shadow CSS is scoped but inheritance and explicit APIs cross boundaries.
18. `:host` styles the host.
19. CSS custom properties are useful theming inputs.
20. `::part` provides explicit external styling hooks.
21. `::slotted()` styles assigned slotted elements within defined boundaries.
22. Templates hold reusable inert markup.
23. Slots enable content projection.
24. Slotting preserves node identity.
25. Named slots create structured component composition contracts.
26. `slotchange` observes slot-assignment changes.
27. Light DOM and shadow DOM are distinct trees.
28. The composed tree is a useful model for rendering/event composition.
29. Shadow DOM affects event retargeting and composed paths.
30. Custom events can be designed as public component APIs.
31. Custom elements do not automatically provide semantic/accessibility behavior.
32. Form-associated custom elements can integrate with forms.
33. `ElementInternals` enables deeper platform integration.
34. Declarative Shadow DOM is useful for server-rendered component structures.
35. Scoped registries can help prevent large-application custom-element name collisions where supported.
36. Web Components can coexist with frameworks rather than simply replacing them.
37. A strong component contract uses attributes, properties, methods, events, slots, CSS custom properties, and parts deliberately.
38. Lifecycle cleanup is mandatory for production components.
39. Shadow DOM is not a memory optimization or security sandbox.
40. The strongest component architecture minimizes public dependence on internal DOM structure.

---

# 63. Concept Connections

## Depends On

```text
Chapter 49 — DOM Architecture
        ↓
Chapter 50 — Browser Events
        ↓
Chapter 51 — Browser Web APIs
        ↓
Chapter 52 — Web Workers / Concurrency
        ↓
Chapter 53 — Web Streams / Data Flow
        ↓
Chapter 54 — Web Components
```

## Builds Toward

```text
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JS Security Engineering
Chapter 78 — Production JS Architecture
Chapter 79 — API Design
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

- Custom Elements
- Shadow DOM
- slots
- templates
- Web IDL
- DOM
- CSSOM
- event retargeting
- composed tree
- accessibility tree
- ElementInternals
- forms
- SSR
- hydration
- framework integration
- design systems

## Concepts Revisited

### Chapter 49 — DOM Architecture

Web Components turn DOM concepts into reusable component architecture:

```text
Element
→ custom element
→ shadow root
→ slot
```

### Chapter 50 — Browser Events

Shadow boundaries make:

```text
retargeting
composed
composedPath
```

practical component concepts.

### Chapter 51 — Browser Web APIs

Custom element registries, templates, observers, and lifecycle APIs demonstrate the broader Web Platform host model.

### Chapter 45 — Memory

Component lifecycle determines whether:

```text
listener
observer
timer
worker
channel
```

remain reachable after disconnect.

### Chapter 37 — Cancellation

Abort signals can simplify component resource cleanup.

---

## Why This Chapter Matters Later

Chapter 55 will apply browser component concepts to networking-driven applications.

Chapter 56/57 will evaluate the security boundary between component encapsulation and true isolation.

Later production chapters depend on designing:

```text
stable contracts
lifecycle ownership
observability
performance
accessibility
```

The central lesson is:

```text
good component = encapsulated implementation + explicit public contract
```

not:

```text
good component = hidden DOM
```

---

# 64. Completion Criteria

## Fundamentals

- [ ] Define Web Components.
- [ ] Explain Custom Elements.
- [ ] Explain Shadow DOM.
- [ ] Explain templates.
- [ ] Explain slots.
- [ ] Distinguish platform components from frameworks.

## Custom Elements

- [ ] Define autonomous custom element.
- [ ] Define customized built-in element.
- [ ] Use `customElements.define()`.
- [ ] Use `get()`.
- [ ] Use `whenDefined()`.
- [ ] Explain upgrade timing.
- [ ] Explain registry collisions.

## Lifecycle

- [ ] Explain constructor.
- [ ] Explain `connectedCallback`.
- [ ] Explain `disconnectedCallback`.
- [ ] Explain `adoptedCallback`.
- [ ] Explain `attributeChangedCallback`.
- [ ] Explain `observedAttributes`.
- [ ] Handle reconnection safely.

## Shadow DOM

- [ ] Create open shadow root.
- [ ] Create closed shadow root.
- [ ] Explain host/root/tree.
- [ ] Explain DOM encapsulation.
- [ ] Explain CSS encapsulation.
- [ ] Explain inheritance across boundary.
- [ ] Explain `:host`.

## Styling

- [ ] Use CSS custom properties.
- [ ] Explain `::part`.
- [ ] Explain `::slotted`.
- [ ] Explain constructable stylesheets conceptually.

## Templates / Slots

- [ ] Use `<template>`.
- [ ] Clone template content.
- [ ] Use default slot.
- [ ] Use named slots.
- [ ] Use fallback content.
- [ ] Explain slot assignment.
- [ ] Explain manual assignment conceptually.
- [ ] Explain `slotchange`.

## Events

- [ ] Explain composed events.
- [ ] Explain retargeting.
- [ ] Use `composedPath`.
- [ ] Design custom event contracts.

## Forms

- [ ] Explain form-associated custom elements.
- [ ] Use `formAssociated`.
- [ ] Explain `ElementInternals`.
- [ ] Handle validation/reset/disabled state.

## Accessibility

- [ ] Design semantics.
- [ ] Design focus.
- [ ] Design keyboard behavior.
- [ ] Design labels/names.
- [ ] Avoid fake-native controls.

## Advanced

- [ ] Explain Declarative Shadow DOM.
- [ ] Explain SSR compatibility.
- [ ] Explain scoped registries.
- [ ] Explain framework integration.

## Performance

- [ ] Measure upgrade cost.
- [ ] Measure component creation.
- [ ] Measure render/update cost.
- [ ] Measure large instance counts.
- [ ] Evaluate Shadow DOM cost.
- [ ] Evaluate slots/parts/style complexity.

## Memory

- [ ] Clean listeners.
- [ ] Clean observers.
- [ ] Clean timers.
- [ ] Clean workers/channels.
- [ ] Diagnose detached components.

## Security

- [ ] Explain why Shadow DOM is not a security boundary.
- [ ] Handle unsafe template data.
- [ ] Design safe event data.
- [ ] Evaluate third-party component trust.

## Principal Judgment

- [ ] Design a stable component API.
- [ ] Defend open vs closed shadow root.
- [ ] Defend Shadow DOM vs light DOM.
- [ ] Defend slots vs properties.
- [ ] Defend parts/custom properties.
- [ ] Defend lifecycle architecture.
- [ ] Defend framework/Web Component boundaries.

---

# 65. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What are Web Components?
2. What are the three core Web Component technologies?
3. What is an autonomous custom element?
4. What is a customized built-in element?
5. What is CustomElementRegistry?
6. What happens when an element is parsed before its definition?
7. What is upgrade?
8. What happens in the constructor?
9. Can connectedCallback run more than once?
10. What is observedAttributes?
11. What is a shadow host?
12. What is a shadow root?
13. Open vs closed Shadow DOM?
14. Is closed Shadow DOM secure?
15. How does CSS encapsulation work?
16. What is :host?
17. What is ::part?
18. What is ::slotted?
19. What is a template?
20. What is a slot?
21. What is a named slot?
22. What is fallback slot content?
23. Does slotting clone a node?
24. What is slotchange?
25. What is composed tree?
26. What happens to event.target across a shadow boundary?
27. What is delegatesFocus?
28. What is a form-associated custom element?
29. What is ElementInternals?
30. What is Declarative Shadow DOM?
31. What are scoped registries?
32. How do Web Components interact with frameworks?
33. How do you design a stable component contract?
34. How do you prevent lifecycle leaks?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Web Components | [ ] | [ ] | [ ] | [ ] |
| Custom Elements | [ ] | [ ] | [ ] | [ ] |
| Autonomous elements | [ ] | [ ] | [ ] | [ ] |
| Customized built-ins | [ ] | [ ] | [ ] | [ ] |
| Registry | [ ] | [ ] | [ ] | [ ] |
| Upgrade | [ ] | [ ] | [ ] | [ ] |
| Constructor | [ ] | [ ] | [ ] | [ ] |
| Connected lifecycle | [ ] | [ ] | [ ] | [ ] |
| Disconnected lifecycle | [ ] | [ ] | [ ] | [ ] |
| Adopted lifecycle | [ ] | [ ] | [ ] | [ ] |
| Attribute lifecycle | [ ] | [ ] | [ ] | [ ] |
| Shadow host | [ ] | [ ] | [ ] | [ ] |
| Shadow root | [ ] | [ ] | [ ] | [ ] |
| Open/closed | [ ] | [ ] | [ ] | [ ] |
| CSS encapsulation | [ ] | [ ] | [ ] | [ ] |
| :host | [ ] | [ ] | [ ] | [ ] |
| custom properties | [ ] | [ ] | [ ] | [ ] |
| ::part | [ ] | [ ] | [ ] | [ ] |
| ::slotted | [ ] | [ ] | [ ] | [ ] |
| template | [ ] | [ ] | [ ] | [ ] |
| default slot | [ ] | [ ] | [ ] | [ ] |
| named slot | [ ] | [ ] | [ ] | [ ] |
| fallback | [ ] | [ ] | [ ] | [ ] |
| slot assignment | [ ] | [ ] | [ ] | [ ] |
| slotchange | [ ] | [ ] | [ ] | [ ] |
| light vs shadow DOM | [ ] | [ ] | [ ] | [ ] |
| composed tree | [ ] | [ ] | [ ] | [ ] |
| retargeting | [ ] | [ ] | [ ] | [ ] |
| delegatesFocus | [ ] | [ ] | [ ] | [ ] |
| form-associated elements | [ ] | [ ] | [ ] | [ ] |
| ElementInternals | [ ] | [ ] | [ ] | [ ] |
| accessibility | [ ] | [ ] | [ ] | [ ] |
| Declarative Shadow DOM | [ ] | [ ] | [ ] | [ ] |
| scoped registries | [ ] | [ ] | [ ] | [ ] |
| framework integration | [ ] | [ ] | [ ] | [ ] |
| component contracts | [ ] | [ ] | [ ] | [ ] |
| lifecycle cleanup | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw:

```text
custom element
→ shadow root
→ slots
→ public contract
```

### Day 2

Explain all lifecycle callbacks without notes.

### Day 7

Build a card with:

```text
named slots
CSS custom properties
parts
custom event
```

### Day 14

Build a form-associated custom element and test accessibility/focus behavior.

### Day 30

Design a production Web Component library for multiple teams, including registry/naming, SSR, styling contracts, lifecycle, versioning, and browser compatibility.

---

# 66. Canonical References and Source Discipline

## Primary HTML Standard

### WHATWG HTML Standard

https://html.spec.whatwg.org/

Use for:

```text
custom elements
custom element lifecycle
custom-element registry
Shadow DOM integration
templates
slots
Declarative Shadow DOM
form-associated custom elements
ElementInternals
HTML parsing
```

---

## Primary DOM Standard

### WHATWG DOM Standard

https://dom.spec.whatwg.org/

Use for:

```text
trees
events
EventTarget
retargeting
composed paths
shadow-tree-related event behavior
```

---

## CSS References

### CSS Scoping

https://drafts.csswg.org/css-scoping/

Use for:

```text
Shadow DOM styling
:host
::slotted
parts
style boundaries
```

---

## Web Components / MDN

### Web Components

https://developer.mozilla.org/en-US/docs/Web/API/Web_components

MDN summarizes the core Web Component technologies and includes current references for Custom Elements, Shadow DOM, templates, slots, scoped registries, custom states, and related APIs. citeturn327281search0

### Using Custom Elements

https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements

Use for practical lifecycle and custom-element integration examples. citeturn327281search5

### Using Shadow DOM

https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM

Use for shadow hosts, roots, encapsulation, open/closed roots, styling, and declarative Shadow DOM examples. citeturn327281search2

### Using Templates and Slots

https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_templates_and_slots

Use for templates, named slots, fallback content, and slot assignment. citeturn327281search1

### `<slot>`

https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/slot

Use for current slot behavior and assignment rules. citeturn327281search3

---

## Source Classification

For each Web Component claim, classify as:

```text
[HTML Standard]
[DOM Standard]
[CSS Scoping]
[Web IDL]
[Browser-specific]
[Measured]
[Historical]
```

Examples:

```text
"connectedCallback runs when connected"
→ [HTML Standard]

"event path is affected by shadow boundaries"
→ [DOM Standard]

"Chrome currently supports scoped registries"
→ [Browser-specific + version-sensitive]

"1000 components render in 30 ms"
→ [Measured]
```

---

## Version-Sensitivity

For advanced component features record:

```text
browser
version
feature
support level
polyfill requirement
SSR mode
framework integration
```

Do not turn browser support tables into timeless guarantees.

---

## Important Discipline

Do not confuse:

```text
Shadow DOM encapsulation
```

with:

```text
security isolation
```

Do not confuse:

```text
Custom Elements
```

with:

```text
framework components
```

Do not confuse:

```text
slot composition
```

with:

```text
copying nodes
```

---

## Source-Derived Notes

MDN describes Web Components as a collection of browser APIs including Custom Elements, Shadow DOM, and HTML templates/slots. citeturn327281search0

MDN describes Shadow DOM as an encapsulated tree attached to a shadow host and explains its CSS/JavaScript isolation characteristics. citeturn327281search2

MDN's templates/slots documentation explains that template content is inert until instantiated and that slots provide placeholders into which consumer-provided content can be projected. citeturn327281search1

MDN documents slot assignment, named slots, fallback content, and newer manual-assignment options as part of current Web Component APIs. citeturn327281search3turn327281search4

MDN's current Web Components reference also documents scoped custom-element registries and related newer features, demonstrating why advanced component architecture should be compatibility-checked. citeturn327281search0turn327281search5

---

# 67. Completion Snapshot

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

- [ ] Web Components
- [ ] Custom Elements
- [ ] autonomous custom elements
- [ ] customized built-ins
- [ ] CustomElementRegistry
- [ ] upgrade
- [ ] lifecycle
- [ ] observed attributes
- [ ] Shadow DOM
- [ ] shadow host
- [ ] shadow root
- [ ] open/closed
- [ ] encapsulation
- [ ] CSS scoping
- [ ] :host
- [ ] custom properties
- [ ] ::part
- [ ] ::slotted
- [ ] template
- [ ] slot
- [ ] named slots
- [ ] fallback
- [ ] slot assignment
- [ ] slotchange
- [ ] light vs shadow DOM
- [ ] composed tree
- [ ] retargeting
- [ ] composed events
- [ ] delegatesFocus
- [ ] form-associated custom elements
- [ ] ElementInternals
- [ ] accessibility
- [ ] Declarative Shadow DOM
- [ ] scoped registries
- [ ] SSR/hydration
- [ ] framework integration
- [ ] component contracts
- [ ] lifecycle cleanup

### I can predict

- [ ] upgrade timing
- [ ] lifecycle callback order
- [ ] reconnect behavior
- [ ] observed attribute behavior
- [ ] open/closed root visibility
- [ ] slot assignment
- [ ] slot fallback
- [ ] event retargeting
- [ ] composedPath behavior
- [ ] attribute/property divergence
- [ ] form participation
- [ ] lifecycle leak behavior

### I can implement

- [ ] Custom Element
- [ ] Shadow Root
- [ ] lifecycle
- [ ] observed attributes
- [ ] template
- [ ] default slot
- [ ] named slot
- [ ] fallback content
- [ ] slotchange
- [ ] event API
- [ ] custom properties
- [ ] parts
- [ ] form-associated element
- [ ] ElementInternals integration
- [ ] Declarative Shadow DOM structure
- [ ] registry model

### I can debug

- [ ] upgrade failures
- [ ] duplicate listeners
- [ ] reconnect leaks
- [ ] slot bugs
- [ ] event-retargeting bugs
- [ ] style leakage
- [ ] accessibility failures
- [ ] form integration bugs
- [ ] registry collisions
- [ ] SSR/hydration duplication
- [ ] detached component retention

### I can defend

- [ ] Web Components vs framework components
- [ ] Shadow DOM vs light DOM
- [ ] open vs closed root
- [ ] slots vs properties
- [ ] parts vs public DOM
- [ ] global vs scoped registry
- [ ] SSR/declarative-shadow strategy
- [ ] lifecycle design
- [ ] accessibility design
- [ ] performance strategy
- [ ] memory strategy
- [ ] security boundary

---

## Final Principal-Level Test

Explain this statement without notes:

> **A production Web Component is not simply a custom HTML tag. It is a platform-integrated component with a defined public contract, a browser-managed lifecycle, optional Shadow DOM encapsulation, explicit content-composition mechanisms, event and styling boundaries, accessibility responsibilities, and resource ownership.**

Your explanation is complete only when you can connect:

```text
HTML
 ↓
custom-element parsing
 ↓
registry / upgrade
 ↓
constructor
 ↓
connectedCallback
 ↓
attributes / properties
 ↓
shadow root
 ↓
slots
 ↓
events / retargeting
 ↓
styles / parts / custom properties
 ↓
form/accessibility integration
 ↓
disconnect
 ↓
cleanup
```

and distinguish:

```text
component encapsulation
```

from:

```text
security isolation
```

and:

```text
public API
```

from:

```text
internal DOM implementation
```

The central mastery target of Chapter 54 is to stop thinking of Web Components as “HTML components” and start thinking of them as **native browser component architecture with explicit contracts, composition boundaries, lifecycle ownership, and platform semantics**.