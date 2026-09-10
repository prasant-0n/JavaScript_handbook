
# Chapter 09 — Functions and First-Class Behavior

> **Status:** `[+] Completed`  
> **Role in curriculum:** Establishes functions as executable values, callable objects, abstraction units, and the foundation for callbacks, higher-order functions, closures, composition, middleware, dependency injection, and asynchronous programming.

---

## 1. Learning Objectives

By the end of this chapter, you should be able to:

- Define a JavaScript function precisely.
- Explain functions as first-class values.
- Distinguish declaration, expression, invocation, and reference.
- Compare function declarations, function expressions, named function expressions, arrow functions, and IIFEs.
- Explain parameters, arguments, defaults, rest parameters, and `arguments`.
- Explain return values and implicit `undefined`.
- Explain callable versus constructable function objects.
- Explain function identity.
- Explain synchronous and asynchronous callbacks.
- Explain higher-order functions and function factories.
- Explain foundational closure behavior.
- Explain pure and impure functions and side effects.
- Explain function composition and strategy functions.
- Understand foundational arrow-function differences, especially lexical `this`.
- Predict function behavior and common callback traps.
- Design production-quality function contracts.

---

# 2. Prerequisites

This chapter depends on:

- Chapter 01 — JavaScript, ECMAScript, and the Runtime Landscape.
- Chapter 02 — Values, Types, and the JavaScript Type System.
- Chapter 05 — Variables, Declarations, and Assignment.
- Chapter 06 — Operators and Expressions.
- Chapter 08 — Control Flow and Iteration.

The next chapters will formalize the ideas introduced here through:

```text
Functions
   ↓
Scope
   ↓
Lexical environments
   ↓
Execution contexts
   ↓
Closures
   ↓
this / invocation
```

---

# 3. What Is a Function?

A JavaScript function is a callable value.

Example:

```js
function add(a, b) {
  return a + b;
}
```

The function can be invoked:

```js
add(2, 3);
```

But it can also be manipulated as data:

```js
const operation = add;

operation(2, 3);
```

So a function has two important roles:

```text
value
+
behavior
```

It can be stored, passed, returned, and compared, while also being executable.

This is why functions are one of the central abstractions of JavaScript.

---

# 4. First-Class Functions

A language treats functions as first-class values when functions can participate in ordinary value operations.

JavaScript permits functions to be:

```text
assigned to variables
stored in arrays
stored in objects
passed as arguments
returned from functions
used as property values
compared by identity
```

Example:

```js
const operations = [
  x => x + 1,
  x => x * 2
];

console.log(operations[0](10));
```

The array is not storing source-code text. It stores function values.

This ability enables:

```text
callbacks
higher-order functions
middleware
event handlers
strategies
decorators
factories
dependency injection
```

---

# 5. Functions Are Objects

At the ECMAScript level, functions are objects with callable behavior.

Therefore:

```js
function greet() {}

greet.description = "Greeting function";

console.log(greet.description);
```

works.

A useful mental model is:

```text
Function object
├── identity
├── properties
├── lexical relationships
├── callable behavior
└── possibly constructable behavior
```

Not every object is callable.

A plain object:

```js
const obj = {};
```

cannot be invoked:

```js
obj(); // TypeError
```

A function can.

---

# 6. Callable vs Constructable

These are different capabilities.

Callable:

```js
fn();
```

Constructable:

```js
new Fn();
```

A traditional function can often be both:

```js
function User() {}

User();
new User();
```

An arrow function is callable but not constructable:

```js
const User = () => {};

User();      // valid
new User();  // TypeError
```

This distinction becomes important when learning constructors, classes, prototypes, and `Proxy`.

Do not define a function merely as “something usable with `new`.”

---

# 7. Function Declaration

Example:

```js
function add(a, b) {
  return a + b;
}
```

This is a function declaration.

Function declarations participate in declaration-instantiation semantics differently from ordinary variable declarations and function expressions.

A common explanation says:

> “Function declarations are moved to the top.”

That is a teaching shortcut, not the precise model.

A better model is:

```text
declaration processing
    ↓
function binding established
    ↓
function value available according to declaration semantics
```

The exact mechanism will be formalized in Chapter 11.

---

# 8. Function Expression

Example:

```js
const add = function (a, b) {
  return a + b;
};
```

Here:

```text
const add
```

creates the binding.

The function expression evaluates to a function value.

That function value initializes the `add` binding.

This is conceptually:

```text
evaluate initializer
      ↓
create function value
      ↓
initialize binding
```

This timing difference matters when comparing declarations with `const`/function expressions.

---

# 9. Named Function Expression

A function expression may have a name:

```js
const factorial = function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1);
};
```

The name:

```text
fact
```

is useful for self-reference.

Advantages include:

```text
recursion
debugging
stack traces
clear self-reference
```

The inner name does not simply create an unrelated outer binding.

---

# 10. Anonymous Function

Example:

```js
const add = function (a, b) {
  return a + b;
};
```

The function expression does not explicitly name itself.

Modern engines may infer function names from surrounding syntax for debugging/introspection.

However, inferred names are not an appropriate replacement for explicit business identifiers.

---

# 11. Arrow Functions

Arrow functions provide concise syntax:

```js
const add = (a, b) => a + b;
```

But they are not merely shorter traditional functions.

Important differences include:

```text
lexical this
no own arguments binding
not constructable
different prototype behavior
different super semantics
```

Use an arrow when its semantics match the problem, not simply because it uses newer syntax.

---

# 12. Arrow Function Expression Body

Example:

```js
const square = x => x * x;
```

The expression result is implicitly returned.

Equivalent intent:

```js
const square = x => {
  return x * x;
};
```

But a block body changes the return behavior.

---

# 13. Arrow Function Block Body

This function:

```js
const square = x => {
  x * x;
};
```

returns:

```text
undefined
```

because the body is a block and contains no `return`.

Correct:

```js
const square = x => {
  return x * x;
};
```

