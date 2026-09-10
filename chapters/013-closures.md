
# Chapter 13 — Closures

> **Chapter Status:** `[+] Completed`
>
> **Prerequisites:** Chapters 09–12 — Functions, Scope, Hoisting/TDZ, and Execution Contexts
>
> **Next:** Chapter 14 — `this`, Invocation, and Binding

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- define a closure precisely as a function together with lexical access to the environment needed by that function;
- distinguish lexical scope from closure behavior;
- explain why a nested function can access an outer binding after the outer call has completed;
- distinguish captured bindings from copied snapshots;
- explain why two factory calls can produce independent state;
- explain why multiple closures created in one invocation can share state;
- trace closure behavior through nested environments and shadowing;
- reason about closure lifetime using reachability rather than stack-frame folklore;
- understand closures in callbacks, timers, promises, event listeners, memoization, debounce, dependency injection, and factories;
- explain `var` versus `let` loop closure behavior;
- identify stale-closure problems;
- identify accidental memory retention through long-lived callbacks;
- distinguish intentional retention from a memory leak;
- compare closure-based state with object properties and private class fields;
- understand how closures interact with `this`, `arguments`, `eval`, and modules;
- distinguish ECMAScript closure semantics from engine implementation strategies;
- implement a simplified closure/environment interpreter;
- debug closure correctness and memory issues systematically;
- defend closure architecture choices at senior/principal level.

---

# 2. Prerequisites

You should already understand:

```text
Chapter 09
functions are values, callbacks, factories

Chapter 10
lexical scope, environments, identifier resolution

Chapter 11
binding creation, initialization, TDZ

Chapter 12
execution contexts, call stack, completion, suspension
```

The key progression is:

```text
function
   ↓
lexical environment
   ↓
binding resolution
   ↓
function survives beyond creator execution
   ↓
captured environment remains reachable
   ↓
closure
```

---

# 3. What Is a Closure?

A closure is a function that retains the lexical access needed to resolve references to variables from its surrounding scope.

Example:

```js
function makeCounter() {
  let count = 0;

  return function counter() {
    count++;
    return count;
  };
}

const counter = makeCounter();

console.log(counter()); // 1
console.log(counter()); // 2
```

The most useful precise explanation is:

```text
counter is a function created in the lexical environment of makeCounter.
That function retains access to the `count` binding.
The active makeCounter execution completes.
The binding remains reachable through the surviving function.
```

Avoid the weaker explanation:

```text
"the function remembers the variable"
```

That phrase is useful as intuition, but not as a complete semantic model.

---

# 4. Why Do Closures Exist?

Closures naturally follow from three language properties:

```text
1. lexical scoping
2. first-class functions
3. functions can outlive the execution that created them
```

Suppose JavaScript allowed:

```js
function outer() {
  const x = 10;

  return () => x;
}
```

If the returned function lost access to `x` after `outer()` returned, lexical scoping would become inconsistent.

Closures enable:

- stateful factories;
- callbacks;
- encapsulation;
- memoization;
- currying and partial application;
- dependency injection;
- event handlers;
- asynchronous continuations;
- module patterns;
- iterators;
- capability-style APIs.

---

# 5. Mental Model

Consider:

```js
function outer() {
  const value = 42;

  return function inner() {
    return value;
  };
}

const fn = outer();
```

Think:

```text
outer invocation
└── lexical environment
    └── value → 42

inner function
└── lexical access to outer environment

fn
└── inner function
    └── retained environment access
```

When `outer()` returns:

```text
outer execution context → completed
```

but:

```text
inner → outer lexical state
```

remains relevant because `inner` can still read `value`.

---

# 6. Closure Captures Bindings, Not Simply Values

This is fundamental.

```js
let x = 1;

const read = () => x;

x = 2;

console.log(read());
```

Result:

```text
2
```

The closure did not freeze the number `1`.

It retained access to the binding:

```text
x
```

whose current value became:

```text
2
```

A real snapshot requires a separate binding:

```js
let x = 1;

const snapshot = x;
const read = () => snapshot;

x = 2;

console.log(read()); // 1
```

Here:

```text
x → 2
snapshot → 1
```

The difference comes from bindings, not from some special “snapshot mode” of closures.

---

# 7. Multiple Closures Sharing One Binding

Example:

```js
function createState() {
  let value = 0;

  return {
    get() {
      return value;
    },

    set(next) {
      value = next;
    },

    increment() {
      value++;
    }
  };
}

const state = createState();

state.increment();
state.increment();

console.log(state.get()); // 2
```

All three methods can access the same binding:

```text
environment
└── value → 2

get ─────────┐
set ─────────┼── same lexical state
increment ───┘
```

This is a core closure pattern for controlled mutable state.

---

# 8. Separate Factory Calls Create Separate State

Example:

```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const a = makeCounter();
const b = makeCounter();

console.log(a()); // 1
console.log(a()); // 2
console.log(b()); // 1
console.log(b()); // 2
```

Why?

Each call creates a new lexical environment:

```text
makeCounter #1
└── count → 2

makeCounter #2
└── count → 2
```

The numeric values match, but the bindings are different.

---

# 9. Scope vs Closure

Scope exists even when a function never escapes.

```js
function f() {
  const x = 10;
  console.log(x);
}
```

This has lexical scope.

A closure becomes especially significant when a function retains lexical access beyond the normal active execution period of the surrounding code:

```js
function f() {
  const x = 10;
  return () => x;
}
```

Useful model:

```text
lexical scope
+
first-class function
+
survival of function
=
observable closure behavior
```

---

# 10. Closure and Identifier Resolution

Closures do not bypass lexical resolution.

Example:

```js
const value = "global";

function outer() {
  const value = "outer";

  return function inner() {
    return value;
  };
}
```

