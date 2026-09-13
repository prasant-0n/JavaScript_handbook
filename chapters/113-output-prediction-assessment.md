# Chapter 113 — Output Prediction Assessment — 25 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 25 output/behavior prediction problems designed to test whether you can execute JavaScript mentally, step by step, across language semantics, objects, scope, `this`, coercion, prototypes, iterators, Promises, async/await, event-loop reasoning, and runtime boundaries.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Specialist · Runtime/Engine Engineer · Node.js Architect · Browser Engineer · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Predict first. Execute second. Explain the mechanism third.**
>
> **Important:** Some questions ask for exact output. Others intentionally ask for ordering, errors, or whether an apparent result can be guaranteed. Do not force every problem into a simple “what prints?” pattern.

---

# 1. Assessment Mission

This assessment tests:

```text
mental execution
+
semantic precision
+
ordering reasoning
+
scope reasoning
+
object/prototype reasoning
+
async scheduling reasoning
```

The goal is not:

```text
memorize common interview outputs
```

The goal is:

```text
simulate the program from its actual rules
```

---

# 2. Instructions

For every problem:

```text
1. Do not run the code.
2. Predict the exact result.
3. Predict any thrown error.
4. Predict ordering.
5. Write the execution trace.
6. State the language/runtime rule.
7. Only then execute/verify.
```

Use:

```text
Prediction:
Actual:
Trace:
Rule:
Confidence:
```

For output-sensitive questions also record:

```text
Environment:
```

where relevant:

```text
ECMAScript
browser
Node.js
ES module
CommonJS
strict mode
```

---

# 3. Scoring

Each question:

```text
0–4 points
```

Total:

```text
100 points
```

Suggested interpretation:

```text
90–100 → Excellent mental execution
80–89  → Strong, targeted gaps
70–79  → Good but inconsistent
60–69  → Significant prediction gaps
<60    → Rebuild semantics before advanced assessment
```

These are learning heuristics, not hiring standards.

---

# 4. Full-Credit Standard

A correct answer should include:

```text
exact output
+
correct order
+
mechanistic explanation
```

For harder questions:

```text
runtime distinction
+
edge case
```

is also required.

---

# 5. Question 1 — Primitive Coercion

```js
console.log("5" + 2);
console.log("5" - 2);
console.log(true + 1);
console.log(null + 1);
console.log(undefined + 1);
```

### Predict

Write all five outputs in order.

### Then explain

Why does:

```text
+
```

behave differently from:

```text
-
```

in these examples?

### Record

```text
Prediction:
Actual:
Trace:
Rule:
Confidence:
```

---

# 6. Question 2 — Equality

```js
console.log(false == 0);
console.log(false === 0);
console.log(null == undefined);
console.log(null === undefined);
console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));
```

### Predict

Write all six outputs.

### Then explain

Identify which comparisons invoke:

```text
coercion
```

and which do not.

---

# 7. Question 3 — `Object.is`

```js
console.log(+0 === -0);
console.log(Object.is(+0, -0));
console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));
```

### Questions

```text
Which lines differ?
Why?
```

Do not answer with:

```text
"Object.is is better."
```

Explain the semantic choice.

---

# 8. Question 4 — `var` Loop Closure

```js
var fns = [];

for (var i = 0; i < 3; i++) {
  fns.push(function () {
    return i;
  });
}

console.log(fns[0]());
console.log(fns[1]());
console.log(fns[2]());
```

### Predict

### Then explain

Why do the functions not retain:

```text
0
1
2
```

?

---

# 9. Question 5 — `let` Loop Closure

```js
const fns = [];

for (let i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns[0]());
console.log(fns[1]());
console.log(fns[2]());
```

### Predict

### Then compare with Question 4

Explain the relevant lexical-environment behavior.

Do not reduce the explanation to:

```text
"let fixes var."
```

---

# 10. Question 6 — TDZ

```js
console.log(a);

let a = 10;
```

### Predict

What happens?

Do not say only:

```text
"undefined"
```

Explain:

```text
binding existence
initialization
Temporal Dead Zone
```

---

# 11. Question 7 — Function Declaration vs Function Expression

```js
sayHello();

function sayHello() {
  console.log("hello");
}

sayWorld();

const sayWorld = function () {
  console.log("world");
};
```

