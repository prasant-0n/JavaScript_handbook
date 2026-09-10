
# Chapter 08 --- Control Flow and Iteration

> **Status:** `[+] Completed`\
> **Role in curriculum:** Establishes how JavaScript selects paths,
> repeats work, exits or skips execution, and traverses data. Control
> flow is where expressions, values, bindings, blocks, iteration
> protocols, and completion behavior begin working together as actual
> program execution.

------------------------------------------------------------------------

## 1. Learning Objectives

By the end of this chapter, the learner should be able to:

-   Explain what control flow means.
-   Explain sequential, conditional, iterative, and abrupt control flow.
-   Use and reason about:
    -   `if`
    -   `else`
    -   `switch`
    -   `for`
    -   `while`
    -   `do...while`
    -   `for...of`
    -   `for...in`
    -   `break`
    -   `continue`
    -   labels
    -   ternary expressions.
-   Distinguish statement-level control flow from expression-level
    selection.
-   Explain truthiness as it affects conditions.
-   Explain block scope inside control structures.
-   Understand `switch` matching semantics and fall-through.
-   Explain loop initialization, condition checks, updates, and
    termination.
-   Explain how `break` and `continue` alter loop control.
-   Explain why `for...of` and `for...in` are fundamentally different.
-   Explain the iterable protocol used by `for...of`.
-   Explain how arrays, strings, Maps, Sets, and custom iterables
    participate in iteration.
-   Explain how inherited enumerable properties affect `for...in`.
-   Recognize why `for...in` is usually not appropriate for arrays.
-   Understand loop variable scope with `var` vs `let`.
-   Understand per-iteration bindings created by lexical loop
    declarations.
-   Understand labels without treating them as a substitute for
    structured design.
-   Explain how abrupt completion interacts with loops and `finally`.
-   Analyze loop performance without relying on simplistic "loop X is
    always faster" claims.
-   Design readable, safe, predictable iteration in production systems.

------------------------------------------------------------------------

# 2. Prerequisites

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.
-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.

The dependency chain is:

``` text
values
  ↓
expressions
  ↓
conditions
  ↓
control flow
  ↓
iteration
  ↓
iterators / generators / async iteration
```

------------------------------------------------------------------------

# 3. What Is Control Flow?

Control flow describes the order in which program instructions are
evaluated.

The simplest program:

``` js
const a = 10;
const b = 20;
const c = a + b;
```

mostly follows sequential flow:

``` text
statement 1
   ↓
statement 2
   ↓
statement 3
```

Control-flow constructs let the program choose among paths or repeat
work:

``` text
             ┌── condition true ──→ branch A
condition ───┤
             └── condition false → branch B
```

or:

``` text
body
 ↓
condition
 ↓
repeat / stop
```

Control flow is therefore the bridge between:

``` text
what code means
```

and:

``` text
which code actually executes
```

------------------------------------------------------------------------

# 4. Major Control-Flow Categories

JavaScript supports several major forms.

## Sequential

Statements execute in sequence.

## Conditional

Choose a path:

``` js
if
switch
?: 
```

## Iterative

Repeat work:

``` js
for
while
do...while
for...of
for...in
```

## Abrupt

Change normal flow:

``` js
break
continue
return
throw
```

`return` and `throw` belong to broader function/error control flow and
will be explored further later.

This chapter focuses primarily on branching and iteration.

------------------------------------------------------------------------

# 5. The `if` Statement

Basic form:

``` js
if (condition) {
  statement;
}
```

Example:

``` js
if (age >= 18) {
  console.log("Adult");
}
```

The condition expression is evaluated.

Its resulting value is converted using Boolean semantics.

Conceptually:

``` text
evaluate condition
      ↓
ToBoolean
      ↓
true?
 ┌────┴────┐
yes        no
 ↓          ↓
body       skip
```

This connects directly to Chapter 07.

------------------------------------------------------------------------

# 6. `if...else`

Example:

``` js
if (age >= 18) {
  console.log("Adult");
} else {
  console.log("Minor");
}
```

Exactly one branch is selected.

The condition is evaluated once for this decision.

Only the selected branch executes.

------------------------------------------------------------------------

# 7. `else if`

JavaScript does not have a distinct `else if` grammar mechanism separate
from nested `if` statements.

It is syntactic structure equivalent to:

``` js
if (a) {
  ...
} else {
  if (b) {
    ...
  } else {
    ...
  }
}
```

Example:

``` js
if (score >= 90) {
  grade = "A";
} else if (score >= 80) {
  grade = "B";
} else {
  grade = "C";
}
```

The conditions are tested in order.

The first matching branch wins.

------------------------------------------------------------------------

# 8. Conditions Use Truthiness

A condition does not require a Boolean value.

Example:

``` js
if ("hello") {
  console.log("runs");
}
```

because:

``` js
Boolean("hello")
```

is:

``` text
true
```

Similarly:

``` js
if (0) {
  ...
}
```

does not execute because:

``` js
Boolean(0)
```

is false.

This is why control-flow reasoning depends on `ToBoolean` semantics.

------------------------------------------------------------------------

# 9. Explicit vs Implicit Boolean Conditions

Both are valid:

``` js
if (value) {
  ...
}
```

and:

``` js
if (Boolean(value)) {
  ...
}
```

The first uses implicit Boolean conversion.

The second makes conversion explicit.

Usually:

``` js
if (value)
```

is clearer.

But in code review, ask:

> Is truthiness actually the intended business rule?

For example, a validation rule may need:

``` js
value !== null && value !== undefined
```

rather than:

``` js
if (value)
```

------------------------------------------------------------------------

# 10. Nested `if`

Example:

``` js
if (authenticated) {
  if (isAdmin) {
    showAdminPanel();
  }
}
```

This is valid.

But unnecessary nesting can reduce readability.

Often:

``` js
if (authenticated && isAdmin) {
  showAdminPanel();
}
```

is easier to understand.

However, merging conditions is not always superior.

When branches have independent business meaning, explicit nested
structure may be clearer.

------------------------------------------------------------------------

# 11. Guard Clauses

Instead of:

``` js
function process(user) {
  if (user) {
    if (user.active) {
      if (user.permissions) {
        return processUser(user);
      }
    }
  }
}
```

a function can often use guard clauses:

``` js
function process(user) {
  if (!user) return;
  if (!user.active) return;
  if (!user.permissions) return;

  return processUser(user);
}
```

This is still control flow, but the structure is flatter.

Guard clauses are especially useful when invalid cases should terminate
early.

------------------------------------------------------------------------

# 12. `else` Is Not Always Necessary

Example:

``` js
if (!user) {
  return;
}

processUser(user);
```

is often clearer than:

``` js
if (!user) {
  return;
} else {
  processUser(user);
}
```

After a guaranteed abrupt exit such as `return`, the `else` adds no
semantic value.

Production style should prioritize clear control-flow structure.

------------------------------------------------------------------------

# 13. `switch`

Basic form:

``` js
switch (value) {
  case 1:
    ...
    break;

  case 2:
    ...
    break;

  default:
    ...
}
```

`switch` is useful when comparing one discriminant against multiple
cases.

------------------------------------------------------------------------

# 14. `switch` Uses Strict-Equality-Like Matching

Case matching follows the language's `switch` semantics, which are based
on strict comparison behavior rather than loose equality coercion.

Example:

``` js
switch (value) {
  case 1:
    console.log("number");
    break;

  case "1":
    console.log("string");
    break;
}
```

For:

``` js
value = "1";
```

the String case matches, not the Number case.

This is an important difference from using `==`.

------------------------------------------------------------------------

# 15. `switch` Fall-Through

Consider:

``` js
switch (value) {
  case 1:
    console.log("one");

  case 2:
    console.log("two");
    break;
}
```

If:

``` js
value === 1
```

execution enters case 1 and continues into case 2 because there is no
`break`.

Output:

``` text
one
two
```

This is called:

``` text
fall-through
```

It is sometimes deliberate but is a common source of bugs.

------------------------------------------------------------------------

# 16. Intentional Fall-Through

Fall-through can be useful:

``` js
switch (status) {
  case "pending":
  case "queued":
    handleWaiting();
    break;

  case "done":
    handleDone();
    break;
}
```

Here both:

``` text
pending
queued
```

share one branch.

This is clearer than accidental fall-through because both cases
intentionally lead to the same code.

------------------------------------------------------------------------

# 17. `default`

A `switch` can have:

``` js
default:
```

which runs when no case matches.

Example:

``` js
switch (status) {
  case "active":
    ...
    break;

  case "inactive":
    ...
    break;

  default:
    ...
}
```

`default` is optional.

For exhaustive domain modeling, a default branch can also be used to
make unexpected values explicit.

------------------------------------------------------------------------

# 18. `switch` Execution Does Not Require Cases in Numeric Order

Cases are labels, not necessarily sorted branches.

This is valid:

``` js
switch (value) {
  case 100:
    ...
    break;

  case 2:
    ...
    break;

  case 50:
    ...
    break;
}
```

Matching is based on comparison, not numeric ordering.

------------------------------------------------------------------------

# 19. `switch` and Block Scope

Case clauses do not automatically give each case its own lexical scope.

This can cause declaration collisions:

``` js
switch (value) {
  case 1:
    const x = 10;
    break;

  case 2:
    const x = 20;
    break;
}
```

This can create a redeclaration problem because both declarations occupy
the same lexical scope associated with the switch statement.

Use explicit blocks if separate lexical scopes are desired:

``` js
switch (value) {
  case 1: {
    const x = 10;
    break;
  }

  case 2: {
    const x = 20;
    break;
  }
}
```

This is a useful production pattern.

------------------------------------------------------------------------

# 20. Ternary Conditional Expression

The conditional operator:

``` js
condition ? a : b
```

selects exactly one expression.

Example:

``` js
const label = active ? "Active" : "Inactive";
```

Only the selected branch expression is evaluated.

Unlike `if`, the conditional operator itself produces a value.

------------------------------------------------------------------------

# 21. `if` vs Ternary

Use:

``` js
if
```

when control flow is the main concern.

Use:

``` js
?:
```

when value selection is simple and clear.

Good:

``` js
const label = active ? "Active" : "Inactive";
```

Less readable:

``` js
const label = a ? b : c ? d : e ? f : g;
```

Deeply nested ternaries are technically valid but often poor production
design.

------------------------------------------------------------------------

# 22. The `for` Loop

Canonical form:

``` js
for (initialization; condition; update) {
  body;
}
```

Example:

``` js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

Conceptually:

``` text
initialization
      ↓
condition
      ↓
 body
      ↓
 update
      ↓
condition
      ↓
...
```

------------------------------------------------------------------------

# 23. `for` Loop Execution Order

For:

``` js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

the flow is:

``` text
let i = 0
 ↓
i < 3
 ↓
body
 ↓
i++
 ↓
i < 3
 ↓
body
 ↓
i++
 ↓
...
```

The update expression executes after the body of each completed
iteration unless control flow prevents reaching it.

This becomes important with `continue`.

------------------------------------------------------------------------

# 24. `for` Initialization Can Be Omitted

Valid:

``` js
let i = 0;

for (; i < 3; i++) {
  console.log(i);
}
```

The loop itself does not declare the variable.

This is useful when the state is initialized elsewhere.

But production code should favor the most local and obvious ownership of
loop state.

------------------------------------------------------------------------

# 25. `for` Condition Can Be Omitted

Valid:

``` js
for (;;) {
  work();
}
```

This is an infinite loop unless control flow exits.

It can be useful for:

-   worker loops,
-   servers,
-   schedulers,
-   event processors.

But termination must be explicit:

``` js
for (;;) {
  if (shouldStop()) break;
  work();
}
```

------------------------------------------------------------------------

# 26. Infinite Loops Must Have a Reason

An infinite loop is not automatically bad.

Bad:

``` js
for (;;) {
  // accidental
}
```

Good:

``` js
for (;;) {
  const job = await queue.next();

  if (job === null) {
    break;
  }

  await process(job);
}
```

The second loop has explicit lifecycle semantics.

Production code should make intentional nontermination obvious.

------------------------------------------------------------------------

# 27. `while`

The `while` loop checks its condition before every iteration.

Example:

``` js
while (condition) {
  work();
}
```

Conceptually:

``` text
condition
 ↓
true?
 ↓ yes
body
 ↓
condition
 ↓
...
```

If the condition is initially false, the body executes zero times.

------------------------------------------------------------------------

# 28. `do...while`

The `do...while` loop executes its body at least once.

Example:

``` js
do {
  work();
} while (condition);
```

Flow:

``` text
body
 ↓
condition
 ↓
repeat / stop
```

This is appropriate when one execution is required before checking
continuation.

------------------------------------------------------------------------

# 29. `while` vs `do...while`

  Loop           Condition checked     Minimum body executions
  -------------- ------------------- -------------------------
  `while`        before body                                 0
  `do...while`   after body                                  1

This is a semantic difference, not merely syntax preference.

------------------------------------------------------------------------

# 30. `break`

`break` terminates the nearest applicable loop or switch.

Example:

``` js
for (let i = 0; i < 10; i++) {
  if (i === 5) {
    break;
  }
}
```

The loop exits when `i` becomes 5.

