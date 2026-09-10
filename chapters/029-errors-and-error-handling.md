
# Chapter 29 — Errors and Error Handling

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what an error is in JavaScript as a runtime-level event and as a program-level value.
- Distinguish syntax errors, runtime exceptions, promise rejections, logical failures, and environmental failures.
- Explain the role of `Error` objects, `name`, `message`, `stack`, `cause`, and custom error types.
- Explain `throw`, `try`, `catch`, and `finally` precisely.
- Predict control flow when exceptions are thrown, caught, rethrown, transformed, or suppressed.
- Distinguish synchronous exceptions from asynchronous failures and rejected promises.
- Explain why `try/catch` around asynchronous work does not automatically catch later promise rejections or timer failures.
- Design error taxonomies suitable for application, library, and infrastructure code.
- Use `Error` subclassing, `cause`, error codes, and structured metadata without creating brittle error contracts.
- Understand the distinction between recovering from an error, translating an error, logging an error, and terminating an operation.
- Explain how `finally` interacts with `return`, `throw`, and abrupt completion.
- Build robust boundaries around external systems such as files, HTTP services, databases, queues, and user input.
- Recognize anti-patterns such as swallowing exceptions, logging and rethrowing blindly, throwing strings, overusing custom classes, and exposing internal details to clients.
- Debug exception paths using stack traces, causal chains, source maps, and deliberate reproduction.
- Explain the difference between expected operational failures and programmer defects.
- Implement a small production-oriented error framework with classification, causes, serialization, and boundary handling.
- Reason about error behavior at principal-engineer level: correctness, reliability, observability, security, compatibility, and operational cost.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

Reading the chapter is not sufficient to claim mastery.

---

## 2. Prerequisites

The learner should already understand:

- JavaScript values and types.
- Functions and lexical scope.
- Execution contexts and call flow.
- Objects and prototypes.
- Classes and constructors.
- Iteration and control flow.
- Promises at a foundational level.
- Async/await basics.
- Modules and imports/exports at a foundational level.
- Basic browser and Node.js runtime concepts.
- Basic debugging with stack traces.

Conceptual dependencies from earlier chapters include:

- Chapter 02 — Values, Types, Type System
- Chapter 06 — Operators, Expressions
- Chapter 08 — Control Flow, Iteration
- Chapter 09 — Functions / First-Class Behavior
- Chapter 10 — Scope / Lexical Environments / Identifier Resolution
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 28 — JSON / Serialization / Structured Clone

Future chapters build directly on this one, especially asynchronous control flow, promises, cancellation, production reliability, observability, testing, security, and system design.

---

## 3. What Is It?

JavaScript error handling is the set of language and application mechanisms used to represent, propagate, classify, observe, recover from, transform, or terminate in response to failures.

At the language level, an error is not a mystical runtime state. JavaScript programs manipulate values, and errors are commonly represented by `Error` objects or other values that can be thrown.

The language provides explicit exception-control mechanisms:

```js
throw value;
```

and:

```js
try {
  // protected operation
} catch (error) {
  // handler
} finally {
  // cleanup
}
```

The important distinction is:

> An exception is a control-flow mechanism; an `Error` object is a value commonly used to describe the failure.

JavaScript does not require a thrown value to be an `Error` instance.

This is legal:

```js
throw "failure";
```

This is also legal:

```js
throw 404;
```

And this is legal:

```js
throw { code: "INVALID_STATE" };
```

But production-quality code normally throws `Error` objects because they provide standardized conventions such as `name`, `message`, and a useful stack representation.

Error handling therefore has multiple layers:

```text
Failure occurs
     ↓
A value represents the failure
     ↓
Control flow becomes abrupt
     ↓
The value propagates outward
     ↓
A boundary catches / translates / recovers / terminates
     ↓
The system records or communicates the outcome
```

A good error architecture must answer five questions:

1. What failed?
2. Where did it fail?
3. Why did it fail?
4. Who can recover from it?
5. What should happen next?

---

## 4. Why Does It Exist?

Without explicit failure propagation, every function would need to encode failure manually in every return value.

For example:

```js
function parseAge(input) {
  if (typeof input !== "string") {
    return ???;
  }

  // ...
}
```

The problem becomes worse when failures travel through multiple layers:

```text
HTTP handler
    ↓
service
    ↓
repository
    ↓
database driver
```

A failure at the database layer may need to reach the service layer and eventually become an HTTP response.

Exceptions provide a built-in non-local control-flow mechanism:

```text
deep operation
    ↓ throws
intermediate function
    ↓ no catch
higher-level boundary
    ↓ catches
translate / recover / report
```

This separates:

- the location where a failure is detected,
- from the location where the application knows what to do about it.

That separation is one of the main reasons exceptions exist.

However, exceptions are not automatically “good.” They have costs:

- control flow becomes less explicit;
- cleanup must be designed carefully;
- broad catches can hide defects;
- asynchronous boundaries change propagation behavior;
- error contracts become part of API design;
- stack traces and metadata can be expensive;
- serialization can accidentally leak internal information.

The engineering question is therefore not:

> “Should we use errors?”

Every serious JavaScript system already has failures.

The useful question is:

> “Where should failures be represented, propagated, transformed, observed, and recovered?”

---

## 5. Mental Model

Use this model:

```text
                FAILURE
                   │
                   ▼
             Represent value
                   │
                   ▼
                 throw
                   │
                   ▼
        ┌─────────────────────┐
        │ abrupt completion   │
        └─────────────────────┘
                   │
             propagate outward
                   │
        ┌──────────┴───────────┐
        │                      │
      catch                  no catch
        │                      │
        ▼                      ▼
 handle / rethrow         caller boundary
        │                      │
        └──────────┬───────────┘
                   ▼
             finally runs
                   │
                   ▼
       normal completion OR
       another abrupt completion
```

The critical concept is **abrupt completion**.

A statement can complete normally, or it can complete abruptly because of:

- `throw`;
- `return`;
- `break`;
- `continue`.

Errors specifically use a throw completion.

You should think of:

```js
throw error;
```

as:

> “Stop the current normal execution path and propagate this abrupt completion until an appropriate handler is found.”

The handler is not necessarily the immediate caller.

Example:

```js
function c() {
  throw new Error("boom");
}

function b() {
  c();
}

function a() {
  b();
}

try {
  a();
} catch (error) {
  console.log(error.message);
}
```

The throw begins in `c`, passes through `b`, passes through `a`, and is handled outside `a`.

The key mental model:

> Exceptions search outward through execution structure until a matching catch boundary is found.

---

## 6. Core Rules

### Rule 1 — Any value can be thrown

```js
throw 123;
throw "bad";
throw { code: "BAD" };
throw new Error("bad");
```

Prefer `Error` objects in application and library code.

### Rule 2 — A throw changes control flow immediately

Code after a synchronous throw in the same control path does not execute.

```js
throw new Error("stop");

console.log("unreachable");
```

### Rule 3 — `catch` receives the exact thrown value

```js
const value = { reason: "bad-input" };

try {
  throw value;
} catch (error) {
  console.log(error === value); // true
}
```

### Rule 4 — The nearest applicable `catch` handles the exception

```js
try {
  try {
    throw new Error("inner");
  } catch (error) {
    throw new Error("outer");
  }
} catch (error) {
  console.log(error.message); // outer
}
```

The first handler transformed the failure by throwing again.

### Rule 5 — `finally` runs when control leaves the protected region

This includes:

- normal completion;
- exception;
- `return`;
- `break`;
- `continue`.

### Rule 6 — `finally` can replace the original outcome

This is dangerous:

```js
function example() {
  try {
    throw new Error("original");
  } finally {
    return "replacement";
  }
}

console.log(example()); // "replacement"
```

The `return` in `finally` suppresses the original throw.

Similarly:

```js
function example() {
  try {
    return "value";
  } finally {
    throw new Error("cleanup-failed");
  }
}
```

The final result is the throw from `finally`, not the original return.

### Rule 7 — Catch only where you can make a meaningful decision

Bad:

```js
try {
  doImportantWork();
} catch {
  // ignore
}
```

This can convert a visible failure into silent corruption.

### Rule 8 — Rethrowing preserves the current error identity

```js
catch (error) {
  throw error;
}
```

This is different from:

```js
catch (error) {
  throw new Error("something failed");
}
```

The second creates a new failure and may hide useful context unless the original is preserved with `cause`.

### Rule 9 — Error messages are for humans, not stable machine contracts

Prefer:

```js
error.code === "USER_NOT_FOUND"
```

over brittle parsing:

```js
error.message.includes("not found")
```

### Rule 10 — Promise rejection is asynchronous control flow

This does not catch a later timer exception:

```js
try {
  setTimeout(() => {
    throw new Error("later");
  }, 0);
} catch (error) {
  // does not run
}
```

The timer callback executes in a later task after the original `try` block has already completed.

### Rule 11 — `await` rethrows a rejected promise into the async function

```js
try {
  await Promise.reject(new Error("failed"));
} catch (error) {
  console.log(error.message);
}
```

This works because `await` observes the promise and resumes the async function with a throw-like failure path.

### Rule 12 — Do not assume every failure is exceptional

Many expected business outcomes are better represented as normal values:

```js
const result = authenticate(credentials);

if (!result.ok) {
  // expected business result
}
```

Use exceptions for failures that are appropriately modeled as exceptional control flow, not simply because a function can fail.

---

## 7. Syntax

### `throw`

```js
throw expression;
```

Examples:

```js
throw new Error("Invalid input");
throw new TypeError("Expected a string");
```

### `try/catch`

```js
try {
  operation();
} catch (error) {
  recover(error);
}
```

### `try/catch/finally`

```js
try {
  operation();
} catch (error) {
  handle(error);
} finally {
  cleanup();
}
```

### `catch` without binding

Modern JavaScript permits:

```js
try {
  operation();
} catch {
  recoverWithoutInspectingTheError();
}
```

Use this when the actual thrown value is intentionally irrelevant.

### Optional catch binding

The omission is significant because:

```js
catch (error)
```

and:

```js
catch
```

have different lexical semantics.

### Error constructors

Common built-in error classes include:

```js
Error
EvalError
RangeError
ReferenceError
SyntaxError
TypeError
URIError
AggregateError
```

The practical use of subclasses is to communicate category, not to create enormous hierarchies.

### `Error` options

Modern code can use:

```js
new Error("Database request failed", {
  cause: originalError
});
```

The `cause` provides causal context without forcing developers to flatten the original error into a string.

---

## 8. Basic Examples

### Basic throw and catch

```js
try {
  throw new Error("Something failed");
} catch (error) {
  console.log(error instanceof Error); // true
  console.log(error.message);           // "Something failed"
}
```

### Type-specific failure

```js
function double(value) {
  if (typeof value !== "number") {
    throw new TypeError("value must be a number");
  }

  return value * 2;
}

try {
  double("10");
} catch (error) {
  if (error instanceof TypeError) {
    console.log("Caller supplied the wrong type");
  }
}
```

### Rethrowing

```js
function loadConfig() {
  try {
    return JSON.parse("{bad json}");
  } catch (error) {
    console.error("Configuration parsing failed");
    throw error;
  }
}
```

### Wrapping with cause

```js
function loadConfig() {
  try {
    return JSON.parse("{bad json}");
  } catch (error) {
    throw new Error("Could not load application configuration", {
      cause: error
    });
  }
}
```

Now the outer error communicates the higher-level context while the original failure remains available.

### Cleanup in `finally`

```js
let connection;

try {
  connection = openConnection();
  return connection.query("SELECT 1");
} finally {
  connection?.close();
}
```

This pattern is conceptually correct but later chapters should connect it with structured resource-management features such as `using` and `DisposableStack`.

---

## 9. Execution Walkthrough

Consider:

```js
function parse(input) {
  return JSON.parse(input);
}

function load() {
  try {
    return parse("{");
  } catch (error) {
    throw new Error("Configuration loading failed", {
      cause: error
    });
  }
}

try {
  load();
} catch (error) {
  console.log(error.message);
  console.log(error.cause.message);
}
```

