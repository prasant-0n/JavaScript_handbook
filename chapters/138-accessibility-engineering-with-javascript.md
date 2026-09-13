# Chapter 138 — Accessibility Engineering With JavaScript

> **JavaScript Mastery — Part XXIII: Browser Platform & Human Interface Engineering**
>
> **Mission:** Master accessibility as an engineering property of JavaScript applications. Learn how semantic HTML, DOM structure, keyboard behavior, focus management, accessible names, ARIA, live regions, forms, dialogs, composite widgets, dynamic rendering, validation, announcements, reduced-motion preferences, testing, and component architecture interact with browsers and assistive technologies.
>
> **Role perspective:** Principal JavaScript Engineer · Accessibility Engineer · Frontend Platform Engineer · Browser Engineer · UI Systems Architect · Design Systems Engineer · QA/Automation Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Use native HTML semantics and browser behavior first. Add ARIA and JavaScript only when they are necessary to expose correct semantics or implement behavior that native controls do not already provide.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] define web accessibility
[ ] explain why accessibility is an engineering concern
[ ] distinguish semantic HTML from visual styling
[ ] explain the accessibility tree conceptually
[ ] explain user agent and assistive technology interaction
[ ] explain accessible names
[ ] explain accessible descriptions
[ ] explain roles
[ ] explain states
[ ] explain properties
[ ] understand native semantics
[ ] understand ARIA
[ ] explain the first rule of ARIA
[ ] choose native HTML instead of custom widgets when possible
[ ] explain keyboard accessibility
[ ] explain Tab navigation
[ ] explain Shift+Tab
[ ] explain focus
[ ] explain focus visibility
[ ] explain logical focus order
[ ] explain tabindex
[ ] understand why positive tabindex values are discouraged
[ ] implement programmatic focus
[ ] implement focus restoration
[ ] implement focus trapping
[ ] implement modal dialogs
[ ] implement non-modal dialogs
[ ] explain roving tabindex
[ ] explain aria-activedescendant
[ ] distinguish DOM focus from active descendant
[ ] explain composite widgets
[ ] implement tabs
[ ] implement listboxes
[ ] implement comboboxes
[ ] implement menus
[ ] implement grids
[ ] implement tree views
[ ] explain keyboard interaction patterns
[ ] implement Escape behavior
[ ] implement Enter/Space behavior
[ ] implement arrow-key navigation
[ ] explain Home/End behavior
[ ] explain typeahead behavior
[ ] explain accessible form labels
[ ] implement validation
[ ] expose error states
[ ] use aria-invalid appropriately
[ ] connect errors with aria-describedby
[ ] explain required fields
[ ] expose status information
[ ] implement live regions
[ ] distinguish status vs alert
[ ] explain aria-live
[ ] explain aria-atomic
[ ] explain aria-relevant
[ ] understand announcement timing
[ ] implement async result announcements
[ ] implement loading announcements
[ ] implement save-state announcements
[ ] support reduced motion
[ ] support prefers-reduced-motion
[ ] avoid inaccessible gesture-only interactions
[ ] make pointer interactions keyboard-equivalent
[ ] understand touch accessibility
[ ] explain screen reader interaction conceptually
[ ] understand name/role/value
[ ] understand semantic state synchronization
[ ] avoid inaccessible custom buttons
[ ] avoid click-only interactions
[ ] avoid div-as-button anti-patterns
[ ] avoid fake links
[ ] preserve native browser behaviors
[ ] understand hidden vs visually hidden
[ ] understand display:none accessibility implications
[ ] understand aria-hidden
[ ] avoid aria-hidden on focusable content
[ ] manage dynamic DOM changes
[ ] manage route-change focus
[ ] manage SPA announcements
[ ] preserve focus during rerendering
[ ] support virtualized lists
[ ] support asynchronous content
[ ] understand disabled vs aria-disabled
[ ] understand inert
[ ] explain pointer-events vs accessibility semantics
[ ] support loading states
[ ] support progressive enhancement
[ ] test keyboard navigation
[ ] test focus order
[ ] test accessible names
[ ] test semantic states
[ ] test dynamic announcements
[ ] use browser accessibility inspection tools
[ ] use automated accessibility testing
[ ] understand limits of automated testing
[ ] perform manual testing
[ ] test with screen readers
[ ] test reduced motion
[ ] test zoom/reflow
[ ] test high contrast/forced colors where applicable
[ ] test touch and pointer alternatives
[ ] design accessible component contracts
[ ] build accessibility primitives
[ ] establish accessibility linting
[ ] establish CI accessibility checks
[ ] establish accessibility regression tests
[ ] explain WCAG concepts
[ ] explain POUR at a high level
[ ] distinguish standards from guidance
[ ] understand WCAG 2.2 principles relevant to JavaScript
[ ] explain keyboard requirements
[ ] explain focus requirements
[ ] explain name/role/value requirements
[ ] explain status message requirements
[ ] explain accessible authentication implications
[ ] design accessible design systems
[ ] review accessibility architecture at principal level


# 2. Prerequisites

You should already understand:

```text
Chapter 33 — Browser Event Loop
Chapter 49 — DOM Architecture
Chapter 50 — Browser Events
Chapter 51 — Browser Web APIs
Chapter 55 — Fetch / HTTP Networking
Chapter 63 — Diagnostics
Chapter 70 — Production Debugging
Chapter 71 — Browser Security
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging Methodology
Chapter 101 — Real-World Production Scenarios
Chapter 137 — Browser Performance APIs & Runtime Instrumentation
```

You should be comfortable with:

```text
DOM
events
forms
CSS
JavaScript state management
async rendering
focus()
keyboard events
custom elements/components
```

---

# 3. What Is Accessibility Engineering?

Accessibility engineering means designing and implementing software so that people with different abilities can:

```text
perceive
understand
navigate
operate
and interact with
```

the application.

For JavaScript systems, accessibility includes:

```text
semantic structure
keyboard behavior
focus management
announcements
state exposure
interaction models
error communication
dynamic content handling
```

Accessibility is not merely:

```text
“add aria-label”
```

It is:

```text
correct semantics
+
correct behavior
+
correct state
+
correct interaction
```

---

# 4. Why Accessibility Matters

A visually perfect interface can fail when:

```text
keyboard focus disappears
screen reader gets no state
button is not actually a button
errors are visually shown but not announced
modal traps the user
dynamic content silently changes
```

Accessibility defects are therefore:

```text
functional defects
```

not cosmetic defects.

---

# 5. Accessibility as a System

Model the interface as:

```text
DOM
 ↓
browser semantics
 ↓
accessibility tree
 ↓
assistive technology
 ↓
user interaction
```

JavaScript changes:

```text
DOM
attributes
focus
state
events
```

which can change what assistive technologies perceive.

---

# 6. Semantic HTML First

Prefer:

```html
<button>Save</button>
```

over:

```html
<div onclick="save()">Save</div>
```

The native button already provides:

```text
role
keyboard activation
focus behavior
form integration
disabled semantics
```

A custom div requires recreating all of that correctly.

---

# 7. Native Semantics Reduce Code

Native HTML can provide:

```text
button
a
input
select
textarea
details
dialog
form
fieldset
legend
```

JavaScript should enhance them rather than replacing them without need.

This reduces:

```text
custom state machine
keyboard code
testing surface
accessibility bugs.
```

---

# 8. The First Rule of ARIA

A foundational principle:

> **Do not use ARIA to recreate semantics that a native HTML element already provides.**

This is often summarized as:

```text
use native HTML whenever possible.
```

ARIA exists to expose semantics for richer interfaces and dynamic application behavior; it does not automatically provide the keyboard behavior of native controls. citeturn335767search0turn335767search1

---

# 9. ARIA Is Not Behavior

This:

```html
<div role="button">Save</div>
```

does not automatically give:

```text
native button behavior.
```

You may still need:

```text
focusability
Enter behavior
Space behavior
disabled behavior
state synchronization
```

Therefore:

```text
ARIA semantics
≠
interaction implementation.
```

---

# 10. Accessibility Tree

The browser maintains an accessibility representation derived from:

```text
DOM
HTML semantics
ARIA
CSS/visibility state
properties
```

Assistive technology consumes this representation rather than simply:

```text
reading raw HTML.
```

---

# 11. Accessibility Tree Mental Model

Think:

```text
DOM tree
    ↓
semantic interpretation
    ↓
accessibility tree
    ↓
screen reader / assistive technology
```

JavaScript must maintain consistency between:

```text
visual UI
DOM state
accessibility state.
```

---

# 12. Name, Role, Value

Many interactive controls expose three central concepts:

```text
Name
Role
Value
```

Example:

```html
<button>Delete</button>
```

Conceptually:

```text
Name = Delete
Role = button
State/value = current button state
```

---

# 13. Accessible Name

An accessible name is the programmatically exposed name of an element.

Examples:

```html
<button>Save</button>
```

or:

```html
<button aria-label="Save document">
  <svg aria-hidden="true">...</svg>
</button>
```

The exact accessible-name computation follows standards and host-language rules.

---

# 14. Accessible Description

A description adds additional explanatory information.

For example:

```html
<input
  aria-describedby="password-help"
>
<p id="password-help">
  Minimum 12 characters.
</p>
```

Name answers:

```text
“What is this?”
```

Description answers:

```text
“What additional information should I know?”
```

---

# 15. Labels

Prefer:

```html
<label for="email">Email</label>
<input id="email" type="email">
```

This creates an explicit relationship.

Avoid:

```text
placeholder = label
```

because placeholder text is not a replacement for a persistent label.

---

# 16. Label vs Placeholder

Label:

```text
identity
```

Placeholder:

```text
example/hint
```

Bad:

```html
<input placeholder="Email">
```

Better:

```html
<label for="email">Email</label>
<input id="email" placeholder="you@example.com">
```

---

# 17. Buttons

Native:

```html
<button type="button">
  Save
</button>
```

Use:

```text
button
```

for:

```text
actions
```

and:

```text
a
```

for:

```text
navigation.
```

---

# 18. Links vs Buttons

### Link

```text
go somewhere
```

### Button

```text
perform an action.
```

Do not use:

```html
<a href="#" onclick="deleteItem()">
```

as a button substitute.

---

# 19. Keyboard Accessibility

Every interactive feature must have a keyboard-accessible path.

W3C APG identifies a common convention:

```text
Tab / Shift+Tab
```

move between components, while other keys such as:

```text
Arrow keys
```

may move within composite widgets. citeturn335767search1

---

# 20. Focus

Focus identifies the element receiving keyboard interaction.

Use:

```js
element.focus();
```

when programmatic focus is required.

Do not move focus:

```text
randomly
```

or:

```text
on every state update.
```

---

# 21. Focus Order

The focus sequence should follow:

```text
logical reading/order of the interface
```

It should not jump:

```text
header
footer
middle form
sidebar
```

unexpectedly.

---

# 22. Positive `tabindex`

Avoid:

```html
tabindex="5"
```

and:

```html
tabindex="10"
```

as a way to manually create a complex tab order.

Prefer:

```html
tabindex="0"
```

when needed, and:

```html
tabindex="-1"
```

for programmatic focus targets that should not enter the normal Tab sequence.

APG strongly discourages positive tabindex values. citeturn335767search1

---

# 23. `tabindex="0"`

A custom focusable element can participate in normal sequential keyboard navigation using:

```html
tabindex="0"
```

But first ask:

```text
Why isn't this a native focusable element?
```

---

# 24. `tabindex="-1"`

Useful for:

```text
programmatic focus
dialog container
error heading
route heading
composite widget management
```

Example:

```js
heading.tabIndex = -1;
heading.focus();
```

---

# 25. Visible Focus

Never remove focus indication without a replacement.

Bad:

```css
:focus {
  outline: none;
}
```

unless an equivalent visible focus treatment exists.

Users need to know:

```text
where keyboard focus is.
```

---

# 26. Focus and Selection

Focus and selection are not always the same.

For a list:

```text
focus
```

may identify the active item while:

```text
selection
```

identifies the chosen value.

Composite widgets must define this distinction clearly. citeturn335767search1

---

# 27. Focus Restoration

When closing a dialog opened from:

```text
Delete button
```

restore focus to:

```text
Delete button
```

when appropriate.

Otherwise focus may fall to:

```text
body
```

causing:

```text
orientation loss.
```

---

# 28. Route-Change Focus

In a SPA:

```text
navigate
```

without a full page load.

The user may remain focused on:

```text
navigation button
```

while the new page content appears elsewhere.

A common pattern is:

```text
move focus to new route heading/main region
```

when the application changes the page context.

---

# 29. Skip Links

Provide a way to skip repeated navigation:

```html
<a href="#main" class="skip-link">
  Skip to main content
</a>
```

This reduces:

```text
repeated keyboard traversal.
```

---

# 30. Landmark Regions

Semantic landmarks can represent:

```text
header
navigation
main
complementary
contentinfo
```

Landmarks allow assistive technology users to navigate the page structure more efficiently. citeturn335767search10

---

# 31. Semantic Layout

Prefer:

```html
<header>
<nav>
<main>
<aside>
<footer>
```

over:

```html
<div class="header">
<div class="nav">
<div class="main">
```

when the semantic elements express the intended meaning.

---

# 32. Heading Structure

Use headings to communicate:

```text
document hierarchy.
```

Avoid:

```text
font-size
```

as the sole semantic model.

Example:

```html
<h1>Orders</h1>

<h2>Recent Orders</h2>

<h2>Archived Orders</h2>
```

---

# 33. Dynamic Heading Updates

When a SPA navigates:

```text
document title
main heading
focus
```

should remain coherent.

Example:

```js
document.title = "Orders — Dashboard";
heading.textContent = "Orders";
heading.focus();
```

---

# 34. `aria-current`

When navigation contains the current route:

```html
<a
  href="/orders"
  aria-current="page"
>
  Orders
</a>
```

This exposes:

```text
current item state
```

to assistive technology.

---

# 35. Expanded State

Disclosure button:

```html
<button
  aria-expanded="false"
  aria-controls="filters"
>
  Filters
</button>
```

When opened:

```js
button.setAttribute("aria-expanded", "true");
```

State must match:

```text
visual state
DOM state
ARIA state.
```

---

# 36. Checked State

For custom checkbox:

```html
<div
  role="checkbox"
  aria-checked="false"
  tabindex="0"
>
  Subscribe
</div>
```

JavaScript must update:

```text
aria-checked
```

when state changes.

Prefer:

```html
<input type="checkbox">
```

when possible.

---

# 37. Disabled vs `aria-disabled`

Native:

```html
<button disabled>
```

has browser-defined disabled semantics.

ARIA:

```html
<button aria-disabled="true">
```

communicates a state but does not always reproduce all native disabled behavior.

Do not treat:

```text
aria-disabled
```

as a universal substitute for:

```text
disabled.
```

---

# 38. `aria-hidden`

Use:

```html
aria-hidden="true"
```

to exclude content from the accessibility tree when appropriate.

Do not put:

```text
focusable interactive content
```

inside an `aria-hidden="true"` subtree.

That creates:

```text
keyboard / accessibility mismatch.
```

---

# 39. Visual Hiding vs Accessibility Hiding

These are different:

```text
display:none
visibility:hidden
aria-hidden
visually-hidden CSS
```

They have different effects on:

```text
layout
rendering
focus
accessibility tree.
```

Understand each before using it.

---

# 40. Visually Hidden Content

Sometimes content should be:

```text
visually hidden
```

but:

```text
available to assistive technologies.
```

Use a well-tested visually-hidden pattern rather than:

```css
display: none;
```

---

# 41. `inert`

The `inert` attribute can make a subtree:

```text
non-interactive
```

and can prevent users from interacting with background content while another context is active.

This is particularly useful with:

```text
modal interfaces.
```

---

# 42. Modal Dialog

A modal dialog should:

```text
move focus inside
keep focus within the modal interaction
provide a name
support Escape where appropriate
restore focus when closed
```

APG describes this as a focus-management responsibility of the author. citeturn335767search8turn335767search1

---

# 43. Dialog Name

A dialog should have an accessible name.

Example:

```html
<div
  role="dialog"
  aria-labelledby="dialog-title"
>
  <h2 id="dialog-title">
    Delete account
  </h2>
</div>
```

---

# 44. Dialog Focus Entry

When opened:

```text
focus moves inside.
```

Choose the initial focus target based on:

```text
task
content
risk
expected user action.
```

Do not always:

```text
focus first input.
```

---

# 45. Dialog Focus Trap

Conceptually:

```text
Tab
last → first

Shift+Tab
first → last
```

Do not let:

```text
keyboard focus escape
```

while the modal is active.

---

# 46. Dialog Close

Escape commonly:

```text
closes dialog
```

when the interaction pattern permits it.

After close:

```text
restore focus to invoking element.
```

APG's dialog pattern explicitly specifies Escape and containment of Tab navigation for modal dialogs. citeturn335767search8

---

# 47. Native `<dialog>`

Modern browsers provide:

```html
<dialog>
```

with built-in behavior for several dialog concerns.

Use it where appropriate rather than rebuilding:

```text
all dialog mechanics.
```

But still validate:

```text
focus
naming
keyboard behavior
styling
browser/AT interoperability.
```

---

# 48. Popovers and Floating UI

Floating interfaces include:

```text
tooltip
popover
menu
listbox
combobox popup
context menu
```

Do not treat them as identical.

Each pattern has different:

```text
focus
keyboard
dismissal
semantics
```

requirements.

---

# 49. Tooltip

A tooltip is generally:

```text
supplementary information
```

not:

```text
primary interaction.
```

Do not build:

```text
critical action
```

inside a tooltip that requires hover.

---

# 50. Menus

An ARIA menu is not simply:

```text
navigation list.
```

Menus have specific interaction conventions.

Use:

```text
menu button
menu
menuitem
```

for menu patterns rather than assigning:

```text
role="menu"
```

to ordinary site navigation.

---

# 51. Tabs

A tab system typically contains:

```text
tablist
tab
tabpanel
```

with a relationship between:

```text
selected tab
```

and:

```text
visible panel.
```

Keyboard behavior can include:

```text
ArrowLeft / ArrowRight
Home / End
Enter / Space
```

depending on the selected tab pattern.

---

# 52. Roving Tabindex

In a composite widget:

```text
one item = tabindex="0"
others = tabindex="-1"
```

Moving within the widget:

```text
old → -1
new → 0
new.focus()
```

This keeps the page-level Tab sequence compact.

---

# 53. Roving Tabindex Example

```js
items[current].tabIndex = -1;

current = nextIndex;

items[current].tabIndex = 0;
items[current].focus();
```

This is a common composite-widget technique. citeturn335767search1

---

# 54. `aria-activedescendant`

Instead of physically moving DOM focus, a composite can keep focus on a controlling element and expose the active item using:

```text
aria-activedescendant.
```

This is common in patterns such as:

```text
combobox
```

and certain list/grid designs. citeturn335767search9turn335767search1

---

# 55. Roving Tabindex vs `aria-activedescendant`

### Roving tabindex

```text
DOM focus moves
```

### `aria-activedescendant`

```text
DOM focus stays
AT active item changes
```

Choose based on:

```text
widget semantics
implementation constraints
browser/AT interoperability
complexity.
```

---

# 56. Combobox

A combobox combines:

```text
input/control
+
popup
```

which may be:

```text
listbox
grid
tree
dialog
```

depending on the pattern. citeturn335767search9

---

# 57. Combobox Complexity

A production combobox must coordinate:

```text
input value
popup visibility
highlighted option
selected value
keyboard
mouse/touch
async results
loading state
empty state
focus
announcement
```

This is a:

```text
state machine.
```

---

# 58. Combobox State Machine

Example:

```text
CLOSED
   ↓
OPEN
   ↓
LOADING
   ↓
HAS_RESULTS
   ↓
SELECTION
   ↓
CLOSED
```

Additional states:

```text
EMPTY
ERROR
NO_MATCH
```

All state transitions should update:

```text
visual UI
DOM
ARIA
focus.
```

---

# 59. Async Search Announcements

When suggestions update dynamically:

```text
visual user
→ sees list
```

while:

```text
screen reader user
→ may need announcements.
```

Live regions can communicate relevant state changes. citeturn335767search2

---

# 60. Live Regions

A live region exposes dynamic content changes to assistive technology.

Example:

```html
<div
  id="status"
  role="status"
  aria-live="polite"
></div>
```

Then:

```js
status.textContent = "Saved.";
```

Live regions are specifically intended for dynamic changes that may otherwise be visually apparent but not programmatically announced. citeturn335767search2

---

# 61. Register Live Region Before Updating

Prefer:

```html
<div id="status" role="status"></div>
```

then:

```js
status.textContent = "Saved.";
```

rather than:

```text
create element
with aria-live
and content simultaneously.
```

Assistive technologies generally announce dynamic changes when the live region is established before the update. citeturn335767search2

---

# 62. `polite` vs `assertive`

### `polite`

```text
announce when appropriate
```

### `assertive`

```text
interrupt more aggressively.
```

Use:

```text
assertive
```

sparingly.

Most status information should not:

```text
interrupt the user's current task.
```

---

# 63. Status vs Alert

Use a status-style message for:

```text
Saved
Loading complete
3 results found
```

Use alert-style behavior for:

```text
important immediate feedback
```

But avoid:

```text
alert on every tiny state update.
```

---

# 64. `aria-atomic`

Controls whether the live region announcement should treat:

```text
whole region
```

or:

```text
changed portion
```

as the relevant update.

Use only when:

```text
the announcement semantics actually require it.
```

---

# 65. `aria-relevant`

Can describe which updates are relevant to a live region.

Possible concepts include:

```text
additions
removals
text
```

Do not set every option by default.

---

# 66. Live Region Message Design

Bad:

```text
“.”
```

or:

```text
“Update”
```

Better:

```text
“3 search results available.”
```

Best:

```text
“3 search results available. Use ArrowDown to review them.”
```

when that guidance is actually necessary and not redundant.

---

# 67. Loading State

Avoid visual-only:

```text
spinner.
```

Expose appropriate state:

```text
button disabled if truly unavailable
status/loading message if useful
progress semantics where meaningful.
```

Do not make:

```text
screen reader wait indefinitely.
```

---

# 68. Progress Bars

Native:

```html
<progress value="40" max="100">
```

may be preferable.

For custom progress:

```text
role="progressbar"
aria-valuenow
aria-valuemin
aria-valuemax
```

must be synchronized with the actual state.

---

# 69. Form Error Architecture

A good error system identifies:

```text
field
problem
correction
```

Example:

```html
<input
  id="email"
  aria-invalid="true"
  aria-describedby="email-error"
/>

<p id="email-error">
  Enter a valid email address.
</p>
```

---

# 70. `aria-invalid`

Set:

```text
aria-invalid="true"
```

when the current value fails validation and this state should be exposed.

Do not set:

```text
aria-invalid="true"
```

on every empty required field before the user interacts unless that is a deliberate validation experience.

---

# 71. Error Summary

For multi-field forms, an error summary can:

```text
explain number of errors
link to fields
receive focus
```

Example:

```text
“Form has 3 errors.”
```

followed by:

```text
Email — invalid format
Password — too short
Country — required
```

---

# 72. Focus the First Error?

Often useful after submit:

```text
focus first invalid field
```

or:

```text
focus error summary
```

depending on:

```text
form size
error count
interaction model.
```

The important property is:

```text
error is discoverable
```

without forcing unnecessary focus movement during normal typing.

---

# 73. Validation Timing

Avoid:

```text
aggressive validation on every keystroke
```

that constantly announces:

```text
error
error
error
```

Better:

```text
submit
blur
meaningful state transition
```

depending on the field.

---

# 74. Required Fields

Prefer native:

```html
<input required>
```

when appropriate.

Do not rely only on:

```html
aria-required="true"
```

if native `required` expresses the same behavior and validation semantics.

---

# 75. Fieldsets and Legends

For related controls:

```html
<fieldset>
  <legend>Notification preferences</legend>
  ...
</fieldset>
```

This provides:

```text
group context.
```

---

# 76. Radio Groups

Native:

```html
<input type="radio">
```

usually provides a better baseline than custom radio systems.

Custom radio groups require:

```text
selection state
keyboard navigation
focus behavior
group semantics.
```

---

# 77. Checkboxes

Native checkbox:

```html
<input type="checkbox">
```

already supports:

```text
Space
checked state
form behavior
disabled state
```

Do not replace it with:

```html
<div role="checkbox">
```

without a strong reason.

---

# 78. Custom Range Widgets

Custom sliders require:

```text
role
value
min
max
step
keyboard
```

and pointer interaction.

Native:

```html
<input type="range">
```

is usually safer.

---

# 79. Keyboard Event Design

Do not depend exclusively on:

```js
event.key === "Enter"
```

for every control.

Understand:

```text
Enter
Space
Arrow keys
Home
End
Escape
Tab
Shift+Tab
```

based on the widget pattern.

---

# 80. PreventDefault Discipline

Poor custom widgets often do:

```js
event.preventDefault();
```