`break` is an abrupt control-flow operation.

It does not simply "set a Boolean."

------------------------------------------------------------------------

# 31. `continue`

`continue` skips the remainder of the current loop iteration and
proceeds to the loop's next iteration according to that loop's
semantics.

Example:

``` js
for (let i = 0; i < 5; i++) {
  if (i % 2 === 0) {
    continue;
  }

  console.log(i);
}
```

Output:

``` text
1
3
```

------------------------------------------------------------------------

# 32. `continue` in a `for` Loop

Important:

``` js
continue;
```

does not mean:

``` text
jump straight to condition
```

For a `for` loop, the update expression is reached according to the loop
semantics.

Example:

``` js
for (let i = 0; i < 3; i++) {
  if (i === 1) {
    continue;
  }

  console.log(i);
}
```

The update:

``` js
i++
```

still occurs.

This distinction matters when reasoning about loop termination.

------------------------------------------------------------------------

# 33. `continue` in `while`

For:

``` js
while (condition) {
  if (skip) {
    continue;
  }

  work();
}
```

there is no automatic update clause.

Therefore the loop's condition is evaluated again.

If state required for termination is updated only in the skipped
portion, an infinite loop can result.

Example:

``` js
let i = 0;

while (i < 3) {
  if (i === 1) {
    continue;
  }

  i++;
}
```

This becomes an infinite loop because when `i` is 1, the update never
occurs.

This is a classic control-flow bug.

------------------------------------------------------------------------

# 34. Labels

JavaScript supports labeled statements:

``` js
outerLoop:
for (...) {
  ...
}
```

A label can be targeted by:

``` js
break outerLoop;
```

or:

``` js
continue outerLoop;
```

Labels are particularly useful for nested loops when you need to exit or
continue an outer loop directly.

------------------------------------------------------------------------

# 35. Labeled Break

Example:

``` js
outer:
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      break outer;
    }
  }
}
```

The `break` exits the labeled outer loop.

Without the label, `break` would only exit the nearest loop.

------------------------------------------------------------------------

# 36. Labeled Continue

Example:

``` js
outer:
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) {
      continue outer;
    }
  }
}
```

This skips to the next iteration of the outer loop.

Labeled control flow can be useful, but excessive use often signals that
the algorithm should be decomposed into functions.

------------------------------------------------------------------------

# 37. `for...of`

`for...of` iterates over **iterable values**.

Example:

``` js
const values = [10, 20, 30];

for (const value of values) {
  console.log(value);
}
```

The values are:

``` text
10
20
30
```

This is fundamentally different from iterating over property names.

------------------------------------------------------------------------

# 38. Iterable Protocol

An object is iterable when it provides the appropriate iteration
mechanism through:

``` js
Symbol.iterator
```

Conceptually:

``` text
iterable
   ↓
iterator
   ↓
next()
   ↓
{ value, done }
```

`for...of` consumes this protocol.

This is a major bridge to Chapter 25.

------------------------------------------------------------------------

# 39. Arrays Are Iterable

Arrays provide an iterator.

Therefore:

``` js
for (const value of [10, 20, 30]) {
  console.log(value);
}
```

iterates:

``` text
10
20
30
```

The loop does not enumerate property names such as:

``` text
"0"
"1"
"2"
```

It consumes the array's iterable behavior.

------------------------------------------------------------------------

# 40. Strings Are Iterable

Strings are iterable too.

Example:

``` js
for (const char of "😀") {
  console.log(char);
}
```

The String iterator is code-point-oriented.

This is one reason `for...of` on strings behaves differently from
indexing.

Recall Chapter 04:

``` text
string.length
  → UTF-16 code units

string iteration
  → Unicode code points
```

------------------------------------------------------------------------

# 41. Maps Are Iterable

Maps iterate their entries by default:

``` js
const map = new Map([
  ["a", 1],
  ["b", 2]
]);

for (const entry of map) {
  console.log(entry);
}
```

Each iteration produces a pair:

``` text
["a", 1]
["b", 2]
```

Destructuring is often used:

``` js
for (const [key, value] of map) {
  console.log(key, value);
}
```

------------------------------------------------------------------------

# 42. Sets Are Iterable

Sets iterate their values:

``` js
const set = new Set([10, 20, 30]);

for (const value of set) {
  console.log(value);
}
```

This reflects the abstraction:

``` text
Set
→ iterable collection of values
```

not:

``` text
object property enumeration
```

------------------------------------------------------------------------

# 43. Typed Arrays Are Iterable

Typed arrays implement iteration as well.

Example:

``` js
const bytes = new Uint8Array([1, 2, 3]);

for (const byte of bytes) {
  console.log(byte);
}
```

This becomes important in binary data processing.

------------------------------------------------------------------------

# 44. Custom Iterable

You can create your own iterable:

``` js
const range = {
  start: 1,
  end: 3,

  *[Symbol.iterator]() {
    for (let i = this.start; i <= this.end; i++) {
      yield i;
    }
  }
};

for (const value of range) {
  console.log(value);
}
```

This produces:

``` text
1
2
3
```

The object participates in the language's iterator protocol.

Chapter 25 will build custom iterators formally.

------------------------------------------------------------------------

# 45. `for...in`

`for...in` iterates over **enumerable property keys**.

Example:

``` js
const user = {
  name: "A",
  age: 25
};

for (const key in user) {
  console.log(key);
}
```

The loop produces property names such as:

``` text
name
age
```

This is fundamentally different from `for...of`.

------------------------------------------------------------------------

# 46. `for...in` Is About Properties

Conceptual model:

``` text
object
 ↓
enumerable property keys
 ↓
iterate keys
```

While:

``` text
for...of
```

means:

``` text
iterable
 ↓
iterator
 ↓
iterate values
```

This distinction is worth memorizing.

------------------------------------------------------------------------

# 47. `for...in` and Arrays

Arrays are objects with enumerable indexed properties.

Therefore:

``` js
const arr = [10, 20, 30];

for (const key in arr) {
  console.log(key);
}
```

can produce:

``` text
0
1
2
```

But the keys are strings:

``` text
"0"
"1"
"2"
```

`for...in` is usually a poor choice for array iteration because:

-   it iterates keys, not values;
-   inherited enumerable properties can participate;
-   ordering is property-enumeration semantics rather than array-value
    semantics;
-   arrays may contain non-index properties.

Prefer:

``` js
for (const value of arr) {
  ...
}
```

when you want array values.

------------------------------------------------------------------------

# 48. `for...in` and Inherited Properties

A major difference:

``` js
for...in
```

can enumerate enumerable properties inherited through the prototype
chain.

Example:

