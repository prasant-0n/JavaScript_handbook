
# Chapter 22 — Arrays

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** Array semantics, indexed properties, length, holes, mutation, iteration, higher-order methods, array-like objects, and engine-aware array design.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what a JavaScript Array is and what it is not.
2. Explain why Arrays are objects with special indexed-property and length semantics.
3. Explain the relationship between:
   - array indices,
   - String property keys,
   - `length`,
   - own properties,
   - holes.
4. Distinguish:
   - dense arrays,
   - sparse arrays,
   - array-like objects.
5. Explain why an Array can contain mixed value types.
6. Explain the special role of the `length` property.
7. Explain how writing an index can change `length`.
8. Explain how shortening `length` deletes indexed elements.
9. Distinguish:
   - deleting an array element,
   - assigning `undefined`,
   - creating a hole.
10. Explain what an array hole is.
11. Predict how holes affect:
   - `forEach`
   - `map`
   - `filter`
   - `reduce`
   - `find`
   - `findIndex`
   - `some`
   - `every`
   - `includes`
   - `indexOf`
   - `for...of`
   - spread
   - `Array.from`
   - `JSON.stringify`
12. Explain which array methods mutate and which return new arrays.
13. Explain:
   - `push`
   - `pop`
   - `shift`
   - `unshift`
   - `splice`
   - `slice`
   - `concat`
   - `copyWithin`
   - `fill`
   - `reverse`
   - `sort`
   - `toReversed`
   - `toSorted`
   - `toSpliced`
   - `with`
14. Explain the difference between:
   - `slice`
   - `splice`
   - `toSpliced`
15. Explain the difference between:
   - `sort`
   - `toSorted`
16. Explain how Array iteration methods handle holes.
17. Explain the difference between:
   - indexed access,
   - iteration,
   - enumeration,
   - method callbacks.
18. Explain array methods that accept array-like objects.
19. Explain `Array.isArray`.
20. Explain `Array.from`.
21. Explain `Array.of`.
22. Explain the behavior of `new Array(...)`.
23. Explain why:
   ```js
   new Array(3)
   ```
   is not equivalent to:
   ```js
   [undefined, undefined, undefined]
   ```
24. Explain `Symbol.isConcatSpreadable` as it relates to Array concatenation.
25. Explain `Symbol.species` where relevant to Array result construction.
26. Understand Array subclassing.
27. Explain how sparse arrays can affect correctness and performance.
28. Explain practical engine concerns without treating V8 implementation details as universal guarantees.
29. Implement selected Array methods from scratch.
30. Debug mutation, aliasing, and sparse-array bugs.
31. Compare Arrays with:
   - objects,
   - Maps,
   - Sets,
   - typed arrays,
   - linked structures.
32. Select the appropriate collection abstraction for production systems.
33. Reason about array performance, memory, and algorithmic complexity.
34. Design safe immutable update patterns.
35. Defend Array design choices at senior/principal engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 02 — Values, Types, Type System
- Chapter 06 — Operators, Expressions
- Chapter 07 — Type Conversion, Coercion, Equality
- Chapter 08 — Control Flow, Iteration
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 21 — Species / Subclassing / Derived Constructors

---

## 3. What Is It?

An Array is an ECMAScript object with specialized behavior for integer-index-like property keys and a special `length` property.

```js
const values = [10, 20, 30];
```

Conceptually, it contains:

```text
index 0 → 10
index 1 → 20
index 2 → 30
length → 3
```

But an Array is still an object.

```js
typeof values; // "object"
```

It can have ordinary named properties:

```js
values.label = "scores";
```

and Symbol-keyed properties:

```js
const metadata = Symbol("metadata");
values[metadata] = { source: "api" };
```

### Critical mental model

An Array is not fundamentally a magical contiguous memory block exposed directly to JavaScript.

At the language level, an Array is an object whose internal semantics give special treatment to array-index-like properties and `length`.

A JavaScript engine may optimize common Array layouts, but the ECMAScript language model is defined independently of a particular implementation strategy.

---

## 4. Why Does It Exist?

Applications frequently need ordered collections.

Common requirements include:

- preserving sequence,
- indexed access,
- iteration,
- adding/removing at the end,
- transforming collections,
- sorting,
- filtering,
- searching.

Arrays provide a standardized abstraction for these operations.

But JavaScript arrays must support much more than simple contiguous integer storage.

They can also be:

```js
const arr = [];
arr[1000000] = "x";
```

or:

```js
const arr = [];
delete arr[0];
```

or:

```js
arr.foo = "bar";
```

or:

```js
arr[Symbol("x")] = true;
```

Therefore the actual language abstraction is more general than:

> "a fixed block of sequential memory."

---

## 5. Mental Model

Use four layers.

### Layer 1 — Ordered indexed object

```text
Array
 ├── "0" → value
 ├── "1" → value
 ├── "2" → value
 └── "length" → 3
```

### Layer 2 — Index semantics

An index-like property such as:

```js
arr[2]
```

is a property access using a String property key derived from the numeric index expression.

### Layer 3 — `length`

`length` tracks a range boundary for indexed properties.

It is not simply:

```text
number of stored values
```

A sparse array can have:

```js
const a = [];
a[10] = "x";

console.log(a.length); // 11
```

while only one indexed property actually exists.

### Layer 4 — methods/protocols

Array methods provide standardized transformations and mutations:

```text
push/pop
map/filter/reduce
slice/splice
sort
iteration
search
copy/update
```

The same Array object therefore combines:

```text
indexed properties
+
length semantics
+
iteration
+
higher-order methods
+
mutation APIs
+
construction/subclassing behavior
```

---

## 6. Core Rules

### Rule 1 — Arrays are objects

```js
Array.isArray([]); // true
typeof [];         // "object"
```

---

### Rule 2 — Array elements are properties

```js
const arr = ["a"];

arr[0] === arr["0"]; // true
```

The index property is represented using a property key.

---

### Rule 3 — `length` is special

```js
const arr = [];

arr[3] = "x";

console.log(arr.length); // 4
```

---

### Rule 4 — `length` is not the count of existing elements

```js
const arr = [];

arr[3] = "x";

console.log(arr.length); // 4
console.log(Object.keys(arr)); // ["3"]
```

There are four index positions in the range `0..3`, but only one own indexed property.

---

### Rule 5 — A hole is absence, not `undefined`

```js
const a = Array(1);
const b = [undefined];

console.log(0 in a); // false
console.log(0 in b); // true
```

---

### Rule 6 — `delete` can create holes

```js
const arr = [1, 2, 3];

delete arr[1];

console.log(arr.length); // 3
```

The index disappears; `length` does not shrink.

---

### Rule 7 — Assigning `undefined` does not create a hole

```js
const arr = [1, 2, 3];

arr[1] = undefined;

console.log(1 in arr); // true
```

---

### Rule 8 — Increasing `length` can create empty slots without creating properties

```js
const arr = [1];

arr.length = 4;

console.log(arr.length); // 4
console.log(Object.keys(arr)); // ["0"]
```