This distinction causes many bugs in callbacks and array transformations.

---

# 14. Returning Object Literals From Arrows

This:

```js
const createUser = name => ({
  name
});
```

returns an object.

This:

```js
const createUser = name => {
  name;
};
```

does not.

The braces in the second form define a block body.

Therefore parentheses are used when an object literal is intended as an expression result.

---

# 15. Invocation vs Reference

These expressions are fundamentally different:

```js
fn
```

and:

```js
fn()
```

The first evaluates the function value/reference.

The second invokes the function.

This distinction appears everywhere.

Wrong in many callback situations:

```js
setTimeout(doWork(), 1000);
```

The function runs immediately.

Usually intended:

```js
setTimeout(doWork, 1000);
```

The first passes the invocation result; the second passes the function value.

---

# 16. Return Values

A function can return a value:

```js
function add(a, b) {
  return a + b;
}
```

The call:

```js
add(2, 3);
```

produces:

```text
5
```

A function can return any appropriate value:

```js
return 42;
return "hello";
return {};
return anotherFunction;
return Promise.resolve(42);
```

The return value is part of the function's contract.

---

# 17. Implicit `undefined`

If execution reaches the end of a function without returning a value:

```js
function test() {}
```

then:

```js
test();
```

produces:

```text
undefined
```

Likewise:

```js
function test() {
  return;
}
```

returns `undefined`.

This matters when a callback API expects a particular result.

---

# 18. `return` and Control Flow

`return` exits the current function invocation.

Example:

```js
function find(values) {
  for (const value of values) {
    if (value === 10) {
      return value;
    }
  }

  return null;
}
```

The `return` does more than provide a value:

```text
it ends the current function execution
```

This is different from:

```text
break → exits a loop/switch
continue → advances loop control
throw → begins exception propagation
```

---

# 19. Parameters and Arguments

Example:

```js
function add(a, b) {
  return a + b;
}

add(10, 20);
```

Here:

```text
a, b
```

are parameters.

```text
10, 20
```

are argument values.

The parameter names become bindings associated with that function invocation.

This distinction is important because each invocation has its own parameter state.

---

# 20. More Arguments Than Parameters

JavaScript allows extra arguments:

```js
function add(a, b) {
  return a + b;
}

add(1, 2, 3, 4);
```

The declared parameters receive:

```text
a → 1
b → 2
```

Additional arguments can be accessed through:

```js
arguments
```

or through explicit rest parameters.

Extra arguments are not automatically errors.

---

# 21. Fewer Arguments Than Parameters

Example:

```js
function add(a, b) {
  return a + b;
}

add(10);
```

The second parameter receives `undefined`.

Conceptually:

```text
a → 10
b → undefined
```

Then:

```js
10 + undefined
```

produces:

```text
NaN
```

unless the function provides another rule.

---

# 22. Default Parameters

Example:

```js
function greet(name = "Guest") {
  return `Hello, ${name}`;
}
```

The default is used when the corresponding argument is:

```text
undefined
```

Thus:

```js
greet();
greet(undefined);
```

use `"Guest"`.

But:

```js
greet(null);
```

does not use the default.

---

# 23. Default Parameters Are Not Falsy Defaults

Consider:

```js
function configure(timeout = 5000) {
  return timeout;
}
```

This preserves:

```js
configure(0);
```

as:

```text
0
```

because default parameters are triggered by `undefined`, not general falsiness.

Contrast:

```js
const timeout = input || 5000;
```

which treats:

```text
0
false
""
NaN
null
undefined
```

according to truthiness semantics.

This is a direct connection to Chapter 07.

---

# 24. Default Parameter Expressions

A default can execute code:

```js
function createUser(name = generateName()) {
  return { name };
}
```

When called without a name, `generateName()` can execute.

Therefore defaults can:

```text
read bindings
call functions
throw errors
have side effects
depend on earlier parameters
```

Use side-effecting defaults deliberately.

---

# 25. Parameter Initialization Order

Parameter initialization follows parameter order.

Example:

```js
function f(a, b = a) {
  return b;
}

console.log(f(10));
```

The default for `b` can use earlier parameter `a`.

Parameter initialization therefore has its own environment/timing rules, which become important later when discussing lexical environments.

---

# 26. Rest Parameters

Rest collects remaining arguments:

```js
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
```

Call:

```js
sum(1, 2, 3, 4);
```

Conceptually:

```text
numbers → [1, 2, 3, 4]
```

Rest is explicit and produces an ordinary Array.

---

# 27. Rest Must Be Last

Valid:

```js
function f(a, b, ...rest) {}
```

Invalid:

```js
function f(...rest, a) {}
```

The semantic meaning of rest is:

```text
collect all remaining arguments
```

so no later formal parameter can follow it.

---

# 28. The `arguments` Object

Traditional functions provide:

```js
arguments
```

Example:

```js
function test(a, b) {
  console.log(arguments[0]);
  console.log(arguments[1]);
  console.log(arguments.length);
}

test(10, 20);
```

`arguments` is array-like.

It is not a normal Array.

For modern APIs, rest parameters are often clearer:

```js
function test(...args) {
  ...
}
```

---

# 29. Arrow Functions and `arguments`

Arrow functions do not create their own `arguments` binding.

If an enclosing function has `arguments`, an arrow can resolve that outer binding lexically.

This is another example of arrow functions differing from traditional functions.

Do not write:

```text
arrow functions have no arguments
```

as an absolute statement.

The precise statement is:

```text
arrow functions do not create their own arguments binding
```

---

# 30. Parameter Destructuring

Function parameters can use patterns:

```js
function greet({ name, age }) {
  return `${name}: ${age}`;
}
```

Call:

```js
greet({
  name: "A",
  age: 25
});
```

The pattern extracts values and creates parameter bindings.

Array patterns work too:

```js
function first([value]) {
  return value;
}
```

---

# 31. Default Destructuring

Example:

```js
function configure({
  host = "localhost",
  port = 3000
} = {}) {
  return { host, port };
}
```

There are two default layers:

```text
whole argument default
property defaults
```

The outer `= {}` handles an undefined whole argument.

The property defaults handle undefined extracted properties.

This is useful for configuration APIs.

---

# 32. Function Arity

Function objects expose `.length`.

Example:

```js
function add(a, b) {}
```

then:

```js
add.length
```

is:

```text
2
```

Default and rest parameters affect `.length`.

Therefore `.length` is not a complete runtime function signature.

It should not be used as a substitute for type validation.

---

# 33. Function `.name`

Functions often expose `.name`:

```js
function add() {}

console.log(add.name);
```

This is useful for:

```text
debugging
logging
profiling
stack traces
```

But function names are metadata.

Do not use:

```js
fn.name
```

as an authorization or security mechanism.

---

# 34. Function Identity

Each function object has identity.

Compare:

```js
const a = () => 1;
const b = () => 1;

console.log(a === b);
```

Result:

```text
false
```

The functions have equivalent source behavior but are different objects.

Now:

```js
const a = () => 1;
const b = a;

console.log(a === b);
```

returns:

```text
true
```

because both bindings identify the same function object.

---

# 35. Why Function Identity Matters

Identity matters in:

```text
event listeners
subscriptions
Map keys
Set entries
memoization
caches
dependency registries
unsubscribe APIs
```

Example:

```js
const handler = () => {
  console.log("clicked");
};

element.addEventListener("click", handler);
element.removeEventListener("click", handler);
```

The same function reference is used for both operations.

Creating a second equivalent arrow does not create the same identity.

---

# 36. Callback

A callback is a function value supplied to another operation to be invoked according to that operation's contract.

Example:

```js
function process(value, callback) {
  return callback(value);
}

process(10, value => value * 2);
```

The callback is not a special language type.

It is a function fulfilling a particular role.

---

# 37. Synchronous Callback

Callbacks can execute immediately.

Example:

```js
[1, 2, 3].forEach(value => {
  console.log(value);
});
```

The callback executes during the `forEach` operation.

Therefore:

```text
callback ≠ asynchronous
```

---

# 38. Asynchronous Callback

Example:

```js
setTimeout(() => {
  console.log("later");
}, 1000);
```

The API arranges for the callback to execute later.

The callback itself is still just a function value.

The asynchronous behavior belongs to the surrounding API/runtime mechanism.

This distinction will become critical in the async chapters.

---

# 39. Callback Contract

A production callback API should define:

```text
when callback runs
how often it runs
arguments passed
return value usage
this behavior
ordering
error behavior
sync/async behavior
cancellation
```

For example, saying:

```text
callback(value)
```

is not enough.

Callers need to understand the lifecycle.

---

# 40. Higher-Order Functions

A higher-order function accepts functions and/or returns functions.

Example:

```js
function execute(operation, value) {
  return operation(value);
}
```

Another:

```js
function withLogging(fn) {
  return (...args) => {
    console.log("calling");
    return fn(...args);
  };
}
```

Higher-order functions are foundational to:

```text
map/filter/reduce
middleware
decorators
strategies
memoization
retry
logging
dependency injection
composition
```

---

# 41. Function Factories

A function factory creates customized functions.

Example:

```js
function createMultiplier(factor) {
  return function multiply(value) {
    return value * factor;
  };
}
```

Usage:

```js
const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5));
console.log(triple(5));
```

Results:

```text
10
15
```

The returned functions retain access to `factor`.

This is closure behavior.

---

# 42. Closure Preview

Consider:

```js
function makeCounter() {
  let count = 0;

  return () => {
    count++;
    return count;
  };
}

const counter = makeCounter();

console.log(counter());
console.log(counter());
```

The outer function returns before `counter()` executes.

Yet the returned function still accesses:

```text
count
```

because the function retains lexical access to the surrounding environment.

Chapter 13 will formalize this.

---

# 43. Independent Closure State

Consider:

```js
const a = makeCounter();
const b = makeCounter();

console.log(a());
console.log(a());
console.log(b());
```

Conceptually:

```text
a → environment A → count
b → environment B → count
```

So the first counter can reach:

```text
2
```

while the second independently begins at:

```text
1
```

This is one of the most useful practical examples of closures.

---

# 44. IIFE

IIFE means:

```text
Immediately Invoked Function Expression
```

Example:

```js
(function () {
  console.log("runs immediately");
})();
```

Arrow version:

```js
(() => {
  console.log("runs immediately");
})();
```

Historically, IIFEs were used to create private scope before modern modules and lexical declarations became common.

---

# 45. Why IIFEs Need Expression Context

This:

```js
function () {}
```

does not form a normal standalone named function declaration.

Parentheses force expression context:

```js
(function () {})
```

Then:

```js
(function () {})();
```

invokes the resulting function value.

The key sequence is:

```text
function expression
→ value
→ invocation
```

---

# 46. Function Factories and Encapsulation

Example:

```js
function createAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit(amount) {
      balance += amount;
    },

    getBalance() {
      return balance;
    }
  };
}
```

The returned object exposes behavior but not the `balance` binding directly.

The state is protected through lexical access.

This is a form of closure-based encapsulation.

Later this will be compared with:

```text
private class fields
modules
WeakMap
composition
```

---

# 47. Pure Functions

A pure function can be described as one that:

```text
produces the same result for the same relevant input
and has no observable external side effects
```

Example:

```js
function square(x) {
  return x * x;
}
```

Pure functions are generally easier to:

```text
test
reason about
cache
compose
parallelize conceptually
```

Purity is not a JavaScript requirement.

It is a design property.

---

# 48. Impure Functions

Example:

```js
let count = 0;

function next() {
  count++;
  return count;
}
```

The result depends on mutable external state.

This is not automatically bad.

Many useful functions must perform:

```text
I/O
logging
database work
event publishing
state updates
```