### Step 1

The top-level `try` calls `load()`.

### Step 2

`load()` calls `parse()`.

### Step 3

`parse()` calls `JSON.parse()`.

### Step 4

Parsing the incomplete input throws a `SyntaxError`.

### Step 5

The exception propagates from `parse()` to `load()`.

### Step 6

The `catch` in `load()` receives the exact `SyntaxError`.

### Step 7

`load()` creates a new `Error` whose `cause` points to the original error.

### Step 8

The new error is thrown.

### Step 9

The exception propagates out of `load()` into the outer `try`.

### Step 10

The outer `catch` receives the wrapping error.

The error chain is conceptually:

```text
Error: Configuration loading failed
  cause ─────► SyntaxError: unexpected end of input
```

This is preferable to reducing everything to:

```text
"something went wrong"
```

because the causal structure is preserved.

---

## 10. Internal Mechanics

Error handling becomes easier once you separate the language-level control mechanism from implementation details.

### 10.1 Abrupt completion

ECMAScript describes evaluation in terms of completion records. A normal completion contains a value; an abrupt completion represents a non-normal exit.

A throw produces an abrupt completion with the thrown value.

Conceptually:

```text
NormalCompletion(value)
```

versus:

```text
ThrowCompletion(error)
```

The engine uses this completion model to propagate control through nested execution constructs.

### 10.2 Propagation through call frames

At runtime, an exception can unwind active execution state until a handler is found.

Conceptually:

```text
frame C  ← throw
   ↑
frame B  ← no handler
   ↑
frame A  ← no handler
   ↑
caller   ← matching catch
```

JavaScript engines optimize this internally, but the semantic effect is equivalent to structured stack unwinding.

Do not assume the internal implementation is literally a simplistic “pop stack frame” algorithm in every engine. The language specification defines observable semantics; the engine decides how to implement them.

### 10.3 Stack traces

`Error.prototype.stack` is widely implemented, but historical details and exact formatting are not equivalent to the core ECMAScript semantics of `throw`.

A stack trace typically helps answer:

```text
What code path led here?
```

It does not automatically answer:

```text
Why did the business operation fail?
```

That distinction is why causal context and structured metadata matter.

### 10.4 Error construction and stack capture

Creating an error object can capture diagnostic information. Stack capture may therefore have non-trivial cost, especially at high volume.

Do not use exception creation as a high-frequency substitute for ordinary branching when the condition is expected and routine.

### 10.5 Cross-realm identity

A built-in error constructor from one realm can have identity relationships different from those in another realm.

For example, browser code can interact with objects originating in another realm such as an iframe.

This means:

```js
value instanceof Error
```

is not universally sufficient for every cross-realm scenario.

Boundary designs should rely on robust tagging/classification strategies rather than assuming one global constructor identity.

---

## 11. ECMAScript / Specification Semantics

The language specification models error behavior through evaluation rules and completion records.

### 11.1 `throw`

The semantics of a throw expression evaluate the operand and then produce a throw completion containing that value.

Conceptually:

```text
evaluate expression
     ↓
value
     ↓
ThrowCompletion(value)
```

### 11.2 `try`

A `try` statement coordinates the evaluation of:

```text
try block
catch clause
finally block
```

The key semantic rule is that `finally` participates in deciding the final completion of the whole statement.

### 11.3 `catch`

A caught throw completion makes its thrown value available to the catch parameter when a binding is present.

Example:

```js
try {
  throw 42;
} catch (x) {
  console.log(x);
}
```

The catch binding receives `42`.

### 11.4 Catch parameter scope

The catch parameter has its own lexical binding behavior.

Example:

```js
try {
  throw "outer";
} catch (error) {
  const message = "inner";
  console.log(error, message);
}
```

The catch binding is not simply another function parameter.

The catch parameter is scoped to the catch clause.

### 11.5 `finally` semantics

Consider:

```js
function f() {
  try {
    return 10;
  } finally {
    console.log("cleanup");
  }
}
```

Conceptually:

1. The `try` block produces a return completion.
2. The `finally` block runs.
3. Because `finally` completes normally, the prior return completion continues.
4. The function returns `10`.

Now:

```js
function f() {
  try {
    return 10;
  } finally {
    return 20;
  }
}
```

The `finally` block produces a new return completion that replaces the previous one.

Similarly:

```js
function f() {
  try {
    throw new Error("A");
  } finally {
    throw new Error("B");
  }
}
```

The final result is the throw of `"B"`.

### 11.6 `return`, `break`, and `continue`

These are also abrupt completions.

Therefore:

```js
for (;;) {
  try {
    break;
  } finally {
    console.log("cleanup");
  }
}
```

runs `finally` before the loop exits.

This common completion model is essential for understanding cleanup correctly.

### 11.7 Error subclasses

`Error` is an ordinary built-in constructor with subclassing behavior that participates in standard object construction and prototype mechanics.

Do not confuse:

```js
error instanceof TypeError
```

with a special engine-level “type system.” The object still has JavaScript object/prototype semantics.

---

## 12. Advanced Behavior

### 12.1 Error causes

Error wrapping is most useful when abstraction boundaries change the meaning of the failure.

Low-level:

```text
ECONNREFUSED
```

Higher-level:

```text
Failed to connect to primary database
```

The higher-level layer should preserve the original cause.

```js
throw new Error("Failed to connect to primary database", {
  cause: error
});
```

This supports layered diagnosis:

```text
API operation failed
    └── service failed
          └── database connection failed
                └── socket refused
```

### 12.2 `AggregateError`

Some operations can fail with multiple independent failures.

Example:

```js
throw new AggregateError(
  [
    new Error("Replica A failed"),
    new Error("Replica B failed")
  ],
  "All replicas failed"
);
```

This is semantically different from a single causal chain.

Use:

```text
cause
```

for one primary causal relationship.

Use:

```text
AggregateError.errors
```

for multiple associated failures.