for every key.

This can break:

```text
scrolling
text input
browser navigation
assistive technology expectations.
```

Prevent default only when the widget must take control of that behavior.

---

# 81. Pointer + Keyboard Parity

If a mouse can:

```text
open menu
```

keyboard should have an equivalent mechanism.

If a pointer can:

```text
select item
```

keyboard users need:

```text
selection path.
```

Accessibility is:

```text
interaction equivalence
```

not:

```text
mouse support + optional keyboard.
```

---

# 82. Touch Accessibility

Touch interactions can fail when:

```text
targets are tiny
gestures have no simple alternative
drag-only actions exist
```

Provide:

```text
buttons
alternative controls
reasonable target sizing
```

and:

```text
non-gesture alternatives.
```

---

# 83. Drag-and-Drop

A pointer drag is not sufficient.

Provide alternatives such as:

```text
move up
move down
select destination
```

using standard controls.

---

# 84. Keyboard Shortcut Design

Keyboard shortcuts can improve productivity but should not become:

```text
the only access path.
```

APG explicitly advises ensuring basic navigation to functionality before relying on shortcuts. citeturn335767search1

---

# 85. Escape Key

Escape commonly means:

```text
dismiss current transient context
```

such as:

```text
modal
menu
popover
combobox popup.
```

But nested interactions need:

```text
clear ownership
```

of Escape.

---

# 86. Focus Visibility During Async Rendering

Bad SPA behavior:

```text
user focuses button
→ state update
→ component remounts
→ focus disappears.
```

Avoid unnecessary:

```text
DOM replacement.
```

Preserve:

```text
stable elements
and focus state.
```

---

# 87. React/Vue/Svelte/Framework Rendering

Frameworks do not automatically solve accessibility.

You still need:

```text
semantic elements
correct props
focus management
ARIA synchronization
keyboard behavior.
```

A component can compile correctly and remain:

```text
inaccessible.
```

---

# 88. Component Accessibility Contract

Every reusable component should document:

```text
role/semantics
keyboard behavior
focus entry
focus exit
state properties
accessible name
description
disabled behavior
announcement behavior
```

---

# 89. Example Component Contract

```text
Component: Combobox

Name:
provided by label

Focus entry:
Tab enters input

ArrowDown:
opens/advances popup

Escape:
closes popup

Enter:
commits selected option

Selected:
aria-selected

Active option:
aria-activedescendant
```

---

# 90. Accessibility Primitive Library

Build reusable primitives:

```text
FocusScope
Dialog
LiveRegion
VisuallyHidden
RovingTabIndex
KeyboardNavigator
FieldError
FieldDescription
Announcer
```

Centralizing difficult mechanics reduces:

```text
duplicated bugs.
```

---

# 91. Focus Scope

A focus scope manages:

```text
entry
containment
exit
restoration
```

for interactive contexts such as:

```text
dialog
popover
command palette.
```

---

# 92. Announcer Primitive

A global announcer can expose:

```js
announce("Saved successfully");
```

implemented using:

```text
pre-mounted live region
```

This avoids repeatedly creating:

```text
ARIA nodes
```

during every event.

---

# 93. Accessibility State Machine

A complex component should model:

```text
state
→ transition
→ DOM update
→ accessibility state update
→ focus action
→ announcement
```

rather than:

```text
random event handler
changes random attributes.
```

---

# 94. Virtualized Lists

Virtualization means:

```text
only visible DOM nodes exist.
```

Accessibility can become complex because:

```text
offscreen items
```

may not exist in the DOM.

The component must expose:

```text
correct position
active item
set size
orientation
```

where the chosen pattern requires it.

---

# 95. Infinite Scroll

Avoid making:

```text
scrolling
```

the only way to discover more content.

Provide:

```text
Load more
```

or another accessible mechanism when appropriate.

---

# 96. Dynamic Content Replacement

When replacing a region:

```js
container.innerHTML = newHtml;
```

you may accidentally destroy:

```text
focus
selection
event state
accessible relationships.
```

Prefer:

```text
targeted updates
```

and:

```text
preserve stable DOM.
```

---

# 97. `innerHTML` and Accessibility

`innerHTML` is not inherently inaccessible.

The concern is:

```text
what semantic/focus state gets replaced.
```

After DOM replacement verify:

```text
labels
IDs
references
focus
tab order
ARIA relationships.
```

---

# 98. ID Stability

ARIA relationships often use IDs:

```html
aria-labelledby="title-123"
aria-describedby="help-123"
aria-controls="menu-123"
```

Dynamic rendering must preserve:

```text
unique
stable
correctly referenced IDs.
```

---

# 99. Duplicate IDs

Duplicate IDs can break:

```text
label relationships
descriptions
controls
active descendants
```

and produce unpredictable accessibility behavior.

Treat:

```text
duplicate IDs
```

as correctness bugs.

---

# 100. Accessible Name Debugging

When a button announces incorrectly, inspect:

```text
text content
aria-label
aria-labelledby
hidden descendants
SVG labels
title attributes
```

Do not guess.

Use:

```text
browser accessibility inspector.
```

---

# 101. SVG Accessibility

Decorative SVG:

```html
<svg aria-hidden="true">
```

when it should not contribute to the accessible name.

Meaningful SVG requires:

```text
appropriate label/semantics.
```

---

# 102. Icon Buttons

Bad:

```html
<button>
  <svg>...</svg>
</button>
```

if the SVG is not contributing a usable accessible name.

Better:

```html
<button aria-label="Close">
  <svg aria-hidden="true">...</svg>
</button>
```

---

# 103. Icon + Visible Text

If text already names the action:

```html
<button>
  <svg aria-hidden="true">...</svg>
  Save
</button>
```

Do not add:

```text
redundant aria-label
```

unless necessary.

---

# 104. Images

Informative:

```html
<img
  src="report.png"
  alt="Quarterly revenue chart"
>
```

Decorative:

```html
<img src="divider.png" alt="">
```

Do not put:

```text
file123.png
```

as useless alt text.

---

# 105. Background Images

Content-critical images should not rely exclusively on:

```css
background-image
```

because meaningful text alternatives become harder.

Use:

```text
semantic image element
```

when the image carries content.

---

# 106. Error Messages and Live Regions

For a dynamically appearing error:

```text
visual red text
```

is not enough.

Ensure:

```text
relationship to field
and/or appropriate status announcement.
```

Avoid announcing the same error:

```text
three times.
```

---

# 107. Toast Notifications

A toast can be:

```text
status
```

or:

```text
alert
```

depending on importance.

Do not rely on:

```text
color
animation
```

to communicate its meaning.

---

# 108. Toast Focus

A non-critical toast should usually:

```text
not steal focus.
```

Focus theft interrupts:

```text
keyboard workflow
screen-reader cursor/navigation
```

unless the interaction genuinely requires immediate attention.

---

# 109. Notifications

Examples:

```text
“Saved.”
```

usually:

```text
status
```

while:

```text
“Payment failed. Action required.”
```

may require:

```text
stronger error exposure
```

depending on product design.

---

# 110. `prefers-reduced-motion`

Use:

```css
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
    transition: none;
  }
}
```

JavaScript can inspect:

```js
matchMedia("(prefers-reduced-motion: reduce)")
```

to alter behavior when animation itself is programmatic.

---

# 111. Reduced Motion Is Not “No Motion”

A respectful implementation may replace:

```text
large parallax
```

with:

```text
small fade
```

rather than disabling:

```text
all visual feedback.
```

---

# 112. Forced Colors / High Contrast

Some users rely on:

```text
forced colors
high contrast
OS/browser contrast settings.
```

Avoid accessibility designs that require:

```text
specific custom colors
```

to convey all state.

Always include:

```text
shape
text
icon
state
```

as appropriate.

---

# 113. Color Is Not the Only Signal

Bad:

```text
red = error
green = success
```

Better:

```text
red + error icon + message
green + success icon + status text
```

---

# 114. Zoom and Reflow

Test at:

```text
200%
400%
```

as relevant to applicable requirements and layouts.

Do not let:

```text
fixed width containers
```

destroy:

```text
content access
```

when users zoom.

---

# 115. Text Resizing

Avoid:

```css
height: 40px;
overflow: hidden;
```

around text whose size may increase.

Prefer:

```text
content-driven sizing.
```

---

# 116. Mobile Screen Readers

Touch screen readers use gesture navigation that may not match:

```text
keyboard Tab.
```

Test with:

```text
TalkBack
VoiceOver
```