``` js
const parent = {
  inherited: 1
};

const child = Object.create(parent);

child.own = 2;

for (const key in child) {
  console.log(key);
}
```

Potential keys include:

``` text
own
inherited
```

depending on enumerability.

This is one reason `for...in` requires care with dictionary-like
objects.

------------------------------------------------------------------------

# 49. Safe Own-Property Filtering

If you intentionally use `for...in` and need only own properties:

``` js
for (const key in obj) {
  if (Object.hasOwn(obj, key)) {
    ...
  }
}
```

This explicitly filters inherited properties.

Depending on the use case, another approach may be clearer:

``` js
for (const key of Object.keys(obj)) {
  ...
}
```

`Object.keys` returns own enumerable string keys.

------------------------------------------------------------------------

# 50. `Object.keys` vs `for...in`

Conceptually:

``` text
Object.keys(obj)
  → array of own enumerable string keys

for...in
  → enumerable string keys across the object's prototype chain
```

The exact enumeration-order semantics should be understood separately in
Chapter 16.

------------------------------------------------------------------------

# 51. Loop Variables and `let`

Consider:

``` js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

The loop declaration creates lexical binding semantics.

In certain loop forms, each iteration has its own per-iteration binding.

This matters for closures.

Example:

``` js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks[0]());
console.log(callbacks[1]());
console.log(callbacks[2]());
```

Conceptually, each callback captures its corresponding iteration
binding.

------------------------------------------------------------------------

# 52. `var` and Loop Closures

Compare:

``` js
const callbacks = [];

for (var i = 0; i < 3; i++) {
  callbacks.push(() => i);
}
```

The callbacks can all observe the same function-scoped `var` binding.

After the loop, that binding has the final value.

Therefore the callbacks can all return:

``` text
3
```

This is not because closures are broken.

It is because the closure captures access to a shared binding.

Chapter 13 will formalize this.

------------------------------------------------------------------------

# 53. Per-Iteration Bindings

The `let` loop behavior is deeper than:

> "let is block-scoped."

For loop semantics, the language can create new lexical bindings for
successive iterations, enabling closures to observe distinct values.

Conceptually:

``` text
iteration 0
  i₀ = 0

iteration 1
  i₁ = 1

iteration 2
  i₂ = 2
```

A closure created in each iteration can capture its corresponding
binding.

This is one of the strongest examples of scope and control flow
interacting.

------------------------------------------------------------------------

# 54. `break` and Loop Cleanup

A loop can contain:

``` js
try
```

and:

``` js
finally
```

Example:

``` js
while (true) {
  try {
    break;
  } finally {
    cleanup();
  }
}
```

The abrupt control flow caused by `break` does not simply bypass
`finally`.

Cleanup semantics still apply.

This becomes important for resource-management reasoning.

------------------------------------------------------------------------

# 55. `continue` and `finally`

Similarly:

``` js
for (;;) {
  try {
    continue;
  } finally {
    cleanup();
  }
}
```

The `finally` block executes before the continuation takes effect.

This illustrates a broader rule:

``` text
abrupt completion
+
cleanup semantics
```

are coordinated by the language.

Chapter 29 and Chapter 30 will revisit this deeply.

------------------------------------------------------------------------

# 56. `return` Inside Loops

A function can return from inside a loop:

``` js
function findUser(users, id) {
  for (const user of users) {
    if (user.id === id) {
      return user;
    }
  }

  return null;
}
```

`return` exits the function, not just the loop.

This can be preferable to:

``` js
let found;

for (...) {
  if (...) {
    found = ...;
    break;
  }
}

return found;
```

when early return makes the algorithm clearer.

------------------------------------------------------------------------

# 57. `throw` Inside Loops

Similarly:

``` js
for (const item of items) {
  if (!isValid(item)) {
    throw new Error("Invalid item");
  }
}
```

`throw` exits normal control flow and begins exception propagation.

Control flow and error handling are therefore closely connected.

------------------------------------------------------------------------

# 58. Loop Termination Invariants

Good loop design starts with a termination argument.

Example:

``` js
for (let i = 0; i < n; i++) {
  ...
}
```

The invariant is roughly:

``` text
i increases
i is bounded above by n
```

Therefore the loop terminates under ordinary arithmetic assumptions.

When writing loops manually, ask:

``` text
What changes every iteration?
What makes progress?
What bounds the state?
What happens at the boundary?
Can the loop become infinite?
```

This is an algorithmic reasoning skill.

------------------------------------------------------------------------

# 59. Off-by-One Errors

Common:

``` js
for (let i = 0; i <= arr.length; i++) {
  ...
}
```

The last valid index is:

``` text
arr.length - 1
```

so:

``` js
i <= arr.length
```

goes one step too far.

Typical form:

``` js
for (let i = 0; i < arr.length; i++) {
  ...
}
```

The actual correct loop depends on the algorithm, but the boundary
condition must be explicit.

------------------------------------------------------------------------

# 60. Empty Collections

A robust loop should handle empty input.

For:

``` js
for (const item of []) {
  ...
}
```

the body executes zero times.

Likewise:

``` js
for (let i = 0; i < 0; i++) {
  ...
}
```

executes zero times.

Never design a loop that assumes at least one item unless the API
contract guarantees it.

------------------------------------------------------------------------

# 61. Mutation During Iteration

Consider:

``` js
const values = [1, 2, 3, 4];

