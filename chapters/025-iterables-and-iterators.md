


# Chapter 25 — Iterables / Iterators

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** The iteration protocol, iterable objects, iterator objects, `next()`, iterator results, built-in consumers, custom iteration, iterator closing, and the bridge from synchronous to asynchronous iteration.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what an iterable is.
2. Explain what an iterator is.
3. Explain the difference between:
   - iterable,
   - iterator,
   - iterator result.
4. Explain why JavaScript separates the iterable and iterator concepts.
5. Explain the role of:
   ```js
   Symbol.iterator
   ```
6. Explain the role of:
   ```js
   next()
   ```
7. Explain the iterator result shape:
   ```js
   {
     value,
     done
   }
   ```
8. Explain why an iterator can also be iterable.
9. Explain built-in iterable consumers including:
   - `for...of`
   - spread syntax
   - destructuring
   - `Array.from`
   - `new Map(...)`
   - `new Set(...)`
10. Explain the difference between:
    - property enumeration,
    - array iteration,
    - generic iteration.
11. Explain why:
    ```js
    for...in
    ```
    is not the same as:
    ```js
    for...of
    ```
12. Explain how Arrays implement iteration.
13. Explain how Strings implement iteration.
14. Explain how Maps implement iteration.
15. Explain how Sets implement iteration.
16. Explain how generators integrate with the iterator protocol.
17. Explain iterator state.
18. Explain why most iterators are stateful objects.
19. Explain the difference between:
    - reusable iterable,
    - single-use iterator.
20. Explain why:
    ```js
    iterator[Symbol.iterator]() === iterator
    ```
    is common.
21. Explain iterator closing.
22. Explain:
    ```js
    return()
    ```
    in iterator protocols.
23. Explain how early termination in `for...of` can trigger iterator cleanup.
24. Explain:
    - `break`
    - `throw`
    - early loop termination
    in relation to iterator closing.
25. Explain the behavior of custom iterators that do not implement `return()`.
26. Explain error propagation from iterator `next()`.
27. Explain what happens when an iterator result is malformed.
28. Explain what happens when `[Symbol.iterator]` is not callable.
29. Explain how Proxies can affect iteration.
30. Explain iterator reentrancy and state-sharing issues.
31. Explain how mutation of an iterable during iteration can affect behavior.
32. Explain array iterator behavior around:
    - holes,
    - later mutations,
    - appended values.
33. Explain Map and Set iterator behavior under mutation.
34. Explain infinite iterables and their appropriate use cases.
35. Explain lazy iteration.
36. Compare:
    - eager Arrays,
    - lazy iterators,
    - generators.
37. Implement a custom iterable.
38. Implement a custom iterator.
39. Implement iterator closing.
40. Implement lazy sequence transformations.
41. Explain why an iterator is not automatically reusable.
42. Explain why `Array.from(iterator)` can consume it.
43. Explain why spreading an iterator consumes it.
44. Explain `Iterator` helpers conceptually where supported by the target runtime.
45. Explain upcoming/evolving iterator features without treating proposals as universal language guarantees.
46. Understand the conceptual relationship between sync and async iteration.
47. Prepare for Chapter 26 on generators and async generators.
48. Debug iteration protocol failures.
49. Compare iteration with direct indexing.
50. Reason about iteration design at principal-engineer level.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 08 — Control Flow / Iteration
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Ordering / Enumeration
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet

Strongly recommended:

- Chapter 19 — Proxy / Reflect
- Chapter 21 — Species / Subclassing

---

## 3. What Is It?

An **iterable** is a value that can provide an iterator through:

```js
Symbol.iterator
```

An **iterator** is an object that provides:

```js
next()
```

which returns iterator results.

A minimal iterator:

```js
const iterator = {
  next() {
    return {
      value: 1,
      done: true,
    };
  },
};
```

A minimal iterable:

```js
const iterable = {
  [Symbol.iterator]() {
    return iterator;
  },
};
```

These concepts are deliberately separate.

### Iterable

Answers:

> "How do I obtain an iterator for this collection/value?"

### Iterator

Answers:

> "What is the next result in this traversal?"

### Iterator result

Answers:

> "Here is the next value and whether iteration has completed."

```js
{
  value: ...,
  done: false,
}
```

or:

```js
{
  value: finalValue,
  done: true,
}
```

---

## 4. Why Does It Exist?

Without a common iteration protocol, every collection would require custom language syntax.

Instead of:

```js
for (const value of collection)
```

being able to consume many different structures, JavaScript could have required:

```text
special Array loop
special String loop
special Set loop
special Map loop
special custom-collection loop
```

The iterable protocol gives all of them a common interface.

```text
consumer
    ↓
get iterator
    ↓
next()
    ↓
{ value, done }
    ↓
repeat
```

This enables:

- Arrays,
- Strings,
- Maps,
- Sets,
- generators,
- custom data structures,
- lazy sequences.

### Why separate iterable and iterator?

A reusable collection can create a fresh iterator every time:

```js
const values = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
  },
};
```

Then:

```js
const a = [...values];
const b = [...values];
```

can traverse the data independently.

A single iterator itself is usually stateful:

```text
iterator
  ↓
current position
```

So:

```js
const iterator = values[Symbol.iterator]();
```

is usually single-use.

---

## 5. Mental Model

Use this architecture:

```text
                    ITERABLE
                       |
              Symbol.iterator
                       |
                       ↓
                   ITERATOR
                       |
                     next()
                       |
                       ↓
              { value, done }
                       |
                repeat until
                   done=true
```

### Example

```js
const iterable = [10, 20];

const iterator = iterable[Symbol.iterator]();

iterator.next();
iterator.next();
iterator.next();
```

Conceptually:

```text
{ value: 10, done: false }
{ value: 20, done: false }
{ value: undefined, done: true }
```

### Core distinction

```text
Iterable
  = produces traversal state

Iterator
  = owns traversal state
```

---

## 6. Core Rules

### Rule 1 — An iterable exposes `[Symbol.iterator]`

```js
typeof value[Symbol.iterator]
```

must be callable for ordinary synchronous iteration.

---

### Rule 2 — An iterator exposes `next()`

```js
iterator.next()
```

returns an iterator result.

---

### Rule 3 — `next()` returns an object

Typical result:

```js
{
  value: 42,
  done: false,
}
```

---

### Rule 4 — `done: true` ends ordinary iteration

Consumers should stop after receiving a completed result.

---

### Rule 5 — `value` can be anything

When `done` is `true`, the `value` property may still contain a meaningful final result for some iterator consumers.

---

### Rule 6 — Iterators are commonly also iterable

Typical pattern:

```js
const iterator = {
  next() {},
  [Symbol.iterator]() {
    return this;
  },
};
```

This allows:

```js
[...iterator]
```