---

### Rule 9 — Decreasing `length` deletes indexed properties at or above the new length

```js
const arr = [1, 2, 3];

arr.length = 1;

console.log(arr); // [1]
```

---

### Rule 10 — Array methods differ in their hole behavior

Do not memorize:

> "Array methods skip empty values."

Some skip holes, some observe absence differently, and some materialize values in other ways.

You must know the specific algorithm.

---

### Rule 11 — Mutating and non-mutating methods must be distinguished

Examples that mutate:

```js
push
pop
shift
unshift
splice
sort
reverse
fill
copyWithin
```

Modern non-mutating change-oriented methods include:

```js
toSorted
toReversed
toSpliced
with
```

---

### Rule 12 — `map` preserves holes

For:

```js
const arr = [1, , 3];
const result = arr.map(x => x * 2);
```

the result is sparse in the corresponding position.

---

### Rule 13 — `for...of` reads values by indexed iteration semantics and can produce `undefined` for holes

This differs from callback-based methods such as `forEach`.

---

### Rule 14 — Array indices are not arbitrary Numbers

```js
arr[1.5] = "x";
```

does not create array index `1.5`.

It creates an ordinary String-keyed property:

```js
arr["1.5"]
```

and does not update `length` as an indexed element would.

---

### Rule 15 — `Array.isArray` is the reliable language-level Array test

Prefer:

```js
Array.isArray(value)
```

over prototype comparisons.

---

## 7. Syntax

### Literals

```js
[]
[1, 2, 3]
["a", 1, true]
```

### Constructor forms

```js
new Array()
new Array(3)
new Array(1, 2, 3)
```

### Static methods

```js
Array.isArray(value)
Array.from(iterableOrArrayLike)
Array.of(1, 2, 3)
```

### Access

```js
arr[0]
arr.at(0)
arr.at(-1)
```

### Mutation

```js
arr.push(value)
arr.pop()
arr.shift()
arr.unshift(value)
arr.splice(start, deleteCount, ...items)
arr.sort(compareFn)
arr.reverse()
arr.fill(value)
arr.copyWithin(target, start, end)
```

### Non-mutating transforms

```js
arr.map(fn)
arr.filter(fn)
arr.slice(start, end)
arr.concat(...)
arr.toSorted(compareFn)
arr.toReversed()
arr.toSpliced(...)
arr.with(index, value)
```

### Search

```js
arr.includes(value)
arr.indexOf(value)
arr.lastIndexOf(value)
arr.find(fn)
arr.findIndex(fn)
arr.findLast(fn)
arr.findLastIndex(fn)
```

### Iteration

```js
arr.forEach(fn)

for (const value of arr) {
}
```

---

## 8. Basic Examples

### Example 1 — Indexed access

```js
const numbers = [10, 20, 30];

console.log(numbers[0]);
console.log(numbers[2]);
```

Output:

```text
10
30
```

---

### Example 2 — `length`

```js
const arr = [];

arr[4] = "x";

console.log(arr.length);
console.log(Object.keys(arr));
```

Output:

```text
5
["4"]
```

---

### Example 3 — Hole versus undefined

```js
const a = [1, , 3];
const b = [1, undefined, 3];

console.log(1 in a);
console.log(1 in b);
```

Output:

```text
false
true
```

---

### Example 4 — Mutation versus copying

```js
const original = [3, 1, 2];

const sorted = original.toSorted();

console.log(original);
console.log(sorted);
```

Output:

```text
[3, 1, 2]
[1, 2, 3]
```

---

### Example 5 — `slice` versus `splice`

```js
const a = [1, 2, 3];

const copied = a.slice(1);

console.log(a);
console.log(copied);
```

`slice` creates a new Array without changing the source.

Now:

```js
const b = [1, 2, 3];

const removed = b.splice(1, 1);

console.log(b);
console.log(removed);
```

`splice` mutates the source and returns removed elements.

---

### Example 6 — `Array(3)`

```js
const a = new Array(3);

console.log(a.length);
console.log(Object.keys(a));
```

Result:

```text
3
[]
```

There are no own indexed properties.

---

## 9. Execution Walkthrough

Consider:

```js
const arr = [10, 20];

arr[2] = 30;
```

### Step 1 — Array literal construction

An Array object is created.

Indexed properties are established:

```text
"0" → 10
"1" → 20
length → 2
```

### Step 2 — Evaluate `arr[2]`

The property key is effectively the String `"2"`.

### Step 3 — Assign `30`

The Array's specialized `[[Set]]` semantics recognize the property as an array index.

### Step 4 — Update `length`

Because the new index is beyond the previous last index, `length` becomes:

```text
3
```

### Step 5 — Final conceptual state

```text
"0" → 10
"1" → 20
"2" → 30
length → 3
```

---

### Hole example

```js
const arr = [10, 20, 30];

delete arr[1];
```

Initial:

```text
0 → 10
1 → 20
2 → 30
length → 3
```

After deletion:

```text
0 → 10
1 → absent
2 → 30
length → 3
```

This is sparse state.

---

## 10. Internal Mechanics

### 10.1 Arrays are exotic objects

At the specification level, Arrays are not merely ordinary objects.

They have specialized Array exotic object behavior.

In particular, the language defines special semantics for:

- indexed properties,
- `length`,
- defining/deleting indexed properties.

The key internal mechanism to study is the Array object's specialized behavior around:

```text
[[DefineOwnProperty]]
```

---

### 10.2 Array index concept

A property key can represent an array index if it satisfies the relevant Array-index criteria.

At a high level:

```text
"0"
"1"
"2"
...
```

can be array-index-like.

But:

```text
"01"
"-1"
"1.5"
"foo"
```

do not act as ordinary array indices in the same way.

This is why:

```js
arr["01"] = "x";
```

does not increase `length` like:

```js
arr[1] = "x";
```

---

### 10.3 `length` as a special property

Array `length` is itself a property with special semantics.

It has a numeric value reflecting the array's indexed range.

It is also non-enumerable.

```js
Object.keys([1, 2, 3]);
// ["0", "1", "2"]
```

`length` is absent from that result.

---

### 10.4 Writing `length`

Increasing:

```js
arr.length = 10;
```

changes the array's length boundary.

It does not necessarily create ten indexed properties.

Decreasing:

```js
arr.length = 1;
```

requires removal of indexed properties that are no longer inside the permitted range.

---

### 10.5 Non-configurable elements can complicate shrinking

Consider:

```js
const arr = [1, 2, 3];

Object.defineProperty(arr, "2", {
  configurable: false,
});

arr.length = 1;
```

The requested shrink encounters a property that cannot be deleted.

Array length semantics therefore include failure/rollback behavior rather than blindly deleting everything.

This is an important example of how:

```text
array length
+
property descriptor invariants
```

interact.

---

### 10.6 Holes and property existence

For:

```js
const arr = [1, , 3];
```

the index:

```js
1
```

is absent.

That means:

```js
1 in arr
```

is false.

But:

```js
arr[1]
```