for (let i = 0; i < values.length; i++) {
  if (values[i] % 2 === 0) {
    values.splice(i, 1);
  }
}
```

Mutation changes indexes while the loop is progressing.

This can cause elements to be skipped.

The general lesson:

> Mutating the collection you are traversing can change the traversal
> model.

Sometimes mutation is intentional and correct, but it must be designed
explicitly.

------------------------------------------------------------------------

# 62. Safer Filtering Instead of In-Place Mutation

Often:

``` js
const filtered = values.filter(value => value % 2 !== 0);
```

is clearer than manually deleting elements while iterating.

But this creates a new array and may have different memory/performance
implications.

The correct decision depends on:

``` text
clarity
allocation
workload
ownership
mutation requirements
```

------------------------------------------------------------------------

# 63. Iterating Object Properties Safely

If you need an object's own enumerable string keys:

``` js
for (const key of Object.keys(obj)) {
  const value = obj[key];
  ...
}
```

If you need key/value pairs:

``` js
for (const [key, value] of Object.entries(obj)) {
  ...
}
```

These APIs make the intended traversal model explicit.

For symbols and all own keys, later APIs include:

``` js
Reflect.ownKeys(obj)
```

Chapter 16 covers these semantics.

------------------------------------------------------------------------

# 64. `for...of` vs `Array.prototype.forEach`

These are not identical.

`for...of` supports:

``` js
break
continue
return
await in async functions
```

according to the surrounding context.

`forEach` is a method callback pattern.

You cannot use:

``` js
break;
```

inside a `forEach` callback to break the outer iteration.

This makes `for...of` often preferable when control flow is central.

------------------------------------------------------------------------

# 65. Async Code Warning

Consider:

``` js
items.forEach(async item => {
  await process(item);
});
```

This does not make `forEach` await all callbacks.

The callbacks are started according to the method semantics, and
`forEach` itself does not await the returned promises.

Prefer:

``` js
for (const item of items) {
  await process(item);
}
```

for sequential asynchronous processing.

Or use explicit concurrency control when parallelism is intended.

This will be covered deeply in Chapter 36 and Chapter 39.

------------------------------------------------------------------------

# 66. Iteration and Backpressure

Iteration becomes a more complex systems concern when the source is:

-   a stream,
-   async iterator,
-   network response,
-   database cursor,
-   queue.

Then:

``` text
producer rate
vs
consumer rate
```

matters.

A simple loop over an in-memory array is not the same as consuming an
infinite or externally controlled data source.

This distinction will lead into async iteration and streaming.

------------------------------------------------------------------------

# 67. Performance Considerations

Do not memorize:

``` text
for is fastest
```

as a universal rule.

Performance depends on:

-   engine,
-   data shape,
-   element types,
-   callback overhead,
-   allocation,
-   iterator protocol cost,
-   optimization state,
-   workload,
-   browser/runtime,
-   algorithmic complexity.

For example:

``` js
for
```

and:

``` js
for...of
```

can have different costs in some workloads.

But the relevant question is:

> Is this loop actually a performance bottleneck?

Measure before rewriting maintainable code into obscure
micro-optimizations.

------------------------------------------------------------------------

# 68. Performance --- Algorithm Dominates Syntax

Compare:

``` js
for (const x of values) {
  expensiveOperation(x);
}
```

and:

``` js
for (let i = 0; i < values.length; i++) {
  expensiveOperation(values[i]);
}
```

The difference between iteration syntaxes may be irrelevant if:

``` text
expensiveOperation
```

dominates runtime.

The correct optimization target is the actual bottleneck.

------------------------------------------------------------------------

# 69. Memory Considerations

Different iteration forms can have different allocation behavior.

Examples:

``` js
Object.keys(obj)
```

creates an array of keys.

``` js
Object.entries(obj)
```

creates an array containing entry pairs.

By contrast:

``` js
for...in
```

does not first require the user to explicitly create an array of all
keys.

But this does not mean `for...in` is always more memory-efficient
overall.

Runtime implementation details and workload matter.

Similarly:

``` js
[...iterable]
```

materializes all values into memory, while:

``` js
for (const value of iterable)
```

can consume incrementally.

------------------------------------------------------------------------

# 70. Security Considerations

Control flow can become security-sensitive when branches implement:

``` text
authorization
validation
rate limiting
tenant selection
security policy
```

Example:

``` js
if (user.isAdmin) {
  grantAccess();
}
```

is only as safe as the trust model of:

``` text
user
isAdmin
```

Do not confuse:

``` text
control-flow correctness
```

with:

``` text
security correctness
```

A branch can execute exactly as written and still implement an insecure
policy.

------------------------------------------------------------------------

# 71. Security --- Prototype Enumeration

Using:

``` js
for (const key in input) {
  ...
}
```

on untrusted objects can expose inherited enumerable properties.

This can become especially important in code that:

-   copies properties,
-   merges configuration,
-   interprets user input as options,
-   constructs authorization objects.

Prefer explicit own-property semantics where appropriate.

------------------------------------------------------------------------

# 72. Production Engineering --- Choose Iteration by Intent

Use:

``` text
for
```

when you need index/control-state semantics.

Use:

``` text
for...of
```

when you want iterable values.

Use:

``` text
for...in
```

when you explicitly want enumerable property keys and understand
prototype behavior.

Use:

``` text
Object.keys / values / entries
```

when own-property traversal is the intended model.

Use:

``` text
array methods
```

when declarative transformation/filtering is clearer.

This is a semantic decision, not a style contest.

------------------------------------------------------------------------

# 73. Production Engineering --- Avoid `for...in` for Arrays

Prefer:

``` js
for (const value of array) {
  ...
}
```

for values.

Use:

``` js
array.forEach(...)
```

for simple callback-based traversal.

Use:

``` js
array.map(...)
```

when producing a transformed array.

Use:

``` js
array.filter(...)
```

when selecting values.

Use a classic `for` when you need:

``` text
index
break
continue
mutation strategy
tight control
```

------------------------------------------------------------------------

# 74. Production Engineering --- Make Termination Obvious

A loop should have a clear answer to:

``` text
How does it stop?
```

For a classic loop:

``` js
for (let i = 0; i < limit; i++) {
  ...
}
```

termination is visually obvious.

For:

``` js
while (condition) {
  ...
}
```

make the state that changes `condition` easy to find.

For:

``` js
for (;;) {
  ...
}
```

document the explicit exit conditions.

------------------------------------------------------------------------

# 75. Common Misconceptions

## Misconception 1 --- `for...in` iterates array values.

False.

It iterates enumerable property keys.

## Misconception 2 --- `for...of` works on every object.

False.

The object must be iterable.

## Misconception 3 --- `for...of` returns property names.

False.

It consumes iterator values.

## Misconception 4 --- `continue` skips the loop update in a `for` loop.

False.

The `for` loop's update step is reached according to its semantics.

## Misconception 5 --- `switch` cases automatically stop.

False.

Without `break` or another abrupt control-flow mechanism, fall-through
occurs.

## Misconception 6 --- `else if` is a special independent control-flow primitive.

It is structurally nested `if` syntax.

## Misconception 7 --- `while` always runs at least once.

False.

`while` can run zero times.

## Misconception 8 --- `do...while` can run zero times.

False.

Its body runs at least once.

## Misconception 9 --- `const` loop variables can never be used in loops.

False.

For example:

``` js
for (const value of values) {
  ...
}
```

is valid because each iteration provides the appropriate binding.

## Misconception 10 --- `forEach(async ...)` waits for all promises.

False.

`forEach` does not await the callback promises.

------------------------------------------------------------------------

# 76. Common Mistakes

### Mistake: off-by-one boundaries

``` js
i <= arr.length
```

often goes too far.

### Mistake: accidental infinite loop

``` js
while (condition) {
  // condition never changes
}
```

### Mistake: `continue` before required state update

Can create infinite loops.

### Mistake: missing `break` in `switch`

Creates accidental fall-through.

### Mistake: `for...in` on arrays

Iterates keys rather than values.

### Mistake: inherited properties

`for...in` can include prototype properties.

### Mistake: mutating a collection while traversing it

Can skip or reorder elements unexpectedly.

### Mistake: async `forEach`

It does not provide sequential or awaited iteration.

------------------------------------------------------------------------

# 77. Comparison Table --- Core Loops

  -------------------------------------------------------------------------------------------
  Construct      Iterates / repeats          Can `break`?  Can `continue`? Typical use
                 based on                                                  
  -------------- ----------------------- ---------------- ---------------- ------------------
  `for`          explicit                             Yes              Yes precise loop
                 init/condition/update                                     control

  `while`        condition                            Yes              Yes condition-driven
                                                                           repetition

  `do...while`   condition after body                 Yes              Yes guaranteed first
                                                                           execution

  `for...of`     iterable values                      Yes              Yes collection/value
                                                                           traversal

  `for...in`     enumerable property                  Yes              Yes property-key
                 keys                                                      traversal

  `switch`       case matching                        Yes              N/A multi-branch
                                                                           selection
  -------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 78. Comparison --- `for...of` vs `for...in`

  Property               `for...of`               `for...in`
  ---------------------- ------------------------ -------------------------------
  Primary model          iteration protocol       property enumeration
  Produces               values                   property keys
  Requires               iterable                 object-like value
  Arrays                 recommended for values   usually avoid
  Strings                supported                property names
  Maps                   entries by default       not map values
  Sets                   values                   not set values
  Prototype properties   not the same concept     can participate
  Custom behavior        `Symbol.iterator`        enumerable property semantics

------------------------------------------------------------------------

# 79. Execution Walkthrough --- `for`

``` js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

