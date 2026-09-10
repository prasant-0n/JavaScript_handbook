

# Chapter 26 — Generators / Async Generators

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** Generator functions, suspended execution, generator objects, `yield`, `yield*`, `next`, `return`, `throw`, generator control flow, lazy computation, and asynchronous generators.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what a generator function is.
2. Distinguish:
   - normal function,
   - generator function,
   - generator object.
3. Explain why a generator function does not execute its body immediately when called.
4. Explain what calling a generator function returns.
5. Explain the role of:
   ```js
   function* fn() {}
   ```
6. Explain the role of:
   ```js
   yield
   ```
7. Explain generator suspension and resumption.
8. Explain why a generator can preserve local variables across `next()` calls.
9. Explain the relationship between generators and the iterator protocol.
10. Explain why generator objects are both:
    - iterators,
    - iterables.
11. Explain:
    ```js
    generator.next()
    ```
12. Explain how values passed into:
    ```js
    next(value)
    ```
    become the result of the suspended `yield` expression.
13. Explain why:
    ```js
    gen.next(value)
    ```
    does not inject the value into the first `yield` before the generator is started.
14. Explain:
    ```js
    generator.return(value)
    ```
15. Explain:
    ```js
    generator.throw(error)
    ```
16. Explain generator completion.
17. Explain generator return values.
18. Explain:
    ```js
    yield* iterable
    ```
19. Explain delegation semantics of `yield*`.
20. Explain how `yield*` interacts with:
    - `next`
    - `return`
    - `throw`
21. Explain how a generator can delegate to another generator.
22. Explain how:
    ```js
    return
    ```
    differs from:
    ```js
    yield
    ```
23. Explain generator cleanup and `finally`.
24. Explain generator closing.
25. Explain what happens when a consumer breaks out of a `for...of` loop over a generator.
26. Explain generator resource-management patterns.
27. Explain generator reentrancy errors.
28. Explain why a running generator cannot be recursively resumed through its own execution state.
29. Explain generator object state.
30. Explain lazy generation.
31. Compare generators with:
    - hand-written iterators,
    - eager Arrays,
    - callbacks,
    - Promises,
    - async iterators.
32. Explain generator composition.
33. Implement:
    - range generators,
    - lazy transforms,
    - recursive traversals,
    - delegation with `yield*`.
34. Implement a generator-based state machine.
35. Debug generator suspension/resumption bugs.
36. Explain async generator functions:
    ```js
    async function* fn() {}
    ```
37. Explain the relationship between async generators and:
    ```js
    Symbol.asyncIterator
    ```
38. Explain `for await...of`.
39. Explain how async generator `next()` differs from synchronous generator `next()`.
40. Explain why async generator results are Promise-based.
41. Explain async generator `return()` and `throw()`.
42. Explain async generator `yield`.
43. Explain awaiting yielded values.
44. Explain error propagation in async generators.
45. Explain async generator cleanup.
46. Compare:
    - Promise,
    - AsyncIterator,
    - AsyncGenerator,
    - Stream.
47. Explain backpressure conceptually in async iteration.
48. Explain lazy asynchronous production.
49. Implement an async generator over paginated data.
50. Implement a cancellable async generator.
51. Explain the performance and memory implications of generators.
52. Explain how generators can accidentally retain data through suspended execution.
53. Identify security risks in unbounded/lazy generator consumers.
54. Design production-safe generator APIs.
55. Reason about generator use at principal-engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 08 — Control Flow / Iteration
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 13 — Closures
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 22 — Arrays
- Chapter 24 — Map / Set / Weak Collections
- Chapter 25 — Iterables / Iterators

Strongly recommended before the async-generator section:

- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions

The async-generator material in this chapter introduces the concepts needed later and should not be mistaken for the complete async execution model.

---

## 3. What Is It?

A generator function is declared with:

```js
function* makeSequence() {
  yield 1;
  yield 2;
  yield 3;
}
```

Calling it:

```js
const generator = makeSequence();
```

does not immediately execute the generator body.

Instead, it creates a **generator object** that can be resumed through:

```js
generator.next()
```

The generator produces iterator results:

```js
generator.next();
// { value: 1, done: false }

generator.next();
// { value: 2, done: false }

generator.next();
// { value: 3, done: false }

generator.next();
// { value: undefined, done: true }
```

### Generator mental model

A generator is a resumable execution state machine:

```text
start
  ↓
run
  ↓
yield
  ↓
SUSPENDED
  ↓
next()
  ↓
resume
  ↓
yield
  ↓
SUSPENDED
  ↓
...
  ↓
return
  ↓
COMPLETED
```

This is the key difference from ordinary functions.

A normal function:

```text
call
 ↓
execute
 ↓
return
```

A generator:

```text
call
 ↓
create generator
 ↓
next()
 ↓
execute
 ↓
yield/suspend
 ↓
next()/throw()/return()
 ↓
resume/close
```

---

## 4. Why Does It Exist?

Generators provide a language-level mechanism for:

- lazy sequences,
- resumable computations,
- custom iterators,
- tree traversal,
- state machines,
- cooperative workflows,
- generator composition.

Before generators, developers had to manually maintain iterator state:

```js
let index = 0;

return {
  next() {
    // complex state management
  },
};
```

Generators let JavaScript express that state machine using ordinary control flow:

```js
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}
```

This improves:

- readability,
- composition,
- correctness,
- control-flow expression.

### Async generators extend the same model

An async generator allows:

```text
asynchronous production
+
lazy iteration
```

through:

```js
async function* stream() {
  yield await getNext();
}
```

and:

```js
for await (const value of stream()) {
}
```

---

## 5. Mental Model

### Synchronous generator

```text
generator function
      ↓
generator object
      ↓
next()
      ↓
execute/resume
      ↓
yield
      ↓
iterator result
      ↓
generator suspended
```

### `next(value)`

```text
previous yield
      ↑
      |
next(value)
      |
      ↓
yield expression evaluates to value
```

The first `next()` starts execution and does not provide a value to a preceding `yield`, because no `yield` is suspended yet.

### Async generator

```text
async generator function
        ↓
async generator object
        ↓
next()
        ↓
Promise for iterator result
        ↓
async execution
        ↓
yield
        ↓
Promise fulfillment/rejection
```

---

## 6. Core Rules

### Rule 1 — A generator function uses `function*`

```js
function* generator() {
}
```

---

### Rule 2 — Calling a generator function does not execute its body immediately

```js
function* test() {
  console.log("body");
}

const gen = test();

console.log("after");
```

Output:

```text
after
```

The body starts on the first resumption.

---

### Rule 3 — A generator call returns a generator object

```js
const gen = test();

typeof gen; // "object"
```

---

### Rule 4 — A generator object is an iterator

```js
gen.next()
```

---

### Rule 5 — A generator object is also iterable

```js
gen[Symbol.iterator]() === gen
```

for standard generator objects.

---

### Rule 6 — `yield` suspends generator execution

```js
function* test() {
  yield 1;
  yield 2;
}
```

Each `yield` pauses execution until the generator is resumed.

---

### Rule 7 — `next()` resumes the generator

```js
gen.next()
```

runs until the next yield or completion.

---