still evaluates to:

```js
undefined
```

because ordinary property access returns `undefined` when a property is not found.

This is why:

```text
absence
```

and:

```text
undefined value
```

must not be confused.

---

### 10.7 Array methods use property existence differently

Callback-oriented array methods often determine whether an index exists before invoking the callback.

For:

```js
[1, , 3].map(fn)
```

the callback is not called for the missing index.

Other operations that obtain values through indexed iteration can observe:

```text
undefined
```

instead.

This difference is fundamental.

---

### 10.8 `Array.prototype.map`

Conceptually:

```text
for each index from 0 to length - 1
    if property exists
        call callback
        create result at that index
```

Thus holes remain holes in the mapped result.

---

### 10.9 `for...of`

Array iteration uses the iterable protocol.

The Array iterator retrieves values by index progression.

For a hole:

```js
const arr = [,];

for (const value of arr) {
  console.log(value);
}
```

the iteration can yield:

```text
undefined
```

The iterator does not simply behave like `forEach`.

---

### 10.10 `for...in`

`for...in` enumerates eligible String-keyed properties according to its own semantics.

It is not an Array-element iterator.

For example:

```js
const arr = [1, , 3];

for (const key in arr) {
  console.log(key);
}
```

The hole is absent and therefore not enumerated.

---

### 10.11 `includes` versus `indexOf`

This classic distinction matters:

```js
[NaN].includes(NaN); // true
[NaN].indexOf(NaN);  // -1
```

They use different equality semantics.

`includes` uses SameValueZero-style comparison.

`indexOf` uses strict-equality-style matching.

---

### 10.12 `find` and holes

Array search methods have historically had subtle differences in how they handle missing elements.

Do not assume:

```text
all callbacks behave exactly like map
```

Read the actual algorithm for the method under study.

For modern ECMAScript, methods such as `find` and `findIndex` have defined behavior that should be tested explicitly when sparse arrays matter.

The production recommendation remains:

> Avoid relying on sparse-array behavior as an implicit control-flow mechanism.

---

### 10.13 Array subclass construction

Array result-producing methods can interact with:

```js
Symbol.species
```

when the specific method's algorithm uses species construction.

This connects directly to Chapter 21.

Do not assume all Array methods use species.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Array exotic objects

The ECMAScript specification models Arrays as exotic objects with special internal method behavior.

The key specialized operation is:

```text
[[DefineOwnProperty]]
```

because defining an indexed property may require updating `length`.

---

### 11.2 `ArrayCreate`

Specification-level construction includes the conceptual abstract operation:

```text
ArrayCreate(length)
```

This creates an Array with the requested initial length.

The indexed properties are not necessarily populated simply because the length is nonzero.

That explains:

```js
new Array(3)
```

as a sparse array of length three.

---

### 11.3 `ArraySetLength`

The specification includes dedicated logic for setting an Array's `length`.

Conceptually:

```text
newLen >= oldLen
    → update length

newLen < oldLen
    → delete indexed properties >= newLen
    → fail if required deletions are prohibited
```

This is more precise than:

> "length just tells you how many elements there are."

---

### 11.4 Canonical numeric/index reasoning

Array-index property recognition is based on specific String-index criteria rather than all numeric-looking values.

This distinction matters when working with:

```js
arr["01"]
arr["1.0"]
arr["-0"]
arr["4294967294"]
arr["4294967295"]
```

A principal-level engineer should understand that the Array index domain is tightly specified rather than simply "non-negative numbers."

---

### 11.5 `LengthOfArrayLike`

Many built-in methods operate on array-like objects using the conceptual abstract operation:

```text
LengthOfArrayLike
```

This supports patterns such as:

```js
const arrayLike = {
  0: "a",
  1: "b",
  length: 2,
};

const result = Array.from(arrayLike);
```

This demonstrates an important design principle:

> Some Array algorithms are generic over array-like inputs instead of requiring actual Arrays.

---

### 11.6 `IsArray`

The specification provides Array detection semantics used by:

```js
Array.isArray(value)
```

This is more precise than checking a prototype chain.

A Proxy wrapping an Array is also treated according to the language's `IsArray` semantics.

---

### 11.7 `ArrayFrom`

`Array.from` accepts:

- iterable inputs,
- array-like inputs.

It therefore sits at the intersection of:

```text
iteration protocol
```

and:

```text
indexed array-like access
```

---

### 11.8 `ArraySpeciesCreate`

Certain Array methods use the conceptual:

```text
ArraySpeciesCreate
```

mechanism.

This connects Array methods to:

```js
Symbol.species
```

and subclassing.

The exact method must be checked before assuming species behavior.

---

### 11.9 Array iterator

Arrays expose an iterator through:

```js
Array.prototype[Symbol.iterator]
```

which is the same function as:

```js
Array.prototype.values
```

for ordinary Array behavior.

This connects Arrays to the broader iterable protocol from Chapter 20.

---

## 12. Advanced Behavior

### 12.1 `new Array(3)` versus `[undefined, undefined, undefined]`

```js
const a = new Array(3);
const b = [undefined, undefined, undefined];

console.log(a.length); // 3
console.log(b.length); // 3

console.log(0 in a); // false
console.log(0 in b); // true
```

This distinction affects iteration methods, serialization, reflection, and copying.

---

### 12.2 `Array.from` can materialize holes

Consider:

```js
const sparse = new Array(3);

const copy = Array.from(sparse);
```

`Array.from` consumes the array's indexed/iterable behavior and can produce an Array containing explicit `undefined` values in positions that were holes in the original.

Check:

```js
0 in sparse;
0 in copy;
```

Do not assume sparse shape is preserved by every copy mechanism.

---

### 12.3 Spread can materialize holes

```js
const sparse = [1, , 3];
const copy = [...sparse];

console.log(copy);
console.log(1 in copy);
```

Because spread consumes the Array iterator, the missing position can become an explicit `undefined` in the new Array.

This is a major distinction:

```text
map → generally preserves holes
spread → iterates values, potentially materializing undefined
```

---

### 12.4 `slice` preserves sparseness

```js
const sparse = [1, , 3];
const copy = sparse.slice();

console.log(1 in copy);
```

The missing indexed property remains missing.

---

### 12.5 `concat` and spreadability

Array concatenation can consult:

```js
Symbol.isConcatSpreadable
```

For ordinary Arrays, the value is spread according to Array concatenation semantics.

An object can participate in the protocol:

```js
const arrayLike = {
  0: "a",
  1: "b",
  length: 2,
  [Symbol.isConcatSpreadable]: true,
};

console.log([].concat(arrayLike));
```

This illustrates protocol-oriented extensibility.

---

### 12.6 `sort` mutates

```js
const values = [3, 1, 2];

const result = values.sort();

console.log(result === values); // true
```

The Array itself is reordered.

---

### 12.7 `toSorted` does not mutate

```js
const values = [3, 1, 2];

const result = values.toSorted();

console.log(result === values); // false
console.log(values);            // [3, 1, 2]
```

This is valuable in state-management and functional-style code.