### 12.3 Custom error classes

Example:

```js
class ValidationError extends Error {
  constructor(message, options) {
    super(message, options);
    this.name = "ValidationError";
  }
}
```

Usage:

```js
throw new ValidationError("Email is required");
```

For application systems, consider whether a class adds meaningful behavior.

A simple structured error can sometimes be enough:

```js
const error = new Error("Email is required");
error.code = "VALIDATION_FAILED";
```

The decision should be based on contract clarity, not style preference alone.

### 12.4 Error codes

Codes are often useful at boundaries:

```js
class AppError extends Error {
  constructor(message, { code, status, cause } = {}) {
    super(message, { cause });
    this.name = "AppError";
    this.code = code;
    this.status = status;
  }
}
```

Then:

```js
throw new AppError("User does not exist", {
  code: "USER_NOT_FOUND",
  status: 404
});
```

This avoids coupling machine logic to message text.

### 12.5 Operational vs programmer errors

A useful engineering distinction:

#### Operational failure

Often expected in production:

- network timeout;
- remote service unavailable;
- file missing;
- invalid user input;
- database deadlock;
- rate limit;
- dependency failure.

#### Programmer defect

Usually indicates broken assumptions:

- accessing an undefined variable due to a bug;
- invalid internal state;
- impossible invariant violation;
- incorrect algorithm;
- faulty null handling.

Not every programmer defect should be “recovered from.”

Sometimes the correct response is:

```text
fail fast
record evidence
restart / isolate
fix root cause
```

### 12.6 Error boundaries

A mature architecture catches errors at meaningful boundaries:

```text
HTTP boundary
Worker boundary
Queue-consumer boundary
CLI boundary
Process boundary
```

A controller may translate:

```text
ValidationError → HTTP 400
AuthenticationError → HTTP 401
NotFoundError → HTTP 404
```

But it should not convert every unknown defect into a misleading success response.

### 12.7 Error translation

Each abstraction layer may have its own vocabulary.

Example:

```text
database error
      ↓
repository error
      ↓
domain error
      ↓
HTTP response
```

Translation should preserve diagnostic context.

Do not indiscriminately flatten:

```js
catch {
  throw new Error("database error");
}
```

because that destroys information.

### 12.8 Error serialization

Never automatically return raw errors to clients:

```js
res.json(error);
```

An error object may contain:

- stack information;
- internal paths;
- implementation details;
- sensitive metadata;
- nested causes;
- connection details.

Instead, define an explicit client-safe shape:

```js
{
  error: {
    code: "USER_NOT_FOUND",
    message: "User not found"
  }
}
```

### 12.9 Error identity and `name`

Some code uses:

```js
error.name === "TypeError"
```

Others use:

```js
error instanceof TypeError
```

Neither should be treated as a universal answer across all environments and boundaries.

Machine contracts should prefer explicit stable codes or discriminators when appropriate.

---

## 13. Edge Cases

### 13.1 Throwing `undefined`

```js
throw undefined;
```

Then:

```js
catch (error) {
  console.log(error); // undefined
}
```

Never assume the catch binding is a useful object.

### 13.2 Throwing strings

Legal:

```js
throw "failed";
```

Problematic because conventional diagnostic properties do not exist.

### 13.3 `finally` masking errors

```js
function run() {
  try {
    throw new Error("real problem");
  } finally {
    cleanup();
  }
}
```

If `cleanup()` throws, the cleanup error can replace the original error.

This matters enormously for production reliability.

### 13.4 Cleanup itself can fail

A cleanup operation is not automatically harmless.

Examples:

- closing a network channel;
- flushing a log;
- committing a transaction;
- releasing an external lock.

Design the policy explicitly:

```text
primary failure
     +
cleanup failure
     ↓
Which failure is primary?
Which should be retained?
Should both be observable?
```

This is one reason later resource-management chapters matter.

### 13.5 Catching everything

```js
try {
  performEverything();
} catch {
  return null;
}
```

This may turn:

```text
unexpected defect
```

into:

```text
apparently valid empty result
```

which is much harder to detect.

### 13.6 Async callback failure

This does not catch:

```js
try {
  setTimeout(() => {
    throw new Error("boom");
  }, 0);
} catch {
  console.log("caught");
}
```

The callback executes after the synchronous protected region finishes.

### 13.7 Promise rejection

This does not catch a rejection unless the promise is observed in the protected asynchronous flow:

```js
try {
  Promise.reject(new Error("boom"));
} catch {
  console.log("caught");
}
```

The rejection is not a synchronous throw from the `try` statement.

By contrast:

```js
try {
  await Promise.reject(new Error("boom"));
} catch (error) {
  console.log("caught");
}
```

works inside an async function.

### 13.8 Cross-realm errors

A browser iframe may have a different intrinsic `Error` constructor.

Therefore, simple `instanceof` checks can be insufficient across realms.

### 13.9 Errors with non-enumerable properties

Standard error properties do not behave like ordinary enumerable application data in every serialization scenario.

This explains why:

```js
JSON.stringify(new Error("boom"))
```

does not necessarily produce the detailed structure a developer expects.

Explicit serialization is usually required.

### 13.10 Error messages can change

Do not build stable application logic around exact engine-generated wording.

This is brittle:

```js
if (error.message === "x is not defined") {
  // ...
}
```

---

## 14. Common Misconceptions

### Misconception 1 — “An error is the same thing as an exception.”

Not exactly.

- `Error` is a constructor / object type convention.
- Exception throwing is a control-flow mechanism.
- Any value can be thrown.

### Misconception 2 — “try/catch catches everything.”

No.

It catches exceptions propagating through the protected synchronous control flow.

It does not automatically capture failures that occur later in another event-loop turn.

### Misconception 3 — “Promise rejection is identical to a synchronous throw.”

They are related conceptually but differ in timing and propagation mechanics.

### Misconception 4 — “finally is only for successful cleanup.”

It runs on both normal and abrupt exits, unless termination occurs outside ordinary JavaScript completion semantics.

### Misconception 5 — “finally cannot change the result.”