Conceptually:

``` text
1. Create loop binding i.
2. Initialize i to 0.
3. Evaluate i < 3.
4. If false, exit.
5. Execute body.
6. Evaluate update i++.
7. Repeat from condition.
```

The exact specification semantics include lexical-environment handling
for the loop.

------------------------------------------------------------------------

# 80. Execution Walkthrough --- `while`

``` js
let i = 0;

while (i < 3) {
  console.log(i);
  i++;
}
```

Conceptually:

``` text
1. Evaluate i < 3.
2. If false, exit.
3. Execute body.
4. Update i.
5. Repeat condition.
```

Unlike a `for` loop, the update is ordinary code in the body.

This makes responsibility for termination more explicit---and easier to
accidentally break.

------------------------------------------------------------------------

# 81. Execution Walkthrough --- `do...while`

``` js
let i = 0;

do {
  console.log(i);
  i++;
} while (i < 3);
```

Conceptually:

``` text
1. Execute body.
2. Update state.
3. Evaluate condition.
4. Repeat or exit.
```

Therefore one body execution is guaranteed.

------------------------------------------------------------------------

# 82. Execution Walkthrough --- `for...of`

``` js
for (const value of values) {
  console.log(value);
}
```

Conceptual high-level flow:

``` text
1. Obtain iterator from iterable.
2. Request next result.
3. Inspect { value, done }.
4. If done, finish.
5. Bind value.
6. Execute body.
7. Request next result.
8. Repeat.
```

The exact iterator-closing behavior on abrupt completion matters and
will be discussed more fully in Chapter 25.

------------------------------------------------------------------------

# 83. Execution Walkthrough --- `for...in`

``` js
for (const key in obj) {
  console.log(key);
}
```

Conceptually:

``` text
1. Obtain enumerable property keys according to for-in semantics.
2. Select the next key.
3. Bind key.
4. Execute body.
5. Continue until enumeration is complete.
```

The exact property-order and prototype/enumerability rules are detailed
in Chapter 16.

------------------------------------------------------------------------

# 84. Execution Walkthrough --- `switch`

``` js
switch (value) {
  case "a":
    handleA();
    break;

  case "b":
    handleB();
    break;

  default:
    handleDefault();
}
```

Conceptually:

``` text
evaluate discriminant
      ↓
compare against cases
      ↓
find matching case / default
      ↓
enter execution
      ↓
fall through until abrupt exit/end
```

This explains why `break` matters.

------------------------------------------------------------------------

# 85. Implementation From Scratch --- Loop Model

Build a tiny interpreter for:

``` text
SET x 0
WHILE x < 3
  PRINT x
  ADD x 1
END
```

Represent each instruction as an object.

The interpreter should support:

``` text
environment/bindings
condition evaluation
jumps
loop repetition
break
```

This is not an attempt to reproduce JavaScript.

The objective is to understand that high-level control-flow syntax
ultimately maps to:

``` text
state
+
conditions
+
transitions
```

------------------------------------------------------------------------

# 86. Implementation Exercise --- Safe Property Iteration

Implement:

``` js
function ownEntries(obj) {
  // ...
}
```

without directly returning:

``` js
Object.entries(obj)
```

The function should:

``` text
iterate own enumerable string keys
preserve key/value relationship
ignore inherited enumerable properties
```

Then compare the implementation against:

``` js
Object.entries(obj)
```

for a test object with inherited properties.

------------------------------------------------------------------------

# 87. Implementation Exercise --- Range Iterable

Implement:

``` js
range(start, end, step = 1)
```

so that:

``` js
for (const value of range(1, 5)) {
  console.log(value);
}
```

produces:

``` text
1
2
3
4
5
```

Then support:

``` js
range(5, 1, -1)
```

and define what happens when:

``` text
step === 0
```

The important engineering challenge is defining the contract and
termination behavior.

------------------------------------------------------------------------

# 88. Implementation Progression

### Guided

Implement:

``` text
range
safeOwnKeys
```

### Partially Guided

Add:

``` text
reverse range
negative steps
input validation
```

### No Reference

Build a custom iterable with explicit iterator state.

### Edge-Case Hardened

Test:

``` text
empty ranges
negative ranges
step larger than distance
wrong step direction
zero step
very large bounds
```

### Production-Grade

Add:

-   deterministic tests,
-   documented inclusive/exclusive semantics,
-   error handling,
-   termination guarantees,
-   performance limits.

------------------------------------------------------------------------

# 89. Debugging Exercises

## Exercise 1

Find the bug:

``` js
for (let i = 0; i <= array.length; i++) {
  console.log(array[i]);
}
```

## Exercise 2

Find the termination bug:

``` js
let i = 0;

while (i < 10) {
  if (i === 5) continue;
  i++;
}
```

## Exercise 3

Explain the fall-through:

``` js
switch (status) {
  case "pending":
    console.log("waiting");

  case "done":
    console.log("complete");
}
```

## Exercise 4

Explain why:

``` js
for (const key in array)
```

is not equivalent to:

``` js
for (const value of array)
```