---

### 12.8 Default `sort` is string-oriented

```js
[10, 2, 1].sort();
```

does not perform numeric ordering by default.

It sorts according to the method's default string conversion/comparison behavior.

Use:

```js
[10, 2, 1].sort((a, b) => a - b);
```

for numeric ascending order.

---

### 12.9 `sort` and `undefined`/holes

Sparse arrays and `undefined` elements have defined but nontrivial sort behavior.

Do not model sort as simply:

```text
convert every element to string
```

There are specific rules for empty slots and `undefined`.

When correctness matters, study the specification algorithm or test the exact case.

---

### 12.10 `with` and immutable indexed replacement

```js
const values = ["a", "b", "c"];

const next = values.with(1, "B");

console.log(values); // ["a", "b", "c"]
console.log(next);   // ["a", "B", "c"]
```

This provides a standard immutable replacement operation for one indexed position.

---

### 12.11 Negative indexing via `at`

```js
const values = ["a", "b", "c"];

values.at(-1); // "c"
```

This is not the same as:

```js
values[-1]
```

because `-1` as a property key is:

```text
"-1"
```

not an Array index from the end.

---

### 12.12 `Array.of`

```js
Array.of(3);
```

produces:

```js
[3]
```

while:

```js
new Array(3);
```

produces an Array of length 3 with no own indexed elements.

This difference is why `Array.of` exists.

---

### 12.13 `Array.from` mapping

```js
const result = Array.from(
  { length: 3 },
  (_, index) => index * 2,
);

console.log(result);
// [0, 2, 4]
```

This can be useful for controlled array construction from array-like data.

---

### 12.14 Array methods are often generic

Some methods are specified in terms of property access and length and can work on array-like objects:

```js
Array.prototype.slice.call(arrayLike);
```

Modern code often prefers clearer alternatives:

```js
Array.from(arrayLike);
```

But knowing genericness is important when reasoning about the platform.

---

### 12.15 Aliasing

```js
const a = [1, 2];
const b = a;

b.push(3);

console.log(a);
```

Output:

```text
[1, 2, 3]
```

Both variables refer to the same Array object.

Copying requires an explicit operation:

```js
const b = [...a];
```

which itself is shallow.

---

### 12.16 Shallow copy

```js
const a = [{ x: 1 }];
const b = [...a];

b[0].x = 2;

console.log(a[0].x); // 2
```

The Array structure is copied, but nested object identity is shared.

---

### 12.17 `structuredClone` versus array spread

```js
const original = [{ x: 1 }];

const shallow = [...original];
const deep = structuredClone(original);
```

Spread copies only the outer Array structure.

`structuredClone` follows a different graph-copying model for supported cloneable values.

---

## 13. Edge Cases

### Edge Case 1 — `-0`

```js
const arr = [];

arr[-0] = "x";

console.log(arr[0]);
console.log(arr["-0"]);
```

Reason carefully about how property-key conversion treats `-0`.

This is a classic test of the distinction between numeric coercion and property-key strings.

---

### Edge Case 2 — Huge indices

```js
const arr = [];

arr[4294967294] = "x";
```

This is near the upper boundary of the Array index range.

Understanding the distinction between:

```text
Array index
```

and:

```text
arbitrary numeric-looking String
```

matters here.

---

### Edge Case 3 — `4294967295`

```js
const arr = [];

arr[4294967295] = "x";
```

This does not behave like an ordinary Array index for `length` purposes.

It becomes an ordinary property key.

---

### Edge Case 4 — Non-index property

```js
const arr = [];

arr.foo = 123;

console.log(arr.length); // 0
```

Arrays can have ordinary properties that are not indexed elements.

---

### Edge Case 5 — Non-integer numeric property

```js
const arr = [];

arr[1.5] = "x";

console.log(arr.length);
```

The `length` does not become `2.5` or otherwise change based on `1.5`.

---

### Edge Case 6 — `delete` versus `splice`

```js
const a = [1, 2, 3];
delete a[1];

const b = [1, 2, 3];
b.splice(1, 1);
```

Both remove the value at index 1, but the resulting structural states differ:

```text
delete → hole remains
splice → later elements shift, length decreases
```

---

### Edge Case 7 — Frozen arrays

```js
const arr = Object.freeze([1, 2, 3]);
```

Mutating operations can fail.

This interacts with:

- strict mode,
- descriptors,
- `length`,
- method semantics.

---

### Edge Case 8 — Sealed arrays

Sealing prevents configuration changes but allows some value writes.

Deletion and length shrinking can therefore behave differently from a normal Array.

---

### Edge Case 9 — Non-writable length

```js
const arr = [1, 2];

Object.defineProperty(arr, "length", {
  writable: false,
});
```

Operations that would change length can fail.

This demonstrates again that Array methods are constrained by property invariants.

---

### Edge Case 10 — Proxy around an Array

A Proxy can intercept property operations around an Array.

This can change observable behavior while still being constrained by proxy invariants.

Array correctness therefore becomes more complex when:

```js
new Proxy(array, traps)
```

is involved.

---

### Edge Case 11 — Array subclass

```js
class MyArray extends Array {}
```

Methods that produce Arrays may interact with:

```js
Symbol.species
```

depending on the method.

---

### Edge Case 12 — Cross-realm Arrays

An Array from another realm may not share the same `Array.prototype`.

Still:

```js
Array.isArray(value);
```

is designed to correctly identify it as an Array.

This is one reason prototype comparisons are weaker.

---

## 14. Common Misconceptions

### Misconception 1 — "Array is a contiguous memory block."

Not at the ECMAScript semantic level.

Engines may use compact representations for common arrays, but the language model permits sparse properties and arbitrary properties.

---

### Misconception 2 — "`length` equals number of elements."

False for sparse arrays.

```js
const arr = [];
arr[100] = "x";

arr.length; // 101
```

---

### Misconception 3 — "A hole is `undefined`."

False.

```text
hole → property absent
undefined → property present with value undefined
```

---

### Misconception 4 — "All array methods skip holes."

False.

Different algorithms treat holes differently.

---

### Misconception 5 — "`delete arr[i]` is equivalent to `splice(i, 1)`."

False.

`delete` creates an absence while preserving length.

---

### Misconception 6 — "`sort()` returns a new Array."

False.

It mutates the receiver and returns that same Array.

---

### Misconception 7 — "Spread creates an exact sparse clone."

False.

Spread consumes the iterable and may materialize `undefined` for holes.

---

### Misconception 8 — "Array indices are all numbers."

At the property-key level, indexed properties are represented by String keys.

---

### Misconception 9 — "`arr[-1]` means last element."

False.

Use:

```js
arr.at(-1)
```

---

### Misconception 10 — "Array methods always require actual Arrays."

Many methods are generic and work with array-like objects.

---

### Misconception 11 — "Array.from only works with Arrays."

False.

It accepts iterables and array-like objects.

---

### Misconception 12 — "`new Array(3)` creates three undefined values."

False.

It creates an Array of length 3 without indexed properties.