### Predict

Does the program:

```text
print
throw
partially execute
```

?

At which point?

---

# 12. Question 8 — `this` in Method Calls

```js
const user = {
  name: "Milan",
  getName() {
    return this.name;
  }
};

console.log(user.getName());

const fn = user.getName;

console.log(fn());
```

### Predict

State the environment assumption for the second call if needed.

Then explain:

```text
method call reference
vs
detached function call
```

---

# 13. Question 9 — Arrow Function `this`

```js
const user = {
  name: "Milan",
  regular() {
    return this.name;
  },
  arrow: () => this.name
};

console.log(user.regular());
console.log(user.arrow());
```

### Predict

Then explain why:

```text
arrow function
```

does not obtain `this` in the same way as:

```text
regular method
```

---

# 14. Question 10 — `bind`

```js
const user = {
  name: "Milan"
};

function getName(prefix) {
  return `${prefix}: ${this.name}`;
}

const bound = getName.bind(user, "User");

console.log(bound());
console.log(getName.call(user, "Account"));
```

### Predict

Then explain:

```text
bind
call
arguments
this
```

---

# 15. Question 11 — Prototype Lookup

```js
const parent = {
  x: 10
};

const child = Object.create(parent);

console.log(child.x);

child.x = 20;

console.log(child.x);
console.log(parent.x);
```

### Predict all outputs.

Then explain:

```text
before own property
after own property
prototype lookup
shadowing
```

---

# 16. Question 12 — `in` vs Own Property

```js
const parent = {
  x: 10
};

const child = Object.create(parent);

child.y = 20;

console.log("x" in child);
console.log(Object.hasOwn(child, "x"));
console.log("y" in child);
console.log(Object.hasOwn(child, "y"));
```

### Predict

Then explain:

```text
prototype property
own property
```

---

# 17. Question 13 — Property Enumerability

```js
const obj = {
  a: 1,
  b: 2
};

Object.defineProperty(obj, "c", {
  value: 3,
  enumerable: false
});

console.log(Object.keys(obj));
console.log(Object.getOwnPropertyNames(obj));
```

### Predict

Then explain why the same property appears in one result but not the other.

---

# 18. Question 14 — Symbols

```js
const key = Symbol("id");

const obj = {
  [key]: 123,
  name: "test"
};

console.log(Object.keys(obj));
console.log(Object.getOwnPropertySymbols(obj));
console.log(obj[key]);
```

### Predict

Then explain why:

```text
Object.keys()
```

does not include the Symbol key.

---

# 19. Question 15 — Iterators

```js
const iterable = {
  values: [10, 20],
  [Symbol.iterator]() {
    let index = 0;

    return {
      next: () => {
        if (index < this.values.length) {
          return {
            value: this.values[index++],
            done: false
          };
        }

        return {
          value: undefined,
          done: true
        };
      }
    };
  }
};

for (const value of iterable) {
  console.log(value);
}
```

### Predict

Then explain the protocol:

```text
iterable
→ Symbol.iterator
→ iterator
→ next()
```

---

# 20. Question 16 — Generator

```js
function* demo() {
  console.log("A");
  yield 1;
  console.log("B");
  yield 2;
  console.log("C");
}

const g = demo();

console.log("D");
console.log(g.next());
console.log("E");
console.log(g.next());
console.log("F");
console.log(g.next());
```

### Predict

Write the exact line-by-line output.

This is an ordering question.

---

# 21. Question 17 — Promise Reaction Ordering

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

### Predict

Then explain:

```text
current synchronous execution
→ Promise reaction job
```

Do not simply say:

```text
"Promises are asynchronous."
```

---

# 22. Question 18 — Multiple Promise Reactions

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");

  Promise.resolve().then(() => {
    console.log("C");
  });

  console.log("D");
});

Promise.resolve().then(() => {
  console.log("E");
});

console.log("F");
```

### Predict the exact order.

This question tests whether you understand that a Promise reaction scheduled during another reaction does not magically execute before already-queued work.

---

# 23. Question 19 — `queueMicrotask`

```js
console.log("A");

queueMicrotask(() => {
  console.log("B");
});

Promise.resolve().then(() => {
  console.log("C");
});