---

### Rule 7 — Iterable does not mean reusable

A value can be iterable yet produce a single-use iterator each time.

---

### Rule 8 — `for...of` uses iteration, not property enumeration

It looks for:

```js
Symbol.iterator
```

rather than:

```js
Object.keys(...)
```

---

### Rule 9 — `for...in` is property enumeration

```js
for (const key in object)
```

operates over enumerable property keys.

It is not a generic iterable consumer.

---

### Rule 10 — Spread consumes the iterator

```js
[...iterable]
```

obtains an iterator and repeatedly calls `next()`.

---

### Rule 11 — Destructuring can consume iterables

```js
const [a, b] = iterable;
```

uses iteration semantics.

---

### Rule 12 — `Array.from` can consume an iterable

```js
Array.from(iterable)
```

materializes results into a new Array.

---

### Rule 13 — `Map` and `Set` constructors consume iterables

```js
new Map(iterableOfEntries)
new Set(iterable)
```

---

### Rule 14 — Iteration can execute user code

Calling:

```js
value[Symbol.iterator]()
```

and then:

```js
iterator.next()
```

invokes code controlled by the iterable.

Do not assume iteration is side-effect free.

---

### Rule 15 — Iterator state is usually mutable

A single iterator object normally advances through a traversal.

---

### Rule 16 — A loop can close an iterator early

When appropriate, early termination causes the consumer to invoke:

```js
iterator.return()
```

if present.

---

### Rule 17 — `return()` is cleanup protocol, not "normal next value"

It is used to signal iterator closing.

---

### Rule 18 — `throw()` is relevant to generator-based iterators

Generic iteration consumers do not simply call `throw()` for every iteration error.

Generator/iterator implementations may expose it for advanced control.

---

### Rule 19 — Invalid iterator results are errors

If `next()` returns a non-object result, a conforming consumer cannot treat it as a valid iterator result.

---

### Rule 20 — Infinite iterables require bounded consumption

```js
function* infinite() {
  let i = 0;
  while (true) yield i++;
}
```

is valid as an iterable source but must not be fully spread:

```js
[...infinite()]; // never completes
```

---

## 7. Syntax

### Iterable

```js
const iterable = {
  [Symbol.iterator]() {
    return iterator;
  },
};
```

### Iterator

```js
const iterator = {
  next() {
    return {
      value: 1,
      done: false,
    };
  },
};
```

### Generator-based iterable

```js
const iterable = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
  },
};
```

### Consume with `for...of`

```js
for (const value of iterable) {
  console.log(value);
}
```

### Manual iteration

```js
const iterator = iterable[Symbol.iterator]();

console.log(iterator.next());
```

### Spread

```js
const values = [...iterable];
```

### Destructuring

```js
const [first, second] = iterable;
```

### Array conversion

```js
const values = Array.from(iterable);
```

---

## 8. Basic Examples

### Example 1 — Manual iterator

```js
function makeIterator() {
  let current = 1;

  return {
    next() {
      if (current <= 3) {
        return {
          value: current++,
          done: false,
        };
      }

      return {
        value: undefined,
        done: true,
      };
    },
  };
}

const iterator = makeIterator();

console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
```

Conceptual output:

```text
{ value: 1, done: false }
{ value: 2, done: false }
{ value: 3, done: false }
{ value: undefined, done: true }
```

---

### Example 2 — Iterable object

```js
const numbers = {
  [Symbol.iterator]() {
    let current = 1;

    return {
      next() {
        if (current <= 3) {
          return {
            value: current++,
            done: false,
          };
        }

        return {
          done: true,
        };
      },
    };
  },
};

console.log([...numbers]);
```

Result:

```text
[1, 2, 3]
```

---

### Example 3 — Reusable iterable

```js
const values = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
  },
};

console.log([...values]);
console.log([...values]);
```

Both traversals work because a fresh iterator is created each time.

---

### Example 4 — Single-use iterator

```js
const iterator = [1, 2, 3][Symbol.iterator]();

console.log([...iterator]);
console.log([...iterator]);
```

The second spread receives no remaining values because the iterator was already consumed.

---

### Example 5 — `for...in` versus `for...of`

```js
const values = ["a", "b"];

for (const key in values) {
  console.log(key);
}

for (const value of values) {
  console.log(value);
}
```

First loop:

```text
0
1
```

Second loop:

```text
a
b
```

---

## 9. Execution Walkthrough

Consider:

```js
const values = [10, 20];

for (const value of values) {
  console.log(value);
}
```

### Step 1 — Obtain iterable method

The loop accesses:

```js
values[Symbol.iterator]
```

### Step 2 — Call the method

It produces an Array iterator object.

Conceptually:

```text
iterator
  current index = 0
```

### Step 3 — Call `next()`

The iterator produces:

```js
{
  value: 10,
  done: false,
}
```

### Step 4 — Assign loop variable

```js
value = 10
```

### Step 5 — Execute loop body

```js
console.log(value);
```

### Step 6 — Repeat

The iterator advances and returns:

```js
{
  value: 20,
  done: false,
}
```

### Step 7 — Final `next()`

The iterator returns a completed result:

```js
{
  value: undefined,
  done: true,
}
```

### Step 8 — Loop exits

The loop completes normally.

---

## 10. Internal Mechanics

### 10.1 Iterable protocol

The synchronous iterable protocol is centered around:

```js
Symbol.iterator
```

A consumer obtains the property and calls it.

The returned value must provide the iterator behavior required by the language operation.

---

### 10.2 Iterator protocol

An iterator provides:

```js
next()
```

Each successful call produces an iterator result.

Minimal valid shape:

```js
{
  value,
  done,
}
```

---

### 10.3 Iterator result object

The result is an ordinary object from the language-level perspective.

It does not need to be a specific built-in class.

This is valid:

```js
return {
  value: 10,
  done: false,
};
```

---

### 10.4 `done`

The `done` field communicates completion.

When:

```js
done === true
```

ordinary iterator-consuming algorithms stop requesting values.

---

### 10.5 Iterator result can contain a final value

```js
{
  value: "final",
  done: true,
}
```

can be meaningful to direct callers of:

```js
iterator.next()
```

Generator return values make this particularly relevant.

---

### 10.6 Iterators often self-iterate

A common implementation:

```js
const iterator = {
  next() {
    // ...
  },

  [Symbol.iterator]() {
    return this;
  },
};
```

This lets the same object participate as both:

```text
iterable
+
iterator
```

---

### 10.7 Iterable versus iterator lifecycle

A reusable iterable:

```text
collection
  ↓
iterator A
  ↓
iterator B
  ↓
iterator C
```

Each iterator owns independent traversal state.

A single iterator:

```text
iterator
  ↓
state 0
  ↓
state 1
  ↓
state 2
```