---

## 15. Common Mistakes

### Mistake 1 — Using `delete` to remove elements

Prefer:

```js
splice()
```

when you mean to close the gap.

Use `delete` only when creating an actual sparse state is intentional.

---

### Mistake 2 — Mutating input unexpectedly

```js
function sortUsers(users) {
  return users.sort(compareUsers);
}
```

This mutates the caller's Array.

Safer:

```js
return users.toSorted(compareUsers);
```

when immutable semantics are desired.

---

### Mistake 3 — Numeric sort bug

Wrong:

```js
numbers.sort();
```

Correct:

```js
numbers.sort((a, b) => a - b);
```

---

### Mistake 4 — Confusing `slice` and `splice`

Remember:

```text
slice → copy a range
splice → mutate/remove/insert
```

---

### Mistake 5 — Accidental aliasing

```js
const b = a;
```

does not copy.

---

### Mistake 6 — Shallow-copy assumptions

```js
const copy = [...original];
```

does not deep-clone nested objects.

---

### Mistake 7 — Ignoring holes

Sparse input can produce surprising callback counts and iteration results.

---

### Mistake 8 — Using arrays for key-based lookup

If the design is:

```text
key → value
```

and not:

```text
ordered numeric position → value
```

a Map or object may be more appropriate.

---

### Mistake 9 — Using Array as a queue with repeated `shift`

Repeated front removal can create unnecessary work for large queues.

Consider a deque/ring-buffer design or a dedicated queue abstraction.

---

### Mistake 10 — Treating `length` assignment as simple truncation in all cases

Descriptors and non-configurable indexed properties can make length operations fail.

---

## 16. Comparison With Related Concepts

| Structure | Ordered | Indexed | Keyed lookup | Unique values | Sparse semantics | Typical use |
|---|---:|---:|---:|---:|---:|---|
| Array | Yes | Yes | No | No | Yes | Ordered collections |
| Object | Insertion/defined key order | Not fundamentally | Yes | No | N/A | Records/dictionaries |
| Map | Insertion order | No | Yes | No | N/A | Arbitrary key/value |
| Set | Insertion order | No | Membership | Yes | N/A | Unique membership |
| TypedArray | Yes | Yes | Numeric index | No | No holes | Binary/numeric data |
| String | Yes | Code-unit indexed | No | N/A | No holes | Text |
| Queue | Yes | Usually logical | Front/back | Depends | Depends | FIFO |
| Linked list | Yes | Sequential | Usually traversal | No | N/A | Specialized sequence |

### Array versus TypedArray

TypedArrays:

- have fixed element types,
- cannot have holes,
- map to binary memory views,
- have numeric conversion/storage semantics.

They are not simply "faster Arrays."

---

### Array versus Map

Use Array when position and order are primary.

Use Map when key identity and lookup are primary.

---

### Array versus Set

Use Array when duplicates and positional access matter.

Use Set when uniqueness and membership are primary.

---

### Array versus Object

Use Objects for record-like structures:

```js
{
  id: 1,
  name: "Asha",
}
```

Use Arrays for sequences:

```js
[
  user1,
  user2,
  user3,
]
```

---

## 17. Performance Considerations

### 17.1 Big-O basics

Typical conceptual complexities:

| Operation | Typical Array complexity |
|---|---:|
| `arr[i]` | O(1) |
| `push` | Amortized O(1) |
| `pop` | O(1) |
| `shift` | O(n) typical |
| `unshift` | O(n) typical |
| `splice` middle | O(n) typical |
| `slice` length n | O(n) |
| `map` length n | O(n) |
| `filter` length n | O(n) |
| `reduce` length n | O(n) |
| `sort` | O(n log n) typical comparison-sort model |

Exact engine strategies can vary.

---

### 17.2 Dense arrays

Engines often optimize arrays that behave like:

```text
dense
homogeneous-ish
index-oriented
```

Do not convert this into a universal guarantee about "packed arrays."

The language specification does not promise V8's internal element kinds.

---

### 17.3 Sparse arrays

Sparse arrays can be more expensive because an engine may need different internal representations.

Patterns such as:

```js
const a = [];
a[10_000_000] = 1;
```

should not be assumed to allocate a contiguous block of ten million JavaScript values.

But they can still be performance-hostile in operations that iterate the whole logical length.

---

### 17.4 Accidental deoptimization

Patterns that mix:

```text
dense indexing
random named properties
holes
very different value representations
```

can make engine optimization harder.

This is engine-specific and should be measured rather than treated as an absolute rule.

---

### 17.5 Front removal

Avoid:

```js
while (queue.length) {
  queue.shift();
}
```

for large workloads if performance matters.

A logical head index can often reduce repeated shifting:

```js
let head = 0;

while (head < queue.length) {
  const item = queue[head++];
}
```

A production queue may instead use a deque/ring buffer.

---

### 17.6 `map` versus manual loops

Do not assume:

> "for loops are always faster than map."

Performance depends on:

- callback overhead,
- engine optimizations,
- workload,
- data representation,
- surrounding code.

Choose based on clarity first and measure when performance is material.

---

## 18. Memory Considerations

### Array storage is not value copying by default

```js
const a = [{ x: 1 }];
```

The Array stores a reference to the object.

Copying the Array does not clone the object.

---

### Sparse arrays and memory

A sparse Array can have a huge `length` with relatively few own indexed properties.

Memory use and traversal cost therefore need separate analysis.

---

### Capacity is implementation-specific

JavaScript does not expose a standard:

```text
array.capacity
```

property.

Do not write designs that assume a specific hidden capacity behavior.

---

### Retention

An Array strongly retains its referenced elements:

```js
const cache = [];
cache.push(largeObject);
```

As long as the element remains reachable through the Array, the object remains reachable.

For memory-sensitive caches, consider lifecycle-aware structures such as WeakMap where semantics permit.

---

## 19. Security Considerations

### Prototype pollution is not an Array-exclusive problem

Arrays inherit from:

```js
Array.prototype
```

so prototype modification can affect behavior.

Avoid depending on mutable global prototypes in security-sensitive code.

---

### Untrusted indices

When processing external input:

```js
arr[userProvidedIndex] = value;
```

validate expectations around index type and range.

Do not assume arbitrary strings are safe or behave as ordinary indices.

---

### Sparse input abuse

An attacker controlling an index such as:

```js
arr[10 ** 9] = value;
```

can produce enormous logical lengths and potentially trigger expensive downstream operations.

Validate user-provided indices and limits.

---

### Sort comparator execution

The comparator passed to:

```js
sort(compareFn)
```

is user code.

It can:

- throw,
- mutate the Array,
- mutate external state,
- trigger getters,
- interact with proxies.

Do not treat sorting as a pure internal primitive.

---

### Getters and proxies

Array algorithms can trigger user-defined:

- getters,
- setters,
- proxies,
- prototype methods.

This means array operations are not necessarily side-effect free even when they look like simple iteration.

---

## 20. Production Usage

### Use case 1 — Ordered records