### Rule 8 — `next(value)` provides a value to the suspended `yield`

```js
function* test() {
  const value = yield "pause";
  yield value;
}
```

After the first:

```js
gen.next()
```

the generator is suspended at the `yield`.

Then:

```js
gen.next(42)
```

causes:

```js
const value = yield "pause";
```

to evaluate as:

```js
const value = 42;
```

---

### Rule 9 — The first `next()` argument is ignored for generator body input

```js
gen.next(100)
```

when the generator is not yet started does not assign `100` to a prior `yield`.

There is no suspended yield yet.

---

### Rule 10 — `return(value)` completes the generator

```js
gen.return(42)
```

produces completion with:

```js
{
  value: 42,
  done: true,
}
```

subject to `finally` and closing semantics.

---

### Rule 11 — `throw(error)` injects an exception at the suspended point

```js
gen.throw(new Error("x"))
```

can be caught by:

```js
try {
  yield;
} catch (error) {
}
```

inside the generator.

---

### Rule 12 — `yield` is not the same as `return`

`yield` suspends.

`return` completes.

---

### Rule 13 — `finally` runs during generator closing

```js
function* resource() {
  try {
    yield 1;
  } finally {
    cleanup();
  }
}
```

Closing the generator runs the `finally` path.

---

### Rule 14 — `for...of` can close a generator early

```js
for (const value of generator()) {
  break;
}
```

The iterator-closing protocol can invoke the generator's `return()` behavior.

---

### Rule 15 — `yield*` delegates iteration

```js
function* outer() {
  yield* inner();
}
```

The outer generator delegates to the inner iterable.

---

### Rule 16 — `yield*` can return a value

If the delegated iterator completes with:

```js
{ value: result, done: true }
```

the `yield*` expression can evaluate to that completion value.

---

### Rule 17 — Generator local state persists across yields

```js
function* counter() {
  let value = 0;

  while (true) {
    yield value++;
  }
}
```

The local variable remains part of the suspended generator state.

---

### Rule 18 — A generator cannot normally be resumed while already executing

Reentrant `next()` calls against a currently executing generator produce an error.

---

### Rule 19 — Generator exceptions can propagate through `next()`

If the generator throws without handling an error, the corresponding `next()` call throws.

---

### Rule 20 — After completion, the generator remains completed

Repeated:

```js
gen.next()
```

returns completed iterator results.

---

### Rule 21 — Async generator calls return AsyncGenerator objects

```js
async function* source() {}

const gen = source();
```

The result is an asynchronous generator object.

---

### Rule 22 — Async generator `next()` returns a Promise

```js
await gen.next()
```

produces an iterator result asynchronously.

---

### Rule 23 — `for await...of` consumes async iterables

```js
for await (const value of source()) {
}
```

---

### Rule 24 — Async generators can use `await`

```js
async function* source() {
  const value = await fetchValue();
  yield value;
}
```

---

### Rule 25 — Async generators can yield Promises without requiring manual Promise unwrapping by the consumer

The async generator protocol handles asynchronous iterator results according to its defined semantics.

---

## 7. Syntax

### Generator function

```js
function* sequence() {
  yield 1;
  yield 2;
}
```

### Generator object

```js
const gen = sequence();
```

### Resume

```js
gen.next();
gen.next(value);
```

### Close

```js
gen.return(value);
```

### Inject exception

```js
gen.throw(error);
```

### Delegation

```js
yield* iterable;
```

### Async generator

```js
async function* sequence() {
  yield 1;
  yield 2;
}
```

### Async consumption

```js
for await (const value of sequence()) {
}
```

### Async manual iteration

```js
const gen = sequence();

const result = await gen.next();
```

---

## 8. Basic Examples

### Example 1 — Basic generator

```js
function* numbers() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = numbers();

console.log(gen.next());
console.log(gen.next());
console.log(gen.next());
console.log(gen.next());
```

---

### Example 2 — Lazy body execution

```js
function* test() {
  console.log("started");
  yield 1;
}

const gen = test();

console.log("created");
console.log(gen.next());
```

Output:

```text
created
started
{ value: 1, done: false }
```

---

### Example 3 — `next(value)`

```js
function* conversation() {
  const answer = yield "What is 2 + 2?";
  yield `You answered ${answer}`;
}

const gen = conversation();

console.log(gen.next());
console.log(gen.next(4));
console.log(gen.next());
```

---

### Example 4 — Generator as iterable

```js
function* values() {
  yield "a";
  yield "b";
}

console.log([...values()]);
```

Result:

```text
["a", "b"]
```

---

### Example 5 — Infinite lazy sequence

```js
function* integers() {
  let value = 0;

  while (true) {
    yield value++;
  }
}
```

Consume only a bounded prefix.

---

## 9. Execution Walkthrough

Consider:

```js
function* demo() {
  console.log("A");

  const x = yield 10;

  console.log("B", x);

  yield x * 2;

  console.log("C");

  return 99;
}
```

### Step 1 — Call generator function

```js
const gen = demo();
```

The body has not run.

---

### Step 2 — First `next()`

```js
gen.next();
```

Execution begins.

Print:

```text
A
```

Then reaches:

```js
yield 10;
```

The generator suspends.

Result:

```js
{
  value: 10,
  done: false
}
```

---

### Step 3 — Second `next(7)`

```js
gen.next(7);
```

The suspended `yield` expression evaluates to:

```text
7
```

Therefore:

```js
x = 7
```

Then:

```text
B 7
```

prints.

Next:

```js
yield x * 2;
```

produces:

```js
14
```

and suspends again.

---

### Step 4 — Third `next()`

The second `yield` completes with `undefined` as its input.

The generator resumes:

```text
C
```

Then:

```js
return 99;
```

completes the generator.

Result:

```js
{
  value: 99,
  done: true
}
```

---

### Step 5 — Further `next()`

```js
gen.next();
```

returns:

```js
{
  value: undefined,
  done: true
}
```

The generator remains completed.

---

## 10. Internal Mechanics

### 10.1 Generator as resumable execution context

A generator must preserve enough execution state to resume later.

Conceptually:

```text
program counter
local bindings
control-flow state
generator status
```

When a generator suspends, that execution state remains associated with the generator object.

---

### 10.2 Generator execution states

A useful conceptual state machine:

```text
suspended-start
      ↓
executing
      ↓
suspended-yield
      ↓
executing
      ↓
completed
```

Additional abrupt/closing paths exist.

---

### 10.3 Suspended local variables

```js
function* counter() {
  let n = 0;

  yield n++;
  yield n++;
}
```

After first yield:

```text
n = 1
```

The state is preserved.

After resumption:

```text
yield 2
```

occurs.

---

### 10.4 `yield` expression value

The expression:

```js
const value = yield 10;
```

has two conceptual phases.

First suspension:

```text
produce 10
```

Later resumption:

```text
yield expression evaluates to next(value)
```

This is one of the most important generator concepts.

---

### 10.5 `throw()`

Suppose:

```js
function* test() {
  try {
    yield 1;
  } catch (error) {
    yield `caught: ${error.message}`;
  }
}
```

Calling:

```js
gen.next();
gen.throw(new Error("boom"));
```