is consumed progressively.

---

### 10.8 Iterator closing

Some consumers perform iterator closing when exiting early.

Conceptually:

```text
consumer
  ↓
iterator.return()
```

if the iterator provides the method.

This gives iterators a cleanup opportunity.

---

### 10.9 `return()` and resource cleanup

Example:

```js
const iterable = {
  [Symbol.iterator]() {
    return {
      next() {
        return {
          value: 1,
          done: false,
        };
      },

      return() {
        console.log("cleanup");
        return {
          done: true,
        };
      },
    };
  },
};
```

Then:

```js
for (const value of iterable) {
  break;
}
```

can invoke the iterator's `return()` during loop closing.

---

### 10.10 Iterator errors

If:

```js
iterator.next()
```

throws:

```js
throw new Error("broken");
```

the consuming operation propagates the error according to its control-flow semantics.

This is ordinary JavaScript abrupt completion interacting with iteration.

---

### 10.11 Non-callable `[Symbol.iterator]`

```js
const value = {
  [Symbol.iterator]: 123,
};

for (const x of value) {
}
```

fails because the iteration protocol expects a callable method.

---

### 10.12 Iterator factory can capture state

```js
function makeRange(start, end) {
  return {
    [Symbol.iterator]() {
      let current = start;

      return {
        next() {
          if (current > end) {
            return { done: true };
          }

          return {
            value: current++,
            done: false,
          };
        },
      };
    },
  };
}
```

The iterator's closure contains traversal state.

---

## 11. ECMAScript / Specification Semantics

### 11.1 `GetIterator`

Specification-level iterable consumers use the conceptual abstract operation:

```text
GetIterator
```

which retrieves the appropriate iterator method and obtains an iterator object.

This is a foundational operation for:

- spread,
- destructuring,
- `for...of`,
- Array construction from iterables,
- many built-in algorithms.

---

### 11.2 `IteratorNext`

The specification has conceptual iterator-step operations around calling:

```text
next
```

and obtaining iterator results.

---

### 11.3 `IteratorComplete`

The result object's:

```js
done
```

property is interpreted using the iterator-completion logic.

Consumers decide whether iteration continues.

---

### 11.4 `IteratorValue`

The value is retrieved from the iterator result.

This separates:

```text
is iteration complete?
```

from:

```text
what value was produced?
```

---

### 11.5 `IteratorClose`

When an iteration-consuming operation exits abruptly or early under the relevant algorithm, it can perform:

```text
IteratorClose
```

which can invoke:

```js
iterator.return
```

when required.

This is important for resource management.

---

### 11.6 Abrupt completion

Iterator operations can fail because:

- obtaining `[Symbol.iterator]` throws,
- calling it throws,
- `next` throws,
- the result is invalid,
- reading `done` throws,
- reading `value` throws,
- cleanup throws.

Iteration is therefore integrated with the language's broader abrupt-completion model.

---

### 11.7 `ForIn/OfHeadEvaluation`

The specification contains separate logic for `for...in` and `for...of`.

This is the formal reason the syntax similarity does not imply semantic similarity.

---

### 11.8 `CreateIterResultObject`

The specification uses a conceptual operation for constructing iterator result objects.

That reinforces the standard result shape:

```text
value
done
```

---

### 11.9 Iterable protocols are structural

An object does not need to inherit from a special `Iterable` base class.

It participates structurally:

```js
obj[Symbol.iterator]
```

provides the protocol.

This is a form of behavioral/structural interface.

---

### 11.10 Array iterator semantics

Array iterators are defined in terms of indexed traversal.

They can observe values that change during iteration according to the Array iterator algorithm.

This means:

```js
const arr = [1, 2];

const iterator = arr[Symbol.iterator]();

arr.push(3);

console.log([...iterator]);
```

can observe later additions.

Do not assume the iterator snapshots the entire Array at creation.

---

### 11.11 String iterator semantics

String iterators operate in a code-point-aware manner for valid surrogate pairs.

This connects iteration with Chapter 23.

---

### 11.12 Map iterator semantics

Map iterators traverse Map entries, keys, or values according to the iterator created.

Mutation during iteration follows defined Map iteration semantics and should be studied explicitly when the collection changes while being consumed.

---

### 11.13 Set iterator semantics

Set iterators traverse Set values according to Set's insertion order semantics.

Mutation during traversal has defined behavior and should not be guessed from Array behavior.

---

### 11.14 Iterator Helpers

Modern ECMAScript development includes iterator-helper capabilities for composing lazy iterator pipelines.

Because runtime support and exact availability can evolve, always distinguish:

```text
standardized language feature
```

from:

```text
runtime support level
```

when using iterator helpers in production.

---

## 12. Advanced Behavior

### 12.1 Reusable iterable

```js
const range = {
  start: 1,
  end: 3,

  [Symbol.iterator]() {
    let current = this.start;

    return {
      next: () => {
        if (current > this.end) {
          return { done: true };
        }

        return {
          value: current++,
          done: false,
        };
      },
    };
  },
};
```

Each call to:

```js
range[Symbol.iterator]()
```

creates independent state.

---

### 12.2 Iterator itself as iterable

```js
function makeIterator() {
  let current = 0;

  return {
    next() {
      return current < 3
        ? { value: current++, done: false }
        : { done: true };
    },

    [Symbol.iterator]() {
      return this;
    },
  };
}
```

---

### 12.3 Lazy transformation

A lazy map-like iterable:

```js
function mapIterable(iterable, transform) {
  return {
    *[Symbol.iterator]() {
      for (const value of iterable) {
        yield transform(value);
      }
    },
  };
}
```

No transformed values are produced until consumed.

---

### 12.4 Lazy filter

```js
function filterIterable(iterable, predicate) {
  return {
    *[Symbol.iterator]() {
      for (const value of iterable) {
        if (predicate(value)) {
          yield value;
        }
      }
    },
  };
}
```

---

### 12.5 Infinite iterable

```js
function integers() {
  let current = 0;

  return {
    *[Symbol.iterator]() {
      while (true) {
        yield current++;
      }
    },
  };
}
```

Consume only a bounded prefix:

```js
const firstFive = [];

for (const value of integers()) {
  firstFive.push(value);

  if (firstFive.length === 5) {
    break;
  }
}
```

---

### 12.6 Mutation during Array iteration

```js
const arr = [1, 2, 3];

for (const value of arr) {
  console.log(value);

  if (value === 1) {
    arr.push(4);
  }
}
```

The behavior demonstrates that Array iteration is not necessarily a frozen snapshot.

Understand the actual iterator algorithm before relying on mutation behavior.

---

### 12.7 Mutation during Set iteration

```js
const set = new Set([1, 2, 3]);

for (const value of set) {
  if (value === 1) {
    set.add(4);
  }

  console.log(value);
}
```