The important question is whether the impurity is intentional and controlled.

---

# 49. Side Effects

A function may cause:

```text
object mutation
filesystem writes
network calls
database writes
event emission
logging
global state changes
timers
```

Example:

```js
function activate(user) {
  user.active = true;
}
```

The caller's object is changed.

This should be visible in the function contract.

---

# 50. Mutation vs Returning a New Value

Mutation:

```js
function activate(user) {
  user.active = true;
  return user;
}
```

New object:

```js
function activate(user) {
  return {
    ...user,
    active: true
  };
}
```

These differ in:

```text
identity
allocation
sharing
memory
predictability
performance
```

Neither is universally correct.

Choose based on ownership and architecture.

---

# 51. Function Composition

Composition feeds one function's result into another.

```js
const double = x => x * 2;
const increment = x => x + 1;

const result = increment(double(5));
```

Flow:

```text
5
 ↓
double
 ↓
10
 ↓
increment
 ↓
11
```

Composition can simplify:

```text
data transformation
validation
middleware
pipelines
```

But over-composition can make control flow difficult to discover.

---

# 52. Strategy Functions

Functions can represent policies.

Example:

```js
function sortItems(items, compare) {
  return [...items].sort(compare);
}
```

The caller supplies behavior:

```js
sortItems(users, (a, b) =>
  a.name.localeCompare(b.name)
);
```

This is often simpler than creating a separate class hierarchy for every policy.

---

# 53. Functions as Dependencies

Example:

```js
function createService(repository) {
  return {
    create(data) {
      return repository.insert(data);
    }
  };
}
```

A function can also be injected:

```js
function createClock(clock) {
  return {
    now() {
      return clock();
    }
  };
}
```

This enables:

```text
dependency injection
testing
decoupling
replaceable behavior
```

The service is explicitly given the capability it needs.

---

# 54. Function Wrappers

A higher-order wrapper can add cross-cutting behavior:

```js
function withLogging(fn) {
  return function (...args) {
    console.log("calling", fn.name);

    const result = fn(...args);

    console.log("result", result);

    return result;
  };
}
```

This is useful for:

```text
logging
timing
metrics
caching
retry
authorization
```

But wrappers can accidentally change semantics.

A production wrapper may need to preserve:

```text
this
errors
async behavior
metadata
identity expectations
```

---

# 55. Arrow Functions and `this`

A common statement is:

> “Arrow functions don't have `this`.”

The precise explanation is:

> Arrow functions do not create their own `this` binding.

Example:

```js
const obj = {
  value: 10,

  method() {
    const arrow = () => this.value;
    return arrow();
  }
};

console.log(obj.method());
```

The arrow uses lexical `this` from the surrounding method execution.

Chapter 14 will define invocation and `this` semantics precisely.

---

# 56. Arrow Functions and Constructability

Arrow functions cannot be used as constructors:

```js
const User = () => {};

new User();
```

throws a `TypeError`.

Traditional functions may be constructable:

```js
function User(name) {
  this.name = name;
}

const user = new User("A");
```

Therefore choose a function form based on the required semantics.

---

# 57. Function Declarations vs Function Expressions

Compare:

```js
greet();

function greet() {
  return "hello";
}
```

with:

```js
greet();

const greet = function () {
  return "hello";
};
```

The first has function-declaration initialization semantics.

The second attempts to access a lexical binding before its initializer has run.

This will be formally explained in Chapter 11.

---

# 58. Function Invocation and Execution State

When:

```js
fn(arg);
```

executes, the runtime must establish function execution state.

Conceptually:

```text
evaluate callable
      ↓
evaluate argument expressions
      ↓
prepare invocation state
      ↓
bind parameters
      ↓
execute function body
      ↓
produce completion
```

The complete specification model will be introduced through execution contexts and environment records.

---

# 59. Per-Invocation Parameter State

Example:

```js
function add(a, b) {
  return a + b;
}

add(1, 2);
add(10, 20);
```

Conceptually:

```text
Invocation #1
a → 1
b → 2

Invocation #2
a → 10
b → 20
```

Each call has its own parameter bindings.

This becomes essential for recursion and closures.

---

# 60. Recursion

A function can invoke itself:

```js
function factorial(n) {
  if (n <= 1) return 1;

  return n * factorial(n - 1);
}
```

Each active invocation contributes execution state.

Unbounded recursion:

```js
function recurse() {
  recurse();
}
```

eventually exhausts the implementation's available call-stack capacity and typically throws a `RangeError`.

The exact limit is engine-dependent.

---

# 61. Recursion Requires a Termination Argument

A recursive function should have a decreasing/progressing measure.

For:

```js
factorial(n)
```

the progress is:

```text
n
→ n - 1
→ n - 2
→ ...
→ 1
```

A production review should ask:

```text
What is the base case?
What makes progress?
Can progress stop?
What is the maximum depth?
```

---

# 62. Function Identity and Event Listeners

This is a practical example:

```js
const handler = () => {
  console.log("click");
};

element.addEventListener("click", handler);
element.removeEventListener("click", handler);
```

Correct because the same function object is supplied.

This does not work as intended:

```js
element.addEventListener("click", () => {
  console.log("click");
});

element.removeEventListener("click", () => {
  console.log("click");
});
```

Those are two different function objects.

---

# 63. Function Identity and Maps

Functions can be Map keys:

```js
const map = new Map();

const fn = () => 42;

map.set(fn, "handler");

console.log(map.get(fn));
```

A different function with equivalent source does not retrieve the same entry.

This follows object identity semantics.

---

# 64. Function Lifetime and Memory

A long-lived function can retain access to captured state.

Example:

```js
function createHandler(largeObject) {
  return () => largeObject.id;
}
```

If the returned function remains reachable, associated lexical state may remain reachable too.

This is not automatically a leak.

But it means function lifetime and captured state should be considered in memory-sensitive code.

The GC chapter will formalize reachability.

---

# 65. Higher-Order Functions as Architecture