`inner()` returns:

```text
"outer"
```

because its lexical resolution path is:

```text
inner environment
   ↓
outer environment
   ↓
global
```

The first matching binding wins.

---

# 11. Closure and Shadowing

Example:

```js
let value = "global";

function outer() {
  let value = "outer";

  return () => value;
}
```

The closure resolves:

```text
value → outer binding
```

not:

```text
global binding
```

If another nested binding shadows it, that new binding becomes relevant.

Closure reasoning therefore depends on the same identifier-resolution rules learned in Chapter 10.

---

# 12. Closure and TDZ

A closure does not bypass TDZ.

Consider:

```js
{
  const read = () => value;

  const value = 10;

  console.log(read());
}
```

This works because:

```text
closure created
→ value binding exists but is uninitialized
→ declaration executes
→ value initialized to 10
→ closure invoked
→ read succeeds
```

But:

```js
{
  const read = () => value;

  console.log(read());

  const value = 10;
}
```

fails because the closure is invoked while the captured binding is still uninitialized.

Result:

```text
ReferenceError
```

Important rule:

> Closure capture does not weaken the initialization rules of the captured binding.

---

# 13. Closure Lifetime

Consider:

```js
function create() {
  const secret = 123;

  return () => secret;
}

const readSecret = create();
```

The active execution of `create()` is complete.

Yet:

```text
readSecret
   ↓
closure
   ↓
lexical state containing secret
```

can remain reachable.

The language cares about observable access.

The engine decides the physical representation.

---

# 14. Closure Lifetime Is Not Stack Lifetime

Incorrect:

```text
"the stack frame of create() stays alive"
```

Better:

```text
"the active execution of create() has completed, but
captured lexical state may remain reachable through the surviving closure."
```

The engine can represent that surviving state in heap contexts or other optimized forms.

Therefore:

```text
call-stack lifetime
≠
captured-state lifetime
```

This is one of the most important ideas in JavaScript memory reasoning.

---

# 15. Reachability Determines Retention

Suppose:

```text
global root
   ↓
handler
   ↓
closure
   ↓
environment
   ↓
largeObject
```

As long as that path remains reachable, `largeObject` may remain alive.

When the root path disappears:

```text
handler unreachable
→ closure unreachable
→ environment unreachable
→ exclusively retained largeObject unreachable
```

the objects may become eligible for garbage collection.

GC timing is implementation-dependent.

---

# 16. What Does a Closure Actually Retain?

Do not use this universal claim:

```text
closure retains a copy of the entire scope
```

The semantic requirement is narrower:

```text
the function must preserve the lexical behavior needed by its referenced bindings
```

The engine may optimize representation.

Therefore distinguish:

```text
semantic captured environment
```

from:

```text
physical memory layout
```

This distinction becomes important in Chapter 48.

---

# 17. Function Factories

Closures make configurable behavior easy:

```js
function makeValidator(minLength) {
  return function validate(input) {
    return input.length >= minLength;
  };
}

const passwordValidator = makeValidator(12);
```

The validator retains:

```text
minLength → 12
```

This is often cleaner than storing configuration globally.

---

# 18. Partial Application

Example:

```js
function add(a, b) {
  return a + b;
}

function addFixed(a) {
  return b => add(a, b);
}

const add10 = addFixed(10);

console.log(add10(5)); // 15
```

The returned function closes over:

```text
a → 10
```

This underlies:

- currying;
- partial application;
- specialized handlers;
- configurable pipelines.

---

# 19. Memoization

A closure can retain a cache:

```js
function memoize(fn) {
  const cache = new Map();

  return function memoized(key) {
    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn(key);
    cache.set(key, result);
    return result;
  };
}
```

The returned function closes over:

```text
fn
cache
```

Critical production observation:

```text
cache lifetime follows closure lifetime
```

If the memoized function is process-long-lived and the key space is unbounded, memory can grow without bound.

---

# 20. Debounce

Example:

```js
function debounce(fn, delay) {
  let timerId;

  return function (...args) {
    clearTimeout(timerId);

    timerId = setTimeout(() => {
      fn(...args);
    }, delay);
  };
}
```

The closure retains:

```text
fn
delay
timerId
```

This is a closure-backed state machine:

```text
call
 ↓
cancel previous timer
 ↓
update timerId
 ↓
later callback
```

---

# 21. Event Handlers

Example:

```js
function createHandler(userId) {
  return event => {
    console.log(userId, event.type);
  };
}

button.addEventListener("click", createHandler(42));
```

The handler captures:

```text
userId
```

The interesting production question is not whether this is “a closure.”

It is:

```text
How long will the handler remain registered?
What else does it retain?
Who unregisters it?
```

---

# 22. Timers

Example:

```js
function scheduleMessage(message) {
  setTimeout(() => {
    console.log(message);
  }, 1000);
}
```

The timer infrastructure retains the callback until execution or cancellation.

The callback retains:

```text
message
```

Therefore:

```text
timer lifetime
+
closure lifetime
+
captured references
```

can affect memory retention.

---

# 23. Promises and Closures

Example:

```js
function loadUser(id) {
  return fetch(`/users/${id}`)
    .then(response => response.json())
    .then(user => ({ id, user }));
}
```

The callbacks close over:

```text
id
```

Promise machinery retains the relevant callbacks until the chain progresses and references can be released.

This becomes important for asynchronous memory analysis.

---

# 24. Async Functions and Closure State

Example:

```js
async function load(id) {
  const user = await fetchUser(id);

  return () => user;
}
```

There are at least two related but distinct ideas:

```text
async continuation state
```

and:

```text
closure state captured by the returned function
```

Do not collapse them.