Set iteration has its own specified semantics.

Do not assume it is identical to Array iteration.

---

### 12.8 Mutation during Map iteration

```js
const map = new Map([
  ["a", 1],
  ["b", 2],
]);

for (const [key, value] of map) {
  if (key === "a") {
    map.set("c", 3);
  }

  console.log(key, value);
}
```

Map's defined traversal semantics matter.

This should be treated as deliberate advanced behavior, not casual coding style.

---

### 12.9 Iterator consumed by `Array.from`

```js
const iterator = [1, 2, 3][Symbol.iterator]();

console.log(Array.from(iterator));
console.log(Array.from(iterator));
```

The second call sees the iterator's remaining state.

---

### 12.10 Iterator consumed by spread

```js
const iterator = [1, 2, 3][Symbol.iterator]();

console.log([...iterator]);
console.log([...iterator]);
```

Second result:

```text
[]
```

---

### 12.11 Partial consumption

```js
const iterator = [10, 20, 30][Symbol.iterator]();

console.log(iterator.next());

console.log([...iterator]);
```

The spread starts from the remaining state.

---

### 12.12 Destructuring closes iterators in relevant early-completion cases

Iterable destructuring can consume only the required values and may close the iterator when the pattern finishes without consuming everything.

Custom iterators with `return()` can make this observable.

This is one of the strongest practical reasons to understand IteratorClose.

---

### 12.13 Throwing getter for `[Symbol.iterator]`

```js
const obj = {
  get [Symbol.iterator]() {
    throw new Error("cannot iterate");
  },
};
```

Any operation requesting the iterator can fail before `next()` is ever called.

---

### 12.14 Throwing iterator method

```js
const obj = {
  [Symbol.iterator]() {
    throw new Error("construction failed");
  },
};
```

Again, iteration fails before producing a value.

---

### 12.15 Malformed iterator result

```js
const obj = {
  [Symbol.iterator]() {
    return {
      next() {
        return 10;
      },
    };
  },
};
```

This is invalid because the iterator result must be an object.

---

### 12.16 Iterator result getter side effects

```js
const iterator = {
  next() {
    return {
      get value() {
        console.log("value read");
        return 1;
      },

      get done() {
        console.log("done read");
        return false;
      },
    };
  },
};
```

Consumers may access these properties as part of the algorithm.

Iteration is therefore observable.

---

## 13. Edge Cases

### Edge Case 1 — Empty iterable

```js
const empty = {
  *[Symbol.iterator]() {}
};

[...empty]; // []
```

---

### Edge Case 2 — Iterator that never returns `done: true`

```js
const infinite = {
  [Symbol.iterator]() {
    return {
      next() {
        return {
          value: 1,
          done: false,
        };
      },
    };
  },
};
```

Any unbounded consumer can run forever.

---

### Edge Case 3 — `done` truthiness

Iterator completion uses the value converted according to the relevant algorithm.

Do not assume the property must literally contain Boolean `true` to represent completion without understanding the specification conversion semantics.

Best practice:

```js
done: true
```

or:

```js
done: false
```

for clarity.

---

### Edge Case 4 — `return()` returns a non-object

Iterator closing expects correct protocol behavior.

A malformed `return()` result can cause an error.

---

### Edge Case 5 — `return()` throws

```js
return() {
  throw new Error("cleanup failed");
}
```

Cleanup failure can replace or affect the original control flow depending on the consuming algorithm and completion state.

This is why cleanup code must be designed carefully.

---

### Edge Case 6 — `[Symbol.iterator]` returns a primitive

```js
const obj = {
  [Symbol.iterator]() {
    return 42;
  },
};
```

Invalid.

An iterator object is required.

---

### Edge Case 7 — `next` is not callable

```js
const obj = {
  [Symbol.iterator]() {
    return {
      next: 123,
    };
  },
};
```

Iteration fails.

---

### Edge Case 8 — Iterator reentrancy

An iterator can accidentally be consumed recursively:

```js
const iterator = {
  next() {
    // calls the same iterator again
  },
};
```

This can corrupt state or create recursion.

---

### Edge Case 9 — Shared iterator

```js
const iterator = values[Symbol.iterator]();

function consume() {
  for (const value of iterator) {
    console.log(value);
  }
}

consume();
consume();
```

The second call gets remaining/empty state.

---

### Edge Case 10 — Two consumers sharing one iterator

```js
const iterator = values[Symbol.iterator]();

const a = iterator.next();
const b = iterator.next();
```

The consumers divide the same traversal state.

---

### Edge Case 11 — Iterator closes early

```js
for (const value of iterable) {
  break;
}
```

A cleanup-aware iterator can observe this through `return()`.

---

### Edge Case 12 — Iterator consumed by multiple APIs

Using the same iterator with:

```js
spread
Array.from
for...of
```

does not reset it.

---

### Edge Case 13 — Array mutation

Mutation can affect later values depending on the Array iterator algorithm.

Do not rely on snapshot semantics.

---

### Edge Case 14 — Primitive strings

Primitive Strings are iterable.

```js
for (const ch of "😀") {
}
```

This follows String iterator semantics from Chapter 23.

---

### Edge Case 15 — Plain object is not iterable

```js
for (const value of { a: 1 }) {
}
```

fails because ordinary Objects do not automatically provide `Symbol.iterator`.

Use:

```js
Object.entries(object)
```

when iteration over properties is intended.

---

## 14. Common Misconceptions

### Misconception 1 — "Iterable and iterator mean the same thing."

False.

Iterable produces an iterator.

Iterator produces iteration results.

---

### Misconception 2 — "Every iterable is reusable."

False.

The iterable may itself return the same stateful iterator.

---

### Misconception 3 — "Every iterator is independent."

False.

An iterator owns state; sharing it shares traversal state.

---

### Misconception 4 — "`for...of` uses Object.keys."

False.

It uses the iterable protocol.

---

### Misconception 5 — "`for...in` is the old version of `for...of`."

False.

They implement different protocols.

---

### Misconception 6 — "Spread just copies the collection."

Not universally.

Spread consumes the iterable.

---

### Misconception 7 — "`Array.from(iterator)` does not mutate the iterator."

It consumes its state.

---

### Misconception 8 — "Iterator results must be instances of IteratorResult."

There is no required user-facing class. The protocol is structural.

---

### Misconception 9 — "done means there is no value."

Not necessarily for direct iterator consumers.

A completed iterator can have a meaningful final return value.

---

### Misconception 10 — "Breaking a loop does nothing to the iterator."

It can invoke iterator closing.

---

### Misconception 11 — "An iterator is always lazy and an iterable is always a collection."

Not necessarily.

These are protocol roles, not performance guarantees.

---

