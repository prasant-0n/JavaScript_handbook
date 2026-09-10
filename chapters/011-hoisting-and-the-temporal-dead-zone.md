

# Chapter 11 — Hoisting and the Temporal Dead Zone

> **Chapter Status:** `[+] Completed`
>
> **Prerequisite:** Chapter 10 — Scope, Lexical Environments, and Identifier Resolution
>
> **Next:** Chapter 12 — Execution Contexts and the Execution Model

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- explain what developers mean by “hoisting” without treating it as literal source-code movement;
- distinguish declaration creation, binding initialization, and assignment;
- explain why `var` can be read before its textual declaration while `let`, `const`, and `class` cannot;
- explain the Temporal Dead Zone (TDZ) as a period in which a lexical binding exists but is not initialized;
- predict whether an identifier access produces a value, `undefined`, a `ReferenceError`, or another error;
- distinguish function declaration instantiation from ordinary variable initialization;
- explain why `const` requires an initializer;
- explain the special behavior of `typeof` for undeclared identifiers and TDZ bindings;
- explain class declaration TDZ behavior;
- compare function declarations, function expressions, and arrow functions before their execution point;
- explain redeclaration rules for `var`, lexical declarations, and mixed `var`/lexical bindings;
- reason about hoisting in scripts, modules, functions, blocks, loops, and parameter environments;
- explain how default parameters create important TDZ-like cases;
- recognize legacy Annex B block-function behavior as compatibility behavior rather than the default mental model;
- connect declaration instantiation to lexical environments, execution contexts, closures, modules, and later runtime chapters;
- debug “Cannot access before initialization” and “is not defined” errors systematically;
- implement a small declaration-environment simulator;
- use prediction, execution tracing, and specification vocabulary instead of memorized “hoisting rules.”

---

# 2. Prerequisites

You should already understand:

1. scope and lexical scope;
2. lexical environments and environment records;
3. identifier resolution and the outer-environment chain;
4. `var`, `let`, and `const` at the source-language level;
5. function declarations and function expressions;
6. block scope and function scope;
7. modules versus scripts at a high level.

Chapter 10 established the key idea:

```text
identifier
   ↓
resolve through the environment chain
   ↓
find a binding
   ↓
read or write that binding
```

Chapter 11 adds the crucial timeline dimension:

```text
binding exists
      ↓
binding initialized?
      ↓
yes ───────────────→ value can be read
no ────────────────→ access may throw
```

This difference is the foundation of the Temporal Dead Zone.

---

# 3. What Is Hoisting?

“Hoisting” is a teaching term used to describe observable behavior that makes some declarations appear to be available before their textual position.

The dangerous mental model is:

```text
Source code:

console.log(x);
var x = 10;
```

becoming:

```text
var x;
console.log(x);
x = 10;
```

as if the JavaScript engine physically moved the line.

That is not the right general model.

A better model is:

```text
Source code
   ↓
Declaration information is processed as part of execution setup
   ↓
Bindings are created according to declaration rules
   ↓
Bindings may be initialized differently
   ↓
Executable statements run
```

For `var`, the binding is initialized to `undefined`.

For `let` and `const`, the lexical binding is created but remains uninitialized until execution reaches the declaration.

For a function declaration, the function binding can be initialized to the function object during declaration instantiation.

For a class declaration, the lexical binding is created but remains uninitialized until the class declaration is evaluated.

Thus:

> **Hoisting is best treated as a high-level description of declaration instantiation and early binding availability, not as code movement.**

---

# 4. Why Does This Exist?

JavaScript must establish the names that a piece of code can access before ordinary statements execute.

Consider:

```js
function greet() {
  return helper();
}

function helper() {
  return "hello";
}
```

When the function body begins executing, both function declarations are already represented by bindings.

This makes declarations useful for building a scope before executing the body.

That design supports:

- function declarations appearing after their uses;
- predictable lexical scoping;
- modules with statically analyzable bindings;
- early detection of invalid redeclarations;
- parameter and block environments;
- closures that capture lexical state;
- static tooling based on source structure.

The language therefore distinguishes several concepts that beginners often collapse into one:

```text
declaration
binding creation
binding initialization
assignment
access
```

Those are not interchangeable.

---

# 5. Mental Model

## 5.1 Think in terms of bindings, not lines

Suppose we have:

```js
console.log(a);
var a = 42;
```

Do not think:

```text
Move `var a` upward.
```

Think:

```text
Before normal statement execution:

Environment
┌───────────────┐
│ a → undefined │
└───────────────┘

Then:

console.log(a)
→ read binding `a`
→ result is undefined

Then:

a = 42
→ update the binding
```

For:

```js
console.log(a);
let a = 42;
```

think:

```text
Before normal statement execution:

Environment
┌───────────────────────────┐
│ a → uninitialized binding │
└───────────────────────────┘

console.log(a)
→ resolve `a`
→ binding exists
→ binding is uninitialized
→ ReferenceError
```

The crucial distinction is:

```text
var
binding created + initialized

let/const/class
binding created + not initialized
```

---

# 6. The Four-Stage Declaration Model

A useful engineering model is to separate four stages.

## Stage 1 — Declaration

Source code tells the runtime that a name is declared.

Example:

```js
let count;
```

## Stage 2 — Binding creation

A corresponding binding is established in the appropriate environment.

Conceptually:

```text
count → [binding]
```

## Stage 3 — Initialization

The binding receives its initial language-level value or becomes available for access.

Examples:

```text
var count
→ initialized to undefined

let count
→ initially uninitialized

const count = 1
→ initialized when declaration executes
```

## Stage 4 — Assignment or further mutation

After initialization, an assignable binding may receive another value:

```js
let count = 1;
count = 2;
```

Conceptually:

```text
creation → uninitialized
initialization → 1
assignment → 2
```

This model resolves many “hoisting” puzzles immediately.

---

# 7. Core Rules

## 7.1 `var` is function-scoped

```js
function demo() {
  var x = 10;
}

console.log(x); // ReferenceError
```

Inside the function, `x` belongs to the function-level environment.

---

## 7.2 `var` is initialized to `undefined`

```js
console.log(x);
var x;
```

Result:

```text
undefined
```

The declaration is processed before the ordinary statement executes, and the binding is initialized to `undefined`.

---

## 7.3 `let` is created but uninitialized

```js
console.log(x);
let x;
```

Result:

```text
ReferenceError
```

The error is not because the name does not exist.

The binding exists.

The error occurs because the binding has not yet been initialized.

---

## 7.4 `const` behaves like `let` with an additional initialization rule

```js
console.log(x);
const x = 10;
```

Result:

```text
ReferenceError
```

And:

```js
const x;
```

is a syntax error because `const` declarations require an initializer.

---

## 7.5 Function declarations can be available before their textual position

```js
sayHello();

function sayHello() {
  console.log("hello");
}
```

The function declaration is initialized during function declaration instantiation.

---

## 7.6 Function expressions are not function declarations

```js
sayHello();

const sayHello = function () {
  console.log("hello");
};
```

Result:

```text
ReferenceError
```

The binding exists but is uninitialized at the point of the call.

---

## 7.7 Arrow functions follow their containing binding rules

```js
run();

const run = () => {
  console.log("running");
};
```

Result:

```text
ReferenceError
```

An arrow function expression does not receive the special function-declaration initialization behavior.

---

## 7.8 Classes have TDZ behavior

```js
const value = new Person();

class Person {}
```

Result:

```text
ReferenceError
```

The class binding is lexical and uninitialized until the class declaration is evaluated.

---

# 8. Syntax

Relevant declaration forms include:

```js
var x;
var x = 1;

let y;
let y = 2;

const z = 3;

function add(a, b) {
  return a + b;
}

class User {
  constructor(name) {
    this.name = name;
  }
}

const fn = function () {};
const arrow = () => {};
```

The syntax looks similar from a source-code perspective, but the declaration-instantiation behavior differs.

---

# 9. Basic Examples

## 9.1 `var` before declaration

```js
console.log(value);
var value = 10;
```

Prediction:

```text
undefined
```

Execution model:

```text
function/global environment setup
→ create `value`
→ initialize `value` to undefined

statement 1
→ read value
→ undefined

statement 2
→ assign 10
```

---

## 9.2 `let` before declaration

```js
console.log(value);
let value = 10;
```

Prediction:

```text
ReferenceError
```

Execution model:

```text
setup
→ create lexical binding `value`
→ leave it uninitialized

statement 1
→ resolve `value`
→ GetBindingValue-like read
→ binding is uninitialized
→ throw ReferenceError
```

The declaration has not been “ignored.”

---

## 9.3 `const` before declaration

```js
console.log(value);
const value = 10;
```

Result:

```text
ReferenceError
```

---

## 9.4 Function declaration before use

```js
console.log(square(5));

function square(n) {
  return n * n;
}
```

Result:

```text
25
```

---

## 9.5 Function expression before use

```js
console.log(square(5));

var square = function (n) {
  return n * n;
};
```

Result:

```text
TypeError: square is not a function
```

Why?

Because:

```text
setup
→ square = undefined

call square(5)
→ attempt to call undefined
→ TypeError
```

This is an important distinction:

```text
ReferenceError:
binding unavailable / access invalid

TypeError:
binding available, but its current value cannot be used as a function
```

---

## 9.6 `var` with function expression

```js
console.log(square);

var square = function () {
  return 25;
};
```

Result:

```text
undefined
```

---

## 9.7 `let` with function expression

```js
console.log(square);

let square = function () {
  return 25;
};
```

Result:

```text
ReferenceError
```

---

# 10. Execution Walkthrough

Consider:

```js
console.log(a);
console.log(b);
greet();

var a = 10;
let b = 20;

function greet() {
  console.log("hello");
}
```

A useful conceptual preparation phase is:

```text
Global / script environment:

a → undefined
b → uninitialized
greet → function object
```

Then execution proceeds top-to-bottom.

### Statement 1

```js
console.log(a);
```

Read:

```text
a → undefined
```

Output:

```text
undefined
```

### Statement 2

```js
console.log(b);
```

Read:

```text
b → uninitialized
```

Result:

```text
ReferenceError
```

Execution stops before:

```js
greet();
```

The important lesson:

> declaration setup occurs before statement evaluation, but initialization timing still matters.

---

# 11. Declaration Instantiation

The phrase **declaration instantiation** describes the process by which execution establishes the bindings associated with declarations before or during the start of a particular execution unit.

The exact specification algorithm differs by context.

Relevant contexts include:

- global scripts;
- functions;
- modules;
- blocks;
- `catch` clauses;
- loop iterations;
- class bodies;
- parameter environments.

There is no single universal “hoisting algorithm.”

Instead, ECMAScript defines context-specific operations that establish and initialize bindings.

---

# 12. `var` Declaration Instantiation

Consider:

```js
function demo() {
  console.log(x);
  var x = 100;
}
```

A useful conceptual model is:

```text
Function declaration setup:

x binding created
x initialized to undefined

Body execution:

console.log(x)
→ undefined

x = 100
```

The engine is not literally rewriting the source file.

The binding is prepared as part of entering the relevant execution context.

---

# 13. Function Declaration Instantiation

Consider:

```js
function demo() {
  return helper();

  function helper() {
    return "ok";
  }
}
```

When `demo` begins execution, the function declaration binding for `helper` is established.

Conceptually:

```text
Enter demo()

Environment:
helper → function object
```

Then:

```js
return helper();
```

can call it.

This is why:

```js
helper();

function helper() {}
```

works in the same applicable declaration scope.

---

# 14. Lexical Declaration Instantiation

Lexical declarations include:

```js
let
const
class
```

The crucial behavior is:

```text
binding creation
      ≠
binding initialization
```

When the lexical scope is established, the binding can exist while remaining uninitialized.

This is what makes the TDZ possible.

Conceptually:

```text
Lexical Environment

x → <uninitialized>
```

A read before initialization is an error.

---

# 15. Temporal Dead Zone (TDZ)

The **Temporal Dead Zone** is the period between:

```text
lexical binding creation
```

and:

```text
lexical binding initialization
```

during which accessing the binding is not allowed.

Example:

```js
{
  // TDZ for value starts for this scope before this line executes.
  console.log(value);
  let value = 10;
}
```

The term “temporal” matters because the restriction depends on execution time.

The term “dead zone” means the binding cannot be legally read during that interval.

A better mental timeline is:

```text
Scope begins
      │
      │
      ├── lexical binding exists
      │
      ├── TDZ
      │
      ├── declaration evaluation
      │
      └── binding initialized
```

After initialization, ordinary reads are allowed.

---

# 16. TDZ Is Not “Variable Does Not Exist”

This distinction is critical.

For:

```js
console.log(x);
let x = 10;
```

there are two very different concepts:

### Undeclared identifier

```js
console.log(notDeclared);
```

There is no corresponding binding in the reachable environment chain.

Result:

```text
ReferenceError: notDeclared is not defined
```

### TDZ binding

```js
console.log(x);
let x = 10;
```

The lexical binding exists but is uninitialized.

Result:

```text
ReferenceError: Cannot access 'x' before initialization
```

Engines may phrase errors differently, but the language-level distinction matters.

---

# 17. `typeof` and the TDZ

A common misconception is:

> `typeof` never throws for an unknown variable.

The actual rule is subtler.

For an undeclared identifier:

```js
console.log(typeof doesNotExist);
```

the result is typically:

```text
"undefined"
```

But for a lexical binding in the TDZ:

```js
console.log(typeof value);
let value = 1;
```

the result is:

```text
ReferenceError
```

Why?

Because the identifier resolves to an existing lexical binding, and that binding is uninitialized.

Mental model:

```text
typeof undeclaredName
→ no binding found
→ special result "undefined"

typeof tdzName
→ binding found
→ binding uninitialized
→ ReferenceError
```

This is a famous edge case worth memorizing only after understanding the binding model.

---

# 18. `const` and Initialization

`const` introduces an immutable binding, but “immutable binding” does not mean:

```text
value itself can never change
```

It means the binding cannot be reassigned after initialization.

Example:

```js
const user = { name: "A" };

user.name = "B";
```

The object can be mutated.

The binding still refers to the same object.

For TDZ reasoning, the important point is:

```text
const binding
→ created
→ uninitialized
→ declaration executes
→ initialized
```

Also:

```js
const x;
```

is invalid because initialization is required as part of the declaration.

---

# 19. Class Declarations

Classes behave as lexical declarations.

Example:

```js
new User();

class User {}
```

The class binding exists but is uninitialized before the class declaration is evaluated.

Therefore:

```text
ReferenceError
```

This creates an important distinction from some function declaration behavior.

Compare:

```js
new User();

function User() {}
```

with:

```js
new User();

class User {}
```

The source looks structurally similar, but declaration semantics differ.

---

# 20. Function Declaration vs Function Expression

Compare these two programs.

### Function declaration

```js
run();

function run() {
  return 1;
}
```

Conceptual setup:

```text
run → function object
```

### Function expression with `const`

```js
run();

const run = function () {
  return 1;
};
```

Conceptual setup:

```text
run → uninitialized
```

Call occurs before initialization.

Result:

```text
ReferenceError
```