where the product population requires it.

---

# 117. Screen Reader Testing

Automated tools can detect:

```text
missing label
missing alt
some ARIA problems
contrast issues
```

but they cannot fully determine:

```text
whether the workflow is understandable
whether announcements happen at the right time
whether focus movement is logical
```

Manual testing remains necessary.

---

# 118. Automated Accessibility Testing

Common tools can integrate into:

```text
unit tests
component tests
browser tests
CI
```

Examples include:

```text
axe-core
Lighthouse accessibility audits
browser accessibility inspectors
```

Treat them as:

```text
bug detectors
```

not:

```text
proof of accessibility.
```

---

# 119. Why Automated Tests Are Incomplete

Automation may not know:

```text
whether focus belongs here
whether wording is understandable
whether interaction order is intuitive
whether a custom widget feels usable
```

Therefore:

```text
automated
+
manual
+
assistive technology testing
```

is stronger.

---

# 120. Keyboard Test Matrix

For every interactive component:

```text
Tab
Shift+Tab
Enter
Space
Escape
Arrow keys
Home
End
typing/typeahead
```

where applicable.

Also verify:

```text
focus visible
focus order logical
focus not lost
state synchronized.
```

---

# 121. Focus Test Harness

Create a test helper:

```js
function assertFocus(element) {
  if (document.activeElement !== element) {
    throw new Error("Unexpected focus");
  }
}
```

Use around:

```text
dialog open
dialog close
route change
validation
menu navigation.
```

---

# 122. Accessible State Assertions

Test:

```js
expect(button.getAttribute("aria-expanded"))
  .toBe("true");
```

and:

```js
expect(input.getAttribute("aria-invalid"))
  .toBe("true");
```

The test should verify:

```text
DOM/ARIA state
```

matches:

```text
application state.
```

---

# 123. Live Region Tests

Test that:

```text
live region exists before update
message is correct
message appears once
```

Do not test only:

```text
textContent
```

without verifying:

```text
timing/lifecycle
```

of the announcement mechanism.

---

# 124. Accessible Name Tests

Test:

```text
button accessible name
input label
dialog name
tab name
menu item name
```

Use browser/automation accessibility APIs where available.

---

# 125. Component Test Contract

Every accessible component should test:

```text
semantics
keyboard
focus
state
announcement
disabled state
error state
```

---

# 126. End-to-End Accessibility Test

A strong test:

```text
open page
→ keyboard navigate
→ activate control
→ observe state
→ perform async operation
→ verify status
→ close context
→ verify focus restoration.
```

---

# 127. Browser Accessibility Inspector

Use developer tools to inspect:

```text
role
name
description
state
focused node
relationships.
```

When debugging accessibility:

```text
inspect the accessibility tree
```

rather than looking only at:

```text
DOM source.
```

---

# 128. Accessibility Tree vs DOM Tree

The accessibility tree is:

```text
semantic projection
```

not:

```text
DOM clone.
```

A DOM element may be:

```text
not exposed
```

or:

```text
merged into semantic representation.
```

---

# 129. Debugging Accessibility

Use this order:

```text
1. Is the correct native element used?
2. Is it reachable by keyboard?
3. Is focus logical?
4. Does it have an accessible name?
5. Does role match behavior?
6. Are state properties synchronized?
7. Are dynamic changes announced if needed?
8. Is focus restored correctly?
9. Does it work with assistive technology?
10. Does automation confirm the contract?
```

---

# 130. Progressive Enhancement

Start with:

```text
semantic HTML
```

then add:

```text
CSS
JavaScript enhancement
ARIA only where necessary.
```

A basic form should remain understandable if:

```text
JavaScript fails
```

when product architecture permits.

---

# 131. Failure Mode: JS Disabled

Ask:

```text
Can user read content?
Can user navigate?
Can user submit important data?
Can user understand errors?
```

Accessibility often improves when:

```text
baseline HTML
```

is solid.

---

# 132. Failure Mode: Hydration

SSR HTML may be accessible:

```text
before JavaScript.
```

Hydration can temporarily or permanently break:

```text
labels
IDs
focus
event handlers
```

Test:

```text
pre-hydration
hydration
post-hydration.
```

---

# 133. Failure Mode: Re-Mounting

A component can re-render by replacing the focused element:

```text
input old
→ destroy
→ input new
```

The user's focus disappears.

Prefer:

```text
stable identity
```

for focused elements.

---

# 134. Failure Mode: Portal

Portals can move elements:

```text
outside visual DOM hierarchy.
```

This can complicate:

```text
focus
aria-controls
descriptions
dialog containment
```

Test the resulting:

```text
accessibility tree
```

not only the source component hierarchy.

---

# 135. Failure Mode: Shadow DOM

Shadow DOM creates encapsulation boundaries.

Accessibility semantics can cross boundaries in defined ways, but components must be tested as:

```text
user-facing controls
```

rather than assuming:

```text
internal DOM = exposed semantics.
```

---

# 136. Custom Element Accessibility

A custom element should expose:

```text
name
role
state
keyboard behavior
focus
```

while maintaining:

```text
internal implementation freedom.
```

Create a stable:

```text
public accessibility contract.
```

---

# 137. Accessibility and Design Systems

A design system should standardize:

```text
button
input
dialog
popover
menu
tabs
tooltip
combobox
alert/status
```

and their:

```text
keyboard patterns
focus behavior
state APIs
```

not just:

```text
colors
spacing
typography.
```

---

# 138. Accessible Component API Design

Prefer APIs that make accessible behavior easy:

```js
<Dialog
  title="Delete account"
  open={open}
  onClose={close}
/>
```

and make difficult mistakes hard.

Avoid APIs that require each consumer to manually remember:

```text
aria-labelledby
focus return
Escape
body inerting.
```

---

# 139. Accessibility by Construction

A principal design system should make:

```text
correctness default.
```

Examples:

```text
Dialog always has a name.
Button requires accessible content.
Field requires label.
Tabs implement keyboard behavior.
```

---

# 140. Linting

Use lint rules to detect:

```text
click-only handlers
missing button semantics
invalid ARIA
missing labels
```

But linting cannot prove:

```text
workflow correctness.
```

---

# 141. CI Accessibility Gate

A reasonable CI pipeline:

```text
lint
→ component accessibility tests
→ automated axe-style scan
→ keyboard E2E
→ visual regression
→ targeted manual review.
```

---

# 142. Accessibility Regression Testing

Regression suite should preserve:

```text
role
name
state
focus
keyboard
announcement
```

across:

```text
refactors
framework upgrades
design changes.
```

---

# 143. Performance vs Accessibility

Do not “optimize” by:

```text
removing semantics
removing labels
removing announcements
```

Instead measure:

```text
DOM cost
observer cost
render cost
announcement frequency
```

and optimize safely.

---

# 144. Accessibility vs Performance

Large live-region updates can create:

```text
excessive announcements
```

while:

```text
huge virtualized DOM
```

can create:

```text
screen-reader complexity.
```

The principal solution balances:

```text
performance
+
accessibility
+
usability.
```

---

# 145. Accessibility vs Security

Do not leak sensitive information through:

```text
aria-label
status messages
live regions
```

that a user was not supposed to access.

Accessibility metadata is still:

```text
application data.
```

---

# 146. Accessibility and Privacy

Avoid announcing:

```text
secret tokens
full personal records
authentication recovery codes
```

through global status channels.

A screen reader user must receive:

```text
necessary information
```

without:

```text
unintended disclosure.
```

---

# 147. Authentication Accessibility

Authentication flows should avoid:

```text
unnecessary memory challenges
keyboard traps
time-dependent inaccessible steps
```

and should communicate:

```text
errors
requirements
next actions.
```

---

# 148. Timeouts

If a session expires:

```text
notify user
```

before destructive timeout where the interaction pattern requires it.

Provide:

```text
extension / recovery
```

where appropriate.

---

# 149. Dragging Alternative

A reorderable list should support:

```text
Move up
Move down
Move to...
```

in addition to:

```text
drag
```

where applicable.

---

# 150. Auto-Refresh

Avoid silently replacing content while the user is reading.

For dynamic updates:

```text
preserve context
announce meaningful changes
allow pause/refresh
```

when required by the experience.

---

# 151. Carousels

Carousels can create accessibility complexity:

```text
auto-advance
focus movement
slide announcements
controls
```

Prefer:

```text
user-controlled navigation
```

and avoid:

```text
unexpected focus stealing.
```

---