### Misconception 12 — "Iterators snapshot data."

Not generally.

The underlying iterable determines mutation behavior.

---

### Misconception 13 — "Infinite iterables are invalid."

They are valid, but consumers must bound traversal.

---

### Misconception 14 — "All iteration is synchronous."

False.

JavaScript also has asynchronous iteration, covered later.

---

## 15. Common Mistakes

### Mistake 1 — Returning the wrong iterator object

```js
[Symbol.iterator]() {
  return {};
}
```

without `next()` creates a broken iterable.

---

### Mistake 2 — Forgetting `return this`

For an iterator intended to be self-iterable:

```js
[Symbol.iterator]() {
  return this;
}
```

is required.

---

### Mistake 3 — Returning a primitive from `next()`

Wrong:

```js
next() {
  return 1;
}
```

Correct:

```js
next() {
  return {
    value: 1,
    done: false,
  };
}
```

---

### Mistake 4 — Reusing a consumed iterator

Create a fresh iterator when independent traversals are required.

---

### Mistake 5 — Assuming `for...of` is equivalent to indexing

This fails for:

- Strings,
- Sets,
- Maps,
- custom iterables.

---

### Mistake 6 — Ignoring iterator cleanup

Resource-producing iterators should consider `return()`.

---

### Mistake 7 — Writing infinite iterables without bounded consumers

```js
[...infinite()]
```

can never finish.

---

### Mistake 8 — Mutating during iteration without understanding semantics

Collection-specific rules matter.

---

### Mistake 9 — Treating custom iterable protocols as purely synchronous data

The iterator may:

- perform I/O,
- mutate state,
- throw,
- allocate,
- trigger user code.

---

### Mistake 10 — Catching errors without preserving cleanup

A cleanup-aware iterator may need to finish its protocol before error handling completes.

---

## 16. Comparison With Related Concepts

| Concept | Provides | Stateful traversal | Typical consumer |
|---|---|---:|---|
| Iterable | `Symbol.iterator` | No/usually no | `for...of`, spread |
| Iterator | `next()` | Yes | iteration consumers |
| Iterator result | `{value, done}` | Per-step result | iterator consumer |
| Array | indexed collection + iterable | Iterator state external | `for...of`, indexing |
| Generator | iterable + iterator behavior | Yes | `for...of`, `next()` |
| Async iterable | `Symbol.asyncIterator` | Async state | `for await...of` |
| Object enumeration | property keys | Enumeration state | `for...in` |

### Iterable versus Iterator

```text
Iterable = factory/provider
Iterator = state machine
```

---

### Array versus iterator

Array:

```text
materialized collection
```

Iterator:

```text
potentially lazy traversal
```

An Array can produce an iterator.

---

### Generator versus hand-written iterator

Generators automatically implement complex iterator state machines.

Hand-written iterators give more explicit control but are more error-prone.

Chapter 26 explores this deeply.

---

### Iterable versus collection

An iterable need not own a collection.

It can represent:

```text
range
stream-like sequence
lazy transformation
infinite sequence
generated values
```

---

## 17. Performance Considerations

### 17.1 Eager versus lazy

Eager:

```js
const result = values.map(transform);
```

materializes the result.

Lazy:

```js
function* mapLazy(iterable, transform) {
  for (const value of iterable) {
    yield transform(value);
  }
}
```

computes values on demand.

---

### 17.2 Iterator overhead

A manual iterator can introduce:

- object allocation,
- repeated method calls,
- protocol checks.

Engines can optimize common patterns, but do not assume protocol abstraction is free.

---

### 17.3 Generator overhead

Generators provide convenient state machines but can have different allocation and scheduling costs than a simple loop.

Measure if the code is performance-critical.

---

### 17.4 Materializing an iterator

```js
Array.from(iterable)
```

has a memory cost proportional to the resulting collection.

Do not materialize an enormous or infinite source accidentally.

---

### 17.5 Spread materialization

```js
[...iterable]
```

also materializes all values.

---

### 17.6 Lazy pipelines

Lazy iteration can reduce:

- memory,
- unnecessary work,
- intermediate collections.

But can increase:

- per-element protocol overhead,
- debugging complexity,
- stack/control-flow complexity.

---

### 17.7 Iterator helper pipelines

Where supported, iterator helpers can express:

```text
lazy transform
filter
take
```

without intermediate Arrays.

Verify runtime support before relying on these APIs in production environments.

---

## 18. Memory Considerations

### Iterator state

A stateful iterator can retain:

- the source object,
- closures,
- intermediate buffers,
- external resources.

A long-lived iterator can therefore retain more memory than expected.

---

### Lazy pipeline retention

```text
source
  ↓
iterator
  ↓
transform closure
  ↓
other objects
```

The iterator can keep the entire dependency chain reachable.

---

### Materialization

Converting:

```js
Array.from(hugeIterable)
```

can create a large memory spike.

Bound or stream the input where possible.

---

### Infinite iterators

An infinite iterator does not necessarily consume infinite memory by itself.

Memory grows based on the state retained by the implementation and consumer.

An infinite lazy sequence can be memory-efficient if consumed incrementally.

---

## 19. Security Considerations

### Untrusted iterable

Calling:

```js
for (const value of untrustedObject)
```

can execute arbitrary code through:

```js
[Symbol.iterator]
next()
return()
value getters
done getters
```

Treat iteration as a code-execution boundary when the object is untrusted.

---

### Infinite iteration denial of service

An attacker-controlled iterable can never finish.

Always bound iterations when consuming untrusted sources.

---

### Resource cleanup

If an iterator wraps resources such as:

```text
file
socket
lock
transaction
stream
```

incorrect closing semantics can produce resource leaks.

---

### Iterator reentrancy

Malicious or complex iterator code can reenter application logic.

Design state machines to be robust against reentrancy where required.

---

### Getter/proxy attacks

Iteration can trigger getters and Proxy traps.

Do not assume reading a value is side-effect free.

---

## 20. Production Usage

### Use case 1 — Custom collection

```js
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  *[Symbol.iterator]() {
    for (let value = this.start; value <= this.end; value++) {
      yield value;
    }
  }
}
```

---

### Use case 2 — Lazy processing

```js
function* mapLazy(iterable, fn) {
  for (const value of iterable) {
    yield fn(value);
  }
}
```

Useful when the source is large or unbounded.

---

### Use case 3 — Bounded consumption

```js
function take(iterable, count) {
  const result = [];

  let consumed = 0;

  for (const value of iterable) {
    result.push(value);

    consumed++;

    if (consumed >= count) {
      break;
    }
  }

  return result;
}
```

This works with infinite iterables.

---

### Use case 4 — Tree traversal

A tree structure can expose:

```js
*[Symbol.iterator]() {
  // depth-first traversal
}
```