### Function expression with `var`

```js
run();

var run = function () {
  return 1;
};
```

Conceptual setup:

```text
run → undefined
```

Call occurs:

```text
undefined(...)
```

Result:

```text
TypeError
```

Thus, “functions are hoisted” is too imprecise to be useful.

The correct question is:

> **Which declaration form created which binding, and what was its initialization state at the time of access?**

---

# 21. Arrow Functions

Arrow functions are expressions.

Example:

```js
const add = (a, b) => a + b;
```

The arrow function object is produced when the initializer expression is evaluated.

Before that point:

```text
add → uninitialized
```

when `const` is used.

Therefore:

```js
add(1, 2);

const add = (a, b) => a + b;
```

throws a `ReferenceError`.

Again:

> The function syntax does not determine hoisting by itself. The declaration form and binding rules do.

---

# 22. Redeclaration Rules

## 22.1 `var` redeclaration

This is allowed in many ordinary declaration contexts:

```js
var x = 1;
var x = 2;
```

After execution:

```text
x === 2
```

The second declaration does not create an entirely unrelated lexical binding.

---

## 22.2 Lexical redeclaration

This is invalid:

```js
let x = 1;
let x = 2;
```

Result:

```text
SyntaxError
```

Likewise:

```js
const x = 1;
let x = 2;
```

is invalid in the same lexical scope.

---

## 22.3 `var` plus lexical declaration collision

Consider:

```js
var x = 1;
let x = 2;
```

This is a syntax error because the declarations conflict in the applicable scope.

The language must not silently treat these as two interchangeable bindings with ambiguous identifier resolution.

---

# 23. `var` Does Not Become Block-Scoped

Consider:

```js
{
  var x = 10;
}

console.log(x);
```

Result:

```text
10
```

The block does not create a new `var` scope.

Compare:

```js
{
  let y = 20;
}

console.log(y);
```

Result:

```text
ReferenceError
```

The block creates lexical scope for `let`.

---

# 24. Function Scope vs Block Scope

This example is useful:

```js
function demo() {
  if (true) {
    var a = 1;
    let b = 2;
    const c = 3;
  }

  console.log(a);
  console.log(b);
  console.log(c);
}
```

Behavior:

```text
a → accessible
b → inaccessible
c → inaccessible
```

Mental model:

```text
Function Environment
│
├── a
│
└── Block Lexical Environment
    ├── b
    └── c
```

The declaration form determines which environment receives the binding.

---

# 25. TDZ and Blocks

The TDZ exists per lexical scope.

Example:

```js
let value = 1;

{
  console.log(value);
  let value = 2;
}
```

Many beginners expect:

```text
1
```

But the inner `let value` shadows the outer binding for the whole inner lexical scope.

So inside the block:

```text
resolve "value"
→ inner lexical binding
→ inner binding is uninitialized
→ ReferenceError
```

This is the direct connection from Chapter 10:

```text
scope resolution first
then binding state
```

The engine does not skip the inner binding just because it is not initialized.

---

# 26. Shadowing + TDZ

Consider:

```js
const outer = "outer";

{
  console.log(outer);
  const outer = "inner";
}
```

The inner `outer` shadows the outer `outer`.

The identifier resolves to the inner binding.

The inner binding is in its TDZ.

Therefore:

```text
ReferenceError
```

Correct reasoning:

```text
Which binding?
→ inner outer

Is it initialized?
→ no

Read legal?
→ no

Result
→ ReferenceError
```

This is much more reliable than saying:

> JavaScript gets confused.

It does not.

The resolution rules are deterministic.

---

# 27. Parameter Bindings and TDZ

Function parameters have their own binding semantics.

Consider:

```js
function demo(a = b, b = 10) {
  return a + b;
}

demo();
```

The default initializer for `a` attempts to access `b` before `b` has been initialized for the parameter initialization sequence.

This produces:

```text
ReferenceError
```

The important principle is:

> Default parameter initializers are evaluated in parameter-initialization order, and later parameters are not magically available as initialized values.

Compare:

```js
function demo(a = 10, b = a) {
  return [a, b];
}

demo();
```

Result:

```text
[10, 10]
```

Here, `a` has already been initialized when `b`'s initializer runs.

---

# 28. Parameter Environment vs Function Body

A function with default parameters may require a distinct parameter environment model from the body environment.

For reasoning purposes:

```text
Call begins
   ↓
parameter bindings established
   ↓
default initializers evaluated as needed
   ↓
function body execution begins
```

This matters for:

- default parameters;
- parameter-name duplication rules;
- `arguments`;
- closures created inside parameter initializers;
- `eval`;
- strictness interactions.

Do not reduce parameter handling to “arguments are just variables at the top of the function.”

That shortcut breaks on edge cases.

---

# 29. Function Parameters Can Refer to Earlier Parameters

Example:

```js
function range(start = 0, end = start + 10) {
  return [start, end];
}

range();
```

Result:

```text
[0, 10]
```

The dependency direction matters:

```text
start initialized
     ↓
end default expression reads start
```

Reverse the dependency:

```js
function range(start = end, end = 10) {
  return [start, end];
}
```

This fails because `end` is not yet initialized when `start`'s initializer is evaluated.

---

# 30. `arguments` and Default Parameters

Default parameters also affect traditional `arguments` mapping behavior.

For example:

```js
function demo(a, b = 2) {
  a = 100;
  return arguments[0];
}
```

Do not assume historical sloppy-mode parameter aliasing behavior from every function form.

Modern reasoning should ask:

```text
Does this function have a simple parameter list?
Are default parameters present?
Is the function strict?
What does the specification define for this parameter environment?
```

This chapter focuses on declaration instantiation, but these parameter details become important in advanced function semantics.

---

# 31. `for` Loops and Per-Iteration Lexical Bindings

`let` and `const` in loops have an important relationship with declaration initialization.

Example:

```js
for (let i = 0; i < 3; i++) {
  console.log(i);
}
```

The language models lexical loop state so that closures can observe the appropriate iteration binding.

Example:

```js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks[0]());
console.log(callbacks[1]());
console.log(callbacks[2]());
```

Expected:

```text
0
1
2
```

This is not simply “the loop variable is copied.”

The language has per-iteration lexical semantics.

A later chapter will connect this more deeply to execution contexts and closures.

---

# 32. Compare `var` and `let` in Loops

With `var`:

```js
const callbacks = [];

for (var i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks[0]());
console.log(callbacks[1]());
console.log(callbacks[2]());
```

Typical result:

```text
3
3
3
```

All callbacks observe the same function-scoped `i` binding.

With `let`, each iteration has the relevant per-iteration lexical binding behavior:

```text
iteration 0 → binding for 0
iteration 1 → binding for 1
iteration 2 → binding for 2
```

This is one of the strongest practical reasons to understand declaration semantics rather than simply memorizing “let fixes loops.”

---

# 33. `catch` Parameters

A `catch` clause can introduce a binding:

```js
try {
  throw new Error("fail");
} catch (error) {
  console.log(error.message);
}
```

The catch parameter is scoped to the catch clause.

That means:

```js
try {
  throw new Error("fail");
} catch (error) {
  // error is available here
}

console.log(error);
```

does not find the catch binding.

This reinforces the Chapter 10 rule:

```text
syntax creates a lexical region
declaration creates a binding in that region
resolution follows environment relationships
```

---

# 34. Global Script vs Module

Top-level behavior differs between scripts and modules.

A classic script has global declaration behavior involving the global environment.

An ES module has module-scoped bindings.

Therefore:

```js
let x = 1;
```

at the top of a module does not behave like a `var` property attached to the global object.

The exact global environment model matters for:

- top-level `var`;
- top-level lexical declarations;
- global object properties;
- `globalThis`;
- module scope;
- import/export bindings.

Modules are deliberately designed to give stronger lexical isolation.

---

# 35. Global `var` and Global Object Properties

In browser-style script environments, global `var` declarations can participate in object-backed global semantics.

Conceptually:

```js
var x = 10;
```

may be associated with:

```js
globalThis.x
```

under relevant global-script rules.

But:

```js
let y = 20;
```

does not mean:

```js
globalThis.y
```

in the same way.

Therefore:

> “Global variable” and “global object property” are not universal synonyms.

This distinction becomes increasingly important in browser architecture and modules.

---

# 36. Imports, Exports, and Live Bindings

Modules use lexical bindings and explicit dependency edges.

An imported binding is not simply a copied snapshot of the source variable.

For example:

```js
// counter.js
export let count = 0;

export function increment() {
  count++;
}
```

and:

```js
// consumer.js
import { count, increment } from "./counter.js";

console.log(count);
increment();
console.log(count);
```

The import observes the exported binding.

The exact module binding machinery is more sophisticated than ordinary variable assignment.

Chapter 64 will revisit this formally.

---

# 37. Hoisting Is Context-Dependent

The statement:

> “JavaScript hoists everything.”

is not useful.

You need to specify:

```text
Which declaration?
Which environment?
Which execution context?
Which realm?
Script or module?
Function body or block?
Strict or non-strict?
Default parameter list?
Legacy Annex B behavior?
```

The word “hoisting” hides those distinctions.

---

# 38. Legacy Block-Level Function Declarations

Modern JavaScript has standardized block-scoped lexical behavior for function declarations in many contexts, but legacy web compatibility rules can introduce special behavior.

These compatibility rules are associated with **Annex B** of ECMAScript.

Example patterns involving functions inside blocks historically varied across browsers.

For production engineering:

- prefer explicit and modern declaration forms;
- do not build architecture around legacy block-function quirks;
- understand Annex B when debugging old code or compatibility-sensitive environments;
- distinguish standardized core behavior from legacy web compatibility behavior.

Do not teach yourself:

```text
function inside block = one universal behavior everywhere
```

without considering the source language mode and compatibility context.

---

# 39. `eval` and Declaration Instantiation

Direct `eval` can interact with the current execution environment and declaration behavior.

Example:

```js
function demo() {
  eval("var x = 10;");
  console.log(x);
}
```

In non-strict contexts, direct `eval` can have effects on variable environment behavior that make static reasoning harder.

Strict mode changes important parts of this behavior.

This is one reason `eval` is problematic for:

- security;
- optimization;
- static analysis;
- refactoring;
- performance predictability;
- maintainability.

A principal-level engineer does not merely ask:

> Can this code use `eval`?

They ask:

```text
What environment does it mutate?
Can its effect be statically known?
Does it cross trust boundaries?
Does it defeat tooling assumptions?
Does it create compatibility obligations?
```

---

# 40. `with` and Name Resolution

The deprecated/problematic `with` statement can alter how identifier lookup behaves by inserting object-property-based resolution into the scope chain.

Example:

```js
with (obj) {
  console.log(value);
}
```

Now the source-level identifier may resolve differently depending on object properties.

This complicates:

- static analysis;
- declaration reasoning;
- optimization;
- debugging;
- refactoring.

Modern production JavaScript should avoid `with`.

It is historically important because it demonstrates why lexical resolution must be distinguished from ordinary object property lookup.

---

# 41. Common Hoisting Myths

## Myth 1 — “JavaScript moves variables to the top.”

Better:

```text
Declarations are processed according to execution-context rules.
```

---

## Myth 2 — “`let` is not hoisted.”

Better:

```text
The lexical binding is established before execution reaches the source declaration,
but it remains uninitialized during the TDZ.
```

---

## Myth 3 — “`const` is not hoisted.”

Same correction:

```text
The binding is established.
Initialization happens at declaration evaluation.
```

---

## Myth 4 — “Function expressions are hoisted.”

Better:

```text
The variable binding may be created early,
but the function object from the expression is produced only when the initializer runs.
```

---

## Myth 5 — “TDZ means JavaScript cannot see the variable.”

Not exactly.

The runtime can resolve the binding.

The access fails because the binding is uninitialized.

---

## Myth 6 — “`typeof` is always safe.”

False for TDZ lexical bindings.

---

## Myth 7 — “All function declarations behave identically in every block and runtime.”

False.

Legacy compatibility semantics exist.

---

# 42. Common Mistakes

## Mistake 1 — Explaining everything with “hoisting”

Weak explanation:

> `let` is not hoisted.

Better explanation:

> `let` creates a lexical binding during scope setup, but the binding remains uninitialized until its declaration is evaluated, producing TDZ behavior before initialization.

---

## Mistake 2 — Ignoring binding state

Weak reasoning:

```text
x exists, so x can be read.
```

Correct reasoning:

```text
x resolves to a binding.
What state is that binding in?
```

---

## Mistake 3 — Confusing `undefined` with uninitialized

These are different:

```text
initialized with the value undefined
```

versus:

```text
uninitialized
```

`var` uses the first state.

`let`/`const`/`class` use the second state before declaration evaluation.

---

## Mistake 4 — Treating function declarations and function expressions as equivalent

They are not.

---

## Mistake 5 — Ignoring shadowing in TDZ examples

A local lexical declaration can shadow an outer binding before the local binding is initialized.

---

# 43. Comparison With Related Concepts

| Concept | Binding Created Early? | Initial Value Before Source Declaration? | Access Before Declaration |
|---|---|---|---|
| `var` | Yes | `undefined` | Allowed |
| `let` | Yes | Uninitialized | `ReferenceError` |
| `const` | Yes | Uninitialized | `ReferenceError` |
| `class` | Yes | Uninitialized | `ReferenceError` |
| Function declaration | Yes, per applicable declaration rules | Function object | Usually allowed in declaration scope |
| `var` + function expression | `var` binding | `undefined` | Access allowed, call fails |
| `const` + function expression | Lexical binding | Uninitialized | `ReferenceError` |
| Arrow function in `const` | Lexical binding | Uninitialized | `ReferenceError` |

The table is a summary, not a replacement for understanding the context-specific specification rules.

---

# 44. Hoisting vs Scope

Scope answers:

```text
Where can this identifier be found?
```

Hoisting/declaration instantiation answers:

```text
When is its binding created and initialized?
```

These questions interact.

Example:

```js
{
  console.log(value);
  let value = 1;
}
```

Scope says:

```text
The identifier resolves to the inner `value`.
```

Declaration timing says:

```text
The inner `value` is uninitialized at the read.
```

Result:

```text
ReferenceError
```

---

# 45. Hoisting vs Initialization

This distinction is worth making explicit.

```js
var x = 10;
```

Conceptually:

```text
binding creation
→ initialization to undefined
→ assignment/initializer evaluation
→ x becomes 10
```

For:

```js
let x = 10;
```

conceptually:

```text
binding creation
→ x uninitialized
→ declaration evaluation
→ initializer evaluated
→ x initialized to 10
```

The source appears simple because several distinct runtime steps are compressed into one line of code.

---

# 46. Hoisting vs Assignment

Consider:

```js
var x = 1;
```

The declaration and assignment are not one indivisible concept.

The `var` declaration establishes the binding.

The initializer causes a value to be assigned as part of declaration execution.

Compare:

```js
var x;
```

Here there is no explicit initializer.

But `x` still has the value:

```text
undefined
```

because the `var` binding is initialized accordingly.

---

# 47. Execution Trace: `var`

Program:

```js
console.log(a);
var a = 5;
console.log(a);
```

Trace:

```text
Declaration setup:
a → undefined

Statement 1:
read a
→ undefined

Statement 2:
evaluate initializer 5
→ assign a = 5

Statement 3:
read a
→ 5
```

Output:

```text
undefined
5
```

---

# 48. Execution Trace: `let`

Program:

```js
console.log(a);
let a = 5;
console.log(a);
```

Trace:

```text
Lexical setup:
a → uninitialized

Statement 1:
resolve a
→ binding found
→ binding uninitialized
→ ReferenceError

Program flow stops.
```

The later initialization is never reached.

---

# 49. Execution Trace: Function Declaration

Program:

```js
console.log(greet());

function greet() {
  return "hello";
}
```

Trace:

```text
Declaration setup:
greet → function object

Statement 1:
resolve greet
→ callable function object
→ invoke
→ "hello"
```

---

# 50. Execution Trace: `var` Function Expression

Program:

```js
console.log(greet());

var greet = function () {
  return "hello";
};
```

Trace:

```text
Setup:
greet → undefined

Call:
greet()
→ attempt to call undefined
→ TypeError
```

The function expression itself has not yet executed.

---

# 51. Execution Trace: `const` Function Expression

Program:

```js
console.log(greet());

const greet = function () {
  return "hello";
};
```

Trace:

```text
Setup:
greet → uninitialized

Call:
resolve greet
→ binding uninitialized
→ ReferenceError
```

---

# 52. Execution Trace: Class Declaration

Program:

```js
const instance = new User();

class User {}
```

Trace:

```text
Setup:
User → uninitialized

Statement 1:
resolve User
→ binding exists
→ uninitialized
→ ReferenceError
```

---

# 53. Formal Specification Vocabulary

The ECMAScript specification does not define a single abstract operation named simply “hoist.”

Instead, it defines algorithms and environment machinery that establish declaration bindings in particular execution contexts.

Important vocabulary includes:

- **Lexical Environment**
- **Environment Record**
- **Global Environment Record**
- **Function Environment Record**
- **Declarative Environment Record**
- **Object Environment Record**
- **Module Environment Record**
- **binding**
- **initialization**
- **resolution**
- **declaration instantiation**

The exact set of abstract operations involved depends on context.

---

# 54. Binding Creation, Initialization, and Assignment

A useful specification-oriented vocabulary is:

```text
Create binding
      ↓
Initialize binding
      ↓
Get / Set binding
```

For conceptual environment simulators, think in terms of operations such as:

```text
CreateMutableBinding
CreateImmutableBinding
InitializeBinding
SetMutableBinding
HasBinding
GetBindingValue
DeleteBinding
```

The specification contains context-specific variants and additional rules, so these names should be treated as semantic concepts and abstract-operation vocabulary rather than as a tiny public JavaScript API.

---

# 55. `GetBindingValue` Mental Model

When code evaluates:

```js
console.log(x);
```

the runtime conceptually must:

1. resolve `x`;
2. locate its environment record;
3. retrieve the binding value;
4. validate whether the binding is initialized;
5. produce the value or throw.

For TDZ cases, the critical point is step 4.

This explains why:

```js
let x;
console.log(x);
```

works once the declaration has executed:

```text
x initialized to undefined
```

while:

```js
console.log(x);
let x;
```

fails:

```text
x uninitialized
```

---

# 56. `CreateMutableBinding`

`let` creates a mutable lexical binding.

Conceptually:

```text
let x
→ mutable binding
→ initially uninitialized
```

Later:

```text
InitializeBinding(x, initialValue)
```

After initialization:

```text
SetMutableBinding(x, newValue)
```

This helps distinguish:

```text
binding mutability
```

from:

```text
value mutability
```

---

# 57. `CreateImmutableBinding`

`const` and certain other lexical constructs use immutable bindings.

Conceptually:

```text
const x = 1
→ create immutable binding
→ initialize once
```

Later assignment:

```js
x = 2;
```

is not a legal update to that binding.

Again, that does not imply the referenced object is deeply immutable.

---

# 58. Function Declaration Instantiation: Conceptual Sequence

For a function body, a simplified model is:

```text
Create function execution environment
        ↓
Process parameters
        ↓
Process declarations
        ↓
Create/initialize relevant bindings
        ↓
Execute function body
```

The actual algorithm includes nuanced handling for:

- parameters;
- `arguments`;
- function declarations;
- `var` declarations;
- lexical declarations;
- strictness;
- duplicate names;
- parameter environments;
- `eval`.

Do not memorize a fake universal sequence as if it were the full specification.

Use the simplified sequence for reasoning, then consult the specification when a corner case demands exactness.

---

# 59. Global Declaration Instantiation: Conceptual Sequence

For a classic global script, the runtime must coordinate:

```text
global object-backed semantics
+
global declarative lexical semantics
```

The language therefore has a richer global environment model than:

```text
global object = all variables
```

This matters for:

- `var`;
- `let`;
- `const`;
- `function`;
- global property conflicts;
- deletability rules;
- modules versus scripts.

Global declaration instantiation is one of the best examples of why simplistic “hoisting” explanations eventually stop being sufficient.

---

# 60. Scripts vs Modules: Execution Timing

Modules are parsed and linked before their evaluation order runs.

That supports static import/export structure.

Conceptually:

```text
parse module
   ↓
resolve dependencies
   ↓
create module bindings
   ↓
link imports/exports
   ↓
evaluate modules in dependency order
```

This is one reason module declarations cannot be explained as “just variables at the top of a file.”

Module bindings are part of a dependency-aware execution model.

---

# 61. Why Modules Improve Reasoning

Modules give you:

- explicit dependencies;
- lexical top-level bindings;
- live import bindings;
- no accidental dependence on global object mutation for normal imports;
- stronger static tooling;
- clearer ownership of state.

That reduces an entire class of “what is this global variable really referring to?” problems.

---

# 62. TDZ and Cyclic Dependencies

Module TDZ behavior becomes particularly important with cyclic dependencies.

A simplified shape:

```text
A imports B
B imports A
```

If a module reads an imported binding before the dependency's initialization has occurred, an access error can occur.

This is not simply:

```text
import copying undefined
```

The module system preserves binding relationships.

This is a later topic, but Chapter 11 gives you the vocabulary needed to understand why cycles can expose uninitialized bindings.

---

# 63. Static vs Dynamic Reasoning

Lexical declarations are designed to make declaration structure more statically visible.

Tools such as:

- linters;
- IDEs;
- refactoring engines;
- bundlers;
- compilers/transpilers;
- module analyzers

can often reason about:

```js
let x;
const y = 1;
function f() {}
```

because these declarations have lexical structure.

Dynamic features such as:

```js
eval
with
```

weaken that predictability.

This is a direct bridge from language semantics to tooling architecture.

---

# 64. Performance Considerations

For ordinary application code, “hoisting” itself is not a performance optimization you should manually target.

Avoid thinking:

> `var` is faster because it is hoisted.

That is not a meaningful engineering rule.

Modern JavaScript engines optimize code using many internal techniques:

- bytecode/interpreter tiers;
- inline caches;
- hidden classes/shapes;
- optimized machine code;
- deoptimization;
- escape analysis and allocation strategies.

Declaration semantics constrain correctness; they are not a simplistic direct performance switch.

More relevant performance considerations include:

- avoiding dynamic scope-changing constructs such as `with`;
- minimizing `eval`;
- keeping module dependencies understandable;
- reducing accidental global state;
- writing code that remains statically analyzable.

---

# 65. Memory Considerations

A binding is a language-level concept.