```js
const users = [
  { id: 1, name: "Asha" },
  { id: 2, name: "Milan" },
];
```

---

### Use case 2 — Immutable state update

Prefer:

```js
const next = items.with(index, updatedItem);
```

or other copy-based operations when state ownership requires immutability.

---

### Use case 3 — Filtering

```js
const activeUsers = users.filter(user => user.active);
```

Readable and naturally expresses transformation.

---

### Use case 4 — Aggregation

```js
const total = prices.reduce(
  (sum, price) => sum + price,
  0,
);
```

Use `reduce` where it remains clearer than a loop; do not force every algorithm into it.

---

### Use case 5 — Queue

For small queues:

```js
queue.push(item);
const next = queue.shift();
```

For high-throughput queues, use a purpose-built deque or head-index approach.

---

### Use case 6 — Deduplication

```js
const unique = [...new Set(values)];
```

This expresses a Set-based uniqueness operation while returning an Array.

---

### Use case 7 — Array-like conversion

```js
const values = Array.from(arguments);
```

In modern functions, rest parameters are usually clearer, but understanding Array-like conversion remains important.

---

### Production rule

Ask:

```text
Do I need order?
Do I need numeric indexing?
Do I need arbitrary key lookup?
Do I need uniqueness?
Do I need binary storage?
Do I need immutability?
Do I need frequent front operations?
```

Then choose the structure.

---

## 21. Implementation From Scratch

### 21.1 Minimal `map`

A simplified educational implementation:

```js
function miniMap(array, callback, thisArg) {
  if (!Array.isArray(array)) {
    throw new TypeError("Expected an Array");
  }

  const result = new Array(array.length);

  for (let i = 0; i < array.length; i++) {
    if (i in array) {
      result[i] = callback.call(thisArg, array[i], i, array);
    }
  }

  return result;
}
```

This intentionally preserves holes.

---

### 21.2 Minimal `filter`

```js
function miniFilter(array, predicate, thisArg) {
  if (!Array.isArray(array)) {
    throw new TypeError("Expected an Array");
  }

  const result = [];

  for (let i = 0; i < array.length; i++) {
    if (!(i in array)) continue;

    const value = array[i];

    if (predicate.call(thisArg, value, i, array)) {
      result.push(value);
    }
  }

  return result;
}
```

---

### 21.3 Minimal `reduce`

```js
function miniReduce(array, callback, initialValue) {
  if (!Array.isArray(array)) {
    throw new TypeError("Expected an Array");
  }

  let i = 0;
  let accumulator;

  if (arguments.length >= 3) {
    accumulator = initialValue;
  } else {
    while (i < array.length && !(i in array)) {
      i++;
    }

    if (i >= array.length) {
      throw new TypeError("Reduce of empty array with no initial value");
    }

    accumulator = array[i++];
  }

  for (; i < array.length; i++) {
    if (!(i in array)) continue;

    accumulator = callback(
      accumulator,
      array[i],
      i,
      array,
    );
  }

  return accumulator;
}
```

This is educational and does not implement every specification detail.

---

### 21.4 Minimal immutable `toSpliced`

Build an educational version:

```js
function miniToSpliced(array, start, deleteCount, ...items) {
  const copy = array.slice();
  copy.splice(start, deleteCount, ...items);
  return copy;
}
```

Then harden it for:

- negative starts,
- oversized delete counts,
- sparse arrays,
- species,
- subclass behavior.

---

### 21.5 Array method test matrix

Create a test table:

```text
Method
- dense input
- sparse input
- undefined values
- inherited properties
- getters
- frozen array
- proxy
- subclass
- throwing callback
```

Use it to compare:

```text
map
filter
reduce
find
includes
slice
concat
sort
toSorted
splice
toSpliced
```

---

### Implementation progression

**Guided**

- `map`
- `filter`
- `reduce`

**Partially Guided**

- `slice`
- `splice`
- `toSpliced`

**No Reference**

- Build a sparse-array-aware transformation library.

**Edge-Case Hardened**

Support:

- holes,
- inherited properties,
- getters,
- invalid callbacks,
- frozen arrays,
- proxies.

**Production-Grade**

Add:

- conformance tests,
- benchmarks,
- type contracts,
- documentation,
- mutation guarantees,
- compatibility matrix.

---

## 22. Debugging Exercises

### Exercise 1 — Missing callback

```js
const values = [1, , 3];

values.forEach(value => {
  console.log(value);
});
```

Explain why the callback does not run for every logical position.

---

### Exercise 2 — Spread changed the shape

```js
const sparse = [1, , 3];
const copy = [...sparse];

console.log(1 in sparse);
console.log(1 in copy);
```

Explain the difference.

---

### Exercise 3 — Accidental mutation

```js
function topThree(values) {
  return values.sort((a, b) => b - a).slice(0, 3);
}
```

Find the mutation bug.

---

### Exercise 4 — Queue performance

```js
function processAll(queue) {
  while (queue.length > 0) {
    process(queue.shift());
  }
}
```

Explain the likely complexity issue for large queues.

---

### Exercise 5 — Hole versus undefined

```js
const a = new Array(2);
const b = [undefined, undefined];

console.log(JSON.stringify(a));
console.log(JSON.stringify(b));
```

Predict the output and explain why serialization does not preserve the exact distinction you might expect.

---

### Exercise 6 — Length semantics

```js
const arr = [1, 2, 3];

delete arr[1];

console.log(arr.length);
console.log(arr[1]);
console.log(1 in arr);
```

---

### Exercise 7 — Non-index key

```js
const arr = [];

arr["01"] = "x";

console.log(arr.length);
console.log(Object.keys(arr));
```

---

## 23. Code Review Exercise

Review:

```js
function normalizeScores(scores) {
  scores.sort((a, b) => a - b);

  return scores
    .filter(Boolean)
    .map(score => ({
      score,
      normalized: score / 100,
    }));
}
```

### Review questions

1. Does the function mutate caller-owned data?
2. Is `filter(Boolean)` semantically correct for score `0`?
3. Are sparse inputs handled intentionally?
4. Is numeric normalization valid for all possible scores?
5. Is sorting necessary before filtering?
6. What does the function promise about ordering?
7. Should the implementation use `toSorted()`?
8. Is the Array the right abstraction?
9. What are the input validation requirements?
10. How would you test:
   - empty arrays,
   - `0`,
   - `NaN`,
   - `Infinity`,
   - sparse arrays,
   - frozen arrays?

---

## 24. Interview Questions

### Fundamentals

1. What is an Array in JavaScript?
2. Why are Arrays technically objects?
3. What is special about the `length` property?
4. What is an array hole?
5. How is a hole different from `undefined`?
6. What happens when you delete an Array element?
7. What happens when you assign a high index?
8. What happens when you reduce `length`?
9. Why is `new Array(3)` sparse?
10. What is `Array.of` used for?
11. Why use `Array.isArray`?
12. What is `Array.from`?

### Methods