Then consumers can use:

```js
for (const node of tree) {
}
```

without exposing traversal implementation.

---

### Use case 5 — Resource-backed iteration

A resource iterator should define cleanup behavior carefully:

```js
return() {
  releaseResource();
  return { done: true };
}
```

This can integrate with early loop termination.

For deterministic resource management, also consider the dedicated resource-management mechanisms studied in Chapter 30.

---

### Use case 6 — Library API

A library can expose an iterable view without exposing its internal data structure:

```js
class Registry {
  #items = new Map();

  *[Symbol.iterator]() {
    yield* this.#items.values();
  }
}
```

Consumers depend on iteration rather than internal storage.

---

### Production rule

When exposing iteration, define:

```text
What is yielded?
Is traversal stable?
Is iteration reusable?
Is it lazy?
What happens on mutation?
How is early termination handled?
Can it be infinite?
What resources are held?
What errors can next() throw?
```

---

## 21. Implementation From Scratch

### 21.1 Basic Range iterator

```js
function range(start, end) {
  let current = start;

  return {
    next() {
      if (current > end) {
        return {
          value: undefined,
          done: true,
        };
      }

      return {
        value: current++,
        done: false,
      };
    },

    [Symbol.iterator]() {
      return this;
    },
  };
}
```

---

### 21.2 Reusable Range iterable

```js
function makeRange(start, end) {
  return {
    [Symbol.iterator]() {
      let current = start;

      return {
        next() {
          if (current > end) {
            return { done: true };
          }

          return {
            value: current++,
            done: false,
          };
        },
      };
    },
  };
}
```

The second version is reusable because each call creates fresh state.

---

### 21.3 Lazy map

```js
function mapLazy(iterable, fn) {
  return {
    *[Symbol.iterator]() {
      for (const value of iterable) {
        yield fn(value);
      }
    },
  };
}
```

---

### 21.4 Lazy filter

```js
function filterLazy(iterable, predicate) {
  return {
    *[Symbol.iterator]() {
      for (const value of iterable) {
        if (predicate(value)) {
          yield value;
        }
      }
    },
  };
}
```

---

### 21.5 Take

```js
function take(iterable, count) {
  if (!Number.isInteger(count) || count < 0) {
    throw new RangeError("count must be a non-negative integer");
  }

  return {
    *[Symbol.iterator]() {
      if (count === 0) {
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
    },
  };
}
```

---

### 21.6 Iterator cleanup experiment

```js
function makeResourceIterable() {
  let closed = false;

  return {
    [Symbol.iterator]() {
      let current = 0;

      return {
        next() {
          if (closed) {
            return { done: true };
          }

          return {
            value: current++,
            done: false,
          };
        },

        return() {
          closed = true;
          console.log("resource closed");
          return { done: true };
        },

        [Symbol.iterator]() {
          return this;
        },
      };
    },
  };
}
```

Then:

```js
for (const value of makeResourceIterable()) {
  console.log(value);

  if (value === 2) {
    break;
  }
}
```

Observe cleanup.

---

### 21.7 Production iterator abstraction

Build a reusable iterable abstraction supporting:

```text
map
filter
take
skip
concat
```

without materializing intermediate Arrays.

Then document:

- laziness,
- reuse,
- error behavior,
- cleanup,
- mutation semantics.

---

### Implementation progression

**Guided**

- manual iterator,
- reusable iterable,
- range.

**Partially Guided**

- lazy map/filter,
- take/skip,
- resource cleanup.

**No Reference**

- build a lazy sequence library.

**Edge-Case Hardened**

Support:

- infinite sources,
- early termination,
- iterator errors,
- cleanup errors,
- invalid counts,
- iterator reuse.

**Production-Grade**

Add:

- tests,
- benchmarks,
- type contracts,
- cancellation strategy,
- observability,
- documentation.

---

## 22. Debugging Exercises

### Exercise 1 — Broken iterator

```js
const iterable = {
  [Symbol.iterator]() {
    return {
      next() {
        return 1;
      },
    };
  },
};

console.log([...iterable]);
```

Identify the protocol violation.

---

### Exercise 2 — Missing `Symbol.iterator`

```js
const iterator = {
  next() {
    return { value: 1, done: true };
  },
};

console.log([...iterator]);
```

Explain why this does not work unless the iterator is also iterable.

---

### Exercise 3 — Reuse bug

```js
const iterator = [1, 2, 3][Symbol.iterator]();

function consume() {
  return [...iterator];
}

console.log(consume());
console.log(consume());
```

Explain the results.

---

### Exercise 4 — Infinite iterable

```js
function* infinite() {
  let i = 0;

  while (true) {
    yield i++;
  }
}

console.log([...infinite()]);
```

Diagnose the operational problem.

---

### Exercise 5 — Cleanup

Create an iterator with:

```js
return()
```

and verify whether:

```js
break
```

causes cleanup.

---

### Exercise 6 — Lazy transformation

```js
const mapped = mapLazy([1, 2, 3], x => x * 2);
```

Verify whether the transform runs when `mapped` is created or when it is consumed.

---

### Exercise 7 — Shared iterator

Two consumers share one iterator.

Determine how values are partitioned.

---

### Exercise 8 — Array mutation

```js
const values = [1, 2, 3];

for (const value of values) {
  if (value === 1) {
    values.push(4);
  }

  console.log(value);
}
```

Predict the behavior before running.

---

## 23. Code Review Exercise

Review:

```js
class DataSource {
  constructor(source) {
    this.source = source;
    this.iterator = source[Symbol.iterator]();
  }

  [Symbol.iterator]() {
    return this.iterator;
  }
}
```

### Questions

1. Is `DataSource` reusable?
2. What happens after one consumer finishes?
3. What happens when two consumers iterate concurrently?
4. Would a fresh iterator be safer?
5. Is `source` itself reusable?
6. Should `iterator` be stored at construction time?
7. What happens if `source[Symbol.iterator]` throws later?
8. Should early termination trigger cleanup?
9. Could this retain a large source unexpectedly?
10. Would a generator-based implementation be clearer?

Propose a production-quality alternative.

---

## 24. Interview Questions

### Fundamentals

1. What is an iterable?
2. What is an iterator?
3. What is an iterator result?
4. Why are iterable and iterator separate concepts?
5. What is `Symbol.iterator`?
6. What does `next()` return?
7. What is `done`?
8. Can an iterator be iterable?
9. Is every iterable reusable?
10. Is every iterator reusable?

### Consumption

11. How does `for...of` work conceptually?
12. How does spread consume an iterable?
13. How does destructuring consume an iterable?
14. How does `Array.from` consume an iterable?
15. How do Map and Set constructors consume iterables?
16. Why does `for...in` differ from `for...of`?
17. Why is a plain Object not iterable by default?