It can. A `return` or `throw` from `finally` can override an earlier completion.

### Misconception 6 — “Error messages are APIs.”

Messages are primarily human-readable diagnostic text.

Stable machine behavior should use structured categories or codes.

### Misconception 7 — “Logging an error means handling it.”

Logging is observation.

Handling means making a decision about what the program should do next.

### Misconception 8 — “Throwing is always better than returning an error object.”

No.

Expected domain outcomes can often be clearer as explicit results.

### Misconception 9 — “A custom Error class automatically makes an API better.”

Only if the class improves a real contract, behavior, or boundary.

### Misconception 10 — “All errors should be converted to a generic error.”

Over-wrapping can destroy useful identity and context unless `cause` or structured metadata preserves them.

---

## 15. Common Mistakes

### Mistake 1 — Swallowing errors

```js
try {
  save();
} catch {
}
```

### Mistake 2 — Logging and rethrowing at every layer

```js
catch (error) {
  console.error(error);
  throw error;
}
```

Repeated across every layer, this creates duplicate noisy logs.

Observe failures where the system has enough context to act meaningfully.

### Mistake 3 — Using exceptions for ordinary branching

Bad:

```js
try {
  return map.get(key).value;
} catch {
  return fallback();
}
```

This hides unrelated defects.

Prefer explicit validation where appropriate.

### Mistake 4 — Catching too broadly

```js
catch (error) {
  return defaultValue;
}
```

This can recover from failures that are not actually recoverable.

### Mistake 5 — Throwing strings

```js
throw "bad";
```

### Mistake 6 — Parsing messages

```js
if (error.message.includes("duplicate")) {
  // ...
}
```

### Mistake 7 — Exposing internal stacks to clients

```js
res.status(500).json({
  message: error.message,
  stack: error.stack
});
```

### Mistake 8 — Losing causes during wrapping

```js
catch (error) {
  throw new Error("Request failed");
}
```

Prefer:

```js
catch (error) {
  throw new Error("Request failed", { cause: error });
}
```

when the original cause is relevant.

### Mistake 9 — Assuming cleanup cannot fail

```js
finally {
  close();
}
```

The cleanup path itself can throw.

### Mistake 10 — Treating every failure as retryable

A malformed request and a temporary network timeout have different retry semantics.

---

## 16. Comparison With Related Concepts

| Concept | Meaning | Typical use |
|---|---|---|
| `throw` | Changes control flow via abrupt completion | Exceptional failure propagation |
| `Error` | Value representing a failure | Diagnostics and classification |
| `return` | Normal function result | Expected outcomes |
| Promise rejection | Asynchronous failure channel | Async APIs |
| `catch` | Handles propagated exceptions | Recovery/translation |
| `finally` | Guaranteed cleanup point for normal JS completion paths | Resource cleanup |
| Result object | Explicit success/failure data | Predictable domain outcomes |
| Logging | Records evidence | Observability |
| Retry | Re-executes an operation | Transient failures |
| Validation | Checks input/state before work | Expected invalid cases |

### Exceptions vs result objects

Exception-style:

```js
function divide(a, b) {
  if (b === 0) {
    throw new RangeError("division by zero");
  }

  return a / b;
}
```

Result-style:

```js
function divide(a, b) {
  if (b === 0) {
    return {
      ok: false,
      code: "DIVISION_BY_ZERO"
    };
  }

  return {
    ok: true,
    value: a / b
  };
}
```

Neither is universally superior.

Ask:

- Is the failure expected?
- Does the caller routinely branch on it?
- Is failure part of normal domain logic?
- Would exceptions create hidden control flow?
- Does the ecosystem already establish one style?

### Error vs assertion

An assertion typically says:

> “This state should be impossible or this invariant must hold.”

Validation says:

> “The input may legitimately be invalid.”

The response strategy can therefore differ:

```text
user input invalid
    → validation result / domain error

internal invariant broken
    → diagnostic failure / fail fast
```

---

## 17. Performance Considerations

Error handling has performance implications, but avoid simplistic statements such as “try/catch is always slow.”

Modern engines optimize many common patterns.

More important considerations include:

### 17.1 Throwing is usually an exceptional path

Do not use exceptions as the normal loop-control mechanism.

Bad:

```js
for (const item of items) {
  try {
    process(item);
  } catch {
    // use throw as branch
  }
}
```

### 17.2 Creating errors can be expensive

An `Error` may capture stack information.

Avoid constructing huge numbers of errors in hot paths unless the failures are genuinely exceptional and worth the diagnostic cost.

### 17.3 Deep causal structures

Long nested causes can increase memory usage and diagnostic processing cost.

### 17.4 Logging costs

Logging an exception can be much more expensive than creating it, especially when:

- stack traces are serialized;
- logs are transmitted remotely;
- metadata is large;
- logging happens at high frequency.

### 17.5 Retry amplification

The largest performance cost may come from policy rather than language mechanics.

For example:

```text
request fails
→ retry
→ retry
→ retry
```

can multiply load on an already-failing dependency.

Error handling and performance must therefore be designed together.

---

## 18. Memory Considerations

Errors are objects.

They can retain references to metadata, causes, request context, or custom properties.

Avoid attaching entire heavyweight object graphs:

```js
error.context = giantRequestObject;
```

This can increase retention and memory pressure.

Prefer narrow metadata:

```js
error.context = {
  requestId,
  operation
};
```

Be especially careful with:

- cyclic structures;
- large request bodies;
- buffers;
- credentials;
- ORM entities;
- open handles.

A long-lived retry queue containing errors can unintentionally retain substantial memory through error graphs.

---

## 19. Security Considerations

Error handling is a security boundary.

### 19.1 Do not leak internal details

Potentially sensitive details include:

- file paths;
- SQL fragments;
- stack traces;
- hostnames;
- tokens;
- internal IDs;
- infrastructure names;
- configuration values.

### 19.2 Do not trust caught values

Because any value can be thrown:

```js
throw maliciousObject;
```

do not assume:

```js
error.message
error.stack
error.code
```