13. Difference between `slice` and `splice`?
14. Difference between `splice` and `toSpliced`?
15. Difference between `sort` and `toSorted`?
16. Which Array methods mutate?
17. Which methods preserve holes?
18. Which operations materialize holes as `undefined`?
19. Difference between `includes` and `indexOf`?
20. Difference between `at(-1)` and `[-1]`?
21. Why does `sort()` give surprising numeric results?

### Specification

22. What is an Array exotic object?
23. What is `ArrayCreate`?
24. What is `ArraySetLength`?
25. What is `LengthOfArrayLike`?
26. What is `ArraySpeciesCreate`?
27. How does an indexed property update `length`?
28. Why does `"01"` not behave like `"1"`?
29. How do non-configurable properties affect length shrinking?
30. Why can Array methods work on array-like objects?

### Runtime/performance

31. Why can sparse Arrays perform differently?
32. Why is repeated `shift()` problematic?
33. Are Arrays contiguous in memory?
34. What are dense/packed versus sparse engine representations?
35. Why should engine-specific optimization claims be benchmarked?

### Principal-level

36. When would you use Array versus Map?
37. When would you use Array versus Set?
38. When would you use Array versus TypedArray?
39. When is a queue abstraction better than an Array?
40. When is an immutable operation preferable to a mutating method?
41. How would you design an API that accepts Arrays without mutating caller state?
42. What invariants should a library maintain around Array inputs?
43. How can malicious indices create resource-exhaustion risk?
44. How should sparse input be handled at API boundaries?
45. When does Array subclassing become a liability?
46. How would you benchmark an Array-heavy workload honestly?
47. How would you review a function containing five chained Array methods?
48. How would you decide between one readable loop and multiple expressive transformations?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
const a = new Array(3);
const b = [undefined, undefined, undefined];

console.log(a.length);
console.log(0 in a);
console.log(0 in b);
```

---

### Exercise B

```js
const arr = [1, , 3];

console.log(arr.map(x => x * 2));
console.log(arr.filter(() => true));
```

Predict the structural differences.

---

### Exercise C

```js
const arr = [1, , 3];

console.log([...arr]);
console.log(Array.from(arr));
```

---

### Exercise D

```js
const arr = [10, 2, 1];

console.log(arr.toSorted());
console.log(arr.sort((a, b) => a - b));
```

Track mutation and ordering.

---

### Exercise E

```js
const arr = [1, 2, 3];

delete arr[1];

console.log(arr.length);
console.log(arr[1]);
console.log(1 in arr);
```

---

### Exercise F

```js
const arr = [];

arr[5] = "x";
arr["01"] = "y";
arr.foo = "z";

console.log(arr.length);
console.log(Object.keys(arr));
```

---

### Exercise G

```js
const arr = [NaN];

console.log(arr.includes(NaN));
console.log(arr.indexOf(NaN));
```

---

### Exercise H

```js
const arr = ["a", "b", "c"];

console.log(arr.at(-1));
console.log(arr[-1]);
```

---

### Exercise I

```js
const arr = [1, 2, 3];

arr.length = 1;

console.log(arr);
```

---

### Exercise J

```js
const arr = [1, 2, 3];

const result = arr.splice(1, 1);

console.log(arr);
console.log(result);
```

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
index
length
property
hole
undefined
```

as separate concepts.

### Level 2 — Explain

Explain why:

```js
new Array(3)
```

is different from:

```js
[undefined, undefined, undefined]
```

### Level 3 — Predict

Predict callback counts and result shape for sparse arrays across:

```text
map
filter
reduce
forEach
for...of
spread
```

### Level 4 — Implement

Implement:

```js
miniMap
miniFilter
miniReduce
miniToSpliced
```

with sparse-array behavior.

### Level 5 — Debug

Find and fix:

- mutation bugs,
- queue performance issues,
- numeric sorting bugs,
- accidental holes,
- caller-state corruption.

### Level 6 — Compare

Compare:

```text
Array
Map
Set
TypedArray
Object
Queue
```

using:

- order,
- access,
- mutation,
- memory,
- complexity,
- semantics.

### Level 7 — Apply

Design a production collection API that:

- never mutates inputs,
- supports large datasets,
- validates index inputs,
- handles sparse input intentionally.

### Level 8 — Defend

Defend whether:

```js
toSorted()
```

is preferable to:

```js
[...arr].sort()
```

for a particular production codebase.

### Level 9 — Principal Judgment

Review an API that accepts user-provided Arrays and decide:

```text
validate?
clone?
freeze?
normalize?
reject sparse?
allow subclasses?
allow proxies?
```

Defend each decision based on the product's trust and correctness model.

---

## 27. Key Takeaways

1. Arrays are objects with specialized indexed-property and length semantics.
2. Array indices are property keys governed by specific index rules.
3. `length` is not the count of present elements in sparse arrays.
4. Holes are missing properties, not stored `undefined` values.
5. `delete arr[i]` can create a hole without changing length.
6. Assigning `undefined` creates a present property with an undefined value.
7. Increasing `length` does not populate every index.
8. Decreasing `length` removes indexed properties that fall outside the new range.
9. Arrays are exotic objects at the specification level.
10. `ArrayCreate` and `ArraySetLength` are important specification concepts.
11. `Array.isArray` is the proper Array test.
12. `Array.from` accepts iterable and array-like inputs.
13. `Array.of` avoids the `new Array(singleNumber)` ambiguity.
14. `slice` copies; `splice` mutates.
15. `toSpliced` provides an immutable-style splice operation.
16. `sort` mutates; `toSorted` does not.
17. Default `sort()` is not numeric sorting.
18. `map` and other callback methods have method-specific hole semantics.
19. Spread and iterable consumption can materialize holes as `undefined`.
20. Array methods may be generic over array-like objects.
21. `includes` and `indexOf` use different equality semantics.
22. `at(-1)` is negative-index access; `arr[-1]` is a String property lookup.
23. Sparse arrays can create correctness and performance problems.
24. Repeated `shift()` is usually unsuitable for large queues.
25. Array subclassing can interact with species construction.
26. Arrays are not interchangeable with Maps, Sets, or TypedArrays.
27. Shallow copies preserve nested object identity.
28. Array operations can execute getters, proxies, and callbacks.
29. Untrusted indices can create denial-of-service risks through huge logical lengths/work.
30. Principal-level Array design requires semantic, performance, and API-contract reasoning.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values / Types
- Chapter 07 — Coercion / Equality
- Chapter 08 — Iteration
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Enumeration
- Chapter 17 — Prototypes
- Chapter 19 — Proxy / Reflect
- Chapter 20 — Symbols
- Chapter 21 — Species / Subclassing

### Builds Toward

- Chapter 23 — Strings / String APIs
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 45 — Memory / GC
- Chapter 48 — V8 Internals / Optimization
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 76 — Composition / Abstraction Design
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 89 — Code Review / Refactoring

### Related Concepts

- Array exotic objects
- `length`
- property descriptors
- holes
- indexed properties
- iterable protocol
- array-like objects
- species
- shallow copying
- mutation
- immutable updates
- typed arrays
- Map/Set