A JavaScript engine may implement environments in optimized representations rather than literally storing one heap object per source scope.

Possible implementation strategies include:

- stack-like frame storage;
- registers;
- context objects;
- optimized captured-variable layouts;
- heap allocation when closures require escaping state.

Therefore:

```text
Lexical Environment
```

is a semantic model, not necessarily a direct one-to-one heap object representation.

This distinction is crucial for connecting Chapter 11 to later engine and garbage-collection chapters.

---

# 66. Security Considerations

Declaration behavior affects security indirectly but significantly.

## 66.1 Accidental globals

Unexpected global writes can create shared mutable state:

```js
user = request.user;
```

in contexts where accidental global creation is possible.

That can cause cross-request state corruption in long-lived processes.

## 66.2 Shadowing mistakes

Security-sensitive code can be harder to review when names are deeply shadowed:

```js
const user = trustedUser();

function process() {
  const user = getInput();
}
```

The name alone does not communicate which binding is being used.

## 66.3 Dynamic code

`eval` can introduce both code-injection risk and declaration-analysis complexity.

## 66.4 Global state

Global mutable bindings can create unexpected communication channels between unrelated parts of an application.

Principal-level rule:

> Make state ownership explicit and minimize ambient mutable state.

---

# 67. Production Usage

In modern production JavaScript:

```text
prefer:
- let
- const
- function declarations where they improve readability
- modules
- explicit imports/exports
```

Use `var` mainly when:

- maintaining legacy code;
- matching deliberate compatibility requirements;
- teaching historical semantics;
- dealing with code where function-scoped behavior is intentionally required.

Use `const` by default when the binding does not need reassignment.

Use `let` when reassignment is part of the design.

Do not choose declarations based on “which one is most hoisted.”

Choose them based on:

- scope;
- lifetime;
- mutability;
- initialization guarantees;
- ownership;
- readability;
- compatibility.

---

# 68. Design Rule: Fail Early on Initialization Errors

TDZ errors are often helpful.

Example:

```js
const config = loadConfig();
```

If code somehow tries to access `config` before initialization, failing immediately is usually better than silently providing:

```text
undefined
```

Silent `undefined` values can propagate and cause distant failures.

TDZ converts a class of subtle bugs into immediate errors.

That is one reason lexical declarations are safer for many application scenarios.

---

# 69. Design Rule: Minimize Declaration Distance

This is not a language requirement, but a production readability rule.

Less clear:

```js
doSomething(value);

// hundreds of lines

let value = createValue();
```

Clearer:

```js
const value = createValue();
doSomething(value);
```

Even though the language can process declarations before source execution in certain ways, humans benefit from nearby initialization.

This reduces cognitive load and makes dataflow easier to review.

---

# 70. Implementation From Scratch

Build a tiny declaration-state simulator.

The simulator does not need to implement JavaScript syntax.

Its job is to model:

```text
binding created
binding initialized
binding read
binding written
```

---

## 70.1 Binding Model

Represent a binding as:

```js
class Binding {
  constructor(kind) {
    this.kind = kind;
    this.initialized = false;
    this.value = undefined;
  }

  initialize(value) {
    if (this.initialized) {
      throw new Error("Binding already initialized");
    }

    this.value = value;
    this.initialized = true;
  }

  get() {
    if (!this.initialized) {
      throw new ReferenceError("Binding is uninitialized");
    }

    return this.value;
  }
}
```

This intentionally models the central TDZ concept.

---

## 70.2 `var` Simulator

```js
const x = new Binding("var");
x.initialize(undefined);

console.log(x.get()); // undefined
```

This models:

```text
create
→ initialize undefined
→ read
```

---

## 70.3 `let` Simulator

```js
const x = new Binding("let");

// x.get(); // ReferenceError

x.initialize(10);

console.log(x.get()); // 10
```

This models:

```text
create
→ uninitialized
→ read → error
→ initialize
→ read → 10
```

---

## 70.4 Immutable Binding Extension

Extend the simulator:

```js
class ImmutableBinding extends Binding {
  constructor(kind) {
    super(kind);
    this.settable = false;
  }

  set() {
    throw new TypeError("Immutable binding");
  }
}
```

Now you can model the difference between:

```text
binding initialization
```

and:

```text
later assignment
```

---

# 71. Guided Implementation Exercise

Write:

```js
function createVarBinding(name) {}
function createLetBinding(name) {}
function initializeBinding(binding, value) {}
function readBinding(binding) {}
function writeBinding(binding, value) {}
```

Requirements:

### `var`

```text
created
→ initialized to undefined
→ writable
```

### `let`

```text
created
→ uninitialized
→ writable after initialization
```

### `const`

```text
created
→ uninitialized
→ writable only through initialization
→ later writes fail
```

Do not implement the whole language.

Implement the declaration state machine.

---

# 72. Partially Guided Exercise

Given:

```js
{
  let a;
  const b = 10;
  var c = 20;
}
```

Build a simulated environment:

```text
Lexical:
a → initialized(undefined)
b → initialized(10)

Function/global var environment:
c → 20
```

Then answer:

```text
Which binding does `a` use?
Which binding does `b` use?
Where does `c` live?
Which declarations are block-scoped?
Which one can be reassigned?
```

---

# 73. No-Reference Implementation Exercise

Build a mini interpreter for only these declarations:

```text
var
let
const
function
```

Support commands:

```text
declare
initialize
read
write
```

Example:

```text
declare var x
read x
initialize x 10
read x
```

Expected:

```text
undefined
10
```

And:

```text
declare let x
read x
```

Expected:

```text
ReferenceError
```

For:

```text
declare const x
write x 10
```

your interpreter should decide that initialization, not ordinary assignment, is required.

---

# 74. Edge-Case Hardening Exercise

Add support for:

- lexical shadowing;
- nested environments;
- `var` function scope;
- block scope;
- function declarations;
- classes;
- `typeof`;
- redeclaration errors;
- parameter initializers;
- per-iteration bindings.

Your target architecture:

```text
Environment
    ↓
Binding table
    ↓
Binding state machine
    ↓
Resolution
    ↓
Read / initialize / write
```

---

# 75. Production-Grade Exercise

Design the binding subsystem so that it provides:

- deterministic errors;
- source-location tracking;
- environment IDs;
- scope IDs;
- declaration kind;
- initialization state;
- binding owner;
- shadowing information;
- debugging traces.

Example debug event:

```text
READ identifier=x
scope=17
resolved_binding=42
kind=let
initialized=false
result=ReferenceError
```

This is much closer to how a useful language-analysis tool would represent the problem.

---

# 76. Debugging Methodology

When you see:

```text
ReferenceError: Cannot access 'x' before initialization
```

do not immediately move the declaration.

Ask these questions.

### Step 1

What is the identifier?

```text
x
```

### Step 2

Where is the nearest lexical declaration?

```text
let x
```

### Step 3

Does it shadow an outer `x`?

```text
yes / no
```

### Step 4

Has that binding been initialized at the point of access?

```text
no
```

### Step 5

Why is the access occurring earlier?

Possible causes:

- source ordering;
- default parameter evaluation;
- cyclic module dependency;
- closure invoked too early;
- class initialization timing;
- block shadowing.

This converts the error from a mystery into a resolution problem.

---

# 77. Debugging Exercise 1

Predict first:

```js
console.log(a);
var a = 10;
```

Answer:

```text
undefined
```

Trace:

```text
a created
a initialized undefined
read a → undefined
a assigned 10
```

---

# 78. Debugging Exercise 2

```js
console.log(a);
let a = 10;
```

Prediction:

```text
ReferenceError
```

Reason:

```text
a binding exists
a uninitialized
read prohibited
```

---