exist or have the expected types.

### 19.3 Avoid unsafe logging

Do not log secrets merely because an error object contains them.

### 19.4 Preserve diagnostic context without exposing it

A good architecture separates:

```text
internal diagnostic representation
```

from:

```text
external client representation
```

### 19.5 Prevent retry storms

An insecure or unreliable failure policy can become a denial-of-service amplifier.

### 19.6 Error-based information disclosure

Different responses can accidentally reveal:

```text
user exists / does not exist
resource exists / missing
credential correct / incorrect
internal topology
```

Normalize externally visible responses where required by the threat model.

---

## 20. Production Usage

### 20.1 HTTP API boundary

A production API should usually separate:

```text
domain error
internal error
client error response
```

Example policy:

```js
function toHttpResponse(error) {
  if (error?.code === "USER_NOT_FOUND") {
    return {
      status: 404,
      body: {
        error: {
          code: "USER_NOT_FOUND",
          message: "User not found"
        }
      }
    };
  }

  return {
    status: 500,
    body: {
      error: {
        code: "INTERNAL_ERROR",
        message: "Internal server error"
      }
    }
  };
}
```

The client does not need the internal stack.

### 20.2 Worker boundary

A queue worker should distinguish:

```text
retryable failure
permanent failure
poison message
programmer defect
```

Catching everything and acknowledging the message can cause silent loss.

### 20.3 Library boundary

Libraries should:

- document what they throw or reject with;
- preserve causes where relevant;
- avoid exposing unstable engine internals as contracts;
- avoid swallowing errors;
- avoid requiring consumers to parse messages.

### 20.4 CLI boundary

A CLI often needs:

```text
developer diagnostics
+
user-friendly stderr
+
non-zero process exit status
```

### 20.5 Service boundary

A service should know:

- which failures are retriable;
- which failures should be surfaced;
- which failures should trip a circuit breaker;
- which failures require alerting;
- which failures are expected noise.

### 20.6 Observability

A high-quality error event may include:

```text
error type
stable code
message
cause chain
request ID
operation
service/component
timestamp
environment
safe context
```

Avoid including secrets.

---

## 21. Implementation From Scratch

Build a small error framework progressively.

### Stage 1 — Guided

Create a base error:

```js
class AppError extends Error {
  constructor(message, {
    code = "INTERNAL_ERROR",
    status = 500,
    cause,
    details
  } = {}) {
    super(message, { cause });

    this.name = "AppError";
    this.code = code;
    this.status = status;
    this.details = details;
  }
}
```

Create subclasses:

```js
class ValidationError extends AppError {
  constructor(message, details) {
    super(message, {
      code: "VALIDATION_ERROR",
      status: 400,
      details
    });

    this.name = "ValidationError";
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, {
      code: "NOT_FOUND",
      status: 404
    });

    this.name = "NotFoundError";
  }
}
```

### Stage 2 — Partially Guided

Add:

- retryability;
- machine-readable category;
- safe public message;
- internal cause;
- operation metadata.

Example target shape:

```js
{
  name,
  code,
  category,
  status,
  retryable,
  message,
  details,
  cause
}
```

### Stage 3 — No Reference

Implement a production-oriented `AppError` without looking at the chapter.

Requirements:

- preserves `cause`;
- supports stable `code`;
- supports safe public representation;
- distinguishes operational failures from programmer defects;
- avoids serializing stack traces by default.

### Stage 4 — Edge-Case Hardened

Add handling for:

- thrown non-Error values;
- nested causes;
- circular metadata;
- absent status;
- invalid error codes;
- unknown external errors;
- cross-boundary classification;
- duplicate logging;
- cleanup failure.

### Stage 5 — Production Grade

Implement:

```js
class ErrorBoundary {
  classify(error) {}
  toPublic(error) {}
  toLogRecord(error) {}
  shouldRetry(error) {}
}
```

Requirements:

- bounded serialization;
- safe redaction;
- causal-chain traversal;
- explicit retry policy;
- stable client error codes;
- no accidental secret leakage;
- no duplicate logging;
- testable classification rules.

---

## 22. Debugging Exercises

### Exercise 1 — Basic propagation

Predict:

```js
function a() {
  throw new Error("A");
}

function b() {
  a();
}

try {
  b();
} catch (error) {
  console.log(error.message);
}
```

Expected reasoning:

```text
A
```

### Exercise 2 — Rethrow

```js
try {
  throw new Error("original");
} catch (error) {
  throw new Error("wrapped", { cause: error });
}
```

Question:

What information must the outer boundary inspect to recover the original failure?

### Exercise 3 — `finally` replacement

```js
function f() {
  try {
    return 1;
  } finally {
    return 2;
  }
}
```

Predict the result and explain the completion flow.

### Exercise 4 — Cleanup failure

```js
function f() {
  try {
    throw new Error("work failed");
  } finally {
    throw new Error("cleanup failed");
  }
}
```

What error reaches the caller? What diagnostic information has been lost?

### Exercise 5 — Async boundary

```js
try {
  setTimeout(() => {
    throw new Error("timer");
  }, 0);
} catch {
  console.log("caught");
}
```

Explain precisely why the catch block does not run.

### Exercise 6 — Promise boundary

```js
try {
  Promise.reject(new Error("rejected"));
} catch {
  console.log("caught");
}
```

Why is this not equivalent to:

```js
try {
  throw new Error("rejected");
} catch {
  console.log("caught");
}
```

### Exercise 7 — Throw any value

```js
try {
  throw null;
} catch (error) {
  console.log(error);
}
```

What assumptions about `error` are unsafe?

---

## 23. Code Review Exercise

Review this production candidate:

```js
async function getUser(req, res) {
  try {
    const user = await database.findUser(req.params.id);

    if (!user) {
      throw new Error("User not found");
    }

    res.json(user);
  } catch (error) {
    console.error(error);

    res.status(500).json({
      message: error.message,
      stack: error.stack
    });
  }
}
```

Identify at least ten issues or design questions.

Possible areas to investigate:

- error classification;
- client-safe serialization;
- status mapping;
- logging policy;
- sensitive information;
- error codes;
- expected vs unexpected failure;
- observability context;
- stable API contracts;
- duplicate or excessive logs;
- domain/business semantics;
- retry behavior at higher layers.

Then redesign the boundary.

---

## 24. Interview Questions

### Foundational

1. What is the difference between an `Error` and an exception?
2. Can JavaScript throw non-Error values?
3. What does `throw` do to control flow?
4. What is the purpose of `try/catch`?
5. Why is `finally` useful?
6. Can `finally` override a return value?
7. Can `finally` replace a thrown error?
8. What is `Error.cause` used for?
9. Why are error codes useful?
10. Why should error messages not usually be parsed?

### Intermediate

11. What happens when a function throws and has no local catch?
12. How does rethrowing differ from wrapping?
13. How would you classify operational vs programmer errors?
14. Why doesn't a synchronous `try/catch` catch a later timer exception?
15. Why can `await` allow `try/catch` to handle promise rejections?
16. What is `AggregateError`?
17. How do custom error classes help?
18. What are the risks of broad catches?
19. How should errors be serialized over HTTP?
20. Why should raw stacks not be exposed to clients?

### Advanced

21. Explain abrupt completion.
22. How does `finally` interact with `return`, `throw`, `break`, and `continue`?
23. Why can cleanup errors mask primary errors?
24. What is an error boundary?
25. How would you design retryable error classification?
26. What are cross-realm implications for `instanceof Error`?
27. How would you preserve diagnostics while changing abstraction-level semantics?
28. How can error handling create memory retention?
29. How can error handling create security vulnerabilities?
30. When should a function return a result instead of throwing?

### Principal-Level

31. Design an error contract for a multi-service platform.
32. How would you prevent duplicate logs across layers?
33. How would you distinguish user-caused failures from infrastructure failures?
34. How should a queue consumer classify errors?
35. How should error handling interact with circuit breakers and retries?
36. How would you design safe causal-chain logging?
37. How would you handle a failure in the cleanup path while preserving the primary error?
38. What should an API guarantee about its error shape?
39. Which parts of an error object should be public versus internal?
40. How would you evolve error codes without breaking clients?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
try {
  throw new Error("A");
} catch (error) {
  console.log("B");
}
console.log("C");
```

Predict:

```text
B
C
```

Then explain why the outer program continues after the catch completes normally.

### Exercise B

```js
function f() {
  try {
    return "A";
  } finally {
    console.log("B");
  }
}

console.log(f());
```

Predict the exact order.

### Exercise C

```js
function f() {
  try {
    return "A";
  } finally {
    return "B";
  }
}

console.log(f());
```

Explain which completion wins.

### Exercise D

```js
function f() {
  try {
    throw new Error("A");
  } finally {
    throw new Error("B");
  }
}

try {
  f();
} catch (error) {
  console.log(error.message);
}
```

Predict:

```text
B
```

Then explain what happened to `"A"`.

### Exercise E

```js
try {
  Promise.resolve().then(() => {
    throw new Error("async");
  });
} catch {
  console.log("caught");
}
```

Predict the output and explain why.

---

## 26. Mastery Exercises

### Exercise 1 — Error taxonomy

Design a taxonomy for:

```text
validation
authentication
authorization
not found
conflict
dependency timeout
dependency unavailable
rate limiting
database serialization failure
programmer defect
unknown failure
```

For each category decide:

- error code;
- public status;
- retryable?;
- alertable?;
- log severity;
- client message.

### Exercise 2 — Error boundary

Build an HTTP error boundary that:

- maps known application errors;
- returns safe messages;
- preserves request IDs;
- logs unexpected failures once;
- never exposes stack traces in production responses.

### Exercise 3 — Worker policy

Build a queue worker policy:

```text
process message
   ↓
success → ack
retryable failure → retry
permanent failure → dead-letter
programmer defect → fail according to worker supervision policy
```

### Exercise 4 — Causal graph

Create a failure chain:

```text
HTTP request
 → service operation
 → repository operation
 → database driver
 → network