### Closing

18. What is iterator closing?
19. What is `IteratorClose`?
20. When does `for...of` call `return()`?
21. What happens if `return()` throws?
22. Why is iterator closing important for resource-backed iterators?

### State

23. Why does an iterator have state?
24. Why does sharing an iterator create contention?
25. Why does an iterable usually create fresh iterators?
26. How can an iterator retain memory?
27. How can mutation affect iteration?

### Advanced

28. How would you build a lazy iterator pipeline?
29. How would you implement an infinite iterable safely?
30. How would you design a tree iterator?
31. How would you make an iterator reentrant-safe?
32. How would you test iterator cleanup?
33. What happens when `next()` returns a primitive?
34. What happens when `Symbol.iterator` is not callable?
35. What is the difference between eager and lazy iteration?
36. What are iterator helper APIs?
37. How do synchronous and asynchronous iteration differ conceptually?

### Principal-level

38. When should a library expose an iterable interface?
39. When should it return an Array instead?
40. What semantics should a mutable collection document during iteration?
41. How should resource-backed iterators handle early termination?
42. How should untrusted iterables be bounded?
43. How would you benchmark an iterator pipeline against Array transforms?
44. What API design makes lazy iteration observable/debuggable?
45. How would you decide whether iterator reuse is desirable or dangerous?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
const iterator = [1, 2][Symbol.iterator]();

console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
```

---

### Exercise B

```js
const iterator = [1, 2, 3][Symbol.iterator]();

console.log([...iterator]);
console.log([...iterator]);
```

---

### Exercise C

```js
const iterable = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
  },
};

console.log([...iterable]);
console.log([...iterable]);
```

---

### Exercise D

```js
const values = [1, 2];

for (const value of values) {
  console.log(value);
}
```

Explain which protocol is involved.

---

### Exercise E

```js
const values = [1, 2];

for (const key in values) {
  console.log(key);
}
```

Explain the difference from Exercise D.

---

### Exercise F

```js
const iterator = [10, 20, 30][Symbol.iterator]();

console.log(iterator.next());

console.log([...iterator]);
```

---

### Exercise G

```js
const iterable = {
  [Symbol.iterator]() {
    return {
      next() {
        return {
          value: 1,
          done: false,
        };
      },
    };
  },
};

console.log([...iterable]);
```

Determine whether it terminates.

---

### Exercise H

```js
const iterable = {
  [Symbol.iterator]() {
    return {
      next() {
        return {
          value: 1,
          done: false,
        };
      },

      return() {
        console.log("closed");
        return { done: true };
      },
    };
  },
};

for (const value of iterable) {
  break;
}
```

Predict whether `"closed"` is printed.

---

### Exercise I

```js
const iterator = {
  count: 0,

  next() {
    if (this.count === 2) {
      return { done: true };
    }

    return {
      value: this.count++,
      done: false,
    };
  },

  [Symbol.iterator]() {
    return this;
  },
};

console.log([...iterator]);
```

---

### Exercise J

```js
const iterator = [1, 2, 3][Symbol.iterator]();

const a = iterator.next();
const b = iterator.next();

console.log(a);
console.log(b);
console.log([...iterator]);
```

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
iterable
iterator
iterator result
```

without collapsing them into one concept.

### Level 2 — Explain

Explain why:

```js
[...iterator]
```

can consume the iterator permanently.

### Level 3 — Predict

Predict manual `next()` sequences.

### Level 4 — Implement

Build:

```text
range
mapLazy
filterLazy
take
```

### Level 5 — Debug

Repair:

- malformed iterator,
- non-callable Symbol.iterator,
- iterator reuse bug,
- missing cleanup.

### Level 6 — Compare

Compare:

```text
Array
iterable
iterator
generator
```

for:

- storage,
- state,
- reuse,
- laziness,
- memory,
- performance.

### Level 7 — Apply

Build a lazy tree traversal API.

### Level 8 — Defend

Choose between:

```text
return Array
return Iterable
return Iterator
```

for a public library API and defend the API semantics.

### Level 9 — Principal Judgment

Design a resource-backed iterable that guarantees:

```text
bounded memory
lazy consumption
early cleanup
error propagation
reusability
observability
```

and explain the trade-offs.

---

## 27. Key Takeaways

1. Iterable and iterator are distinct protocol roles.
2. An iterable exposes `Symbol.iterator`.
3. An iterator exposes `next()`.
4. `next()` returns an iterator result.
5. Iterator results contain `value` and `done`.
6. Iterators are usually stateful.
7. An iterable can create a fresh iterator for each traversal.
8. An iterator itself is often also iterable.
9. `for...of` uses the iterable protocol.
10. `for...in` uses property enumeration.
11. Spread consumes an iterable.
12. Destructuring consumes an iterable.
13. `Array.from` consumes an iterable.
14. Map and Set constructors consume iterables.
15. Iteration is structural and protocol-based.
16. Plain Objects are not iterable by default.
17. Infinite iterables are valid but require bounded consumers.
18. Iterator results must satisfy protocol requirements.
19. Iterator closing can invoke `return()`.
20. `return()` provides a cleanup hook.
21. Iterator errors integrate with JavaScript abrupt completion.
22. Iterators are not snapshots by default.
23. Collection-specific mutation semantics matter.
24. Lazy iteration can reduce intermediate memory use.
25. Lazy iteration can increase per-item protocol overhead.
26. Iterators can retain source data and closures.
27. Untrusted iterables can execute arbitrary code.
28. Untrusted/infinite iterables require bounded consumption.
29. Resource-backed iterators must implement deliberate cleanup semantics.
30. Iterator design is both a protocol problem and a lifecycle problem.

---

## 28. Concept Connections

### Depends On

- Chapter 08 — Control Flow / Iteration
- Chapter 15 — Objects / Property Semantics
- Chapter 16 — Property Keys / Enumeration
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 24 — Map / Set / Weak Collections

### Builds Toward

- Chapter 26 — Generators / Async Generators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Async Fundamentals
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 40 — Observables / Reactive
- Chapter 53 — Web Streams / Data Flow
- Chapter 60 — Node Streams
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 80 — Library Authoring
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance

### Related Concepts

- `Symbol.iterator`
- `next()`
- iterator result
- `IteratorClose`
- `return()`
- generators
- lazy evaluation
- infinite sequences
- Array iteration
- Map/Set iteration
- asynchronous iteration
- resource cleanup
- structural protocols

### Concepts Revisited

**Chapter 20:**  
`Symbol.iterator` turns a Symbol from an abstract protocol identifier into a concrete iteration mechanism.

**Chapter 22:**  
Arrays expose iteration independently from their indexed-property model.

**Chapter 23:**  
String iteration is code-point-aware while indexed access is code-unit-oriented.