The async operation may retain state needed to resume execution, while the returned function may retain a different set of state after the async function completes.

---

# 25. `var` Loop Closure Behavior

Classic example:

```js
const fns = [];

for (var i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns.map(fn => fn()));
```

Result:

```text
[3, 3, 3]
```

Why?

There is one relevant `var` binding:

```text
i
```

and every callback resolves to that same binding.

Timeline:

```text
i = 0
create callback A

i = 1
create callback B

i = 2
create callback C

i = 3
loop ends

A → same i → 3
B → same i → 3
C → same i → 3
```

---

# 26. `let` Loop Closure Behavior

Now:

```js
const fns = [];

for (let i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns.map(fn => fn()));
```

Result:

```text
[0, 1, 2]
```

The specification provides per-iteration lexical binding behavior.

Conceptually:

```text
iteration 0 → i₀
iteration 1 → i₁
iteration 2 → i₂
```

Each callback has access to the corresponding iteration binding.

This is not merely “JavaScript copies the number.”

---

# 27. Stale Closures

A stale-closure problem occurs when a function retains a binding from an older lexical execution context than the developer intended.

A generic shape is:

```text
render / invocation #1
   ↓
closure A captures binding A

render / invocation #2
   ↓
closure B captures binding B
```

Later:

```text
closure A runs
```

and sees state associated with its original binding.

The language is behaving correctly.

The architecture or lifecycle may be wrong.

---

# 28. Debugging Stale Closures

Ask:

```text
1. Where was the callback created?
2. Which invocation created it?
3. Which bindings does it reference?
4. Which exact binding does each name resolve to?
5. When was the callback registered?
6. When is it invoked?
7. Was a newer callback supposed to replace it?
```

This methodology works across:

- UI frameworks;
- event systems;
- server callbacks;
- promises;
- timers.

---

# 29. Closures and Objects

Closure-based state:

```js
function createCounter() {
  let count = 0;

  return {
    increment() {
      count++;
    },

    value() {
      return count;
    }
  };
}
```

Object-property state:

```js
const counter = {
  count: 0,

  increment() {
    this.count++;
  }
};
```

Neither is universally superior.

Compare:

```text
encapsulation
serialization
debugging
reflection
inheritance
API discoverability
state ownership
lifecycle
```

Choose based on system requirements.

---

# 30. Closures and Private Fields

Modern classes can express private state:

```js
class Counter {
  #count = 0;

  increment() {
    return ++this.#count;
  }

  value() {
    return this.#count;
  }
}
```

Closure state and private fields overlap in purpose.

Closures often fit:

```text
factory + capability + encapsulated lexical state
```

Private class fields often fit:

```text
identity + instances + methods + class-oriented architecture
```

---

# 31. Closure-Based Module Pattern

Before modern modules were standardized, an IIFE plus closures was often used to create private module state:

```js
const counterModule = (() => {
  let count = 0;

  return {
    increment() {
      return ++count;
    },

    current() {
      return count;
    }
  };
})();
```

Modern ES modules are preferable for module boundaries, but closures remain useful inside modules for local encapsulation.

---

# 32. Closures and Capability Security

A closure can expose a narrow capability:

```js
function makeReader(store) {
  return path => store.read(path);
}
```

The caller receives:

```text
read capability
```

rather than unrestricted access to:

```text
store
```

This can reduce ambient authority.

But a closure is not a substitute for:

- process isolation;
- worker isolation;
- OS permissions;
- authentication;
- authorization.

The security benefit comes from controlled references and API surface.

---

# 33. Dependency Injection

Closures can capture dependencies:

```js
function makeService(repository, logger) {
  return {
    create(data) {
      logger.log("creating");
      return repository.save(data);
    }
  };
}
```

This can be a simpler alternative to introducing a class solely to store dependencies.

Evaluate:

```text
lifecycle
testability
state ownership
extensibility
instrumentation
team conventions
```

before choosing.

---

# 34. Middleware Factories

Common server-side pattern:

```js
function requireRole(role) {
  return function middleware(request, response, next) {
    if (request.user.role !== role) {
      return response.status(403).end();
    }

    next();
  };
}
```

The returned middleware captures:

```text
role
```

This makes factories concise and composable.

---

# 35. Route Handler Factories

Example:

```js
function makeHandler(service) {
  return async function handler(request, response) {
    const result = await service.run(request.params.id);
    response.json(result);
  };
}
```

The process-long-lived handler may retain:

```text
service
```

That may be entirely correct.

The principal question is:

```text
Does the retention graph match the intended ownership graph?
```

---

# 36. Request-Scoped Retention Risk

Dangerous pattern:

```js
function register(request, emitter) {
  emitter.on("done", () => {
    console.log(request.user.id);
  });
}
```

If:

```text
emitter = process lifetime
```

and:

```text
listener = never removed
```

then:

```text
emitter
 ↓
listener
 ↓
closure
 ↓
request
```

can retain request-associated state far beyond its intended lifetime.

This is a practical closure-lifetime bug.

---

# 37. Capture Only What You Need

Instead:

```js
function register(request, emitter) {
  const userId = request.user.id;

  emitter.on("done", () => {
    console.log(userId);
  });
}
```

the closure may retain a much smaller object graph.

This is not a guarantee that memory is solved—the listener still needs lifecycle management.

But the ownership boundary is clearer.

Production rule:

> Long-lived callbacks should capture the smallest useful state.

---

# 38. Closure Retention vs Memory Leak

A closure retaining data is not automatically a leak.

Distinguish:

```text
intentional retention
bounded retention
unintentional retention
unbounded retention
```

Example:

```text
memoization cache
```

may intentionally retain values.

It becomes problematic when:

```text
growth is unbounded
or
lifetime exceeds design
```