throws the error at the suspended point.

The generator can handle it.

---

### 10.6 `return()`

```js
gen.return(42);
```

requests generator completion.

But `finally` blocks can intercept the closing process:

```js
function* test() {
  try {
    yield 1;
  } finally {
    yield 2;
  }
}
```

A return/close operation can therefore continue through finally logic.

This is why closing is not simply:

```text
set done=true
```

---

### 10.7 Generator `return()` and `finally`

Consider:

```js
function* test() {
  try {
    yield 1;
  } finally {
    console.log("cleanup");
  }
}
```

After:

```js
gen.next();
gen.return("done");
```

the `finally` executes before the generator is ultimately completed.

---

### 10.8 `yield*` delegation

```js
function* inner() {
  yield 1;
  yield 2;
}

function* outer() {
  yield* inner();
  yield 3;
}
```

The outer generator delegates the traversal to `inner`.

Result:

```text
1
2
3
```

---

### 10.9 `yield*` is more than a loop

It delegates protocol operations.

The outer generator can forward:

```text
next
throw
return
```

to the delegated iterator where the protocol supports them.

That makes `yield*` a control-flow composition mechanism.

---

### 10.10 `yield*` return value

```js
function* inner() {
  yield 1;
  return 42;
}

function* outer() {
  const result = yield* inner();

  yield result;
}
```

The `yield*` expression receives the delegated generator's completion value:

```text
42
```

---

### 10.11 Generator iterable identity

Generator objects implement:

```js
Symbol.iterator
```

to return themselves.

Therefore:

```js
const gen = numbers();

gen[Symbol.iterator]() === gen; // true
```

This makes generator objects directly consumable by iterable consumers.

---

### 10.12 Reentrancy

A generator cannot be resumed while already executing.

Conceptually:

```text
generator state = executing
        ↓
another next()
        ↓
invalid reentrant resume
```

This protects the integrity of the generator's execution state.

---

### 10.13 Generator exceptions and completion

A generator can end through:

```text
normal return
uncaught throw
return()
throw()
```

The resulting state becomes completed.

---

### 10.14 Generator object retention

A suspended generator may retain references to objects reachable from:

- local variables,
- closures,
- iterator state,
- captured environments.

Therefore leaving many generators suspended can retain significant memory.

---

## 11. ECMAScript / Specification Semantics

### 11.1 GeneratorFunction

A generator function has specialized function semantics.

Calling it creates a Generator object rather than immediately executing the generator body.

---

### 11.2 Generator object internal state

The specification models generator objects with internal state representing execution status and associated execution context.

A useful conceptual set of states includes:

```text
suspended-start
suspended-yield
executing
completed
```

---

### 11.3 GeneratorStart

Specification-level generator initialization includes the conceptual:

```text
GeneratorStart
```

operation.

The generator's execution state is initialized without immediately running to completion.

---

### 11.4 GeneratorResume

Calling:

```js
gen.next(value)
```

conceptually resumes the generator execution using its stored context.

---

### 11.5 GeneratorResumeAbrupt

Operations such as:

```js
gen.throw(error)
```

or closing behavior can resume the generator with an abrupt completion.

---

### 11.6 GeneratorYield

When:

```js
yield value
```

is reached, generator execution suspends and returns an iterator result.

The execution context is preserved for future resumption.

---

### 11.7 GeneratorResumeNext

Subsequent iterator operations continue the suspended generator.

---

### 11.8 `yield` and Completion Records

A generator's control-flow model integrates with ECMAScript Completion Records:

```text
normal
return
throw
```

This is why generator control operations compose with:

- `try`
- `catch`
- `finally`
- abrupt completion.

---

### 11.9 `YieldExpression`

The language grammar has a distinct `YieldExpression`.

Its semantics include:

```text
evaluate operand
suspend generator
produce iterator result
later receive completion value
```

---

### 11.10 `YieldDelegate`

`yield*` has its own detailed delegation semantics.

It:

1. obtains an iterator from the delegated value,
2. repeatedly performs iterator operations,
3. forwards inputs,
4. handles abrupt completions,
5. eventually evaluates to the delegated completion value.

---

### 11.11 `Generator.prototype.next`

Returns iterator results and advances generator execution.

---

### 11.12 `Generator.prototype.return`

Requests completion with the supplied value.

`finally` clauses can affect the observable result.

---

### 11.13 `Generator.prototype.throw`

Injects an exception into the suspended generator.

---

### 11.14 AsyncGenerator objects

Async generators use specialized asynchronous generator semantics.

Their iterator methods produce Promises for iterator results.

---

### 11.15 `AsyncGeneratorStart`

Async generator initialization creates an asynchronous generator execution state.

---

### 11.16 Async generator request queue

An async generator must coordinate asynchronous requests.

Conceptually:

```text
next/throw/return requests
           ↓
       request queue
           ↓
   async execution
           ↓
Promise resolution/rejection
```

This is important because multiple consumers can request operations while the generator is awaiting asynchronous work.

---

### 11.17 AsyncGeneratorYield

An async generator's `yield` integrates with asynchronous completion.

The consumer receives a Promise for an iterator result.

---

### 11.18 AsyncGeneratorResumeNext

Asynchronous generator requests are processed according to their ordering and completion state.

This is more complex than simply:

```js
async function* = generator + await
```

although that is a useful first mental model.

---

### 11.19 Async iterator protocol

Async iteration uses:

```js
Symbol.asyncIterator
```

and an iterator method whose `next()` produces a Promise for an iterator result.

---

### 11.20 `for await...of`

The asynchronous loop obtains an async iterator and awaits each iteration step as required by its semantics.

It also participates in iterator closing when the loop exits early.

---

## 12. Advanced Behavior

### 12.1 Generator return value versus yielded values

```js
function* test() {
  yield 1;
  return 2;
}
```

Consumption with spread:

```js
[...test()]
```

returns:

```text
[1]
```

The final return value is not yielded through ordinary iteration.

Manual:

```js
const gen = test();

gen.next(); // value 1
gen.next(); // value 2, done true
```

---

### 12.2 `for...of` does not expose generator return value

```js
for (const value of test()) {
  console.log(value);
}
```

prints only:

```text
1
```

because normal iteration stops when `done` becomes true.

---

### 12.3 Capturing generator return value manually

```js
const gen = test();

gen.next();
const result = gen.next();

console.log(result.value);
```

This reveals the completion value.

---

### 12.4 Passing values into generators

```js
function* calculator() {
  const a = yield "first";
  const b = yield "second";

  return a + b;
}
```

Drive carefully:

```js
const gen = calculator();

gen.next();
gen.next(10);
gen.next(20);
```

The first `next()` starts the generator.

Subsequent values resolve suspended yield expressions.

---

### 12.5 Sending `undefined`

```js
gen.next(undefined);
```

still resumes the suspended generator.

Do not confuse:

```text
no argument
```

with:

```text
argument explicitly equal to undefined
```

at the JavaScript call level unless the generator's logic distinguishes them.

---

### 12.6 Throwing into `yield`

```js
function* test() {
  try {
    yield 1;
  } catch {
    yield 2;
  }
}
```

Calling:

```js
gen.next();
gen.throw(new Error());
```