console.log("D");
```

### Predict

Then explain how:

```text
queueMicrotask
```

and:

```text
Promise reactions
```

participate in the microtask/job processing model of the host/runtime.

Do not claim that all host runtimes have identical event-loop implementation details.

---

# 24. Question 20 — `async` Function

```js
async function f() {
  console.log("A");
  return 42;
}

console.log("B");

const p = f();

console.log("C");

p.then(value => {
  console.log(value);
});

console.log("D");
```

### Predict exact output.

Then explain:

```text
function call
Promise creation
synchronous body execution
fulfillment reaction
```

---

# 25. Question 21 — `await`

```js
async function f() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

f();

console.log("D");
```

### Predict.

Then explain precisely what:

```text
await
```

does to the continuation of the async function.

---

# 26. Question 22 — `await` and Additional Promise Work

```js
async function f() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  await Promise.resolve();

  console.log("C");
}

console.log("D");

f();

Promise.resolve().then(() => {
  console.log("E");
});

console.log("F");
```

### Predict

This is intentionally harder.

Before checking the answer, create a queue timeline:

```text
synchronous phase
queue after f() pauses
queue after first continuation
queue after second await
```

---

# 27. Question 23 — `Promise.all`

```js
const a = Promise.resolve("A");
const b = Promise.resolve("B");

Promise.all([a, b]).then(values => {
  console.log(values);
});

console.log("C");
```

### Predict.

Then explain:

```text
why Promise.all does not synchronously print the array
```

and:

```text
why the resulting array preserves input order
```

even though individual Promise settlement timing can differ.

---

# 28. Question 24 — Exceptions in Async Code

```js
async function f() {
  throw new Error("boom");
}

console.log("A");

const p = f();

console.log("B");

p.catch(error => {
  console.log(error.message);
});

console.log("C");
```

### Predict.

Then explain the difference between:

```text
throw inside async function
```

and:

```text
throw inside synchronous function
```

from the caller's perspective.

---

# 29. Question 25 — Principal-Level Mixed Ordering

```js
console.log("1");

setTimeout(() => {
  console.log("2");

  Promise.resolve().then(() => {
    console.log("3");
  });
}, 0);

Promise.resolve().then(() => {
  console.log("4");
});

queueMicrotask(() => {
  console.log("5");
});

(async function () {
  console.log("6");

  await null;

  console.log("7");
})();

console.log("8");
```

### Predict

Write the exact output order.

Then explain the execution in phases:

```text
synchronous execution
→ microtasks/jobs
→ timer callback/task
→ microtasks/jobs created by timer
```

Do not treat:

```text
setTimeout(..., 0)
```

as:

```text
"runs immediately after current code"
```

---

# 30. Output Prediction Trace Sheet

Use this for every question.

```md
## Question #

### Environment
- ECMAScript / Browser / Node:
- Module / Script:
- Strict mode:
- Other assumptions:

### Prediction
```text
...
```

### Actual
```text
...
```

### Match
- [ ] Exact
- [ ] Partially correct
- [ ] Incorrect

### Trace
1.
2.
3.
4.

### Rule
-

### Misconception
-

### Confidence
1 / 2 / 3 / 4 / 5

### Related Concepts
-

### Production Relevance
-
```

---

# 31. Behavior Classification

After each problem classify the source of the result:

```text
[ ] pure ECMAScript language semantics
[ ] host scheduling
[ ] browser behavior
[ ] Node.js behavior
[ ] module system
[ ] environment-sensitive behavior
```

This prevents overgeneralization.

---

# 32. Prediction Failure Taxonomy

When your prediction is wrong, classify the failure:

```text
A — syntax misunderstanding
B — coercion misunderstanding
C — scope misunderstanding
D — closure misunderstanding
E — this misunderstanding
F — prototype misunderstanding
G — descriptor/property misunderstanding
H — iterator/generator misunderstanding
I — Promise/job misunderstanding
J — event-loop misunderstanding
K — host/runtime assumption
L — module/runtime assumption
M — insufficient execution tracing
```

---

# 33. The "Why Was I Wrong?" Rule

Never write:

```text
"I forgot."
```

Instead write:

```text
"I assumed X, but the actual rule is Y."
```

Example:

```text
I assumed await blocks the function synchronously.