A real diagnosis needs:

```text
root
→ closure
→ environment
→ retained object
→ expected lifetime
→ actual lifetime
```

---

# 39. Heap Retainer Path

A memory profiler may reveal:

```text
Window
  ↓
event listener
  ↓
function
  ↓
context/environment
  ↓
large array
```

The useful question is not:

> “Why do closures leak?”

It is:

> “Why is this closure still reachable, and what object graph does it retain?”

That question leads to actionable remediation.

---

# 40. Closure Memory and Garbage Collection

Suppose:

```js
function makeHandler(data) {
  return () => data.id;
}
```

If the returned function becomes unreachable and no other reference path retains `data`, both may become eligible for collection.

GC does not occur because:

```text
function returned
```

or:

```text
scope ended
```

by themselves.

Eligibility is based on reachability.

---

# 41. Closure Performance

Avoid blanket claims such as:

```text
closures are slow
closures always allocate
closures are expensive
```

The actual cost depends on:

```text
creation rate
capture set
escape behavior
call frequency
lifetime
engine optimization
hot-path characteristics
```

Modern engines optimize many closure patterns effectively.

Use:

```text
semantic understanding
→ profiling
→ targeted optimization
```

rather than folklore.

---

# 42. Closure Allocation

Do not assume:

```text
one source-level arrow function
=
one permanently heap-allocated object
```

An engine may:

- inline functions;
- specialize contexts;
- eliminate allocations;
- store state in optimized frames;
- use registers;
- materialize objects only when necessary.

These are implementation strategies.

Do not make architectural promises based on a particular optimization.

---

# 43. Closure Optimization

Potential engine strategies can include:

```text
inlining
escape analysis
scalar replacement
context specialization
stack/register allocation
dead-state elimination
```

The exact techniques vary by engine and version.

The language guarantee remains:

```text
observable lexical behavior must be preserved
```

not:

```text
a closure must be represented as a heap object
```

---

# 44. Closure and `this`

Closures and `this` are related but distinct.

Example:

```js
const obj = {
  value: 10,

  makeReader() {
    return () => this.value;
  }
};
```

The arrow function lexically gets access to the surrounding function's `this`.

That is different from capturing:

```js
const value = 10;
```

as an ordinary lexical binding.

Chapter 14 will analyze `this` formally.

---

# 45. Closure and `arguments`

An arrow function can access the surrounding ordinary function's `arguments`:

```js
function outer() {
  return () => arguments[0];
}

console.log(outer(42)());
```

The inner arrow does not create its own `arguments`.

It uses lexical access to the surrounding function's `arguments`-related state.

---

# 46. Closure and Default Parameters

Parameters can be captured:

```js
function makePrefix(prefix = "user") {
  return name => `${prefix}:${name}`;
}

const format = makePrefix("admin");

console.log(format("42")); // admin:42
```

The closure retains access to the parameter binding:

```text
prefix → "admin"
```

This is another reason parameter initialization belongs in the closure mental model.

---

# 47. Closure and `eval`

Dynamic evaluation can interact with surrounding environments:

```js
function outer() {
  let x = 1;

  return () => eval("x");
}
```

Such code complicates:

- static analysis;
- optimization;
- refactoring;
- security;
- reasoning about captured state.

Production guidance:

```text
avoid dynamic evaluation unless there is a clear and reviewed need
```

---

# 48. Closure and `with`

`with` can alter identifier lookup by allowing object properties to participate in name resolution.

That makes closure reasoning less statically obvious.

Modern JavaScript should avoid it.

This is a historical example of why lexical closure reasoning is cleaner when name resolution remains lexical.

---

# 49. Closure and Serialization

Functions and their lexical environments are not ordinary JSON data.

This does not work as a persistence mechanism:

```js
const fn = (() => {
  const secret = 42;
  return () => secret;
})();

JSON.stringify(fn);
```

You cannot expect:

```text
function code + captured lexical state
```

to serialize transparently.

For persistence/RPC:

```text
serialize data
+
reconstruct behavior
```

rather than attempting to transport closures as data.

---

# 50. Closure and Structured Clone

Functions are not generally transferable through structured cloning like ordinary serializable data.

Therefore:

```text
closure
→ worker
```

is not a generic transport model.

Use:

```text
messages
structured-cloneable data
transferables
shared-memory mechanisms where appropriate
```

and recreate behavior in the destination environment.

---

# 51. Closure and Workers

A worker has its own execution environment.

A function created in one agent does not simply become callable in another agent.

Communication occurs through explicit mechanisms.

Thus:

```text
closure = in-process lexical capability
worker message = cross-agent protocol
```

These should not be conflated.

---

# 52. Debugging Methodology

When a closure behaves unexpectedly, follow:

```text
1. Find the function that runs.
2. Find where that function was created.
3. List the outer identifiers it references.
4. Resolve each identifier to its exact binding.
5. Identify which invocation created that binding.
6. Trace mutations to those bindings.
7. Determine when the callback was registered.
8. Determine when it was invoked.
9. Determine what keeps the callback reachable.
10. Inspect retained objects if memory is involved.
```

This method works for correctness and memory problems.

---

# 53. Debugging Exercise 1 — Shared State

```js
function createState() {
  let count = 0;

  return {
    inc: () => ++count,
    read: () => count
  };
}

const state = createState();

state.inc();
state.inc();

console.log(state.read());
```

Questions:

```text
Which binding is shared?
How many closure functions access it?
Why is the answer 2?
```

---

# 54. Debugging Exercise 2 — Separate Environments

```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const a = makeCounter();
const b = makeCounter();

console.log(a());
console.log(b());
```

Expected:

```text
1
1
```

Explain why the closures do not share `count`.

---