resumes by throwing at the suspended yield point.

---

### 12.7 Throwing when not suspended

A completed or inappropriate generator state can cause different behavior than a suspended generator.

Always track:

```text
current generator state
```

before reasoning about `throw()`.

---

### 12.8 Return through `finally`

```js
function* test() {
  try {
    yield 1;
  } finally {
    console.log("cleanup");
  }
}
```

Calling:

```js
gen.next();
gen.return(99);
```

runs cleanup before completion.

---

### 12.9 Yield from `finally`

```js
function* test() {
  try {
    yield 1;
  } finally {
    yield 2;
  }
}
```

Closing can itself become suspended because `finally` contains a yield.

This is advanced but demonstrates that generator control flow is a real execution model, not merely an iterator convenience.

---

### 12.10 Delegation to Array

```js
function* values() {
  yield* [1, 2, 3];
}
```

`yield*` works with any synchronous iterable, not only generators.

---

### 12.11 Delegation chain

```js
function* a() {
  yield 1;
}

function* b() {
  yield* a();
}

function* c() {
  yield* b();
}
```

This creates composable lazy traversal.

---

### 12.12 Recursive tree traversal

```js
function* walk(node) {
  yield node;

  for (const child of node.children) {
    yield* walk(child);
  }
}
```

This is one of the strongest practical generator patterns.

---

### 12.13 Async generator over paginated data

```js
async function* fetchPages(fetchPage) {
  let page = 1;

  while (true) {
    const result = await fetchPage(page);

    for (const item of result.items) {
      yield item;
    }

    if (!result.nextPage) {
      return;
    }

    page = result.nextPage;
  }
}
```

Consumption:

```js
for await (const item of fetchPages(fetchPage)) {
  process(item);
}
```

The consumer can stop without materializing all pages.

---

### 12.14 Async generator error propagation

```js
async function* source() {
  yield await getValue();
  throw new Error("failed");
}
```

A consumer using:

```js
for await (const value of source()) {
}
```

receives the asynchronous error through the loop's error path.

---

### 12.15 Async generator cleanup

```js
async function* source() {
  try {
    yield 1;
    yield 2;
  } finally {
    await cleanup();
  }
}
```

Early termination can invoke async iterator closing behavior.

This is essential for resource-backed async sequences.

---

### 12.16 Async generator concurrency

A single async generator should not be assumed to process independent `next()` calls as parallel work.

Requests are coordinated by the async generator's protocol.

For application-level concurrency, design it explicitly rather than trying to force concurrent consumers through one generator instance.

---

### 12.17 Backpressure concept

A consumer:

```js
for await (const item of source()) {
  await process(item);
}
```

naturally couples production and consumption because the next iteration is requested after processing completes.

This is a useful backpressure-like pattern.

It is not equivalent to every stream backpressure mechanism.

---

### 12.18 Async generator versus Promise

Promise:

```text
one eventual result
```

Async generator:

```text
zero or more eventual results over time
```

This distinction is fundamental.

---

## 13. Edge Cases

### Edge Case 1 — First `next(value)`

```js
function* test() {
  const x = yield 1;
  return x;
}

const gen = test();

console.log(gen.next(42));
console.log(gen.next(42));
```

The first `42` does not become `x`.

The second does.

---

### Edge Case 2 — Generator return value lost through spread

```js
function* test() {
  yield 1;
  return 2;
}

console.log([...test()]);
```

The result is:

```text
[1]
```

---

### Edge Case 3 — `yield*` captures return value

```js
function* inner() {
  return 42;
}

function* outer() {
  const value = yield* inner();

  yield value;
}
```

The outer generator can yield `42`.

---

### Edge Case 4 — Generator closes through break

```js
function* test() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("cleanup");
  }
}
```

Breaking from:

```js
for (const value of test()) {
  break;
}
```

can trigger cleanup.

---

### Edge Case 5 — Generator with `finally` that yields

A closing operation may suspend again if `finally` itself yields.

This can produce more than one observable step during closure.

---

### Edge Case 6 — Reentrancy

```js
function* test() {
  yield 1;
}
```

Calling `next()` recursively while the generator is executing is invalid.

---

### Edge Case 7 — Generator retained in memory

```js
const generators = [];

for (let i = 0; i < 100000; i++) {
  generators.push(largeGenerator());
}
```

Suspended generators can retain their captured execution state.

---

### Edge Case 8 — Infinite generator spread

```js
[...infiniteGenerator()]
```

does not terminate.

---

### Edge Case 9 — Async infinite generator

```js
async function* infinite() {
  while (true) {
    yield await nextValue();
  }
}
```

Use bounded consumption and cancellation.

---

### Edge Case 10 — Async cleanup rejection

If an async generator's cleanup path rejects, the consumer can observe the cleanup failure as part of iterator closing.

Resource cleanup code therefore requires explicit error policy.

---

### Edge Case 11 — `throw()` after completion

A completed generator does not return to its body.

Its subsequent control operations have completed-generator semantics rather than restarting execution.

---

### Edge Case 12 — `return()` before first `next()`

Calling:

```js
const gen = test();

gen.return(10);
```

can complete the generator without running the body.

Do not assume `finally` from never-entered execution always runs as if the body had started; reason from the actual generator state.

---

### Edge Case 13 — Async generator `next()` ordering

Multiple:

```js
gen.next()
gen.next()
gen.next()
```

calls can queue requests rather than executing synchronously in parallel.

---

### Edge Case 14 — Consumer breaks during async iteration

```js
for await (const value of source()) {
  break;
}
```

can trigger async iterator closing.

---

### Edge Case 15 — Async generator yielding promises

The async generator protocol incorporates promise handling.

Do not manually `await` every yielded value at the consumer unless the data contract requires additional transformation.

---

## 14. Common Misconceptions

### Misconception 1 — "Calling a generator executes it."

False.

Calling creates the generator object.

---

### Misconception 2 — "Generator is just syntax sugar for an Array."

False.

Generators are lazy and resumable.

---

### Misconception 3 — "yield returns from the generator."

False.

`yield` suspends; `return` completes.

---

### Misconception 4 — "The first next(value) sends value into yield."

False.

The generator has not yet reached a suspended yield.

---

### Misconception 5 — "Generator return values are yielded."

False.

They appear in the final iterator result but are not ordinary iteration values.

---

### Misconception 6 — "Spread captures generator return value."

False.

Spread consumes yielded values until `done`.

---

### Misconception 7 — "`yield*` is just a nicer for loop."

Incomplete.

It delegates iterator protocol operations and can propagate control methods and capture the delegated return value.

---

### Misconception 8 — "Generators are automatically memory efficient."

Not always.

A suspended generator can retain substantial state.

---

### Misconception 9 — "Async generator is just an async function returning an array."

False.

It lazily produces a sequence of asynchronous results.

---

### Misconception 10 — "Async generator makes processing parallel."

False.

It provides asynchronous sequencing, not automatic parallelism.

---

### Misconception 11 — "for await...of means everything runs concurrently."

False.

By default it sequentially awaits iteration results.

---

### Misconception 12 — "Breaking from a generator loop is just a local loop exit."

It can invoke iterator closing and generator cleanup.