**Chapter 24:**  
Map and Set demonstrate different iterator shapes over collections.

**Chapter 19:**  
Proxies can intercept `[Symbol.iterator]`, `next`, `return`, and accessed iterator-result properties.

### Why This Chapter Matters Later

Iteration is one of JavaScript's most important abstraction boundaries.

It lets code consume:

```text
arrays
strings
maps
sets
trees
ranges
generators
streams
custom collections
```

through one protocol.

The deeper lesson is:

> JavaScript's iteration model separates the source of traversal from the traversal state itself.

That distinction becomes essential for:

- generators,
- asynchronous iteration,
- streaming,
- lazy algorithms,
- resource cleanup,
- concurrency,
- library design.

---

## Track A — Core Theory

### Level 1 — Intuition

> An iterable gives you an iterator; an iterator gives you the next value.

### Level 2 — Syntax

Know:

```js
Symbol.iterator
next()
{ value, done }
```

### Level 3 — Practical

Build:

- range,
- custom iterable,
- lazy transformations.

### Level 4 — Edge Cases

Understand:

- single-use iterators,
- malformed results,
- `return()`,
- infinite sources,
- mutation during iteration.

### Level 5 — Runtime/Internal

Understand:

- iterator state,
- iterator-result objects,
- consumer algorithms,
- iterator closing,
- abrupt completion.

### Level 6 — Specification Semantics

Be comfortable with:

- `GetIterator`
- `IteratorNext`
- `IteratorComplete`
- `IteratorValue`
- `IteratorClose`
- `CreateIterResultObject`

### Level 7 — Performance/Security

Reason about:

- lazy versus eager,
- protocol overhead,
- retention,
- infinite loops,
- untrusted iteration,
- resource cleanup.

### Level 8 — Production Engineering

Design:

- lazy APIs,
- reusable iterables,
- resource-aware iterators,
- bounded consumers.

### Level 9 — Interview/Reasoning

Answer:

> Why are iterable and iterator separate concepts, and why is that separation useful?

### Level 10 — Principal Judgment

Evaluate:

> Should this public API return an Array, an Iterable, or an Iterator?

Consider:

```text
reusability
laziness
memory
consumer expectations
cleanup
errors
concurrency
observability
```

---

## Track B — Implementation

The implementation ladder is:

```text
1. Manual iterator
        ↓
2. Self-iterable iterator
        ↓
3. Reusable iterable
        ↓
4. Lazy map/filter
        ↓
5. take/skip
        ↓
6. Iterator cleanup
        ↓
7. Infinite source
        ↓
8. Lazy tree traversal
        ↓
9. Lazy sequence library
        ↓
10. Production iterator abstraction
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain:

```text
Iterable
    ↓
Iterator
    ↓
IteratorResult
```

### Drill 2

Explain why:

```js
const iterator = iterable[Symbol.iterator]();
```

can only be consumed once.

### Drill 3

Explain why:

```js
[...iterable]
```

is not a passive copy operation.

### Drill 4

Explain why iterator cleanup matters for:

```text
files
sockets
locks
transactions
streams
```

### Drill 5

Design a lazy API and explain when it should instead return an Array.

### Drill 6

Explain how an untrusted iterable can become a denial-of-service vector.

### Drill 7

Compare a generator-based implementation with a hand-written iterator.

---

## 29. Completion Criteria

Mark Chapter 25 `[+] Completed` only when the learner can:

- [ ] Define iterable.
- [ ] Define iterator.
- [ ] Define iterator result.
- [ ] Explain `Symbol.iterator`.
- [ ] Explain `next()`.
- [ ] Explain `value` and `done`.
- [ ] Explain reusable versus single-use iteration.
- [ ] Explain `for...of`.
- [ ] Explain `for...in`.
- [ ] Explain spread consumption.
- [ ] Explain iterable destructuring.
- [ ] Explain `Array.from`.
- [ ] Explain Map/Set iterable construction.
- [ ] Explain iterator closing.
- [ ] Explain `return()`.
- [ ] Explain abrupt completion.
- [ ] Explain malformed iterator results.
- [ ] Explain infinite iterables.
- [ ] Explain mutation during iteration.
- [ ] Explain Array iterator behavior.
- [ ] Explain Map/Set iterator behavior.
- [ ] Explain iterator state.
- [ ] Explain lazy iteration.
- [ ] Explain iterator memory retention.
- [ ] Implement a custom iterator.
- [ ] Implement a reusable iterable.
- [ ] Implement lazy transformations.
- [ ] Implement iterator cleanup.
- [ ] Debug iteration protocol violations.
- [ ] Design bounded consumption.
- [ ] Compare Array, Iterable, Iterator, and Generator.
- [ ] Defend a production iteration API.

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

> When should a production API expose materialized data, a reusable iterable, or a stateful iterator—and how should it define mutation, laziness, errors, cleanup, memory, and resource ownership?

---

## Chapter 25 Retrieval Set

### Retrieval 1

What is the difference between an iterable and an iterator?

### Retrieval 2

What does:

```js
Symbol.iterator
```

provide?

### Retrieval 3

What must:

```js
next()
```

return?

### Retrieval 4

Why is:

```js
done
```

different from:

```js
value
```

### Retrieval 5

Why can an iterator be single-use?

### Retrieval 6

Why does:

```js
[...iterator]
```

consume it?

### Retrieval 7

Why does:

```js
for...in
```

differ from:

```js
for...of
```

### Retrieval 8

What is `IteratorClose`?

### Retrieval 9

When might `return()` run?

### Retrieval 10

Why can an iterator retain memory?

### Retrieval 11

Why are infinite iterables valid but dangerous to consume eagerly?

### Retrieval 12

When should a public API return an Array rather than an Iterable?

---

## Chapter 25 Final Mental Model

Remember:

```text
                ITERABLE
                    |
             Symbol.iterator
                    |
                    ↓
                ITERATOR
                    |
                  next()
                    |
                    ↓
             { value, done }
                    |
             done === false
                    |
                    ↓
              consume value
                    |
                  repeat
                    |
             done === true
                    |
                    ↓
                  stop
```

Early termination:

```text
consumer
   ↓
break / throw / early exit
   ↓
IteratorClose
   ↓
iterator.return()
   ↓
cleanup
```

Reusable design:

```text
collection/iterable
      |
      +---- iterator A
      |
      +---- iterator B
      |
      +---- iterator C
```

Single-use design:

```text
iterator
   ↓
state 0
   ↓
state 1
   ↓
state 2
   ↓
done
```

Finally:

> Iteration is not merely "looping over values." It is a protocol for producing traversal state, values, completion, and—when necessary—cleanup. Understanding that protocol is the key to generators, lazy sequences, streams, asynchronous iteration, and robust collection APIs.