# 55. Debugging Exercise 3 — Mutation

```js
let x = 1;

const read = () => x;

x = 99;

console.log(read());
```

Explain:

```text
closure captured binding
→ binding was reassigned
→ closure observes current value
```

---

# 56. Debugging Exercise 4 — TDZ

```js
{
  const read = () => value;

  console.log(read());

  const value = 10;
}
```

Trace:

```text
value binding created
→ uninitialized
→ closure created
→ closure invoked
→ read resolves value
→ value still uninitialized
→ ReferenceError
```

---

# 57. Debugging Exercise 5 — Retention

```js
function setup(emitter, request) {
  emitter.on("done", () => {
    console.log(request.id);
  });
}
```

Questions:

```text
What captures request?
What keeps the closure alive?
What releases it?
What happens if emitter is process-long-lived?
```

---

# 58. Code Review Exercise — Hidden Mutable State

Review:

```js
function createStore() {
  let state = {};

  return {
    get() {
      return state;
    },

    set(next) {
      state = next;
    }
  };
}
```

Potential problem:

```text
get() exposes the internal object reference.
```

A caller can mutate state without using `set`.

Possible defensive approach:

```js
function createStore() {
  let state = {};

  return {
    get() {
      return { ...state };
    },

    set(next) {
      state = { ...next };
    }
  };
}
```

Whether copying is appropriate depends on data size and required semantics.

The deeper lesson:

> Hiding the binding does not automatically protect the referenced object from mutation.

---

# 59. Code Review Exercise — Listener Lifecycle

Review:

```js
function attach(emitter, request) {
  emitter.on("message", () => {
    process(request);
  });
}
```

Ask:

```text
Who owns the listener?
When is it removed?
Can multiple requests register listeners forever?
Does each listener retain a request graph?
```

A production API should make ownership and cleanup explicit.

---

# 60. Code Review Exercise — Narrow Capture

Review:

```js
function attach(emitter, request) {
  const requestId = request.id;

  emitter.on("message", () => {
    processById(requestId);
  });
}
```

This can reduce retained state compared with capturing the entire request object.

But:

```text
listener cleanup
```

still matters.

---

# 61. Implementation From Scratch — Environment

Build:

```js
class Environment {
  constructor(parent = null) {
    this.parent = parent;
    this.bindings = new Map();
  }

  define(name, value) {
    this.bindings.set(name, value);
  }

  hasOwn(name) {
    return this.bindings.has(name);
  }

  get(name) {
    if (this.bindings.has(name)) {
      return this.bindings.get(name);
    }

    if (this.parent) {
      return this.parent.get(name);
    }

    throw new ReferenceError(`${name} is not defined`);
  }

  set(name, value) {
    if (this.bindings.has(name)) {
      this.bindings.set(name, value);
      return;
    }

    if (this.parent) {
      this.parent.set(name, value);
      return;
    }

    throw new ReferenceError(`${name} is not defined`);
  }
}
```

This models the central environment-chain operation:

```text
current environment
→ parent
→ parent
→ ...
```

---

# 62. Implementation — Closure Function

Create:

```js
class ClosureFunction {
  constructor(name, body, environment) {
    this.name = name;
    this.body = body;
    this.environment = environment;
  }

  call(args = []) {
    return this.body({
      args,
      environment: this.environment
    });
  }
}
```

This is intentionally simplified.

The key relationship is:

```text
function
+
captured environment
```

---

# 63. Guided Closure Simulation

```js
const globalEnv = new Environment();
globalEnv.define("x", 100);

const readX = new ClosureFunction(
  "readX",
  ({ environment }) => environment.get("x"),
  globalEnv
);

console.log(readX.call()); // 100
```

This simulates lexical capture.

---

# 64. Nested Environment Simulation

```js
const globalEnv = new Environment();
globalEnv.define("globalValue", 1);

const outerEnv = new Environment(globalEnv);
outerEnv.define("outerValue", 2);

const innerEnv = new Environment(outerEnv);
innerEnv.define("innerValue", 3);
```

Lookup:

```text
innerValue → innerEnv
outerValue → outerEnv
globalValue → globalEnv
```

This mirrors the conceptual lexical environment chain.

---

# 65. Guided Counter Implementation

Build:

```js
function makeCounter() {
  const env = new Environment();
  env.define("count", 0);

  return () => {
    const next = env.get("count") + 1;
    env.set("count", next);
    return next;
  };
}
```

Then:

```js
const a = makeCounter();
const b = makeCounter();

console.log(a()); // 1
console.log(a()); // 2
console.log(b()); // 1
```

This proves:

```text
factory call
→ new environment
→ new binding
```

---

# 66. Partially Guided Implementation

Add:

```text
createBinding
readBinding
writeBinding
createClosure
invokeClosure
```

The simulator must support:

```text
nested environments
shadowing
mutable captured state
separate factory invocations
```

Test:

```text
global x
outer x
inner closure
```

and verify that shadowing is resolved from the nearest environment.

---

# 67. No-Reference Implementation

Build a tiny interpreter supporting:

```text
define
assign
create function
capture current environment
call function
return function
```

Required proof:

```text
x = 10
create closure f
x = 20
call f
```

Expected:

```text
20
```

Then:

```text
create factory
call factory twice
update first closure
read second closure
```

Expected:

```text
second state is independent
```

---

# 68. Edge-Case Hardening

Add support for:

- shadowing;
- TDZ state;
- immutable bindings;
- nested closures;
- recursion;
- `var` versus `let` loop models;
- exception propagation;
- closure destruction;
- simulated references;
- retention diagnostics.

Track:

```text
closureId
environmentId
creator
capturedNames
creationLocation
```

Example:

```text
closure=17
environment=42
captures=[requestId, service]
createdAt=module.js:120
```