```

Preserve causes at each abstraction boundary.

### Exercise 5 — Cleanup correctness

Write code that:

1. performs an operation;
2. attempts cleanup;
3. preserves primary failure information;
4. records cleanup failure;
5. exposes deterministic behavior to the caller.

Do not solve this by simply swallowing the cleanup error.

### Exercise 6 — Error serializer

Implement:

```js
serializeError(error)
```

Requirements:

- accepts arbitrary thrown values;
- safely identifies `Error` objects;
- includes stable fields;
- preserves a bounded cause chain;
- redacts sensitive metadata;
- does not expose internal stack traces by default.

### Exercise 7 — Property-based thinking

Generate arbitrary thrown values and verify that your boundary never crashes while trying to report an error.

---

## 27. Key Takeaways

1. Exceptions are a control-flow mechanism; `Error` is a value type/convention.
2. Any JavaScript value can be thrown, so caught values must not be blindly trusted.
3. `throw` creates an abrupt completion that propagates outward.
4. `catch` handles a propagated throw completion.
5. `finally` executes during normal cleanup paths and abrupt exits.
6. `finally` can override earlier return or throw completions.
7. `cause` preserves lower-level context while allowing higher-level semantic wrapping.
8. `AggregateError` represents multiple associated failures rather than a single cause.
9. Errors should be classified at meaningful boundaries.
10. Error messages are for humans; stable machine contracts should use structured codes or discriminators.
11. Promise rejection and asynchronous callback failures require different propagation reasoning from synchronous throws.
12. Cleanup can fail and can accidentally mask the primary failure.
13. Catching broadly without a recovery strategy can hide defects.
14. Logging is not the same as handling.
15. Error handling is also a performance, memory, security, reliability, and API-design concern.
16. Production systems should separate internal diagnostics from externally visible error responses.
17. The right place to catch an error is where the system has enough context to make a meaningful decision.
18. Error architecture should preserve information rather than flattening every failure into a generic string.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values, Types, Type System
- Chapter 06 — Operators, Expressions
- Chapter 08 — Control Flow, Iteration
- Chapter 09 — Functions / First-Class Behavior
- Chapter 10 — Scope / Lexical Environments / Identifier Resolution
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 28 — JSON / Serialization / Structured Clone

### Builds Toward

- Chapter 30 — Resource Management / Cleanup (`using`, `await using`, DisposableStack)
- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 39 — Concurrency / Parallelism
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 99 — Myths / Misconceptions
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 105 — Node REST API
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

### Related Concepts

- Completion records
- Control flow
- Call stacks
- Promises
- Async/await
- Resource cleanup
- Cancellation
- Retries
- Circuit breakers
- Logging
- Tracing
- Observability
- API contracts
- Validation
- Assertions
- Domain errors
- Process supervision

### Concepts Revisited

This chapter intentionally revisits:

- objects;
- classes;
- prototypes;
- lexical scoping;
- execution contexts;
- abrupt control flow;
- promises;
- async functions;
- JSON serialization.

### Why This Chapter Matters Later

Almost every production subsystem eventually crosses an error boundary.

HTTP clients fail.

Databases fail.

Files disappear.

Queues contain invalid messages.

User input is malformed.

Dependencies time out.

Workers crash.

Cleanup fails.

Distributed systems partially succeed.

Without a rigorous error model, later chapters on asynchronous programming, reliability, observability, testing, performance, security, and system design become collections of isolated practices.

The central principle is:

> Do not merely catch errors. Design failure behavior.

---

## 29. Completion Criteria

Mark Chapter 29 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Explain the difference between an error value and an exception.
- [ ] Explain abrupt completion.
- [ ] Explain synchronous exception propagation.
- [ ] Explain `try`, `catch`, and `finally`.
- [ ] Explain `Error`, subclasses, `cause`, and `AggregateError`.
- [ ] Explain why thrown values can be arbitrary.
- [ ] Explain operational vs programmer failures.

### Predictive Mastery

- [ ] Predict nested throw/catch behavior.
- [ ] Predict `finally` behavior with `return`.
- [ ] Predict `finally` behavior with `throw`.
- [ ] Predict cleanup masking.
- [ ] Predict synchronous vs asynchronous catch behavior.
- [ ] Predict promise rejection behavior.

### Implementation

- [ ] Build custom error classes.
- [ ] Build structured error codes.
- [ ] Preserve causal chains.
- [ ] Build an error boundary.
- [ ] Serialize errors safely.
- [ ] Design retry classification.
- [ ] Handle arbitrary thrown values.

### Debugging

- [ ] Read a stack trace.
- [ ] Follow a causal chain.
- [ ] Identify swallowed exceptions.
- [ ] Identify incorrect broad catches.
- [ ] Find async propagation mistakes.
- [ ] Detect sensitive error leakage.

### Production Engineering

- [ ] Design client-safe error responses.
- [ ] Define a stable error contract.
- [ ] Prevent duplicate logs.
- [ ] Distinguish retryable and non-retryable failures.
- [ ] Design worker failure behavior.
- [ ] Account for cleanup failures.
- [ ] Protect diagnostics from memory and security problems.

### Interview Readiness

- [ ] Answer foundational questions without memorized wording.
- [ ] Explain abrupt completion in your own model.
- [ ] Predict `finally` precedence.
- [ ] Explain async error propagation.
- [ ] Defend when to throw versus return.
- [ ] Design an error architecture for a production service.
- [ ] Defend the design against performance, security, reliability, and compatibility concerns.

### Track A — Core Theory

- [ ] Understand exception semantics.
- [ ] Understand completion-based control flow.
- [ ] Understand error object/prototype behavior.
- [ ] Understand causal and aggregate error representation.
- [ ] Understand boundary-based handling.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented implementation reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed debugging scenarios.
- [ ] Completed code review exercise.
- [ ] Completed architecture reasoning.
- [ ] Defended trade-offs under changing requirements.

### Mastery Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Do not mark `[*] Mastered` until the learner can independently:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# Chapter 29 — Revision / Retrieval Record

Use this section for future spaced-retrieval sessions.

### Retrieval Prompts

1. What exactly is thrown by `throw new Error("x")`?
2. What is an abrupt completion?
3. What happens if no `catch` handles a throw?
4. Why can `finally` replace a return value?
5. Why can cleanup mask the primary failure?
6. Why does synchronous `try/catch` not catch a later timer exception?
7. How does `await` change promise-rejection handling?
8. When should `cause` be used?
9. When is `AggregateError` more appropriate than `cause`?
10. Why are error codes more stable than messages?
11. How should an HTTP boundary classify errors?
12. How should a queue worker classify failures?
13. What information should never be exposed to clients?
14. When is returning a result better than throwing?
15. How would you prevent error handling from becoming a reliability problem?

### Weak Areas

```text
- 
- 
- 
```

### Revision Queue

```text
- [ ] Revisit abrupt completion and finally
- [ ] Revisit synchronous vs asynchronous propagation
- [ ] Revisit error classification
- [ ] Revisit cause and aggregate errors
- [ ] Revisit production error boundaries
```

### Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

### Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 29 — Canonical References and Source Discipline

For future detailed study of this chapter, use this source hierarchy:

1. ECMAScript specification — language-level semantics such as `throw`, `try`, completion records, constructors, and built-in error objects.
2. JavaScript engine documentation / implementation notes — implementation and diagnostic details such as stack-trace behavior and performance characteristics.
3. Host runtime documentation — browser or Node.js behavior for event-loop boundaries, process-level handling, worker behavior, and runtime-specific diagnostics.
4. Application architecture documentation — error taxonomy, API contracts, retry policies, observability, and operational behavior.

When a claim is engine-specific, label it as engine-specific.

When a claim is host-specific, label it as browser-specific, Node-specific, or otherwise runtime-specific.

Do not present implementation behavior as universal ECMAScript semantics.

---

# Chapter 29 — Completion Snapshot

```text
Chapter: 29
Title: Errors and Error Handling
Part: V — Errors / Cleanup
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