# 79. Debugging Exercise 3

```js
{
  const x = "outer";

  {
    console.log(x);
    const x = "inner";
  }
}
```

Prediction:

```text
ReferenceError
```

Reason:

```text
inner lexical binding shadows outer binding
inner binding uninitialized
read → ReferenceError
```

---

# 80. Debugging Exercise 4

```js
console.log(typeof missing);
```

Prediction:

```text
"undefined"
```

Now:

```js
console.log(typeof missing);
let missing = 1;
```

At the point of the `typeof`, `missing` is a TDZ binding.

Prediction:

```text
ReferenceError
```

---

# 81. Debugging Exercise 5

```js
sayHi();

const sayHi = () => "hi";
```

Prediction:

```text
ReferenceError
```

Why not `TypeError`?

Because:

```text
sayHi
→ lexical binding
→ uninitialized
```

The engine cannot obtain the function value at all.

---

# 82. Debugging Exercise 6

```js
sayHi();

var sayHi = () => "hi";
```

Prediction:

```text
TypeError
```

Why?

```text
sayHi
→ initialized undefined
→ call attempted
→ undefined is not callable
```

---

# 83. Debugging Exercise 7

```js
const x = 1;

{
  console.log(x);
  let x = 2;
}
```

Prediction:

```text
ReferenceError
```

Reason:

```text
inner x shadows outer x
inner x is uninitialized
```

---

# 84. Code Review Exercise

Review:

```js
function process() {
  console.log(config);

  const local = build();
  let config = loadConfig();

  return local;
}
```

Problems:

1. `config` is read before initialization.
2. The declaration appears far from the intended use.
3. The variable ordering obscures initialization dependencies.

A clearer design is:

```js
function process() {
  const config = loadConfig();
  const local = build(config);

  return local;
}
```

The second version makes initialization and dependency flow obvious.

---

# 85. Code Review Exercise: Legacy `var`

Review:

```js
function handle(request) {
  if (request.admin) {
    var role = "admin";
  }

  if (role === "admin") {
    grantAccess();
  }
}
```

The code may work because `var` is function-scoped.

But that is part of the problem.

A more explicit design:

```js
function handle(request) {
  const role = request.admin ? "admin" : "user";

  if (role === "admin") {
    grantAccess();
  }
}
```

The improvement is not merely stylistic.

It makes state:

- initialized once;
- block-independent;
- easier to reason about;
- less prone to accidental reuse.

---

# 86. Interview Questions

## Beginner

1. What is hoisting?
2. Is `let` hoisted?
3. What is the TDZ?
4. Why does `var` return `undefined` before declaration?
5. Why does `let` throw before declaration?
6. Are function declarations hoisted?
7. Are arrow functions hoisted?

## Intermediate

8. Explain `var` function scope versus `let` block scope.
9. Why does `typeof` throw for a TDZ binding?
10. Why can `var` be redeclared?
11. Why are `let` and `const` redeclarations errors?
12. Explain function declaration versus function expression hoisting.
13. Why can a shadowed variable produce a TDZ error?

## Advanced

14. What is declaration instantiation?
15. What environment records participate in global declaration behavior?
16. Why are parameter defaults relevant to TDZ reasoning?
17. Why do `for (let ...)` loops have per-iteration bindings?
18. How do module bindings differ from ordinary local bindings?
19. How can cyclic module dependencies expose uninitialized bindings?
20. What role do `eval` and `with` play in complicating static scope analysis?

## Principal

21. Explain why “hoisting” is an incomplete architectural model.
22. How does lexical declaration semantics improve static tooling?
23. What bugs does TDZ prevent compared with `undefined`-initialized bindings?
24. How would you represent binding states in a debugger or language server?
25. How would you explain declaration instantiation to an engineer who only knows runtime-level folklore?
26. How would you distinguish ECMAScript guarantees from V8 implementation details when discussing hoisting?

---

# 87. Predict-the-Output Exercises

Predict before reading the answer.

---

## Exercise A

```js
console.log(a);
var a = 1;
```

---

## Exercise B

```js
console.log(a);
let a = 1;
```

---

## Exercise C

```js
console.log(a);
const a = 1;
```

---

## Exercise D

```js
hello();

function hello() {
  console.log("hello");
}
```

---

## Exercise E

```js
hello();

var hello = function () {
  console.log("hello");
};
```

---

## Exercise F

```js
hello();

let hello = function () {
  console.log("hello");
};
```

---

## Exercise G

```js
console.log(typeof x);
let x = 10;
```

---

## Exercise H

```js
console.log(typeof x);
```

---

## Exercise I

```js
const x = "outer";

{
  console.log(x);
  const x = "inner";
}
```

---

## Exercise J

```js
function f(a = b, b = 10) {
  return a + b;
}

console.log(f());
```

---

## Exercise K

```js
function f(a = 10, b = a) {
  return a + b;
}

console.log(f());
```

---

## Exercise L

```js
const callbacks = [];

for (let i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

---

## Exercise M

```js
const callbacks = [];

for (var i = 0; i < 3; i++) {
  callbacks.push(() => i);
}

console.log(callbacks.map(fn => fn()));
```

---

# 88. Mastery Exercises

## Level 1 — Understand

Explain, without using the word “hoisting”:

```js
console.log(x);
let x = 1;
```

---

## Level 2 — Explain

Draw an environment table for:

```js
var a = 1;
let b = 2;
const c = 3;
```

at:

```text
before execution
during execution
after initialization
```

---

## Level 3 — Predict

Given ten mixed examples, predict:

```text
value
undefined
ReferenceError
TypeError
SyntaxError
```

before running them.

---

## Level 4 — Implement

Implement the mini binding simulator from Section 73.

---

## Level 5 — Debug

Given a project containing:

```text
ReferenceError: Cannot access 'config' before initialization
```

trace:

```text
declaration
scope
shadowing
evaluation order
module edges
```

and identify the exact binding that is uninitialized.

---

## Level 6 — Defend

Defend this statement:

> `let` is hoisted, but not initialized.

Do not use folklore. Use:

```text
binding creation
lexical environment
initialization
read
ReferenceError
```

---

# 89. Principal-Level Reasoning Problems

## Problem 1 — Diagnose a false mental model

An engineer says:

> “The browser sees `let`, so it simply refuses to create it until execution reaches the declaration.”

Explain why this is incomplete.

Your explanation should mention:

```text
lexical environment
binding creation
uninitialized state
identifier resolution
initialization timing
```

---

## Problem 2 — Explain an error without naming “TDZ”

Explain:

```js
console.log(config);