---

# 69. Production-Grade Closure Analyzer

Design a runtime/static analyzer that reports:

```text
Function
Creation Site
Captured Bindings
Registration Site
Owner
Estimated Lifetime
Cleanup Path
Large Retainers
```

Example:

```text
Handler: onMessage
Captures: request, service
Registered on: processEmitter
Expected lifetime: request
Observed owner lifetime: process
Cleanup: none

Potential retention mismatch detected.
```

Static analysis cannot always know actual runtime lifetime, so production tooling should combine code analysis with profiling where necessary.

---

# 70. Interview Questions

## Beginner

1. What is a closure?
2. Why can an inner function access an outer variable?
3. Does a closure capture a value or a binding?
4. Can closures maintain state?
5. Give a real-world closure example.

## Intermediate

6. Why do separate `makeCounter()` calls produce separate counters?
7. Why do several methods returned from one invocation share state?
8. Explain `var` versus `let` in loop closures.
9. Can a closure outlive its creating function?
10. How is closure behavior related to lexical scope?

## Advanced

11. Can closures retain memory?
12. What determines how long captured state remains reachable?
13. Why is a closure not the same thing as a stack frame?
14. How do event listeners interact with closure lifetime?
15. What is a stale closure?
16. How do closures interact with async callbacks?
17. How are closures represented by engines?

## Principal

18. How would you diagnose a closure-related memory retention problem?
19. When would you choose closure state over class private fields?
20. How would you design callback ownership to prevent accidental retention?
21. How do closures support capability-style APIs?
22. Which closure claims are ECMAScript semantics versus engine details?
23. How would you instrument a production system to identify long-lived closures?
24. How would you distinguish a legitimate cache from a memory leak?

---

# 71. Predict-the-Output Exercises

Predict before execution.

## Exercise A

```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const c = makeCounter();

console.log(c());
console.log(c());
```

## Exercise B

```js
function makeCounter() {
  let count = 0;
  return () => count;
}

const a = makeCounter();
const b = makeCounter();

console.log(a());
console.log(b());
```

## Exercise C

```js
let x = 1;
const f = () => x;
x = 5;

console.log(f());
```

## Exercise D

```js
let x = 1;
const snapshot = x;
const f = () => snapshot;
x = 5;

console.log(f());
```

## Exercise E

```js
const fns = [];

for (var i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns.map(fn => fn()));
```

## Exercise F

```js
const fns = [];

for (let i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns.map(fn => fn()));
```

## Exercise G

```js
function outer() {
  let value = 1;

  return {
    get: () => value,
    set: next => {
      value = next;
    }
  };
}

const state = outer();

state.set(9);
console.log(state.get());
```

## Exercise H

```js
{
  const read = () => value;
  const value = 10;

  console.log(read());
}
```

## Exercise I

```js
{
  const read = () => value;
  console.log(read());
  const value = 10;
}
```

## Exercise J

```js
const fns = [];

for (var i = 0; i < 3; i++) {
  fns.push(() => i);
}

i = 99;

console.log(fns[0]());
```

---

# 72. Mastery Exercises

## Level 1 — Understand

Explain closure using:

```text
function
lexical environment
binding
capture
lifetime
reachability
```

without relying on the phrase:

```text
"remembers variables"
```

---

## Level 2 — Explain

Explain why:

```js
const a = makeCounter();
const b = makeCounter();
```

creates independent state.

---

## Level 3 — Predict

Predict closure behavior for:

```text
reassignment
shadowing
loops
multiple closures
async callbacks
timers
```

---

## Level 4 — Implement

Build the closure interpreter.

---

## Level 5 — Debug

Given a heap retainer path, identify:

```text
root
→ closure
→ environment
→ object
```

and explain whether retention is intended.

---

## Level 6 — Defend

Defend:

> A closure retains lexical access to bindings; it does not necessarily copy an entire scope or keep the original execution stack frame alive.

---

# 73. Principal-Level Reasoning Problems

## Problem 1 — Binding vs snapshot

Given:

```js
let x = 1;
const f = () => x;
x = 2;
```

Explain why:

```text
f() → 2
```

and identify the captured entity.

---

## Problem 2 — “Closures leak memory”

Challenge this statement.

Correct reasoning:

```text
closures create references
references create retention paths
retention is a problem only when lifetime exceeds intent or growth is unbounded
```

---

## Problem 3 — Stack frame misconception

An engineer says:

> “The stack frame of `makeCounter()` is still alive because the callback uses `count`.”

Correct them using:

```text
active execution context ends
captured lexical state survives
physical representation is engine-specific
```

---

# 74. Principal-Level Design Exercise

Choose a representation for a configurable component:

```text
closure
object
class with private fields
module-level state
```

Defend your choice using:

```text
encapsulation
lifecycle
serialization
debugging
testability
inheritance
extensibility
memory retention
team conventions
```

There is no universal answer.

---

# 75. Production Closure Review Checklist

For every long-lived callback, ask:

```text
1. What bindings are captured?
2. Are any captured objects unnecessarily large?
3. Who owns the callback?
4. Who releases the callback?
5. What is its intended lifetime?
6. Can listeners accumulate?
7. Can caches grow without bound?
8. Could request-scoped data be retained?
9. Can a smaller value be captured instead?
10. Have actual retention paths been verified?
```

---

# 76. Security Checklist

For capability-style closure APIs:

```text
1. What references are exposed?
2. Can hidden state be mutated indirectly?
3. Is the capability narrower than the underlying resource?
4. Could secrets be retained longer than needed?
5. Can untrusted code receive the closure?
6. What happens when the closure is retained?
7. Is stronger isolation required?
```

---

# 77. Performance Decision Framework