# 152. Accessible Infinite Data

For live feeds:

```text
“20 new items available”
```

may be better than:

```text
immediately injecting 20 items above the user's position.
```

Let users:

```text
choose when to load
```

where appropriate.

---

# 153. Scroll Position

SPA transitions should preserve or intentionally reset:

```text
scroll position
focus position
reading context.
```

Do not let:

```text
focus
```

and:

```text
scroll
```

fight each other.

---

# 154. Accessibility State as Source of Truth

Avoid:

```text
visual open = true
aria-expanded = false
```

Instead derive both from:

```text
one application state.
```

Example:

```js
const open = state.isOpen;

button.toggleAttribute("aria-expanded", open);
panel.hidden = !open;
```

---

# 155. Single-State Principle

Use:

```text
application state
```

to derive:

```text
visual state
ARIA state
focus behavior
announcements.
```

This reduces:

```text
state divergence.
```

---

# 156. Accessibility Event Architecture

A complex interaction can be modeled:

```text
USER ACTION
    ↓
STATE TRANSITION
    ↓
DOM UPDATE
    ↓
ARIA UPDATE
    ↓
FOCUS UPDATE
    ↓
ANNOUNCEMENT
```

Missing one stage can create:

```text
partial accessibility.
```

---

# 157. Implementation From Scratch — Focus Manager

Build:

```text
FocusManager
```

with:

```text
capture()
focus()
restore()
contains()
```

State:

```js
{
  previousActiveElement,
  currentScope
}
```

---

# 158. Focus Manager Milestone 1

Implement:

```js
capture();
```

which records:

```js
document.activeElement
```

before opening a transient UI.

---

# 159. Focus Manager Milestone 2

Implement:

```js
focusFirst(scope);
```

that finds:

```text
first valid focus target.
```

Account for:

```text
disabled
hidden
inert
tabindex
```

where appropriate.

---

# 160. Focus Manager Milestone 3

Implement:

```js
restore();
```

which:

```text
checks target still exists
checks target is focusable/usable
falls back safely
```

---

# 161. Focus Trap Milestone

Build:

```text
Tab containment
Shift+Tab containment
Escape
```

Then test:

```text
first
middle
last
```

focus positions.

---

# 162. Live Announcer Implementation

Build:

```js
const region =
  document.createElement("div");

region.setAttribute("role", "status");
region.setAttribute("aria-live", "polite");

document.body.append(region);
```

Then:

```js
function announce(message) {
  region.textContent = "";
  queueMicrotask(() => {
    region.textContent = message;
  });
}
```

Test carefully with actual assistive technology; announcement delivery behavior can vary.

---

# 163. Roving Tabindex Implementation

Implement:

```js
function activate(index) {
  items.forEach((item, i) => {
    item.tabIndex = i === index ? 0 : -1;
  });

  items[index].focus();
}
```

Then add:

```text
ArrowRight
ArrowLeft
Home
End
```

based on the widget pattern.

---

# 164. Accessible Tabs Implementation

Implement:

```text
tablist
tab
tabpanel
```

with:

```text
aria-selected
aria-controls
aria-labelledby
tabpanel tabindex when needed
```

and:

```text
roving tabindex.
```

---

# 165. Accessible Dialog Implementation

Build:

```text
open()
focus()
contain()
close()
restore()
```

Test:

```text
mouse
keyboard
Escape
screen reader
nested dialog if supported.
```

---

# 166. Accessible Combobox Implementation

Build states:

```text
closed
open
loading
empty
results
selected
error
```

Support:

```text
typing
ArrowDown
ArrowUp
Enter
Escape
selection
```

and expose:

```text
name
expanded
popup
active descendant
```

as required by the selected pattern.

---

# 167. Accessibility Test Harness

Build reusable helpers:

```js
expectRole()
expectName()
expectFocused()
expectAria()
pressKey()
announceAndFlush()
```

This turns:

```text
manual principles
```

into:

```text
repeatable engineering tests.
```

---

# 168. Manual Test Matrix

For every production component test:

```text
mouse
keyboard
touch
screen reader
zoom
reduced motion
forced colors/high contrast where relevant
```

---

# 169. Screen Reader Matrix

Where product support requires:

```text
NVDA + Firefox/Chrome
JAWS + Chrome/Edge
VoiceOver + Safari
TalkBack + Android browser
```

Do not assume:

```text
one screen reader
```

represents:

```text
all assistive technology.
```

---

# 170. Browser Compatibility

Accessibility behavior is a combination of:

```text
HTML
DOM
browser
ARIA
OS accessibility APIs
assistive technology
```

Therefore:

```text
“works in Chrome”
```

does not guarantee:

```text
“works everywhere.”
```

---

# 171. Standards Hierarchy

Use:

```text
HTML Standard
WAI-ARIA
ARIA Authoring Practices Guide
WCAG
browser documentation
assistive technology testing
```

as appropriate.

APG is guidance and examples, not itself the normative ARIA standard. citeturn335767search3

---

# 172. WCAG Mental Model

WCAG is commonly organized around:

```text
Perceivable
Operable
Understandable
Robust
```

For JavaScript engineers, especially relevant concerns include:

```text
keyboard operation
focus
labels
names
status messages
error identification
consistent interaction
compatibility.
```

---

# 173. WCAG Is Not “A Checklist of ARIA”

Meeting accessibility expectations requires:

```text
content
design
interaction
code
testing
```

not just:

```text
ARIA attributes.
```

---

# 174. Accessibility Conformance vs Usability

A technically conforming interface can still be:

```text
confusing
slow
verbose
hard to learn.
```

Accessibility engineering should therefore target:

```text
conformance
+
usability.
```

---

# 175. User Research

Where possible, test with:

```text
people who use assistive technologies.
```

Real users reveal problems that:

```text
checklists
```

and:

```text
automation
```

miss.

---

# 176. Accessibility Bug Taxonomy

Classify bugs:

```text
semantic
keyboard
focus
announcement
state
label/name
contrast/visual
timing
responsive/reflow
AT compatibility
```

This improves:

```text
triage
ownership
root-cause analysis.
```

---

# 177. Accessibility Incident Response

When a production accessibility regression occurs:

```text
1. identify affected workflow
2. identify assistive technology/population
3. reproduce
4. inspect accessibility tree
5. identify broken state/focus/semantics
6. patch
7. regression test
8. audit adjacent components
```

---

# 178. Accessibility Metrics

Useful engineering metrics include:

```text
accessibility defect rate
keyboard test coverage
component compliance
open critical accessibility bugs
time-to-fix
regressions per release
```

Avoid treating:

```text
“automated score = 100”
```

as the accessibility KPI.

---

# 179. Accessibility Definition of Done

A component is not done until:

```text
[ ] semantic element chosen
[ ] accessible name defined
[ ] keyboard path implemented
[ ] focus behavior defined
[ ] state exposed
[ ] dynamic updates handled
[ ] errors handled
[ ] screen reader tested
[ ] automated checks pass
[ ] browser compatibility assessed
[ ] regression tests added
```

---

# 180. Code Review Exercise

Review:

```js
card.addEventListener("click", openDetails);

card.setAttribute("role", "button");
card.setAttribute("tabindex", "0");

card.addEventListener("keydown", event => {
  if (event.key === "Enter") {
    openDetails();
  }
});
```

Find problems:

```text
div/card may be replaceable with native button
Space behavior missing
semantic action unclear
focus style not addressed
possible duplicate interaction with nested controls
event semantics may diverge from native button
```

Then redesign with:

```html
<button type="button">
```

when the action is genuinely a button.

---

# 181. Code Review Exercise — Modal

Review:

```js
modal.hidden = false;

document.body.append(modal);

modal.querySelector("input").focus();

modal.addEventListener("keydown", e => {
  if (e.key === "Escape") {
    modal.hidden = true;
  }
});
```

Find:

```text
no focus containment
no focus restoration
no dialog name
no background inertness/appropriate modality handling
possible listener accumulation
no initial-focus strategy
no close-button semantics
```

---

# 182. Code Review Exercise — Live Region

Review:

```js
function showToast(message) {
  const div = document.createElement("div");

  div.setAttribute("aria-live", "assertive");
  div.textContent = message;

  document.body.append(div);
}
```

Find:

```text
assertive may be too aggressive
announcement region may be created too late
lifecycle/cleanup not defined
duplicate announcements possible
toast may steal visual attention
message sensitivity not considered.
```

---

# 183. Debugging Exercises

## Exercise A — Button Announces “Button”