const config = loadConfig();
```

without using the phrase “Temporal Dead Zone.”

A strong answer should say:

```text
The lexical environment contains a binding for `config`.
At the point of access, that binding has not yet been initialized.
Identifier resolution finds the binding, but retrieving its value is invalid.
Therefore a ReferenceError is thrown.
```

---

## Problem 3 — Compare two APIs

Why is this:

```js
var state;
```

less protective against accidental early reads than:

```js
let state;
```

before its declaration executes?

Reason in terms of failure mode:

```text
undefined
vs
immediate ReferenceError
```

---

## Problem 4 — Module debugging

Given a cyclic import graph, determine:

```text
which module evaluates first
which binding is initialized
which read occurs too early
why the error is about initialization rather than missing name resolution
```

This prepares you for Chapter 64.

---

# 90. Concept Connections

## Depends On

- Chapter 01 — JavaScript, ECMAScript, and the Runtime Landscape
- Chapter 05 — Variables, Declarations, and Assignment
- Chapter 07 — Type Conversion, Coercion, and Equality
- Chapter 08 — Control Flow and Iteration
- Chapter 09 — Functions and First-Class Behavior
- Chapter 10 — Scope, Lexical Environments, and Identifier Resolution

---

## Builds Toward

- Chapter 12 — Execution Contexts and the Execution Model
- Chapter 13 — Closures
- Chapter 14 — `this`, Invocation, and Binding
- Chapter 18 — Classes and Object-Oriented JavaScript
- Chapter 44 — Realms, Agents, and Execution Isolation
- Chapter 64 — ES Modules
- Chapter 65 — CommonJS and Interoperability
- Chapter 88 — Debugging Methodology
- Chapter 95 — Legacy JavaScript

---

## Related Concepts

- lexical environments;
- environment records;
- binding state;
- shadowing;
- declaration instantiation;
- function execution;
- module linking;
- live bindings;
- default parameter initialization;
- per-iteration environments.

---

## Concepts Revisited

Chapter 10 established:

```text
scope determines which binding is found
```

Chapter 11 adds:

```text
binding state determines whether the found binding may be read
```

Therefore:

```text
identifier resolution
+
initialization state
=
observable access behavior
```

---

## Why This Chapter Matters Later

Without this chapter, later topics become confusing:

```text
closures
modules
classes
execution contexts
parameter environments
cyclic dependencies
debugging ReferenceErrors
```

With this chapter, the phrase:

```text
Cannot access 'x' before initialization
```

stops being folklore and becomes a precise state-machine problem.

---

# 91. Completion Criteria

### Understand

- Define hoisting as a teaching abstraction.
- Define declaration instantiation.
- Define TDZ.
- Distinguish declaration from initialization.
- Distinguish initialization from assignment.

### Explain

- Explain `var`.
- Explain `let`.
- Explain `const`.
- Explain classes.
- Explain function declarations.
- Explain function expressions.
- Explain arrow functions.
- Explain `typeof` and TDZ.
- Explain redeclaration rules.
- Explain shadowing + TDZ.
- Explain parameter-default ordering.

### Predict

You can correctly predict the result of:

- `var` before declaration;
- `let` before declaration;
- `const` before declaration;
- function declaration before use;
- function expression before use;
- arrow function before use;
- class before declaration;
- `typeof` with undeclared identifiers;
- `typeof` with TDZ bindings;
- default parameter dependencies;
- `for (let)` closure behavior.

### Implement

You can implement a binding-state simulator that supports:

```text
create
initialize
read
write
shadow
```

and correctly models:

```text
var
let
const
```

### Debug

You can diagnose a `ReferenceError` by tracing:

```text
identifier
→ scope
→ binding
→ shadowing
→ initialization state
→ evaluation order
```

### Interview

You can explain the topic without relying on:

```text
"JavaScript moves declarations to the top."
```

Instead, you use:

```text
binding creation
lexical environment
initialization
declaration instantiation
TDZ
execution order
```

### Principal Judgment

You can determine when:

- `var` is legacy compatibility rather than a preferred default;
- `let` is appropriate because reassignment is intentional;
- `const` improves invariants;
- TDZ provides a useful fail-fast property;
- modules are preferable to global state;
- dynamic constructs make scope analysis harder;
- a bug is actually a scope-resolution problem rather than a timing problem.

---

# 92. Key Takeaways

1. **Hoisting is a teaching metaphor, not literal source-code movement.**
2. **Declaration, binding creation, initialization, and assignment are separate concepts.**
3. **`var` bindings are initialized to `undefined` during relevant declaration setup.**
4. **`let`, `const`, and `class` bindings can exist before initialization.**
5. **The period before lexical initialization is the Temporal Dead Zone.**
6. **TDZ means the binding exists but cannot yet be read.**
7. **`typeof` is not universally safe; it throws for TDZ bindings.**
8. **Function declarations and function expressions follow different declaration semantics.**
9. **Arrow functions are expressions and do not receive function-declaration instantiation behavior.**
10. **Shadowing can cause a TDZ error even when an outer variable has already been initialized.**
11. **`var` redeclaration differs from lexical redeclaration.**
12. **Default parameter initialization order can expose uninitialized bindings.**
13. **Per-iteration lexical environments explain `for (let)` closure behavior.**
14. **Scripts and modules have different top-level declaration models.**
15. **`eval` and `with` complicate declaration and scope reasoning.**
16. **ECMAScript describes context-specific declaration instantiation rather than one universal “hoisting phase.”**
17. **A reliable debugging method is: resolve the identifier, locate its binding, then inspect initialization state.**
18. **TDZ is not merely a quirk; it provides fail-fast protection against certain accidental early reads.**
19. **Language-level environments are semantic models, not necessarily literal heap objects.**
20. **Principal-level JavaScript reasoning replaces folklore with binding, environment, and execution models.**

---

# 93. Final Mastery Drill

Without running the code, explain each program completely.

### Drill 1

```js
console.log(a);
var a = 10;
```

### Drill 2

```js
console.log(a);
let a = 10;
```

### Drill 3

```js
foo();

function foo() {
  return "ok";
}
```

### Drill 4

```js
foo();

var foo = function () {
  return "ok";
};
```

### Drill 5

```js
foo();

const foo = () => "ok";
```

### Drill 6

```js
const x = "outer";

{
  console.log(x);
  const x = "inner";
}
```

### Drill 7

```js
console.log(typeof x);
let x = 10;
```

### Drill 8

```js
function f(a = b, b = 10) {
  return a + b;
}

f();
```

### Drill 9

```js
function f(a = 10, b = a) {
  return a + b;
}

f();
```

### Drill 10

```js
const fns = [];

for (let i = 0; i < 3; i++) {
  fns.push(() => i);
}

console.log(fns.map(fn => fn()));
```

For every answer, use this exact reasoning order:

```text
1. What declarations exist?
2. Which environment owns each binding?
3. Which bindings are created during setup?
4. Which are initialized immediately?
5. Which remain uninitialized?
6. Which identifier does each read resolve to?
7. What is the binding state at that exact moment?
8. What result or error follows?
9. Why does that behavior matter for production code?
```

---

# 94. Chapter Completion Record

**Chapter:** 11 — Hoisting and the Temporal Dead Zone

**Status:** `[+] Completed`

**Strong Areas Expected:**

- declaration versus initialization;
- `var` initialization;
- lexical TDZ;
- function declaration instantiation;
- class TDZ;
- function expression behavior;
- `typeof` edge case;
- shadowing + TDZ;
- default parameter ordering.

**Revision Triggers:**

- confusing binding existence with binding initialization;
- using “not hoisted” as the explanation for lexical declarations;
- confusing `undefined` with an uninitialized binding;
- incorrectly predicting function-expression calls;
- forgetting that shadowing can produce TDZ errors;
- treating module bindings as ordinary copied values.

**Evidence of Mastery:**

- predict at least 90% of output exercises without execution;
- explain TDZ using binding state rather than folklore;
- implement the declaration-state simulator;
- debug at least three real-world early-initialization failures;
- explain script/module differences at a conceptual level;
- defend declaration choices using scope, mutability, initialization, and production trade-offs.

---

# 95. Transition to Chapter 12

Chapter 11 explains **how declarations and bindings become available**.

Chapter 12 will move one level deeper:

```text
How does JavaScript represent the currently executing code?
How is an execution context entered?
What state exists while a function runs?
How do lexical environments, variable environments, and execution contexts interact?
What changes on function calls?
What happens during nested execution?
How does this connect to the call stack?
```

The next conceptual layer is:

```text
declarations
   ↓
bindings
   ↓
lexical environments
   ↓
execution contexts
   ↓
call stack
   ↓
runtime execution
```

That is the bridge from language-level declaration semantics to the execution model of JavaScript engines.