Higher-order functions can implement:

```text
middleware
retry
logging
metrics
authorization
caching
decorators
validation
```

For example:

```js
function requireRole(role, handler) {
  return function (user, ...args) {
    if (user.role !== role) {
      throw new Error("Forbidden");
    }

    return handler(user, ...args);
  };
}
```

The wrapper represents policy around another behavior.

In production, authorization wrappers must be designed with explicit trust boundaries and error semantics.

---

# 66. Function Contracts

A robust function contract should define:

```text
Inputs
  types
  shape
  optionality
  defaults
  validation

Outputs
  type
  shape
  identity
  ownership

Side effects
  mutation
  I/O
  events

Errors
  thrown
  returned
  retryable

Timing
  synchronous
  asynchronous

Concurrency
  reentrancy
  shared state

Security
  trust boundary
  authority

Performance
  expected cost
  hot-path assumptions
```

Small function bodies can still have complicated contracts.

---

# 67. Production Function Boundaries

Do not choose function boundaries solely by line count.

Ask:

```text
Does this function represent one coherent behavior?
Does it own a meaningful piece of state?
Does it have a distinct failure boundary?
Does it require independent testing?
Does it have a different retry/transaction policy?
```

For example:

```text
calculateInvoice
saveInvoice
sendInvoiceEmail
publishInvoiceEvent
```

may belong to different boundaries because their failure semantics differ.

---

# 68. Pure Calculation + Side Effect Boundary

A useful architecture:

```js
function calculateTotal(items) {
  return items.reduce(
    (total, item) => total + item.price,
    0
  );
}

async function saveInvoice(invoice) {
  // external I/O
}
```

The calculation is locally testable.

The persistence function owns external state.

Separating these concerns often improves reliability, although the final architecture must match the domain.

---

# 69. Security — Functions Are Capabilities

A function reference can represent authority.

If you pass:

```js
createWriter(writeToDatabase);
```

you are granting the callee a capability to perform the supplied operation.

Therefore function references can be treated as:

```text
behavior
+
authority
```

This is useful in dependency injection and capability-oriented design.

It also means arbitrary untrusted functions must never be accepted as harmless data.

---

# 70. Security — Function Metadata Is Not Trust

Do not make authorization decisions such as:

```js
if (handler.name === "adminHandler") {
  grantAccess();
}
```

Function names can change and do not constitute authenticated identity or permission.

Use explicit trusted policy data instead.

---

# 71. Performance Considerations

Function calls have overhead, but modern JavaScript engines optimize many common patterns.

Avoid claims such as:

> “Function calls are always slow.”

Actual performance depends on:

```text
call frequency
JIT optimization
inlining
argument patterns
closures
allocation
polymorphism
algorithmic cost
I/O
```

Usually:

```text
correct abstraction
+
measurement
```

is better than prematurely eliminating helper functions.

---

# 72. Higher-Order Function Performance

Higher-order functions can introduce:

```text
callback invocation
closure allocation
temporary objects
iterator/callback overhead
```

But they can also improve:

```text
clarity
reuse
testability
composition
```

The correct decision is workload-dependent.

Measure actual bottlenecks before replacing clear abstractions.

---

# 73. Memory Considerations

Be alert to retained function references in:

```text
event listeners
timers
subscriptions
caches
module state
workers
queues
```

Ask:

```text
How long does the function live?
What does it capture?
Who owns it?
How is it released?
```

This will become important when studying closures, garbage collection, and resource cleanup.

---

# 74. Common Misconceptions

### “A function is just a block of code.”

Incomplete. It is an object/value with callable behavior.

### “All functions are constructors.”

False. Arrow functions are not constructable.

### “Arrow functions are just shorter syntax.”

False. They have different semantics.

### “Arrow functions have no `this`.”

Misleading. They do not create an own `this` binding.

### “Callback means asynchronous.”

False.

### “Extra arguments cause an error.”

False.

### “Missing arguments cause an error.”

Not generally; missing parameters receive `undefined`.

### “Rest and `arguments` are the same.”

False.

### “Identical function source means identical function.”

False. Identity is separate.

### “Function calls are always expensive.”

False. Performance depends on the engine and workload.

### “A closure is simply a nested function.”

Incomplete. The important behavior is retained lexical access.

---

# 75. Common Mistakes

### Mistake: invoking instead of passing

```js
register(handler());
```

instead of:

```js
register(handler);
```

### Mistake: forgetting `return` in arrow block bodies

```js
const f = x => {
  x * 2;
};
```

### Mistake: using truthiness where default semantics are required

### Mistake: mutating parameters without documenting it

### Mistake: wrapping methods without considering `this`

### Mistake: recreating callback functions when identity matters for removal

### Mistake: assuming `.length` gives a complete function signature

### Mistake: using function names as trust/authorization data

---

# 76. Comparison Table — Function Forms

| Form | Own `this` binding | Own `arguments` | Constructable | Typical use |
|---|---|---|---|---|
| Function declaration | Yes, invocation-dependent | Yes | Usually yes | named reusable behavior |
| Function expression | Yes, invocation-dependent | Yes | Usually yes | function values |
| Named function expression | Yes, invocation-dependent | Yes | Usually yes | recursion/debugging |
| Arrow function | No; lexical | No | No | callbacks/lexical context |
| IIFE | Depends on contained form | Depends | Depends | immediate scoped execution |

---

# 77. Comparison Table — Parameter Features

| Feature | Purpose |
|---|---|
| `a` | ordinary parameter binding |
| `a = 1` | default when argument is `undefined` |
| `...args` | collect remaining arguments |
| `arguments` | special traditional-function argument object |
| `{ name }` | object destructuring |
| `[a, b]` | array-pattern destructuring |

---

# 78. Execution Walkthrough — Declaration

```js
function add(a, b) {
  return a + b;
}
```

Conceptually:

```text
declaration processing
      ↓
function binding established
      ↓
function value available
      ↓
add(2, 3)
      ↓
invocation state
      ↓
a → 2
b → 3
      ↓
a + b
      ↓
5
```

The detailed execution-context mechanics come later.

---

# 79. Execution Walkthrough — Function Expression

```js
const add = function (a, b) {
  return a + b;
};
```

Conceptually:

```text
binding exists according to lexical declaration rules
      ↓
initializer executes
      ↓
function value created
      ↓
binding initialized
      ↓
later invocation
```

This explains why accessing the lexical binding before initialization can fail.

---

# 80. Execution Walkthrough — Callback

```js
function execute(value, callback) {
  return callback(value);
}

execute(10, x => x * 2);
```

Conceptually:

```text
evaluate execute
evaluate 10
evaluate arrow function
      ↓
invoke execute
      ↓
value → 10
callback → function value
      ↓
callback(10)
      ↓
20
      ↓
return 20
```

---

# 81. Execution Walkthrough — Default Parameter

```js
function greet(name = "Guest") {
  return name;
}

greet();
```

Conceptually:

```text
call
 ↓
argument absent
 ↓
parameter value is undefined
 ↓
default initializer applies
 ↓
name → "Guest"
 ↓
return "Guest"
```

For:

```js
greet(null);
```

the parameter receives `null`; the default does not apply.

---

# 82. Execution Walkthrough — Rest

```js
function collect(first, ...rest) {
  return [first, rest];
}

collect(1, 2, 3);
```

Conceptually:

```text
first → 1
rest → [2, 3]
```

---

# 83. Execution Walkthrough — Factory and Closure

```js
function makeAdder(x) {
  return y => x + y;
}

const add10 = makeAdder(10);
```

Conceptually:

```text
makeAdder invocation
 ↓
x → 10
 ↓
create arrow function
 ↓
retain lexical access to x
 ↓
return function
 ↓
add10 → returned function
```

Later:

```js
add10(5);
```

can use:

```text
x = 10
y = 5
```

and produce:

```text
15
```

---

# 84. Implementation From Scratch — `map`

Implement:

```js
function myMap(array, callback) {
  // ...
}
```

Requirements:

```text
new result array
callback(value, index, array)
preserve intended iteration semantics
do not mutate the original array
```

Do not use native `map` inside the implementation.

The exercise is about understanding:

```text
higher-order functions
callback contracts
iteration
result construction
```

---

# 85. Implementation — `filter`

Implement:

```js
function myFilter(array, predicate) {
  // ...
}
```

Requirements:

```text
invoke predicate
convert its result to Boolean semantics
collect matching values
return new array
```

Test:

```js
myFilter([1, 2, 3, 4], x => x % 2 === 0);
```

Expected:

```text
[2, 4]
```

---

# 86. Implementation — `reduce`

Implement:

```js
function myReduce(array, reducer, initialValue) {
  // ...
}
```

Study:

```text
accumulator
current value
index
array
initial value
empty-array behavior
callback errors
```

Then compare your design with native `reduce`.

The goal is to understand the callback contract rather than merely reproduce syntax.

---

# 87. Implementation — `once`

Implement:

```js
function once(fn) {
  // ...
}
```

Desired semantics:

```text
first call → invoke fn
later calls → return stored result
```

Then define behavior for:

```text
fn throws
different arguments
different this
undefined result
async result
```

This is where a tiny abstraction becomes a real engineering problem.

---

# 88. Implementation — Memoization

Start with:

```js
function memoize(fn) {
  // ...
}
```

Begin with primitive arguments.

Then analyze why generic memoization needs decisions around:

```text
object identity
multiple arguments
mutation
cache lifetime
eviction
errors
async results
```

This prepares the learner for caching and performance engineering.

---

# 89. Implementation — Composition

Implement:

```js
function compose(...fns) {
  // ...
}

function pipe(...fns) {
  // ...
}
```

Define the execution direction explicitly.

For example:

```js
compose(f, g)(x)
```

may mean:

```text
f(g(x))
```

while `pipe` may mean the reverse conceptual direction.

A good abstraction documents this rather than expecting users to infer it.

---

# 90. Implementation Progression

### Guided

Implement:

```text
myMap
myFilter
myReduce
once
```

### Partially Guided

Add:

```text
validation
callback argument contracts
error propagation
```

### No Reference

Build a reusable function-utility module.

### Edge-Case Hardened

Test:

```text
empty arrays
sparse arrays
callback exceptions
extra arguments
missing arguments
identity-sensitive callbacks
```

### Production-Grade

Add:

```text
documentation
tests
explicit compatibility guarantees
performance measurements where justified
```

---

# 91. Debugging Exercises

### Exercise 1

Why is the result `undefined`?

```js
const double = x => {
  x * 2;
};
```

### Exercise 2

Why does this invoke immediately?

```js
setTimeout(doWork(), 1000);
```

### Exercise 3

Why are these different?

```js
const a = () => 1;
const b = () => 1;

a === b;
```

### Exercise 4

Why does this mutate caller-visible state?

```js
function change(user) {
  user.name = "B";
}
```

### Exercise 5

Why does this not replace the caller's binding?

```js
function change(user) {
  user = { name: "B" };
}
```

### Exercise 6

Why do two factory instances have independent state?

```js
const a = makeCounter();
const b = makeCounter();
```

### Exercise 7

Why can this listener not be removed with the second arrow?

```js
element.addEventListener("click", () => {});
element.removeEventListener("click", () => {});
```

### Exercise 8

Why is an arrow function unsuitable here?

```js
const User = () => {};
new User();
```

---

# 92. Code Review Exercise — Callback API

Review:

```js
function execute(task) {
  return task();
}
```

Ask:

```text
Must task be callable?
Can it throw?
Is execution synchronous?
Can it return a Promise?
What arguments should task receive?
What this behavior is required?
Can it be cancelled?
How many times is it invoked?
```

The implementation is small; the contract is the real API.

---