---

### Misconception 13 — "Generators can be resumed recursively."

Normally no.

The generator execution state cannot be reentered while already executing.

---

## 15. Common Mistakes

### Mistake 1 — Using a generator when eager data is simpler

Generators add control-flow complexity.

Do not use them only because they are advanced.

---

### Mistake 2 — Materializing immediately

```js
[...generator()]
```

throws away the main memory/laziness advantage if all data is needed anyway.

---

### Mistake 3 — Unbounded consumption

Never blindly spread an infinite or attacker-controlled generator.

---

### Mistake 4 — Ignoring cleanup

Resource-backed generators should use:

```js
try/finally
```

and define closing behavior.

---

### Mistake 5 — Sharing one generator among multiple consumers

A generator object is a stateful iterator.

Independent consumers should usually get independent generator instances.

---

### Mistake 6 — Assuming async iteration means parallelism

Use explicit concurrency primitives when multiple tasks should overlap.

---

### Mistake 7 — Swallowing generator errors

Generator errors often represent source failures or invariant violations.

Preserve error context.

---

### Mistake 8 — Holding suspended generators indefinitely

This can retain closures and large object graphs.

---

### Mistake 9 — Creating deep `yield*` abstraction layers without observability

Lazy control flow can become hard to debug when many generators delegate to one another.

---

### Mistake 10 — Blocking work inside an async generator

Async syntax does not make CPU-heavy synchronous work non-blocking.

---

## 16. Comparison With Related Concepts

| Concept | Produces | Lazy | Resumable | Async | Typical purpose |
|---|---|---:|---:|---:|---|
| Array | Values | No | No | No | Materialized collection |
| Iterator | Iterator results | Usually | Yes | Optional protocol variant | Custom traversal |
| Generator | Iterator results | Yes | Yes | No | Lazy control flow |
| Async Iterator | Promise of iterator results | Yes | Yes | Yes | Async sequence |
| Generator + `yield*` | Delegated sequence | Yes | Yes | No | Composition |
| Promise | One eventual result | N/A | No | Yes | One async result |
| Stream | Data flow | Usually | Usually | Often | Continuous/buffered data |

### Generator versus hand-written iterator

Generator:

```text
less boilerplate
natural control flow
automatic state preservation
```

Hand-written iterator:

```text
more explicit state
more control
more boilerplate
```

---

### Generator versus callback

Callback:

```text
push values into consumer
```

Generator:

```text
consumer requests next value
```

This makes control direction different.

---

### Generator versus Promise

Generator:

```text
multiple lazily produced values
```

Promise:

```text
one eventual result
```

---

### Async generator versus stream

Async generator:

```text
language-level sequence protocol
```

Stream:

```text
data-flow abstraction with buffering/backpressure semantics
```

A stream can often be adapted into an async iterable.

---

## 17. Performance Considerations

### 17.1 Generator overhead

Generators maintain resumable execution state.

Compared with a simple loop, they can introduce:

- generator object allocation,
- state transitions,
- iterator result creation,
- protocol method calls.

Do not assume generator syntax is free.

---

### 17.2 When laziness helps

Generators can avoid materializing:

```js
million-item array
```

when the consumer only needs the first few values.

---

### 17.3 When laziness hurts

For tiny hot loops, generator/protocol overhead may exceed the value of abstraction.

Benchmark actual workloads.

---

### 17.4 Async generator overhead

Async generators add:

- Promise creation/management,
- async request coordination,
- suspension/resumption,
- iterator-result handling.

They are useful when the data source is naturally asynchronous.

---

### 17.5 `yield*`

Delegation adds protocol forwarding.

Usually this is valuable for clarity and composition.

For very hot paths, benchmark if it matters.

---

### 17.6 Lazy pipeline versus eager pipeline

Lazy:

```text
source → map → filter → take
```

can stop after the needed values.

Eager:

```text
source → Array → Array → Array → take
```

may allocate intermediate collections.

---

### 17.7 Async iteration and concurrency

Sequential:

```js
for await (const item of source) {
  await process(item);
}
```

limits concurrency to approximately one in-flight processing operation at a time.

If controlled parallelism is required, use an explicit concurrency limiter.

---

## 18. Memory Considerations

### Suspended execution retains state

A generator retains:

```text
local bindings
control state
closures/references
```

until completion or collection.

---

### Generator leaks

A common memory bug:

```js
const active = new Set();

function register(generator) {
  active.add(generator);
}
```

If completed generators are never removed, the Set strongly retains them and their reachable state.

---

### Async generators and in-flight promises

An async generator may retain:

- pending Promises,
- buffers,
- fetched pages,
- external resources.

Cancellation and cleanup must be deliberate.

---

### Infinite lazy sources

Infinite generation itself need not consume infinite memory.

The danger comes from:

- retaining generated values,
- materializing them,
- unbounded consumer queues.

---

### Lazy pipeline retention

A generator pipeline can keep the source alive longer than expected.

Analyze:

```text
source
 ↓
iterator
 ↓
transform closure
 ↓
consumer
```

as a reachability graph.

---

## 19. Security Considerations

### Untrusted generators

An attacker-controlled generator can:

- run forever,
- throw repeatedly,
- allocate unbounded data,
- perform expensive work.

Bound consumption and input size.

---

### Async infinite sources

A remote producer can effectively behave as:

```text
never-ending data source
```

Use:

- cancellation,
- timeouts,
- quotas,
- maximum item counts,
- rate limits.

---

### Resource leaks

Generators that wrap:

- files,
- database cursors,
- sockets,
- locks,

must define cleanup behavior.

---

### Exception-based control flow

`throw()` can cross generator boundaries.

Do not allow untrusted generator code to manipulate sensitive control flow without validation.

---

### Lazy code hides work

A function that returns an iterable may look cheap:

```js
const values = expensiveSource();
```

but the expensive work may happen later at iteration time.

This affects:

- authorization timing,
- transaction lifetime,
- logging,
- metrics,
- security checks.

Document when work actually occurs.

---

## 20. Production Usage

### Use case 1 — Large sequences

```js
function* chunks(array, size) {
  for (let i = 0; i < array.length; i += size) {
    yield array.slice(i, i + size);
  }
}
```

Useful when consumers only need incremental batches.

---

### Use case 2 — Tree traversal

```js
function* traverse(node) {
  yield node;

  for (const child of node.children) {
    yield* traverse(child);
  }
}
```

---

### Use case 3 — Lazy file/log processing

Conceptually:

```js
function* lines(source) {
  // produce one line at a time
}
```

For actual Node streams/files, use the appropriate async or stream APIs rather than buffering the entire file.

---

### Use case 4 — Pagination

```js
async function* users(fetchPage) {
  let cursor = undefined;

  while (true) {
    const page = await fetchPage(cursor);

    yield* page.items;

    if (!page.nextCursor) {
      return;
    }

    cursor = page.nextCursor;
  }
}
```

---

### Use case 5 — Async resource wrapper

```js
async function* records(client) {
  const resource = await client.open();

  try {
    while (true) {
      const record = await resource.next();

      if (record == null) {
        return;
      }

      yield record;
    }
  } finally {
    await resource.close();
  }
}
```