Actual:
await suspends the async function continuation and schedules continuation work according to Promise/job semantics.
```

---

# 34. Trace Discipline

For difficult ordering questions, create:

```text
Synchronous Queue
Microtask / Job Queue
Task / Timer Queue
```

Example:

```text
SYNC:
A
B

MICROTASK/JOBS:
C
D

TASK:
timer callback

MICROTASK/JOBS AFTER TASK:
E
```

Do not use:

```text
"the event loop does magic"
```

as an explanation.

---

# 35. Exact Output vs Environment-Specific Output

Some snippets cannot be answered honestly without specifying:

```text
browser
Node.js
script
module
strictness
```

When that happens:

```text
state assumptions
```

before predicting.

---

# 36. Important Assessment Rule

A technically correct answer can still be incomplete if it:

```text
predicts output
```

but cannot explain:

```text
why
```

The assessment measures:

```text
mental execution
```

not:

```text
pattern recognition
```

---

# 37. Interview Translation

After solving each question, answer:

```text
How would I explain this in an interview in 30 seconds?
```

Then:

```text
How would I explain it to a senior engineer?
```

Then:

```text
What edge case would I mention?
```

---

# 38. Implementation Translation

For every async question ask:

```text
How could this behavior cause a production bug?
```

Examples:

```text
ordering bug
race
stale state
unhandled rejection
incorrect timeout
resource leak
```

---

# 39. Mastery Standard

You have strong prediction ability when you can:

```text
read
→ simulate
→ predict
→ explain
→ verify
→ correct
```

without repeatedly relying on:

```text
trial-and-error execution
```

---

# 40. Scoring Rubric

Per question:

## 4 points

```text
exact output
correct ordering
correct mechanism
clear explanation
```

## 3 points

```text
correct result
minor explanation gap
```

## 2 points

```text
partially correct
major reasoning gap
```

## 1 point

```text
guessing
major misconception
```

## 0 points

```text
wrong
or
no meaningful attempt
```

---

# 41. Critical Topics

Treat repeated misses in these areas as mandatory revision:

```text
coercion
scope
closures
this
prototypes
property descriptors
iterators
generators
Promise reactions
await
microtasks/jobs
host event loop
```

---

# 42. Anti-Memorization Rule

Do not memorize:

```text
"output = B A"
```

Memorize the reasoning:

```text
sync work completes
→ queued reaction remains pending
→ host/runtime reaches job processing
→ reaction executes
```

This transfers to unfamiliar examples.

---

# 43. Second-Order Prediction

After predicting output, predict:

```text
what happens if one line changes?
```

Examples:

```text
Promise.resolve()
→ setTimeout

var
→ let

regular function
→ arrow function

own property
→ prototype property
```

This tests whether your understanding is structural.

---

# 44. Counterfactual Exercise

For Questions 1–25, choose five where you were wrong.

For each:

```text
Original:
Wrong assumption:
Minimal change:
New output:
Why:
```

---

# 45. Oral Retrieval

Answer all questions without writing code.

For each snippet state:

```text
first output
second output
third output
```

then explain the transitions.

---

# 46. Whiteboard Retrieval

For Questions:

```text
18
22
25
```

draw:

```text
execution stack
Promise jobs
microtasks
timer/task
continuations
```

before speaking the answer.

---

# 47. Production Translation

Map the questions to real bugs:

```text
Q1–3 → input/coercion surprises
Q4–6 → scope/closure bugs
Q8–10 → callback/method binding bugs
Q11–14 → authorization/property bugs
Q15–16 → iteration bugs
Q17–25 → async ordering/race bugs
```

---

# 48. Advanced Challenge

Take Question 25 and rewrite it using:

```text
setImmediate
process.nextTick
MessageChannel
queueMicrotask
setTimeout
```

only when your environment supports them.

Then explicitly label:

```text
ECMAScript semantics
Node-specific semantics
browser-specific semantics
```

Do not transfer a Node-specific ordering result into a browser claim.

---

# 49. Debugging Exercise

Take one failed prediction.

Implement a minimal test that:

```text
prints
timestamps
event labels
queue stages
```

Then compare:

```text
mental model
actual runtime
```

---

# 50. Retrieval Record

```md
# Chapter 113 — Output Prediction — Retrieval Record

## Attempt
- Date:
- Duration:
- Score:
- Percentage:
- Status before:
- Status after:

## Question Scores
- Q1:
- Q2:
- Q3:
- Q4:
- Q5:
- Q6:
- Q7:
- Q8:
- Q9:
- Q10:
- Q11:
- Q12:
- Q13:
- Q14:
- Q15:
- Q16:
- Q17:
- Q18:
- Q19:
- Q20:
- Q21:
- Q22:
- Q23:
- Q24:
- Q25:

## Most Difficult
-

## Misconceptions
-

## Environment Confusions
-

## Async Ordering Gaps
-

## Scope / Object Gaps
-

## Questions Requiring Rebuild
-

## Questions Requiring Runtime Verification
-

## Next Review
-
```

---

# 51. Dependency Scorecard

```text
Language semantics
[ ] strong
[ ] medium
[ ] weak

Scope / closures
[ ] strong
[ ] medium
[ ] weak

Objects / prototypes
[ ] strong
[ ] medium
[ ] weak

Iteration
[ ] strong
[ ] medium
[ ] weak

Promises / async
[ ] strong
[ ] medium
[ ] weak

Event loop
[ ] strong
[ ] medium
[ ] weak

Runtime distinction
[ ] strong
[ ] medium
[ ] weak
```

---

# 52. Dependency Graph

```text
Chapters 01–30
        ↓
language + objects + errors
        ↓
Chapters 31–44
        ↓
async + jobs + host/runtime semantics
        ↓
Chapters 45–70
        ↓
memory + engine + browser + Node
        ↓
Chapters 71–101
        ↓
algorithms + production judgment
        ↓
Chapters 102–111
        ↓
projects
        ↓
Chapter 112 — Conceptual Assessment
        ↓
Chapter 113 — Output Prediction
        ↓
Chapter 114 — Debugging Assessment
        ↓
Chapter 115 — Async/Event Loop Assessment
```

---

# 53. Concept Connections

## Re-tests

```text
coercion
scope
TDZ
closures
this
prototypes
property semantics
Symbols
iterators
generators
Promises
async/await
jobs
microtasks
timers
browser/Node runtime differences
```

## Builds Toward

```text
debugging
async reasoning
performance diagnosis
race detection
production incident analysis
```

## Why This Chapter Matters

Output prediction is a compact form of execution simulation.

If you can accurately predict:

```text
what runs
when it runs
what state it sees
```

you have a strong foundation for:

```text
debugging
concurrency reasoning
race analysis
performance analysis
```

---

# 54. Spaced Retrieval Schedule

### Day 0

Attempt all 25.

### Day 1

Redo all incorrect questions.

### Day 3

Redo:

```text
Q4–10
```

for:

```text
scope
closures
this
```

and:

```text
Q17–25
```

for:

```text
async ordering
```

### Day 7

Solve all 25 orally.

### Day 14

Solve:

```text
Q18
Q22
Q25
```

from memory with a queue diagram.

### Day 30

Create five new output-prediction problems yourself.

---

# 55. Completion Criteria

Mark:

```text
[+] Completed
```

only when:

```text
[ ] all 25 attempted
[ ] all outputs recorded
[ ] all errors recorded
[ ] all traces written
[ ] wrong assumptions documented
[ ] environment assumptions documented
[ ] critical async gaps remediated
```

Mark:

```text
[*] Mastered
```

when you can:

```text
[ ] predict unfamiliar snippets
[ ] explain every output
[ ] distinguish language vs host behavior
[ ] trace async ordering
[ ] recognize race conditions
[ ] defend the reasoning without executing code
```

---

# 56. Final Mental Model

Output prediction is:

```text
program text
     ↓
parse / semantics
     ↓
bindings + values + objects
     ↓
execution
     ↓
state changes
     ↓
scheduled work
     ↓
job/microtask/task processing
     ↓
observable result
```

For synchronous code:

```text
read
→ execute
→ mutate state
→ continue
```

For asynchronous code:

```text
read
→ execute until suspension/scheduling
→ register continuation
→ return to current execution
→ finish current work
→ process queued work according to runtime/host rules
→ resume continuation
```

The deepest skill is:

```text
never guess the output from familiarity
```

Instead:

```text
derive it from the rules
```

> **Mastery reminder:** If you can predict unfamiliar JavaScript correctly and explain every step, you are no longer memorizing behavior—you are executing the language model in your head.