# 93. Code Review Exercise — Mutation

Review:

```js
function activate(user) {
  user.active = true;
  return user;
}
```

Identify:

```text
input mutation
same-identity return
side effect
```

Compare with:

```js
function activate(user) {
  return {
    ...user,
    active: true
  };
}
```

The second changes identity and allocation behavior.

Neither is universally correct.

---

# 94. Code Review Exercise — Wrapper

Review:

```js
function withLogging(fn) {
  return (...args) => {
    console.log("calling");
    return fn(...args);
  };
}
```

Questions:

```text
Does it preserve this?
What happens if fn throws?
What if fn returns a Promise?
Should rejection be logged?
Should function metadata be preserved?
Does wrapping change identity?
```

The lesson is:

> A wrapper changes an API contract unless its behavior is deliberately preserved.

---

# 95. Code Review Exercise — Responsibility

Review:

```js
function createOrder(data) {
  validate(data);
  normalize(data);
  calculateTotals(data);
  saveToDatabase(data);
  sendEmail(data);
  updateSearch(data);
  publishEvent(data);
}
```

Do not split it merely because it is long.

Instead ask whether operations have different:

```text
failure boundaries
transaction boundaries
retry policies
latencies
dependencies
observability
```

Those differences provide stronger reasons for separating functions.

---

# 96. Interview Questions

## Junior

1. What is a JavaScript function?
2. What does first-class function mean?
3. Function declaration vs expression?
4. What is a callback?
5. What is a higher-order function?
6. Parameter vs argument?
7. What happens to a missing argument?
8. What is returned when no `return` executes?

## Mid-Level

9. Traditional function vs arrow?
10. Default parameter?
11. Rest parameter?
12. `arguments`?
13. Why does an arrow have no own `arguments`?
14. What is an IIFE?
15. What is a function factory?
16. `fn` vs `fn()`?
17. Why is callback not synonymous with asynchronous?
18. What is function identity?

## Senior

19. Callable vs constructable?
20. Explain lexical `this` for arrows.
21. Explain per-invocation parameter bindings.
22. Explain factory-created retained state.
23. Why are identical function expressions different objects?
24. Design a callback contract.
25. Compare mutation and returning a new value.
26. Explain risks of wrapping a method.

## Staff / Principal

27. How should function boundaries align with failure boundaries?
28. When should behavior be represented as a function parameter rather than inheritance?
29. How do higher-order functions implement cross-cutting concerns?
30. How can callback APIs create reliability problems?
31. How do function references represent capabilities?
32. How can retained closures affect memory?
33. How would you review a codebase for hidden callback side effects?
34. When does functional composition improve architecture and when does it become over-abstraction?

---

# 97. Predict-the-Output Exercises

Predict before executing:

### 1

```js
function add(a, b) {
  return a + b;
}

console.log(add(2, 3));
```

### 2

```js
const add = function (a, b) {
  return a + b;
};

console.log(add(2, 3));
```

### 3

```js
const add = (a, b) => a + b;

console.log(add(2, 3));
```

### 4

```js
const add = (a, b) => {
  a + b;
};

console.log(add(2, 3));
```

### 5

```js
function test(a, b) {
  console.log(a, b);
}

test(10);
```

### 6

```js
function test(a = 10) {
  return a;
}

console.log(test());
console.log(test(undefined));
console.log(test(null));
console.log(test(0));
```

### 7

```js
function test(a, ...rest) {
  return [a, rest];
}

console.log(test(1, 2, 3, 4));
```

### 8

```js
function test(a) {
  return arguments.length;
}

console.log(test(1, 2, 3));
```

### 9

```js
const a = () => 1;
const b = a;
const c = () => 1;

console.log(a === b);
console.log(a === c);
```

### 10

```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const a = makeCounter();
const b = makeCounter();

console.log(a());
console.log(a());
console.log(b());
```

### 11

```js
function execute(fn) {
  return fn();
}

console.log(execute(() => 42));
```

### 12

```js
function execute(fn) {
  return fn;
}

console.log(execute(() => 42));
```

### 13

```js
const obj = {
  value: 10,

  method() {
    const arrow = () => this.value;
    return arrow();
  }
};

console.log(obj.method());
```

### 14

```js
function makeAdder(x) {
  return y => x + y;
}

const add10 = makeAdder(10);

console.log(add10(5));
```

### 15

```js
const fn = function inner() {
  return inner.name;
};

console.log(fn());
```

---

# 98. Mastery Exercises

## Exercise A — First-Class Functions

Demonstrate one function being:

```text
assigned
stored in an array
stored in an object
passed as an argument
returned
used as a Map key
attached with a property
```

Explain function identity in every case.

## Exercise B — Function Form Matrix

Compare:

```text
declaration
expression
named expression
arrow
IIFE
```

for:

```text
syntax
timing
this
arguments
constructability
prototype
use cases
```

## Exercise C — Callback Contract

Design:

```js
runTask(task, options)
```

Specify:

```text
input
output
timing
invocation count
errors
cancellation
timeout
observability
```

## Exercise D — Closure Factory

Implement:

```js
createValidator(schema)
```

returning:

```js
validate(input)
```

Then explain the captured state.

## Exercise E — Wrapper

Implement:

```js
withRetry(fn, options)
```

and explicitly define:

```text
attempts
retryable errors
delay
backoff
sync/async behavior
return/rejection
```

## Exercise F — Composition

Implement `compose` and `pipe`, then explain their execution direction.

## Exercise G — Identity-Safe Subscription

Implement:

```js
subscribe(event, handler)
unsubscribe(event, handler)
```

and explain why unsubscription requires a function identity contract.

---

# 99. Production Function Design Framework

Before creating a function, ask:

```text
1. What responsibility does it own?
2. What inputs does it accept?
3. What are the valid input states?
4. What does it return?
5. Who owns the returned value?
6. Does it mutate inputs?
7. What side effects occur?
8. Can it throw?
9. Is it sync or async?
10. Can it be retried?
11. Does it depend on external state?
12. Can dependencies be injected?
13. Is its state shared?
14. Does it capture state?
15. What security authority does it possess?
16. What performance assumptions exist?
```