The exact resource API is domain-specific, but the lifecycle pattern is important.

---

### Use case 6 — Bounded lazy processing

```js
async function takeAsync(iterable, count) {
  const result = [];

  if (count <= 0) {
    return result;
  }

  for await (const value of iterable) {
    result.push(value);

    if (result.length >= count) {
      break;
    }
  }

  return result;
}
```

---

### Production rule

Before exposing a generator API, document:

```text
What is yielded?
When does work happen?
Is it reusable?
Is it stateful?
Can it be infinite?
What does return() do?
What does early break do?
What resources are held?
How are errors propagated?
How is cancellation handled?
Can consumers safely run concurrently?
```

---

## 21. Implementation From Scratch

### 21.1 Range generator

```js
function* range(start, end) {
  for (let value = start; value <= end; value++) {
    yield value;
  }
}
```

---

### 21.2 Lazy map generator

```js
function* mapGenerator(iterable, transform) {
  for (const value of iterable) {
    yield transform(value);
  }
}
```

---

### 21.3 Lazy filter generator

```js
function* filterGenerator(iterable, predicate) {
  for (const value of iterable) {
    if (predicate(value)) {
      yield value;
    }
  }
}
```

---

### 21.4 Lazy take

```js
function* takeGenerator(iterable, count) {
  if (count <= 0) {
    return;
  }

  let consumed = 0;

  for (const value of iterable) {
    yield value;

    consumed++;

    if (consumed >= count) {
      return;
    }
  }
}
```

---

### 21.5 Tree traversal

```js
function* depthFirst(root) {
  yield root;

  for (const child of root.children ?? []) {
    yield* depthFirst(child);
  }
}
```

---

### 21.6 State machine generator

```js
function* machine() {
  let state = "idle";

  while (true) {
    const input = yield state;

    if (input === "start") {
      state = "running";
    }

    if (input === "stop") {
      state = "stopped";
      return state;
    }
  }
}
```

This is educational. Production state machines should make transitions and invalid states explicit.

---

### 21.7 Async pagination generator

```js
async function* paginate(fetchPage) {
  let cursor = undefined;

  while (true) {
    const page = await fetchPage(cursor);

    for (const item of page.items) {
      yield item;
    }

    if (!page.nextCursor) {
      return;
    }

    cursor = page.nextCursor;
  }
}
```

---

### 21.8 Async generator with cleanup

```js
async function* readResource(resource) {
  try {
    while (true) {
      const item = await resource.next();

      if (item === undefined) {
        return;
      }

      yield item;
    }
  } finally {
    await resource.close();
  }
}
```

---

### 21.9 Cancellable async generator

Use an explicit cancellation signal:

```js
async function* poll(signal, fetchValue) {
  while (!signal.aborted) {
    const value = await fetchValue(signal);

    if (signal.aborted) {
      return;
    }

    yield value;
  }
}
```

This pattern requires the underlying operation to actually honor cancellation.

---

### Implementation progression

**Guided**

- range,
- map,
- filter,
- take.

**Partially Guided**

- tree traversal,
- generator state machine,
- `yield*`.

**No Reference**

- build a lazy sequence library.

**Edge-Case Hardened**

Support:

- early return,
- throw,
- cleanup,
- infinite input,
- reentrancy,
- errors.

**Production-Grade**

Add:

- async iteration,
- cancellation,
- concurrency limits,
- resource lifecycle,
- observability,
- benchmarks,
- compatibility tests.

---

## 22. Debugging Exercises

### Exercise 1 — Lazy body

```js
function* test() {
  console.log("run");
  yield 1;
}

const gen = test();

console.log("created");
```

Explain why `"run"` has not printed.

---

### Exercise 2 — `next(value)`

```js
function* test() {
  const value = yield 1;
  yield value;
}

const gen = test();

console.log(gen.next(10));
console.log(gen.next(20));
```

Determine which value reaches the `yield`.

---

### Exercise 3 — Return value

```js
function* test() {
  yield 1;
  return 2;
}

console.log([...test()]);
```

Where did `2` go?

---

### Exercise 4 — Throw

```js
function* test() {
  try {
    yield 1;
  } catch (error) {
    yield error.message;
  }
}

const gen = test();

console.log(gen.next());
console.log(gen.throw(new Error("boom")));
```

Trace the control flow.

---

### Exercise 5 — Cleanup

```js
function* test() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("cleanup");
  }
}

for (const value of test()) {
  break;
}
```

Determine why cleanup happens.

---

### Exercise 6 — Reentrancy

Create a generator that attempts to call its own `next()` while executing.

Determine the failure mode.

---

### Exercise 7 — `yield*`

```js
function* inner() {
  yield 1;
  return 99;
}

function* outer() {
  const value = yield* inner();
  yield value;
}
```

Predict the sequence.

---

### Exercise 8 — Memory retention

Create a generator that captures a large object, suspend it, and inspect why retaining the generator can retain the object.

---

### Exercise 9 — Async cleanup

Create an async generator with:

```js
try/finally
```

and verify cleanup on early `break`.

---

### Exercise 10 — Infinite source

```js
async function* infinite() {
  let i = 0;

  while (true) {
    yield i++;
  }
}
```

Write a bounded async consumer.

---

## 23. Code Review Exercise

Review:

```js
async function* records(client) {
  let page = 1;

  while (true) {
    const result = await client.fetch(page);

    for (const record of result.records) {
      yield record;
    }

    if (!result.nextPage) {
      return;
    }

    page = result.nextPage;
  }
}
```

### Questions

1. Is this lazy?
2. When does network work happen?
3. Is the source reusable?
4. What happens if the consumer breaks early?
5. Is pagination state retained?
6. Does the client support cancellation?
7. What happens if `client.fetch()` throws?
8. What happens if `yield` is consumed slowly?
9. Can the producer buffer unbounded data?
10. Does the API need explicit concurrency control?
11. What observability should be added?
12. How would you test early closure?

Propose a production-grade version.

---

## 24. Interview Questions

### Fundamentals

1. What is a generator function?
2. What does calling a generator function return?
3. Does the generator body execute immediately?
4. What does `yield` do?
5. What does `next()` do?
6. Is a generator an iterator?
7. Is a generator iterable?
8. What is a generator's execution state?
9. Why are generators useful for lazy sequences?
10. Why is `yield` different from `return`?

### `next` / `return` / `throw`

11. What does `next(value)` do?
12. Why is the first `next(value)` input not assigned to a yield?
13. What does `generator.return(value)` do?
14. What does `generator.throw(error)` do?
15. How does `try/catch` interact with `throw()`?
16. How does `finally` interact with `return()`?
17. What happens after a generator completes?
18. Can a generator be restarted?

### Delegation

19. What does `yield*` do?
20. Is `yield*` just a loop?
21. How does `yield*` forward `next()`?
22. How does it interact with `throw()`?
23. How does it interact with `return()`?
24. How can `yield*` produce a return value?
25. Why is delegation useful for recursive traversal?

### Async generators

26. What is an async generator?
27. What does calling an async generator return?
28. What does async generator `next()` return?
29. What is `Symbol.asyncIterator`?
30. What is `for await...of`?
31. How does async generator `yield` interact with `await`?
32. How are async generator errors propagated?
33. How does async generator cleanup work?
34. Does async generator mean concurrent execution?
35. How does an async generator differ from a Promise?