## Exercise 5

Explain the difference between:

``` js
for (const value of map)
```

and:

``` js
for (const key in map)
```

## Exercise 6

Explain why:

``` js
items.forEach(async item => {
  await process(item);
});
```

does not make the surrounding function automatically await all
processing.

## Exercise 7

Find the bug:

``` js
for (const key in input) {
  result[key] = input[key];
}
```

when `input` may have inherited enumerable properties.

------------------------------------------------------------------------

# 90. Code Review Exercise

Review:

``` js
for (const key in config) {
  options[key] = config[key];
}
```

Questions:

``` text
Should inherited properties be copied?
Should Symbols be copied?
Should only own enumerable properties be copied?
Should getters run?
Could prototype pollution be involved?
Would Object.entries be clearer?
Should keys be allowlisted?
```

A stronger implementation might be:

``` js
for (const [key, value] of Object.entries(config)) {
  options[key] = value;
}
```

But even this does not automatically solve every security concern.

The correct implementation depends on the data contract.

------------------------------------------------------------------------

# 91. Code Review Exercise --- Loop Choice

Review:

``` js
array.forEach(item => {
  if (shouldStop(item)) {
    return;
  }

  process(item);
});
```

A developer claims:

> "return here breaks the loop."

It does not.

It returns from the callback invocation.

If actual loop termination is required, use:

``` js
for (const item of array) {
  if (shouldStop(item)) {
    break;
  }

  process(item);
}
```

This is an excellent example of choosing syntax according to
control-flow intent.

------------------------------------------------------------------------

# 92. Interview Questions

## Junior

1.  What is control flow?
2.  What is the difference between `if` and `switch`?
3.  What does `break` do?
4.  What does `continue` do?
5.  What is the difference between `while` and `do...while`?
6.  What is a ternary operator?

## Mid-Level

7.  What is the difference between `for...of` and `for...in`?
8.  Why should `for...in` usually not be used for arrays?
9.  What is an iterable?
10. What is the role of `Symbol.iterator`?
11. Why can `for...in` see inherited properties?
12. Why can `let` behave differently from `var` inside loops?

## Senior

13. Explain per-iteration lexical bindings.
14. Explain `continue` behavior in `for` vs `while`.
15. Explain `switch` fall-through.
16. How does `for...of` consume an iterator?
17. Why can mutation during iteration be dangerous?
18. Why does `forEach(async ...)` not provide awaited sequential
    iteration?
19. How would you choose among `for`, `for...of`, array methods, and
    `for...in`?

## Staff / Principal

20. Design a loop policy for a high-throughput Node.js service.
21. How would you reason about iterator overhead vs algorithmic cost?
22. How do control-flow decisions affect observability and error
    handling?
23. How can property enumeration become a security issue?
24. How would you audit a large codebase for infinite-loop risks?
25. How would you design an iterator API for a production library?
26. When should explicit loops be preferred over functional array
    methods?

------------------------------------------------------------------------

# 93. Predict-the-Output Exercises

Predict before running.

### 1

``` js
if ("hello") {
  console.log("yes");
}
```

### 2

``` js
if (0) {
  console.log("yes");
} else {
  console.log("no");
}
```

### 3

``` js
switch ("1") {
  case 1:
    console.log("number");
    break;

  case "1":
    console.log("string");
    break;

  default:
    console.log("default");
}
```

### 4

``` js
switch (1) {
  case 1:
    console.log("A");

  case 2:
    console.log("B");
    break;
}
```

### 5

``` js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

### 6

``` js
let i = 0;

while (i < 3) {
  console.log(i);
  i++;
}
```

### 7

``` js
let i = 0;

do {
  console.log(i);
  i++;
} while (i < 0);
```

### 8

``` js
for (let i = 0; i < 5; i++) {
  if (i === 2) continue;
  console.log(i);
}
```

### 9

``` js
for (let i = 0; i < 5; i++) {
  if (i === 2) break;
  console.log(i);
}
```

### 10

``` js
const values = [10, 20, 30];

for (const value of values) {
  console.log(value);
}
```

### 11

``` js
const values = [10, 20, 30];

for (const key in values) {
  console.log(key);
}
```

### 12

``` js
const text = "😀";

for (const value of text) {
  console.log(value);
}
```

### 13

``` js
const map = new Map([
  ["a", 1],
  ["b", 2]
]);

for (const value of map) {
  console.log(value);
}
```

### 14

``` js
const obj = {
  a: 1,
  b: 2
};

for (const key in obj) {
  console.log(key, obj[key]);
}
```

### 15

``` js
const parent = { inherited: 1 };
const child = Object.create(parent);
child.own = 2;

for (const key in child) {
  console.log(key);
}
```

### 16

``` js
const callbacks = [];