A screen reader says:

```text
“button”
```

without an action name.

Inspect:

```text
accessible name
child SVG
aria-hidden
text
```

and fix it.

---

## Exercise B — Focus Vanishes

After opening a menu:

```text
focus disappears.
```

Inspect:

```text
mount timing
DOM replacement
tabindex
focus call
```

---

## Exercise C — Modal Escape

Escape closes the dialog but:

```text
focus remains in removed DOM.
```

Fix:

```text
focus restoration.
```

---

## Exercise D — Search Results

Visual UI shows:

```text
5 results
```

screen reader gets:

```text
nothing.
```

Implement:

```text
status/live region.
```

---

## Exercise E — Validation

Visual form shows:

```text
Email invalid.
```

but screen reader cannot associate it with:

```text
Email input.
```

Fix:

```text
aria-invalid
aria-describedby
```

and the associated error element.

---

## Exercise F — Keyboard Trap

A custom picker works with:

```text
mouse
```

but:

```text
Tab
```

cannot escape.

Find:

```text
focus containment
tabindex
keydown prevention.
```

---

# 184. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```html
<div
  role="button"
  tabindex="0"
>
  Save
</div>
```

Question:

```text
Does this automatically behave exactly like a native button?
```

---

### Exercise 2

```html
<button aria-expanded="false">
  Filters
</button>
```

JavaScript opens the panel but does not update:

```text
aria-expanded.
```

Predict:

```text
visual vs semantic state mismatch.
```

---

### Exercise 3

```html
<div aria-live="polite">
  Loading...
</div>
```

Then replace the element itself.

Predict:

```text
why announcement behavior may differ from updating
an already-existing live region.
```

---

### Exercise 4

A modal opens and focus remains on:

```text
background button.
```

Predict:

```text
what a keyboard and screen-reader user experiences.
```

---

### Exercise 5

A component continuously sets:

```js
input.focus();
```

after every render.

Predict:

```text
why typing and navigation become frustrating.
```

---

# 185. Interview Questions

### Fundamentals

```text
1. What is accessibility engineering?
2. What is semantic HTML?
3. What is the accessibility tree?
4. What is an accessible name?
5. Why is native HTML preferred?
```

### ARIA

```text
6. What is ARIA?
7. What does ARIA not automatically provide?
8. What is the first rule of ARIA?
9. What is aria-expanded?
10. What is aria-hidden?
```

### Keyboard

```text
11. What is tabindex?
12. Why are positive tabindex values discouraged?
13. What is roving tabindex?
14. What is aria-activedescendant?
15. How would you keyboard-enable a custom component?
```

### Focus

```text
16. What is focus management?
17. Why restore focus after a modal?
18. How do you manage SPA route focus?
19. How do you prevent focus loss during rerendering?
20. When should focus move programmatically?
```

### Dynamic Content

```text
21. What are live regions?
22. What is aria-live?
23. What is the difference between status and alert?
24. When would you use aria-atomic?
25. Why should a live region exist before its content changes?
```

### Forms

```text
26. How do you build accessible validation?
27. What is aria-invalid?
28. What is aria-describedby?
29. Why is placeholder not a label?
30. How do error summaries work?
```

### Principal

```text
31. How would you design an accessible component library?
32. How would you test a custom combobox?
33. How would you debug an accessibility regression in a SPA?
34. How would you balance performance and live announcements?
35. How would you make an accessibility contract part of CI?
36. How would you design focus management across portals and modals?
37. How would you support both keyboard and screen-reader navigation for a complex grid?
```

---

# 186. Mastery Exercises

### Exercise 1 — Accessible Dialog

Build:

```text
dialog
focus entry
focus containment
Escape
restore
accessible name
background inertness
```

### Exercise 2 — Accessible Tabs

Build:

```text
tablist
tabs
panels
roving tabindex
selection state
keyboard navigation
```

### Exercise 3 — Accessible Combobox

Build:

```text
input
async suggestions
loading
empty
selection
keyboard
aria-expanded
aria-controls
aria-activedescendant or roving pattern
```

### Exercise 4 — Form System

Build:

```text
label
description
validation
aria-invalid
describedby
error summary
focus management
status.
```

### Exercise 5 — Accessibility Harness

Build:

```text
role/name assertions
focus assertions
keyboard helpers
live-region assertions
```

### Exercise 6 — Design System Contract

Create an accessibility contract for:

```text
Button
Dialog
Menu
Tabs
Combobox
Tooltip
Toast
FormField
```

### Exercise 7 — Accessibility Incident

Simulate:

```text
release
→ keyboard users cannot submit checkout
```

Diagnose:

```text
semantic
focus
event
state
testing
```

issues.

---

# 187. Track A — Core Theory

Master:

```text
semantic HTML
accessibility tree
accessible name
description
role
state
ARIA
keyboard interfaces
focus
tabindex
focus restoration
composite widgets
live regions
forms
dynamic DOM
virtualization
screen readers
testing
WCAG concepts
```

Deliverable:

```text
explain exactly what a user and assistive technology receive from a component.
```

---

# 188. Track B — Implementation

Build:

```text
FocusManager
FocusScope
Dialog
LiveRegion
Announcer
RovingTabIndex
Tabs
Menu
Listbox
Combobox
AccessibleFormField
ErrorSummary
Keyboard test harness
Accessibility CI checks
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

# 189. Track C — Interview / Reasoning

Practice:

```text
“Why isn't role=button enough?”

“How would you manage focus in a modal?”

“When should you use aria-activedescendant?”

“Why are positive tabindex values discouraged?”

“How would you build an accessible combobox?”

“How do live regions work?”

“How do you test accessibility beyond automation?”

“What accessibility contracts belong in a design system?”
```

Deliverable:

```text
semantic reasoning
+
interaction reasoning
+
focus reasoning
+
AT reasoning
+
trade-off.
```

---

# 190. Principal Decision Framework

For every component ask:

```text
1. What is the native HTML equivalent?
2. Why is a custom control necessary?
3. What is its accessible name?
4. What is its role?
5. What are its states?
6. What is the keyboard contract?
7. Where does focus enter?
8. Where does focus move?
9. Where does focus leave?
10. Is there dynamic state that must be announced?
11. What happens when JavaScript fails?
12. What happens on rerender/remount?
13. What happens with touch?
14. What happens with zoom/reflow?
15. What happens with reduced motion?
16. What happens with a screen reader?
17. What browser/AT combinations matter?
18. What can automation detect?
19. What still requires manual testing?
20. How is the contract enforced in the component API?
21. How is it enforced in CI?
22. What is the regression strategy?
```

---

# 191. Production Accessibility Checklist

```text
[ ] native element considered first
[ ] semantic HTML correct
[ ] accessible name defined
[ ] accessible description defined where needed
[ ] role correct
[ ] state correct
[ ] keyboard access implemented
[ ] focus visible
[ ] focus order logical
[ ] programmatic focus justified
[ ] focus restoration implemented
[ ] no positive tabindex
[ ] aria-hidden used safely
[ ] inert/containment handled appropriately
[ ] labels associated
[ ] errors associated
[ ] aria-invalid synchronized
[ ] live regions intentional
[ ] announcements not excessive
[ ] dynamic content discoverable
[ ] pointer has keyboard equivalent
[ ] drag has alternative
[ ] reduced motion supported
[ ] zoom/reflow tested
[ ] forced colors/high contrast considered
[ ] screen reader tested
[ ] automated accessibility testing runs
[ ] keyboard E2E runs
[ ] accessibility tree inspected
[ ] component contract documented
[ ] CI regression checks present
[ ] browser/AT compatibility documented
```

---

# 192. Specification / Source Discipline

Primary sources:

```text
HTML Standard
WAI-ARIA
WAI-ARIA Authoring Practices Guide
WCAG
browser accessibility documentation
assistive technology documentation
```

Use APG to understand:

```text
patterns
keyboard interaction
focus management
roles/states
```

APG explicitly describes itself as guidance synthesized from multiple specifications rather than the normative standard itself. citeturn335767search3

WAI-ARIA defines semantics for roles, states, properties, live regions, and related accessibility mappings, while authors still need to implement keyboard interaction for custom widgets. citeturn335767search0turn335767search1

---

# 193. Current Platform Notes

As of September 2026:

```text
semantic HTML remains the preferred foundation
for accessible web interfaces.

WAI-ARIA continues to provide semantics for
custom/dynamic application interfaces.

APG continues to provide authoring guidance
for common interactive patterns.

Modern browsers support major accessibility
platform primitives including focus management,
ARIA, inert, dialog-related capabilities,
and accessibility inspection.

Exact assistive-technology behavior varies by:
browser
OS
screen reader
device
widget implementation.
```

Do not promise:

```text
universal behavior
```

from:

```text
one browser + one screen reader.
```

---

# 194. Common Misconceptions

### Misconception 1

```text
“ARIA makes anything accessible.”
```

Reality:

```text
ARIA changes semantics.
JavaScript still needs to implement behavior.
```

### Misconception 2

```text
“Tabindex=0 makes a div accessible.”
```

Reality:

```text
focusability ≠ complete widget semantics/behavior.
```

### Misconception 3

```text
“Lighthouse 100 means accessibility is solved.”
```

Reality:

```text
automated tools catch only a subset of issues.
```

### Misconception 4

```text
“Screen readers just read the DOM.”
```

Reality:

```text
assistive technologies consume browser/platform
accessibility representations.
```

### Misconception 5

```text
“Visually hidden means inaccessible.”
```

Reality:

```text
visual visibility and accessibility exposure
are related but distinct concepts.
```

---

# 195. Common Mistakes

```text
[ ] using div instead of button
[ ] using link as button
[ ] missing visible labels
[ ] positive tabindex
[ ] focus theft
[ ] focus loss
[ ] no focus restoration
[ ] aria-hidden focus traps
[ ] inconsistent aria-expanded
[ ] live-region spam
[ ] click-only interactions
[ ] no keyboard alternative for drag
[ ] replacing focused DOM nodes
[ ] random IDs breaking relationships
[ ] duplicate IDs
[ ] incomplete custom comboboxes
[ ] treating menu as navigation
[ ] assuming automation proves accessibility
[ ] assuming one screen reader is enough
[ ] relying only on color
[ ] motion without reduced-motion handling
```

---

# 196. Performance Considerations

Accessibility implementation can affect:

```text
DOM size
layout
rendering
announcement frequency
event listeners
observer count
```

Optimize by:

```text
stable DOM
bounded state
minimal announcement frequency
reused accessibility primitives
delegated events
efficient virtualization
```

but never remove necessary semantics simply to reduce:

```text
attribute count.
```

---

# 197. Memory Considerations

Watch for:

```text
focus manager retaining DOM references
modal stacks never cleared
event listeners accumulating
live-region queues growing
keyboard handler registries leaking
component unmount cleanup missing.
```

Use:

```text
cleanup
WeakMap where appropriate
bounded arrays
disposable scopes
```

---

# 198. Security Considerations

Accessibility metadata can expose:

```text
sensitive application state
private notifications
user data
```

Do not announce:

```text
passwords
tokens
hidden account details
```

through global live regions.

Treat accessibility output as:

```text
user-visible data.
```

---

# 199. Reliability Considerations

A production accessibility layer should tolerate:

```text
component unmount
navigation
async race
double close
missing focus target
cancelled request
portal relocation
browser differences.
```

Every operation should be:

```text
safe to repeat or safely reject
```

where practical.

---

# 200. Final Accessibility Mental Model

Use:

```text
NATIVE HTML
    ↓
SEMANTICS
    ↓
ACCESSIBILITY TREE
    ↓
KEYBOARD / POINTER / TOUCH
    ↓
FOCUS
    ↓
STATE
    ↓
DYNAMIC ANNOUNCEMENT
    ↓
ASSISTIVE TECHNOLOGY
    ↓
USER OUTCOME
```

For a complex component:

```text
user action
→ state transition
→ DOM update
→ semantic state update
→ focus update
→ announcement
→ verification.
```

---

# 201. Accessibility Debugging Mental Model

When a component fails:

```text
Is the element semantically correct?
        ↓
Can the user reach it?
        ↓
Can the user operate it?
        ↓
Can the user identify it?
        ↓
Can the user understand its state?
        ↓
Can the user discover dynamic changes?
        ↓
Can the user recover from errors?
        ↓
Can focus move predictably?
        ↓
Does assistive technology receive the same truth?
```

---

# 202. Dependency Graph

```text
Chapter 33
Browser Event Loop
        ↓
Chapter 49
DOM
        ↓
Chapter 50
Events
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 55
Fetch
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Production Debugging
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
Chapter 88
Debugging
        ↓
Chapter 101
Production Scenarios
        ↓
Chapter 133
Service Workers
        ↓
Chapter 137
Browser Performance APIs
        ↓
Chapter 138
Accessibility Engineering With JavaScript
```

Cross-cutting:

```text
DOM semantics
keyboard events
focus
forms
browser security
performance
testing
design systems
observability
```

---

# 203. Concept Connections

## Depends On

```text
DOM
HTML semantics
browser events
focus
forms
CSS
browser accessibility APIs
JavaScript state management
testing
```

## Builds Toward

```text
accessible design systems
inclusive component libraries
production frontend platforms
robust UI architecture
assistive-technology compatibility
```

## Related Concepts

```text
WAI-ARIA
ARIA APG
WCAG
screen readers
keyboard navigation
focus management
live regions
semantic HTML
design systems
```

## Concepts Revisited

```text
DOM
Events
Forms
Async Rendering
State Machines
Testing
Performance
Security
Observability
Component Architecture
```

## Why This Chapter Matters

A JavaScript application is not accessible because:

```text
it renders
```

or:

```text
it passes an automated scan.
```

It is accessible when the user can:

```text
find the interface
understand what it is
operate it
understand state
recover from errors
navigate predictably
```

using the interaction modality and assistive technology available to them.

---

# 204. Retrieval Record

```md
# Chapter 138 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Semantic HTML
-

## Accessibility Tree
-

## Accessible Name
-

## Roles / States / Properties
-

## ARIA
-

## Keyboard
-

## Focus
-

## tabindex
-

## Focus Restoration
-

## Dialogs
-

## Composite Widgets
-

## Roving Tabindex
-

## aria-activedescendant
-

## Combobox
-

## Tabs
-

## Menus
-

## Live Regions
-

## Forms
-

## Validation
-

## SPA Navigation
-

## Reduced Motion
-

## Screen Readers
-

## Automated Testing
-

## Manual Testing
-

## Design System
-

## WCAG
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

# 205. Spaced Retrieval Schedule

### Day 0

Study:

```text
semantic HTML
ARIA
accessible names
keyboard/focus
```

### Day 1

Explain:

```text
native vs custom
tabindex
focus restoration
live regions.
```

### Day 3

Build:

```text
accessible dialog.
```

### Day 7

Build:

```text
tabs
menu
roving tabindex.
```

### Day 14

Build:

```text
combobox
+
form validation.
```

### Day 21

Run:

```text
screen-reader
keyboard
automated
```

testing on the entire mini application.

### Day 30

Perform a:

```text
principal accessibility architecture review
```

without notes.

---

# 206. Completion Criteria

You may mark this chapter:

```text
[~] In Progress
```

when you can:

```text
explain semantics
implement keyboard behavior
```

Mark:

```text
[?] Needs Revision
```

when:

```text
you repeatedly confuse ARIA semantics with behavior
or lose focus during component changes.
```

Mark:

```text
[+] Completed
```

when you can:

```text
build and test accessible components without a reference.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
design
implement
debug
test
review
and defend
```

an accessibility architecture across:

```text
keyboard
screen reader
mobile
desktop
SPA routing
dialogs
forms
async content
custom widgets
```

Reading alone does not mark mastery.

---

# 207. Final Principal Principle

> **Accessibility is the synchronization of meaning, interaction, state, and focus across the visual interface, browser semantics, and assistive technology.**

The engineering sequence is:

```text
choose correct native semantics
→ define accessible name/state
→ provide keyboard access
→ manage focus
→ expose dynamic changes
→ preserve state consistency
→ test with automation
→ test manually
→ validate assistive technology
→ enforce the component contract
→ monitor regressions
```

The most important distinction is:

```text
SEMANTICS
≠
BEHAVIOR
≠
VISUALS
```

A robust component aligns all three:

```text
visual state
=
DOM state
=
accessibility state
```

and provides:

```text
mouse
+
keyboard
+
touch
+
assistive technology
```

paths to the same user goal.

At principal level, ask:

```text
“What is the simplest native semantic model that exposes the
correct meaning and behavior, how does focus move through the
interaction, what state must assistive technology know, what
changes dynamically, how will users discover those changes,
and how will we prove the contract still works after the next
refactor?”
```

That is accessibility engineering.