### Concepts Revisited

**Chapter 15:**  
Array elements demonstrate how specialized objects can still use property descriptors and ordinary property identity.

**Chapter 16:**  
Array index ordering and property enumeration become concrete through indexed properties and holes.

**Chapter 19:**  
Proxies can intercept Array operations, but invariants still apply.

**Chapter 20:**  
Array iteration uses `Symbol.iterator`, and concatenation can use `Symbol.isConcatSpreadable`.

**Chapter 21:**  
Array subclassing can invoke species-aware result construction.

### Why This Chapter Matters Later

Arrays are one of the most frequently used JavaScript abstractions, but they also expose many of the language's deepest object semantics.

A strong Array model becomes a bridge between:

```text
objects
+
property keys
+
specialized internal methods
+
iteration
+
higher-order functions
+
subclassing
+
performance
```

That foundation is required for understanding collections, algorithms, serialization, engines, and production application design.

---

## Track A — Core Theory

### Level 1 — Intuition

> An Array is an object with ordered integer-index-like properties and a special length boundary.

### Level 2 — Syntax

Know:

```js
[]
new Array()
new Array(length)
Array.of()
Array.from()
arr.at()
```

### Level 3 — Practical

Know when to use:

```text
push/pop
map/filter/reduce
slice/splice
sort/toSorted
```

### Level 4 — Edge Cases

Understand:

- holes,
- sparse arrays,
- high indices,
- non-index properties,
- frozen/sealed arrays,
- non-writable length,
- proxies,
- subclasses.

### Level 5 — Runtime/Internal

Understand:

- Array exotic objects,
- specialized property definition,
- length synchronization,
- engine representation caveats.

### Level 6 — Specification Semantics

Be comfortable with:

- `ArrayCreate`
- `ArraySetLength`
- `LengthOfArrayLike`
- `IsArray`
- `ArraySpeciesCreate`

### Level 7 — Performance/Security

Reason about:

- sparse versus dense workloads,
- front-removal costs,
- memory retention,
- malicious indices,
- callbacks/getters/proxies.

### Level 8 — Production Engineering

Design:

- immutable update APIs,
- queue abstractions,
- input validation,
- collection choice,
- benchmark methodology.

### Level 9 — Interview/Reasoning

Answer:

> Why is `new Array(3)` not the same as `[undefined, undefined, undefined]`, and why does that difference matter?

### Level 10 — Principal Judgment

Evaluate:

> Would you allow sparse Arrays through a public API, normalize them, reject them, or document them?

The correct answer depends on the API's domain and invariants.

---

## Track B — Implementation

The implementation ladder is:

```text
1. indexed Array model
        ↓
2. map/filter/reduce
        ↓
3. slice/splice
        ↓
4. sparse-array handling
        ↓
5. immutable transformations
        ↓
6. queue/deque abstraction
        ↓
7. conformance test matrix
        ↓
8. benchmark suite
        ↓
9. production collection API
        ↓
10. hardened public contract
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain:

```text
length
≠
number of present elements
```

### Drill 2

Explain why:

```js
delete arr[1]
```

and:

```js
arr.splice(1, 1)
```

are semantically different.

### Drill 3

Explain why:

```js
[1, , 3].map(x => x)
```

and:

```js
[...([1, , 3])]
```

can produce differently structured Arrays.

### Drill 4

Explain why:

```js
Array.isArray(value)
```

is preferable to:

```js
value instanceof Array
```

in cross-realm code.

### Drill 5

Choose between:

```text
Array
Map
Set
TypedArray
Queue
```

for a given workload and defend the choice with:

```text
semantics
complexity
memory
mutation
security
operational constraints
```

---

## 29. Completion Criteria

Mark Chapter 22 `[+] Completed` only when the learner can:

- [ ] Explain Arrays as exotic objects.
- [ ] Explain array-index property semantics.
- [ ] Explain `length`.
- [ ] Distinguish holes from `undefined`.
- [ ] Explain `delete` versus `splice`.
- [ ] Explain increasing/decreasing length.
- [ ] Explain descriptor constraints on length changes.
- [ ] Predict sparse-array callback behavior.
- [ ] Explain iteration behavior for holes.
- [ ] Explain spread/materialization behavior.
- [ ] Explain `Array.isArray`.
- [ ] Explain `Array.from`.
- [ ] Explain `Array.of`.
- [ ] Explain `ArrayCreate`.
- [ ] Explain `ArraySetLength`.
- [ ] Explain `LengthOfArrayLike`.
- [ ] Explain `ArraySpeciesCreate`.
- [ ] Distinguish mutating and non-mutating methods.
- [ ] Explain `slice`, `splice`, `toSpliced`.
- [ ] Explain `sort`, `toSorted`.
- [ ] Explain numeric sorting.
- [ ] Explain `includes` versus `indexOf`.
- [ ] Explain `at(-1)` versus `arr[-1]`.
- [ ] Explain shallow-copy behavior.
- [ ] Explain subclassing/species interactions.
- [ ] Identify Array performance traps.
- [ ] Identify Array security/resource-exhaustion risks.
- [ ] Implement selected Array methods.
- [ ] Debug sparse and mutation bugs.
- [ ] Compare Arrays with other collection types.
- [ ] Make a production-level collection choice.

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

> When is an Array the correct production data structure, how should its mutation/sparse behavior be controlled, and when should another collection or abstraction replace it?

---

## Chapter 22 Retrieval Set

### Retrieval 1

What is the difference between:

```js
new Array(3)
```

and:

```js
[undefined, undefined, undefined]
```

### Retrieval 2

What is a hole?

### Retrieval 3

What happens to `length` after:

```js
delete arr[2]
```

### Retrieval 4

What happens to indexed properties when:

```js
arr.length = 2
```

reduces the length?

### Retrieval 5

Why is:

```js
arr[1.5]
```

not an ordinary Array index?

### Retrieval 6

Why can:

```js
[1, , 3].map(...)
```

and:

```js
[...[1, , 3]]
```

produce different structural results?

### Retrieval 7

Why is:

```js
includes(NaN)
```

different from:

```js
indexOf(NaN)
```

### Retrieval 8

Why is repeated:

```js
shift()
```

often a poor large-scale queue strategy?

---

## Chapter 22 Final Mental Model

Remember:

```text
                       ARRAY
                         |
        +----------------+----------------+
        |                |                |
    indexed keys      length          protocols
        |                |                |
      "0" "1"       range boundary    iterator
      "2" ...                         species
        |
    present OR hole
```

And:

```text
hole
  ≠
undefined
```

Also:

```text
slice       → copy
splice      → mutate
toSpliced   → immutable-style copy + splice

sort        → mutate
toSorted    → copy + sort
```

Finally:

> An Array is a specialized JavaScript object whose language semantics are defined around indexed properties, a special `length` property, iteration, and collection-oriented algorithms. Dense storage may be an engine optimization, but sparse arrays, holes, descriptors, proxies, subclassing, and protocol hooks are all part of the language model.