When considering closure-heavy code, evaluate:

| Dimension | Questions |
|---|---|
| Creation | How frequently are closures created? |
| Capture | What state must remain reachable? |
| Calls | Is the closure on a hot path? |
| Lifetime | How long do instances survive? |
| Allocation | Does profiling show meaningful allocation cost? |
| Optimization | Can the engine optimize this pattern? |
| Readability | Does avoiding a closure make the design worse? |
| Measurement | Is there evidence for optimization? |

Rule:

```text
Do not replace clear closure-based design
just because closures are theoretically expensive.
```

Measure first.

---

# 78. Common Misconceptions

## Misconception 1

> A closure is a copy of the whole scope.

Not a useful universal model.

## Misconception 2

> Closures always capture snapshots.

False; captured bindings can change.

## Misconception 3

> Closures always cause heap allocation.

Not a semantic guarantee.

## Misconception 4

> Closures keep stack frames alive forever.

False as an implementation model.

## Misconception 5

> Closures automatically prevent mutation.

They only hide bindings unless controlled operations are exposed.

## Misconception 6

> Every closure-retention problem is a leak.

Retention can be intentional and bounded.

## Misconception 7

> `let` fixes all closure bugs.

It solves important loop-binding issues, but stale-state and lifecycle problems remain possible.

---

# 79. Common Mistakes

Avoid:

```text
"closure stores variables"
"closure stores a whole scope"
"closure keeps the stack frame"
"closure always allocates on heap"
"closure causes memory leak"
```

Prefer:

```text
function retains lexical access
binding remains reachable
environment relationships remain observable
engine chooses representation
lifetime follows reachability
```

---

# 80. Memory Analysis Workflow

For a suspected closure retention issue:

```text
1. Identify the unexpected retained object.
2. Find its GC root.
3. Walk the retainer path.
4. Identify the closure.
5. Identify the captured binding/environment.
6. Identify why the closure remains reachable.
7. Determine intended lifetime.
8. Remove the ownership mismatch.
9. Re-profile.
```

This is more reliable than deleting closures blindly.

---

# 81. Debugging Stale State Workflow

For a callback returning unexpected old state:

```text
creation site
→ captured binding
→ creator invocation
→ callback registration time
→ callback invocation time
→ mutations
→ replacement / cleanup
```

If there are multiple invocations or renders, label them:

```text
instance A
instance B
instance C
```

Then trace which closure belongs to which instance.

---

# 82. Closure and Reentrancy

Closures often retain access to mutable state that can be mutated by callbacks.

Example:

```js
function createManager() {
  let state = "idle";

  return {
    update() {
      state = "updating";
      notify();
      state = "ready";
    },

    state: () => state
  };
}
```

If `notify()` invokes code that calls `update()` again, the same captured state may be re-entered.

This creates an invariant problem:

```text
shared lexical state
+
reentrant callback
=
nested mutation risk
```

Closure design therefore intersects directly with reentrancy.

---

# 83. Closure and API Ownership

A closure can create an API where only selected operations are exposed:

```text
private binding
   ↓
public methods
```

This is powerful for invariants.

But the design must still specify:

```text
who owns lifecycle
who can call operations
when state is released
whether operations are reentrant
```

A private binding does not eliminate architectural responsibilities.

---

# 84. Closure and Async Lifecycle

A closure passed into:

```text
timer
promise
event emitter
WebSocket
job queue
stream
```

may survive substantially longer than the function that created it.

The relevant question becomes:

```text
Which runtime component owns the callback?
```

Examples:

```text
timer → timer subsystem
emitter → emitter
promise chain → promise/job machinery
DOM listener → target/listener registry
```

Closure lifetime follows from those references.

---

# 85. Closure and Cancellation

Suppose:

```js
function start(request, signal) {
  return new Promise((resolve, reject) => {
    const handler = () => resolve(request.id);

    signal.addEventListener("abort", handler);
  });
}
```

If cancellation occurs and the listener is not removed appropriately, the callback may retain:

```text
request
```

longer than expected.

Therefore closure lifecycle and cancellation design are linked.

This prepares for Chapter 37.

---

# 86. Closure and Resource Cleanup

A robust subscription API often returns an explicit cleanup function:

```js
function subscribe(source, state) {
  const handler = value => {
    state.value = value;
  };

  source.on("data", handler);

  return () => {
    source.off("data", handler);
  };
}
```

The cleanup closure retains `handler` and possibly `state`.

The cleanup contract makes ownership visible.

---

# 87. Production-Grade Example

A robust resource-owning factory might look like:

```js
function createSubscription(source, dependency) {
  let active = true;

  const handler = value => {
    if (!active) return;
    dependency.process(value);
  };

  source.on("data", handler);

  return {
    stop() {
      if (!active) return;
      active = false;
      source.off("data", handler);
    }
  };
}
```

This closure-backed design makes:

```text
lifecycle state
handler
dependency
cleanup
```

explicit.

A complete production implementation would also define:

```text
error handling
idempotency
concurrency
listener exceptions
shutdown semantics
```

---

# 88. Specification-Oriented Vocabulary

Important terms:

```text
lexical environment
environment record
binding
identifier resolution
lexical scope
closure
function object
execution context
environment reachability
captured state
```

The ECMAScript specification does not define a developer API called:

```text
new Closure()
```

Closure is a semantic consequence of functions and lexical environments.

---

# 89. Specification vs Engine

ECMAScript guarantees observable lexical behavior.

Engines may implement it with:

```text
stack slots
registers
heap contexts
optimized frames
inlining
specialized representations
```

Therefore statements like:

```text
"V8 stores every closure in a heap object"
```

should not be presented as universal language truths.

When discussing engine behavior, label it:

```text
V8-specific
engine-specific
version-sensitive
implementation-detail
```

---

# 90. Completion Criteria

### Understand

You can define:

- closure;
- captured binding;
- lexical environment;
- reachability;
- closure lifetime.

### Explain

You can explain:

- why returned functions access outer state;
- why separate factory calls have separate state;
- why multiple closures can share one state;
- why captured bindings observe mutation;
- why `var` and `let` differ in loop closures;
- why closures do not keep stack frames alive by requirement.

### Predict

You can correctly predict:

- mutation;
- shadowing;
- loops;
- multiple factory instances;
- TDZ;
- timers;
- event callbacks;
- promise callbacks.

### Implement

You can build:

```text
environment
binding lookup
captured function
closure invocation
state sharing
separate factory environments
```

### Debug

You can diagnose:

```text
stale closure
retained request
long-lived listener
unbounded memoization
unexpected shared state
```

### Principal Judgment

You can choose between:

```text
closure
object
class/private field
module state
```

using lifecycle, encapsulation, debugging, memory, and extensibility considerations.

**Evidence of mastery:**

- 90%+ prediction accuracy;
- working closure simulator;
- successful retainer-path analysis;
- correct explanation of binding versus snapshot;
- correct `var`/`let` loop reasoning;
- clear closure design defense.

---

# 91. Key Takeaways

1. **A closure is a function with lexical access to surrounding bindings.**
2. **Closures are a natural consequence of lexical scope plus first-class functions.**
3. **Closures access bindings; they do not automatically freeze value snapshots.**
4. **Multiple closures can share one lexical environment and therefore share mutable state.**
5. **Separate factory invocations create separate environments and independent state.**
6. **A closure can outlive the active execution that created it.**
7. **Captured state lifetime is a reachability problem, not a stack-frame problem.**
8. **Engines are free to optimize closure representation.**
9. **`var` loop closures commonly share one binding; `let` has per-iteration lexical semantics.**
10. **Closures are central to factories, callbacks, memoization, debounce, dependency injection, and encapsulation.**
11. **Long-lived closures can retain large object graphs.**
12. **Retention is not automatically a memory leak; intended lifetime and growth matter.**
13. **Event listeners, timers, promises, and subscriptions determine callback lifetime.**
14. **Capturing only the required state can reduce accidental retention.**
15. **Closures can create useful capability boundaries but are not a replacement for stronger isolation.**
16. **Closure behavior and `this` are related but distinct concepts.**
17. **Closure correctness depends on lexical resolution and binding state.**
18. **Stale closures are usually a lifecycle/binding-instance problem, not a mysterious language failure.**
19. **Production closure analysis should use profiling and retainer paths rather than folklore.**
20. **Principal-level closure design is about semantics, ownership, lifetime, and trade-offs.**

---

# 92. Final Mastery Drill

For every example, answer in this order:

```text
1. Where is the function created?
2. Which lexical environment is associated with it?
3. Which bindings does it reference?
4. Which exact binding does each name resolve to?
5. Is that binding shared with another closure?
6. Is the binding mutable?
7. What mutates it?
8. What keeps the closure reachable?
9. What state survives the creator execution?
10. What is the intended lifetime?
```

### Drill 1

```js
function createCounter() {
  let count = 0;

  return () => ++count;
}
```

### Drill 2

```js
const a = createCounter();
const b = createCounter();
```

### Drill 3

```js
let x = 1;
const read = () => x;
x = 9;
```

### Drill 4

```js
const fns = [];

for (var i = 0; i < 3; i++) {
  fns.push(() => i);
}
```

### Drill 5

```js
const fns = [];

for (let i = 0; i < 3; i++) {
  fns.push(() => i);
}
```

### Drill 6

```js
function setup(emitter, request) {
  emitter.on("done", () => process(request));
}
```

### Drill 7

```js
function setup(emitter, request) {
  const requestId = request.id;

  emitter.on("done", () => processById(requestId));
}
```

### Drill 8

```js
{
  const read = () => value;
  console.log(read());
  const value = 10;
}
```

For each drill, explain both:

```text
language-level semantics
```

and:

```text
production implications
```

---

# 93. Transition to Chapter 14

Chapter 13 explains:

```text
how functions retain lexical access to surrounding bindings.
```

Chapter 14 moves to a different but closely related question:

```text
What determines `this`?
Why does call-site syntax matter?
Why do methods, plain calls, constructors, and explicit calls behave differently?
Why do arrow functions not receive their own dynamic `this`?
How do call/apply/bind work?
How does `new` alter invocation?
```

The next conceptual layer is:

```text
function
   ↓
invocation form
   ↓
execution context
   ↓
`this`
   ↓
binding semantics
```

Closures answer:

```text
"What lexical state can this function access?"
```

Chapter 14 will answer:

```text
"What receiver/context does this invocation provide?"
```

---

# 94. Chapter Completion Record

**Chapter:** 13 — Closures

**Status:** `[+] Completed`

**Strong Areas Expected:**

- lexical capture;
- shared bindings;
- independent closure environments;
- loop closures;
- closure lifetime;
- memory retention;
- stale closures;
- callback ownership;
- closure-based encapsulation.

**Revision Triggers:**

- saying closures capture snapshots by default;
- saying closures keep stack frames alive;
- saying every closure is a heap allocation;
- blaming all retention on “memory leaks”;
- forgetting that multiple closures can share one binding;
- confusing hidden bindings with immutable values;
- confusing closure capture with `this`.

**Evidence of Mastery:**

- predict 90%+ closure outputs without execution;
- explain binding capture precisely;
- implement a closure environment simulator;
- identify a callback retention path;
- explain `var` versus `let` loop capture;
- choose closure vs object/class with explicit trade-offs.

---