---

# 100. Function Review Checklist

Review:

```text
Inputs:
  type
  validation
  defaults
  mutation

Outputs:
  shape
  identity
  ownership

Errors:
  thrown
  returned
  retryable

Side effects:
  I/O
  state mutation
  events

Dependencies:
  explicit
  injectable

State:
  local
  shared
  captured

Async:
  timing
  cancellation
  retries

Security:
  authority
  trust boundary

Performance:
  hot path
  allocation
  invocation frequency

Maintainability:
  naming
  cohesion
  abstraction level
```

---

# 101. Concept Connections

## Depends On

- Chapters 01–08, especially values, bindings, expressions, control flow, and iteration.

## Builds Toward

- Chapter 10 — Scope, Lexical Environments, and Identifier Resolution.
- Chapter 11 — Hoisting and the Temporal Dead Zone.
- Chapter 12 — Execution Contexts and the Execution Model.
- Chapter 13 — Closures.
- Chapter 14 — `this`, Invocation, and Binding.
- Chapter 15 — Objects and Property Semantics.
- Chapter 18 — Classes and Object-Oriented JavaScript.
- Chapter 19 — Proxy and Metaprogramming.
- Chapter 25 — Iterables and Iterators.
- Chapter 35 — Promises.
- Chapter 36 — Async/Await.
- Chapter 39 — Concurrency and Parallelism.
- Chapter 74 — Functional Programming.
- Chapter 76 — Composition and Abstraction Design.
- Chapter 77 — Design Patterns.
- Chapter 78 — Production JavaScript Architecture.
- Chapter 80 — Library Authoring.
- Chapter 83 — Observability.
- Chapter 84 — Reliability.
- Chapter 85 — Performance.
- Chapter 86 — Testing.
- Chapter 88 — Debugging.

## Related Concepts

```text
callable objects
parameters
arguments
callbacks
higher-order functions
factories
closures
composition
side effects
purity
dependency injection
function identity
capabilities
```

## Concepts Revisited Later

Later chapters will formalize:

```text
lexical environments
execution contexts
references
this
internal [[Call]]
internal [[Construct]]
closures
iterator/generator functions
Promise callbacks
async function behavior
```

## Why This Chapter Matters Later

Functions are the primary unit of executable behavior in JavaScript.

A mature mental model must treat a function as:

```text
a value
+
an object
+
callable behavior
+
a lexical relationship
+
an API contract
```

This one model connects the foundations of JavaScript to closures, event systems, asynchronous programming, middleware, functional programming, dependency injection, and production architecture.

---

# 102. Key Takeaways

1. A function is an executable value.
2. Functions are first-class values.
3. Function values are objects with callable behavior.
4. Callable and constructable are distinct capabilities.
5. Function declarations and expressions have different initialization/timing semantics.
6. Arrow functions are behaviorally different from traditional functions.
7. Arrow functions do not create their own `this` binding.
8. Arrow functions do not create their own `arguments` binding.
9. Arrow functions are not constructable.
10. Callbacks are functions used according to another API's contract; they are not inherently asynchronous.
11. Higher-order functions accept and/or return functions.
12. Parameters become bindings for a specific invocation.
13. Missing arguments generally produce `undefined`.
14. Default parameters apply when an argument is `undefined`.
15. Rest parameters collect remaining arguments into an Array.
16. `arguments` is a traditional-function, array-like mechanism.
17. `fn` and `fn()` are fundamentally different.
18. Function identity matters for listeners, subscriptions, Maps, Sets, caches, and registries.
19. Function factories can produce independent state.
20. Closure behavior comes from retained lexical access to surrounding bindings.
21. IIFEs execute function expressions immediately.
22. Pure functions can make reasoning and testing easier.
23. Side effects and mutation should be intentional and documented.
24. Function wrappers can implement cross-cutting behavior but may change semantics.
25. Functions are useful for strategy injection and dependency injection.
26. A function reference can represent capability/authority.
27. Captured state affects function lifetime and memory reachability.
28. Function boundaries should reflect meaningful responsibility and failure semantics.
29. Performance claims about functions should be measured rather than based on folklore.
30. Function design is API design.

---

# 103. Completion Criteria

### Understand

- Define functions and first-class behavior.
- Explain all major function forms.
- Explain parameters, arguments, defaults, rest, and `arguments`.
- Explain callable vs constructable.

### Explain

- Explain callbacks and higher-order functions.
- Explain function identity.
- Explain factories.
- Explain foundational closure behavior.
- Explain lexical `this` for arrows.
- Explain mutation and side effects.

### Predict

- Predict return/default/rest/arguments behavior.
- Predict function identity.
- Predict callback passing versus invocation.
- Predict basic factory/closure behavior.

### Implement

- Implement `map`, `filter`, `reduce`, `once`, composition, and wrappers.
- Implement identity-safe subscription/unsubscription.

### Debug

- Diagnose missing returns.
- Diagnose callback invocation mistakes.
- Diagnose mutation/reassignment confusion.
- Diagnose listener identity issues.
- Diagnose captured-state behavior.

### Apply

- Design clear function contracts.
- Use functions for strategies and dependency injection.
- Make side effects and mutation explicit.

### Compare

- declaration vs expression
- traditional vs arrow
- callback vs asynchronous callback
- rest vs `arguments`
- function reference vs invocation
- callable vs constructable
- factory vs constructor
- pure vs impure
- mutation vs copy-and-return
- composition vs inheritance

### Defend

- Defend a function form based on semantics.
- Defend a function boundary based on responsibility.
- Defend a callback contract.
- Explain identity requirements.
- Explain when a higher-order abstraction is justified and when it becomes overengineering.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 10 — Scope, Lexical Environments, and Identifier Resolution