### Performance/memory

36. When do generators reduce memory?
37. When can generators be slower?
38. How can suspended generators cause retention?
39. Why is spreading an infinite generator dangerous?
40. How does lazy evaluation change operational timing?

### Principal-level

41. When should a public API return a generator instead of an Array?
42. When should it return an Iterable instead of exposing a Generator object?
43. How should generator APIs document reuse semantics?
44. How should resource-backed generators handle cancellation?
45. How would you design an async generator over a paginated API?
46. How would you introduce bounded concurrency around an async generator?
47. How would you prevent attacker-controlled infinite generation?
48. How would you observe and debug lazy work?
49. When is a Stream a better abstraction than an AsyncGenerator?
50. How would you decide whether lazy evaluation is worth its complexity?
51. How would you test iterator closing and cleanup guarantees?
52. What happens if consumers share one generator instance?
53. How would you design a generator API that is safe under early termination?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
function* test() {
  console.log("start");
  yield 1;
}

const gen = test();

console.log("created");
console.log(gen.next());
```

---

### Exercise B

```js
function* test() {
  const value = yield 1;
  yield value;
}

const gen = test();

console.log(gen.next(10));
console.log(gen.next(20));
console.log(gen.next(30));
```

---

### Exercise C

```js
function* test() {
  yield 1;
  return 2;
}

const gen = test();

console.log([...gen]);
console.log(gen.next());
```

---

### Exercise D

```js
function* inner() {
  yield 1;
  return 42;
}

function* outer() {
  const result = yield* inner();
  yield result;
}

console.log([...outer()]);
```

---

### Exercise E

```js
function* test() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("cleanup");
  }
}

for (const value of test()) {
  console.log(value);
  break;
}
```

---

### Exercise F

```js
function* test() {
  try {
    yield 1;
  } catch (error) {
    yield error.message;
  }
}

const gen = test();

console.log(gen.next());
console.log(gen.throw(new Error("boom")));
```

---

### Exercise G

```js
function* counter() {
  let value = 0;

  while (value < 3) {
    yield value++;
  }
}

const gen = counter();

console.log(gen.next());
console.log(gen.next());
console.log(gen.next());
console.log(gen.next());
```

---

### Exercise H

```js
async function* values() {
  yield 1;
  yield 2;
}

(async () => {
  for await (const value of values()) {
    console.log(value);
  }
})();
```

Predict the sequence and explain why the consumer is asynchronous.

---

### Exercise I

```js
async function* test() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("cleanup");
  }
}

(async () => {
  for await (const value of test()) {
    break;
  }
})();
```

Explain cleanup.

---

### Exercise J

```js
function* test() {
  const a = yield "A";
  const b = yield "B";
  return a + b;
}

const gen = test();

console.log(gen.next());
console.log(gen.next(10));
console.log(gen.next(20));
```

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
generator function
generator object
yield
next
return
throw
```

### Level 2 — Explain

Explain why:

```js
function* fn() {}
fn();
```

does not execute the body.

### Level 3 — Predict

Predict generator states after sequences of:

```text
next
next(value)
throw
return
```

### Level 4 — Implement

Build:

```text
range
map
filter
take
tree traversal
```

with generators.

### Level 5 — Debug

Fix:

- incorrect `next(value)` assumptions,
- missed cleanup,
- shared generator state,
- infinite consumption,
- reentrancy.

### Level 6 — Compare

Compare:

```text
Array
Iterable
Iterator
Generator
AsyncGenerator
Stream
Promise
```

using:

- laziness,
- state,
- memory,
- reuse,
- async behavior,
- backpressure,
- cleanup.

### Level 7 — Apply

Build an async pagination generator.

### Level 8 — Defend

Decide whether a library API should expose:

```text
Generator
Iterable
Iterator
Array
AsyncIterable
AsyncGenerator
Stream
```

and justify the contract.

### Level 9 — Principal Judgment

Design a production async generator that supports:

```text
pagination
cancellation
bounded concurrency
early cleanup
timeouts
metrics
errors
retries
```

without turning the generator into an untestable control-flow monolith.

---

## 27. Key Takeaways

1. A generator function is declared with `function*`.
2. Calling a generator function creates a generator object without immediately executing its body.
3. Generator objects are iterators.
4. Generator objects are also iterable.
5. `yield` suspends execution.
6. `next()` resumes execution.
7. `next(value)` provides the value for the previously suspended `yield`.
8. The first `next(value)` does not provide input to a prior yield.
9. Generator locals survive suspension.
10. `return()` requests completion.
11. `throw()` injects an error at the suspended point.
12. `yield` and `return` are different control-flow operations.
13. `finally` participates in generator cleanup.
14. `for...of` can close a generator on early termination.
15. `yield*` delegates to another iterable.
16. `yield*` forwards protocol control and can capture a delegated return value.
17. Generator return values are not yielded through ordinary iteration.
18. Spread normally discards the final generator return value from its collected results.
19. A generator cannot be normally reentered while already executing.
20. Generators provide resumable execution state.
21. Suspended generators can retain substantial memory.
22. Generators are useful for lazy sequences and recursive traversal.
23. Async generators combine asynchronous production with lazy iteration.
24. Async generator `next()` produces a Promise for an iterator result.
25. Async iteration uses `Symbol.asyncIterator`.
26. `for await...of` consumes async iterables.
27. Async generators do not automatically create parallelism.
28. Async iteration can naturally coordinate production and consumption.
29. Async generator cleanup should use deliberate lifecycle handling.
30. Infinite generators require bounded consumers.
31. Untrusted generators require resource/time/item limits.
32. Stream and AsyncGenerator are related but not identical abstractions.
33. Principal-level generator design is about control flow, lifecycle, memory, and API contracts—not just syntax.

---

## 28. Concept Connections

### Depends On

- Chapter 08 — Control Flow / Iteration
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 13 — Closures
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 22 — Arrays
- Chapter 24 — Map / Set / Weak Collections
- Chapter 25 — Iterables / Iterators

### Builds Toward

- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async / Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 40 — Observables / Reactive
- Chapter 53 — Web Streams / Data Flow
- Chapter 60 — Node Streams
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 78 — Production JS Architecture
- Chapter 80 — Library Authoring
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 87 — Deterministic Async Testing

### Related Concepts

- execution suspension
- execution context preservation
- iterator protocol
- iterable protocol
- generator protocol
- `yield`
- `yield*`
- completion records
- `next`
- `return`
- `throw`
- `finally`
- async iterator protocol
- async generator request queue
- lazy evaluation
- backpressure
- cancellation
- streaming

### Concepts Revisited

**Chapter 08:**  
Generators turn iteration control flow into resumable language-level execution.

**Chapter 12:**  
Generators make execution contexts visibly persistent across multiple resumption points.

**Chapter 13:**  
Generator suspension can retain closure/environment state, making lifetime analysis important.

**Chapter 20:**  
Generators automatically participate in the iterable protocol through `Symbol.iterator`.