for (var i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

### 17

``` js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

### 18

``` js
let i = 0;

while (i < 3) {
  if (i === 1) continue;
  i++;
}
```

Determine whether this terminates.

### 19

``` js
outer:
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      break outer;
    }

    console.log(i, j);
  }
}
```

### 20

``` js
const result = true ? "A" : "B";
console.log(result);
```

------------------------------------------------------------------------

# 94. Mastery Exercises

## Exercise A --- Control-Flow Trace

Take:

``` js
for (let i = 0; i < 5; i++) {
  if (i === 2) continue;

  if (i === 4) break;

  console.log(i);
}
```

Write the exact control-flow sequence, including:

``` text
condition
body
continue
update
break
termination
```

## Exercise B --- Loop Selection

For each task, choose the best iteration construct and defend it:

``` text
1. Traverse array values until a match is found.
2. Transform every array value.
3. Iterate own object keys.
4. Iterate Map entries.
5. Run until an external condition becomes false.
6. Guaranteed one initial attempt.
7. Nested search that must exit two loops.
8. Process asynchronous jobs sequentially.
```

## Exercise C --- Termination Proof

Take a custom loop and write:

``` text
loop invariant
progress measure
termination condition
boundary case
```

## Exercise D --- Prototype Safety

Create an object with:

``` text
own enumerable properties
inherited enumerable properties
non-enumerable properties
symbol properties
```

Compare:

``` js
for...in
Object.keys
Object.values
Object.entries
Reflect.ownKeys
for...of
```

Explain exactly what each observes.

## Exercise E --- Closure + Loop

Create examples using both:

``` js
var
let
```

inside loops with callbacks.

Explain the binding model rather than simply saying:

> "let fixes the loop."

------------------------------------------------------------------------

# 95. Production Decision Framework

Before selecting an iteration construct, ask:

``` text
1. Am I traversing values or property keys?
2. Is the source iterable?
3. Do I need the index?
4. Do I need break/continue?
5. Is traversal synchronous or asynchronous?
6. Is the collection finite?
7. Can the collection mutate during traversal?
8. Can inherited properties matter?
9. Do I need own properties only?
10. Do I need one pass or a transformation?
11. What is the termination argument?
12. What is the error-handling behavior?
13. What memory does the traversal require?
14. Does the actual workload justify optimization?
```

The answer should determine the construct.

------------------------------------------------------------------------

# 96. Concept Connections

## Depends On

-   Chapter 01 --- JavaScript, ECMAScript, and the Runtime Landscape.
-   Chapter 02 --- Values, Types, and the JavaScript Type System.
-   Chapter 03 --- Numbers, Floating Point, and BigInt.
-   Chapter 04 --- Strings, Unicode, and Text Semantics.
-   Chapter 05 --- Variables, Declarations, and Assignment.
-   Chapter 06 --- Operators and Expressions.
-   Chapter 07 --- Type Conversion, Coercion, and Equality.

## Builds Toward

-   Chapter 09 --- Functions and First-Class Behavior.
-   Chapter 10 --- Scope, Lexical Environments, and Identifier
    Resolution.
-   Chapter 11 --- Hoisting and the Temporal Dead Zone.
-   Chapter 12 --- Execution Contexts and the Execution Model.
-   Chapter 13 --- Closures.
-   Chapter 22 --- Arrays.
-   Chapter 25 --- Iterables and Iterators.
-   Chapter 26 --- Generators and Async Generators.
-   Chapter 29 --- Errors and Error Handling.
-   Chapter 30 --- Resource Management and Cleanup.
-   Chapter 36 --- Async/Await.
-   Chapter 38 --- Async Iteration and Streaming.
-   Chapter 39 --- Concurrency and Parallelism.
-   Chapter 45 --- Memory Management and Garbage Collection.
-   Chapter 56 --- Browser Security.
-   Chapter 57 --- JavaScript Security Engineering.

## Related Concepts

``` text
truthiness
branching
iteration
scope
bindings
iterables
iterators
enumeration
completion
abrupt completion
closures
termination
short-circuiting
```

## Concepts Revisited Later

-   Iterator protocols.
-   Iterator closing.
-   Generator control flow.
-   Async iteration.
-   Error propagation.
-   `finally` cleanup.
-   Collection semantics.
-   Property enumeration.
-   Closure capture in loops.
-   Event-loop interaction with asynchronous loops.

## Why This Chapter Matters Later

Control flow is where the language starts expressing algorithms and
state transitions.

Advanced JavaScript requires reasoning about more than:

``` text
"this line runs next"
```

You must understand:

``` text
which branch is selected
which expressions are evaluated
which bindings exist
when iteration state changes
what can terminate or skip execution
what cleanup runs on abrupt completion
```

These concepts reappear in functions, closures, async systems,
iterators, generators, streams, and production architecture.

------------------------------------------------------------------------

# 97. Key Takeaways

1.  Control flow determines which parts of a program execute and in what
    order.
2.  `if` and `switch` provide conditional branching.
3.  Conditions use Boolean conversion semantics.
4.  `switch` uses strict-comparison-style matching rather than loose
    equality coercion.
5.  `switch` can fall through between cases unless control flow exits.
6.  Case clauses share lexical scope unless explicit blocks are
    introduced.
7.  The ternary operator is an expression that selects one of two branch
    expressions.
8.  `for`, `while`, and `do...while` implement different repetition
    semantics.
9.  `while` can execute zero times.
10. `do...while` executes its body at least once.
11. `break` exits the nearest applicable loop or switch.
12. `continue` skips to the next iteration according to the loop's
    semantics.
13. In a `for` loop, `continue` does not eliminate the loop's update
    step.
14. Labels allow targeted `break` and `continue` for nested control
    flow.
15. `for...of` consumes iterable values.
16. `for...in` enumerates enumerable property keys.
17. `for...in` can include inherited enumerable properties.
18. `for...in` is generally not the right abstraction for traversing
    array values.
19. `Object.keys`, `Object.values`, and `Object.entries` make
    own-property traversal explicit.
20. Strings, arrays, Maps, Sets, and typed arrays can be iterated with
    `for...of`.
21. Custom iterables participate through `Symbol.iterator`.
22. `let` loop semantics can create per-iteration bindings that closures
    capture separately.
23. `var` can cause multiple loop callbacks to observe one shared
    binding.
24. Mutating a collection while iterating can invalidate assumptions
    about traversal.
25. Loop correctness requires an explicit termination argument.
26. Off-by-one errors and accidental infinite loops are common
    control-flow bugs.
27. `forEach(async ...)` does not automatically await callback promises.
28. Performance differences among loop constructs are workload- and
    engine-dependent.
29. Control-flow choice should be driven by semantic intent, not
    simplistic performance folklore.
30. Control flow used for security decisions must be reviewed as policy,
    not merely as syntax.

------------------------------------------------------------------------

# 98. Completion Criteria

### Understand

-   Explain all core branching and loop constructs.
-   Explain `for...of` vs `for...in`.
-   Explain `break`, `continue`, and labels.
-   Explain iterable-based traversal.

### Explain

-   Explain switch fall-through.
-   Explain per-iteration lexical bindings.
-   Explain truthiness in conditions.
-   Explain termination and off-by-one errors.
-   Explain why `forEach(async ...)` does not await all callbacks.

### Predict

-   Predict branch selection.
-   Predict exact loop output.
-   Predict `break` / `continue` behavior.
-   Predict property enumeration vs iterable traversal.

### Implement

-   Build a small control-flow interpreter.
-   Build a custom iterable.
-   Build a safe own-property iteration utility.
-   Build a range iterable with a termination contract.

### Debug

-   Find infinite loops.
-   Find skipped iterations.
-   Find off-by-one errors.
-   Find switch fall-through bugs.
-   Find prototype-enumeration issues.
-   Find incorrect async iteration patterns.

### Apply

-   Choose the right iteration construct based on data model and
    control-flow requirements.
-   Design loops with explicit termination and error behavior.

### Compare

-   `if` vs `switch`
-   `if` vs ternary
-   `for` vs `while`
-   `while` vs `do...while`
-   `for...of` vs `for...in`
-   `for...of` vs `forEach`
-   `Object.keys` vs `for...in`
-   `break` vs `return`
-   `break` vs `continue`

### Defend

-   Defend a loop construct based on semantics, correctness,
    readability, memory, performance, and security.
-   Explain the exact control-flow path for a nontrivial loop.
-   Defend a termination argument for custom iteration logic.

**Chapter status:** `[+] Completed`

**Next chapter in sequence:** Chapter 09 --- Functions and First-Class
Behavior