**Chapter 25:**  
The iterator protocol explains the surface behavior of `next`, `return`, and `throw`.

**Chapter 31+:**  
Async generators build on asynchronous execution and Promise semantics developed later.

### Why This Chapter Matters Later

Generators provide the conceptual bridge from:

```text
ordinary synchronous functions
```

to:

```text
resumable lazy computation
```

and then:

```text
asynchronous lazy computation
```

That bridge is essential for understanding:

- lazy pipelines,
- async iteration,
- streams,
- parsers,
- paginated APIs,
- event processing,
- tree traversal,
- resource-backed iteration,
- concurrency control.

---

## Track A — Core Theory

### Level 1 — Intuition

> A generator is a function whose execution can pause and later continue from the exact suspended point.

### Level 2 — Syntax

Know:

```js
function* fn() {}
yield
next()
return()
throw()
yield*
async function* fn() {}
for await...of
```

### Level 3 — Practical

Build:

- range,
- lazy map/filter,
- tree traversal,
- pagination.

### Level 4 — Edge Cases

Understand:

- first next argument,
- return values,
- throw,
- finally,
- early close,
- reentrancy,
- infinite sources.

### Level 5 — Runtime/Internal

Understand:

- preserved execution state,
- generator lifecycle,
- suspension/resumption,
- request queue for async generators,
- memory retention.

### Level 6 — Specification Semantics

Be comfortable with:

- GeneratorStart
- GeneratorResume
- GeneratorResumeAbrupt
- GeneratorYield
- YieldExpression
- `yield*`
- async-generator request processing
- iterator closing.

### Level 7 — Performance/Security

Reason about:

- generator overhead,
- lazy memory savings,
- retained suspended state,
- unbounded sources,
- attacker-controlled generation,
- resource cleanup.

### Level 8 — Production Engineering

Design:

- lazy APIs,
- async pagination,
- resource-backed generators,
- cancellation,
- bounded consumption,
- observability.

### Level 9 — Interview/Reasoning

Answer:

> Why is a generator more than an iterator implementation convenience?

### Level 10 — Principal Judgment

Evaluate:

> Should this API expose a generator, iterable, async iterable, Promise, or Stream?

Consider:

```text
laziness
state
reuse
memory
cleanup
backpressure
concurrency
consumer expectations
```

---

## Track B — Implementation

The implementation ladder is:

```text
1. Basic generator
        ↓
2. range
        ↓
3. lazy map/filter
        ↓
4. take
        ↓
5. yield*
        ↓
6. recursive traversal
        ↓
7. generator state machine
        ↓
8. async generator
        ↓
9. pagination
        ↓
10. cancellation + production lifecycle
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain why:

```js
gen.next(10)
```

on a fresh generator does not assign `10` to a `yield` expression.

### Drill 2

Explain:

```text
yield
vs
return
```

in terms of generator state.

### Drill 3

Explain why generator return values disappear from:

```js
[...gen]
```

### Drill 4

Explain why `yield*` is a protocol delegation mechanism rather than merely loop syntax.

### Drill 5

Explain how `finally` makes generator cleanup observable.

### Drill 6

Compare:

```text
Generator
Promise
AsyncGenerator
Stream
```

for a paginated API.

### Drill 7

Design a bounded async generator for untrusted remote data.

---

## 29. Completion Criteria

Mark Chapter 26 `[+] Completed` only when the learner can:

- [ ] Explain generator functions.
- [ ] Explain generator objects.
- [ ] Explain lazy body execution.
- [ ] Explain generator state.
- [ ] Explain `yield`.
- [ ] Explain `next()`.
- [ ] Explain `next(value)`.
- [ ] Explain first-next semantics.
- [ ] Explain `return()`.
- [ ] Explain `throw()`.
- [ ] Explain generator completion.
- [ ] Explain generator return values.
- [ ] Explain generator iterable behavior.
- [ ] Explain `yield*`.
- [ ] Explain delegated control operations.
- [ ] Explain `yield*` return values.
- [ ] Explain generator `finally`.
- [ ] Explain iterator closing.
- [ ] Explain generator reentrancy.
- [ ] Explain memory retention from suspended generators.
- [ ] Implement lazy generators.
- [ ] Implement recursive traversal.
- [ ] Implement a generator state machine.
- [ ] Explain async generators.
- [ ] Explain `Symbol.asyncIterator`.
- [ ] Explain `for await...of`.
- [ ] Explain async generator Promise results.
- [ ] Explain async generator error propagation.
- [ ] Explain async generator cleanup.
- [ ] Explain async generator request ordering.
- [ ] Implement async pagination.
- [ ] Implement cancellation-aware async generation.
- [ ] Compare generators with streams and Promises.
- [ ] Design bounded lazy consumption.
- [ ] Debug generator lifecycle bugs.
- [ ] Defend a production generator API.

### Mastery Gate

Mastery requires:

```text
Understand
   ↓
Explain
   ↓
Predict
   ↓
Implement
   ↓
Debug
   ↓
Apply
   ↓
Compare
   ↓
Defend
```

The final defense should answer:

> When is a generator or async generator the right abstraction for a production API, and how should laziness, state, cleanup, cancellation, memory, errors, concurrency, and consumer expectations shape the design?

---

## Chapter 26 Retrieval Set

### Retrieval 1

Why does:

```js
function* fn() {}
fn();
```

not execute the body?

### Retrieval 2

What does `yield` do to generator execution?

### Retrieval 3

What does the second:

```js
gen.next(value)
```

do differently from the first?

### Retrieval 4

What happens to:

```js
return 42;
```

inside a generator?

### Retrieval 5

Why does spread not include the generator return value?

### Retrieval 6

What is `yield*`?

### Retrieval 7

How does `throw()` enter a generator?

### Retrieval 8

Why does `finally` matter for generator cleanup?

### Retrieval 9

How can a generator retain memory while suspended?

### Retrieval 10

Why can an async generator model pagination better than returning one giant Array?

### Retrieval 11

Why does an async generator not automatically imply parallel processing?

### Retrieval 12

When is a Stream a better abstraction than an AsyncGenerator?

---

## Chapter 26 Final Mental Model

Remember:

```text
             generator function
                    |
                  call()
                    |
             generator object
                    |
                  next()
                    |
               execute
                    |
                 yield
                    |
              SUSPENDED
                    |
             next(value)
                    |
                 resume
                    |
                yield ...
                    |
               return/throw
                    |
               COMPLETED
```

Delegation:

```text
outer generator
      |
   yield*
      |
      ↓
inner iterable/iterator
      |
 next / throw / return
      |
      ↓
outer control flow
```

Async generator:

```text
async generator
      ↓
async iterator
      ↓
next()
      ↓
Promise<IteratorResult>
      ↓
await asynchronous work
      ↓
yield
      ↓
consumer
```

Production lifecycle:

```text
lazy source
   ↓
bounded consumer
   ↓
processing
   ↓
early exit?
   ↓
iterator close
   ↓
cleanup
   ↓
release resources
```

Finally:

> A generator is a resumable execution machine that happens to implement the iterator protocol. An async generator extends that machine into asynchronous production. The real engineering power comes from controlling when work happens, how much state is retained, how consumers advance it, and how execution is closed.

