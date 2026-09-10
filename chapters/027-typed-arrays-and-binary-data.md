
# Chapter 27 — Typed Arrays / Binary Data

## Chapter Status

`[~] In Progress`

**Part:** IV — Data Structures  
**Primary theme:** Raw binary memory, `ArrayBuffer`, `SharedArrayBuffer`, typed-array views, `DataView`, byte order, binary parsing, memory layout, and safe text/binary boundaries.

---

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

1. Explain what binary data means in JavaScript.
2. Explain why ordinary JavaScript Arrays are not the same as typed arrays.
3. Explain the relationship between:
   - `ArrayBuffer`
   - `SharedArrayBuffer`
   - typed arrays
   - `DataView`
   - `Buffer` in Node.js.
4. Explain the difference between:
   - raw memory/storage,
   - a view over that storage,
   - a decoded numeric value.
5. Explain why:
   ```js
   new ArrayBuffer(8)
   ```
   does not itself provide a sequence of JavaScript Numbers.
6. Explain typed-array element types:
   - `Int8Array`
   - `Uint8Array`
   - `Uint8ClampedArray`
   - `Int16Array`
   - `Uint16Array`
   - `Int32Array`
   - `Uint32Array`
   - `Float16Array`
   - `Float32Array`
   - `Float64Array`
   - `BigInt64Array`
   - `BigUint64Array`
7. Explain typed-array element size and binary representation.
8. Explain why typed arrays do not support ordinary Array holes.
9. Explain how typed arrays differ from ordinary Arrays.
10. Explain the role of:
    ```js
    ArrayBuffer.prototype.byteLength
    ```
11. Explain:
    ```js
    view.byteLength
    view.byteOffset
    view.length
    ```
12. Explain multiple views over the same buffer.
13. Explain aliasing between typed-array views.
14. Explain why changing one view can change another.
15. Explain `DataView`.
16. Explain why `DataView` exists when typed arrays already exist.
17. Explain endianness.
18. Explain:
    - little-endian,
    - big-endian,
    - platform/default byte order,
    - explicit byte order.
19. Explain why `DataView` lets applications specify endianness.
20. Explain common `DataView` methods:
    - `getInt8`
    - `getUint8`
    - `getInt16`
    - `getUint16`
    - `getInt32`
    - `getUint32`
    - `getFloat16`
    - `getFloat32`
    - `getFloat64`
    - `getBigInt64`
    - `getBigUint64`
    - corresponding setters.
21. Explain alignment versus unaligned binary access.
22. Explain why JavaScript binary data is not automatically a C struct.
23. Explain typed-array constructors and their overloads.
24. Explain:
    ```js
    TypedArray.from()
    TypedArray.of()
    ```
25. Explain typed-array `subarray`.
26. Explain typed-array `slice`.
27. Explain the crucial difference:
    - `subarray` creates another view,
    - `slice` creates copied storage.
28. Explain ArrayBuffer transfer and detachment concepts.
29. Explain:
    - `transfer()`
    - `transferToFixedLength()`
    where supported by the target runtime.
30. Explain detached buffers at a conceptual level.
31. Explain resizable ArrayBuffers.
32. Explain growable SharedArrayBuffers where supported.
33. Explain:
    - `maxByteLength`
    - `resizable`
    - `resize()`
34. Explain how resizing interacts with views.
35. Explain fixed-length versus length-tracking views conceptually.
36. Explain `SharedArrayBuffer`.
37. Explain why shared memory requires stronger concurrency reasoning.
38. Explain the role of:
    ```js
    Atomics
    ```
    at a conceptual level.
39. Explain why ordinary typed-array access on shared memory does not automatically provide atomic synchronization.
40. Distinguish:
    - binary representation,
    - atomic synchronization,
    - memory ordering.
41. Explain security implications of shared memory and Spectre-related browser isolation requirements.
42. Explain how Node.js `Buffer` relates to `Uint8Array`.
43. Explain browser binary APIs:
    - Fetch `ArrayBuffer`,
    - Blob,
    - File,
    - Web Streams,
    - TextEncoder/TextDecoder.
44. Explain how binary data moves through HTTP/networking systems.
45. Explain common binary parsing patterns.
46. Explain fixed-width integer parsing.
47. Explain floating-point parsing.
48. Explain byte-level serialization.
49. Implement binary encoders/decoders.
50. Implement a binary packet parser.
51. Implement a simple binary protocol.
52. Debug:
    - offset errors,
    - length errors,
    - endianness bugs,
    - signedness errors,
    - aliasing bugs.
53. Compare typed arrays with:
    - normal Arrays,
    - Strings,
    - Buffer,
    - DataView.
54. Reason about memory alignment, bandwidth, copying, and allocation.
55. Identify resource-exhaustion and malformed-input risks in binary parsers.
56. Defend production binary-data design decisions.

---

## 2. Prerequisites

Recommended prerequisite chapters:

- Chapter 02 — Values / Types / Type System
- Chapter 03 — Numbers / Floating Point / BigInt
- Chapter 07 — Type Conversion / Coercion / Equality
- Chapter 15 — Objects / Property Semantics
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 24 — Map / Set / Weak Collections
- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators

Important future chapters:

- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 39 — Concurrency / Parallelism
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 59 — Node Core APIs
- Chapter 60 — Node Streams
- Chapter 96 — WebAssembly / Native Interoperability

---

## 3. What Is It?

JavaScript has several abstractions for working with binary data.

The most important distinction is:

```text
storage
+
view
+
interpretation
```

### ArrayBuffer

An `ArrayBuffer` represents a fixed-length raw binary data block.

```js
const buffer = new ArrayBuffer(8);
```

This gives storage capacity for:

```text
8 bytes
```

but does not itself say how those bytes represent:

```text
integers
floats
characters
packets
pixels
```

---

### Typed array

A typed array provides a typed indexed view over binary storage.

```js
const buffer = new ArrayBuffer(8);

const view = new Uint32Array(buffer);
```

Now the same bytes are interpreted as fixed-width unsigned 32-bit integers.

---

### DataView

`DataView` provides flexible byte-level access:

```js
const view = new DataView(buffer);
```

You can read different numeric representations at different offsets:

```js
view.getUint16(0, true);
view.getInt32(2, false);
view.getFloat64(0, true);
```

This is especially useful for binary formats where:

- fields have different widths,
- alignment is not uniform,
- byte order is explicitly defined.

---

### SharedArrayBuffer

`SharedArrayBuffer` provides shared memory that can be accessed by multiple agents when the host/runtime permits it.

It is a concurrency primitive, not simply:

```text
ArrayBuffer but bigger.
```

---

### Node.js Buffer

In Node.js:

```js
Buffer
```

is a byte-oriented type built around `Uint8Array` semantics with additional Node-specific APIs.

```js
const buffer = Buffer.from("hello");
```

---

## 4. Why Does It Exist?

Ordinary JavaScript values are excellent for application logic but inefficient or semantically unsuitable for many binary tasks.

Binary protocols require exact control over:

```text
byte width
signedness
offset
endianness
memory layout
encoding
```

Examples include:

- network packets,
- files,
- cryptographic inputs,
- image/audio data,
- WebAssembly memory,
- device protocols,
- compression formats,
- database pages,
- serialization formats.

A normal Array:

```js
[1, 2, 3, 4]
```

stores JavaScript Number values.

It is not equivalent to four bytes.

A typed array:

```js
new Uint8Array([1, 2, 3, 4])
```

expresses:

```text
four 8-bit unsigned values
```

This difference matters for:

- memory,
- interoperability,
- exact representation,
- binary protocols.

---

## 5. Mental Model

Use this picture:

```text
                 BINARY DATA
                      |
              +-------+-------+
              |               |
           storage          storage
              |               |
         ArrayBuffer   SharedArrayBuffer
              |
        +-----+------+
        |            |
   TypedArray      DataView
        |            |
 typed interpretation flexible byte access
```

### Important distinction

```text
ArrayBuffer
    = storage

TypedArray
    = typed view over storage

DataView
    = flexible numeric/byte view over storage
```

### Aliasing

```text
          same buffer
       +---------------+
       | 00 00 00 00   |
       +---------------+
          ↑         ↑
       Uint8      Uint32
       view       view
```

Different views can observe the same underlying bytes.

---

## 6. Core Rules

### Rule 1 — `ArrayBuffer` is raw byte storage

```js
const buffer = new ArrayBuffer(16);
```

allocates 16 bytes of ArrayBuffer storage.

---

### Rule 2 — Typed arrays are views

```js
const view = new Uint8Array(buffer);
```

The view does not require independent storage when constructed over an existing buffer.

---

### Rule 3 — Typed arrays have fixed element types

```js
new Int16Array(...)
```

has 16-bit signed integer elements.

---

### Rule 4 — Typed arrays have no holes

```js
const arr = new Uint8Array(3);
```

Every indexed position has a defined element value.

Fresh elements are initialized according to the typed-array type's rules.

---

### Rule 5 — Typed arrays use fixed-width numeric/BigInt storage

Assignments are converted to the element type.

```js
const arr = new Uint8Array(1);

arr[0] = 300;

console.log(arr[0]);
```

The result is determined by Uint8 conversion semantics, not ordinary Number storage.

---

### Rule 6 — Typed arrays can share storage

```js
const buffer = new ArrayBuffer(8);

const bytes = new Uint8Array(buffer);
const ints = new Uint32Array(buffer);
```

Both reference the same underlying bytes.

---

### Rule 7 — `byteLength` is bytes

```js
view.byteLength
```

reports the byte size of the view.

---

### Rule 8 — `byteOffset` is the view's starting byte offset

```js
view.byteOffset
```

identifies where the view begins within its buffer.

---

### Rule 9 — `length` is element count

```js
view.length
```

counts typed elements rather than bytes.

For:

```js
new Uint32Array(buffer)
```

with a 16-byte view:

```text
byteLength = 16
length = 4
```

---

### Rule 10 — Typed-array element width determines length

```text
Uint8    → 1 byte
Uint16   → 2 bytes
Uint32   → 4 bytes
Float64  → 8 bytes
BigInt64 → 8 bytes
```

The total view size must align with the element width under the typed-array construction rules.

---

### Rule 11 — `DataView` is byte-oriented

`DataView` indexes by byte offsets.

```js
view.getUint32(4, true);
```

reads four bytes starting at byte offset 4.

---

### Rule 12 — `DataView` can specify endianness

```js
view.getUint32(offset, true);  // little-endian
view.getUint32(offset, false); // big-endian
```

---

### Rule 13 — Endianness is about byte order, not numeric type

The number:

```text
0x12345678
```

can be encoded as:

```text
12 34 56 78
```

or:

```text
78 56 34 12
```

depending on byte order.

---

### Rule 14 — `subarray()` shares storage

```js
const sub = typed.subarray(1, 3);
```

The returned view uses the same underlying buffer.

---

### Rule 15 — `slice()` copies values into new storage

```js
const copy = typed.slice(1, 3);
```

The resulting typed array is independent.

---

### Rule 16 — Typed arrays are iterable

```js
for (const value of typed) {
}
```

---

### Rule 17 — Typed arrays integrate with many Array-like methods

Modern typed arrays provide methods such as:

```js
map
filter
reduce
slice
subarray
set
```

but their result and conversion semantics differ from ordinary Arrays in important ways.

---

### Rule 18 — `set()` copies values into an existing typed-array region

```js
target.set(source, offset);
```

---

### Rule 19 — `Array.from` creates ordinary Arrays unless called through a relevant constructor mechanism

```js
Array.from(new Uint8Array([1, 2]));
```

produces a normal Array.

---

### Rule 20 — `Uint8ClampedArray` clamps values

It is designed for byte-like values where values are constrained to the 0–255 range with the specified clamping behavior.

---

### Rule 21 — BigInt typed arrays use BigInt values

```js
const a = new BigInt64Array(2);

a[0] = 10n;
```

Do not assign an ordinary Number where a BigInt is required.

---

### Rule 22 — Number typed arrays and BigInt typed arrays are distinct families

```text
Int32Array
```

uses Number values.

```text
BigInt64Array
```

uses BigInt values.

---

### Rule 23 — ArrayBuffer and typed-array views are not strings

Binary data must be explicitly decoded.

```js
new TextDecoder().decode(bytes);
```

---

### Rule 24 — Buffer is Node-specific

Browser code should not assume:

```js
Buffer
```

exists unless the runtime explicitly provides it.

---

### Rule 25 — Shared memory requires synchronization discipline

Writing to a shared typed array does not automatically establish application-level synchronization.

---

## 7. Syntax

### ArrayBuffer

```js
const buffer = new ArrayBuffer(16);
```

### Typed arrays

```js
new Uint8Array(8)
new Int16Array(4)
new Uint32Array(2)
new Float64Array(2)
new BigInt64Array(2)
```

### Typed array from existing buffer

```js
const view = new Uint32Array(
  buffer,
  byteOffset,
  elementLength,
);
```

### DataView

```js
const view = new DataView(buffer);
```

### DataView access

```js
view.getUint8(offset)
view.getUint16(offset, littleEndian)
view.getUint32(offset, littleEndian)
view.getFloat32(offset, littleEndian)
view.getFloat64(offset, littleEndian)
```

### DataView writes

```js
view.setUint8(offset, value)
view.setUint16(offset, value, littleEndian)
view.setUint32(offset, value, littleEndian)
view.setFloat32(offset, value, littleEndian)
view.setFloat64(offset, value, littleEndian)
```

### Typed-array metadata

```js
typed.length
typed.byteLength
typed.byteOffset
typed.buffer
```

### Shared storage

```js
typed.set(source, offset)
```

### Views

```js
typed.subarray(start, end)
typed.slice(start, end)
```

### Static constructors

```js
Uint8Array.from(iterable)
Uint8Array.of(1, 2, 3)
```

---

## 8. Basic Examples

### Example 1 — Uint8Array

```js
const bytes = new Uint8Array([10, 20, 30]);

console.log(bytes.length);
console.log(bytes.byteLength);
```

Output:

```text
3
3
```

---

### Example 2 — Uint32Array

```js
const values = new Uint32Array([1, 2, 3]);

console.log(values.length);
console.log(values.byteLength);
```

Output:

```text
3
12
```

Each element uses 4 bytes.

---

### Example 3 — Shared buffer between views

```js
const buffer = new ArrayBuffer(4);

const bytes = new Uint8Array(buffer);
const ints = new Uint32Array(buffer);

bytes[0] = 1;

console.log(ints[0]);
```

The exact numeric result depends on the platform's typed-array byte-order behavior, illustrating why `DataView` is preferred when a binary format specifies byte order explicitly.

---

### Example 4 — Subarray aliasing

```js
const values = new Uint8Array([1, 2, 3, 4]);

const sub = values.subarray(1, 3);

sub[0] = 99;

console.log(values);
```

The original view changes because the subarray shares storage.

---

### Example 5 — Slice copying

```js
const values = new Uint8Array([1, 2, 3, 4]);

const copy = values.slice(1, 3);

copy[0] = 99;

console.log(values);
console.log(copy);
```

The original remains unchanged because the sliced result owns copied storage.

---

## 9. Execution Walkthrough

Consider:

```js
const buffer = new ArrayBuffer(8);
const bytes = new Uint8Array(buffer);
const values = new Uint32Array(buffer);

bytes[0] = 0x78;
bytes[1] = 0x56;
bytes[2] = 0x34;
bytes[3] = 0x12;
```

### Step 1 — Allocate storage

The ArrayBuffer represents 8 bytes:

```text
00 00 00 00 00 00 00 00
```

### Step 2 — Create Uint8 view

```text
bytes
byteOffset = 0
byteLength = 8
length = 8
```

### Step 3 — Create Uint32 view

Assuming the buffer length is compatible:

```text
values
byteOffset = 0
byteLength = 8
length = 2
```

### Step 4 — Write byte values

The first four bytes become:

```text
78 56 34 12
```

### Step 5 — Read as Uint32

The same four bytes can be interpreted as one 32-bit value.

The actual number depends on the typed-array view's defined byte-order behavior.

### Step 6 — Why DataView matters

If the binary protocol says:

```text
little-endian
```

write:

```js
new DataView(buffer).getUint32(0, true);
```

If it says:

```text
big-endian
```

use:

```js
new DataView(buffer).getUint32(0, false);
```

The protocol, not the host machine, determines the correct interpretation.

---

## 10. Internal Mechanics

### 10.1 ArrayBuffer as backing storage

An ArrayBuffer provides raw binary storage.

It has a byte length.

Typed arrays and DataView can create views over it.

---

### 10.2 View metadata

A view can be understood using:

```text
buffer
byteOffset
byteLength
element length
element size
```

For example:

```js
const buffer = new ArrayBuffer(16);
const view = new Uint32Array(buffer, 4, 2);
```

Conceptually:

```text
buffer:   00 01 02 03 04 05 06 07 ...
                    ↑
                byteOffset=4

view:
element 0 → bytes 4..7
element 1 → bytes 8..11
```

---

### 10.3 Multiple views

```js
const buffer = new ArrayBuffer(8);

const bytes = new Uint8Array(buffer);
const words = new Uint16Array(buffer);
const floats = new Float32Array(buffer);
```

All may interpret the same raw storage differently.

This is extremely useful for binary protocols, but can become difficult to reason about if the abstraction is not documented.

---

### 10.4 Typed-array conversion

Writing to a typed array converts the assigned value according to its element type.

For example:

```js
const bytes = new Uint8Array(1);

bytes[0] = 256;
```

does not retain the Number `256` as a full-width JavaScript Number.

The result is stored in the element representation.

---

### 10.5 Signed versus unsigned

```text
Int8Array
```

interprets the 8-bit storage using signed two's-complement integer semantics.

```text
Uint8Array
```

interprets it as unsigned.

The same byte:

```text
FF
```

can therefore mean:

```text
255
```

or:

```text
-1
```

depending on interpretation.

---

### 10.6 Floating-point typed arrays

`Float32Array` and `Float64Array` store IEEE-style binary floating-point values according to their defined element representation.

This connects directly to Chapter 03.

---

### 10.7 BigInt typed arrays

```js
new BigInt64Array(...)
```

stores signed 64-bit integer values represented as BigInts.

The API therefore uses:

```js
1n
```

rather than:

```js
1
```

---

### 10.8 `DataView`

DataView does not impose one element type on the entire view.

Instead, each access chooses:

```text
offset
numeric width
signedness
floating/integer interpretation
endianness
```

Example:

```js
view.getUint16(0, true);
view.getFloat32(2, false);
```

---

### 10.9 Unaligned access

DataView allows byte offsets that are not necessarily aligned to the natural size of the numeric type.

For example:

```js
view.getUint32(1, true);
```

reads four bytes starting at byte 1.

This is highly useful for compact binary formats.

---

### 10.10 Typed-array alignment constraints

Typed-array views have element-size alignment requirements for their byte offset and total view size.

For example, a `Uint32Array` view cannot arbitrarily start at a byte offset that violates its element alignment requirements.

DataView is the tool for flexible byte offsets.

---

### 10.11 `subarray`

```js
const sub = typed.subarray(start, end);
```

creates a new view over the same buffer.

Therefore:

```text
same storage
different view boundaries
```

---

### 10.12 `slice`

```js
const copy = typed.slice(start, end);
```

creates independent storage containing copied elements.

---

### 10.13 `set`

```js
target.set(source, offset);
```

copies source elements into target storage.

If source and target overlap through the same buffer, the defined copying semantics must be followed; do not invent manual assumptions about temporary buffers without checking the algorithm.

---

### 10.14 `ArrayBuffer.isView`

```js
ArrayBuffer.isView(value)
```

can identify whether a value is a supported view such as:

- typed array,
- `DataView`.

This is different from:

```js
value instanceof ArrayBuffer
```

because a view is not itself an ArrayBuffer.

---

## 11. ECMAScript / Specification Semantics

### 11.1 ArrayBuffer

The ECMAScript model treats ArrayBuffer as binary data storage with internal state representing its byte length and backing data.

The specification does not require a particular physical memory implementation.

---

### 11.2 SharedArrayBuffer

SharedArrayBuffer differs because its backing data can be shared among agents subject to host/runtime rules.

Its semantics therefore interact with the memory model and Atomics.

---

### 11.3 ArrayBufferView

The conceptual category:

```text
ArrayBufferView
```

covers views over ArrayBuffer-like storage, including typed arrays and DataView.

At specification level, views have relationships involving:

```text
[[ViewedArrayBuffer]]
byteOffset
byteLength
```

and typed-array-specific internal state.

---

### 11.4 TypedArray exotic objects

Typed arrays have specialized semantics for indexed property access and element conversion.

They are not ordinary Arrays.

---

### 11.5 Integer-indexed exotic objects

Typed arrays are examples of integer-indexed exotic objects.

This explains why:

```js
typed[0]
```

has specialized semantics tied to binary element storage.

---

### 11.6 Canonical numeric index strings

Typed-array indexed access involves specialized numeric-index property semantics.

This differs from ordinary Array index handling.

Principal-level reasoning should distinguish:

```text
ordinary Array index
```

from:

```text
typed-array integer index
```

---

### 11.7 `GetValueFromBuffer`

Specification algorithms conceptually read values from backing storage through operations analogous to:

```text
GetValueFromBuffer
```

The operation accounts for:

- byte position,
- numeric type,
- raw bytes,
- memory representation,
- shared versus non-shared storage.

---

### 11.8 `SetValueInBuffer`

Writing typed data conceptually uses:

```text
SetValueInBuffer
```

which converts and stores numeric/BigInt data according to the selected type.

---

### 11.9 DataView accessors

DataView methods are defined around explicit byte offsets and numeric type interpretation.

This is why the caller can specify endianness for multi-byte reads/writes.

---

### 11.10 Endianness

Typed-array element storage semantics do not provide an API for selecting arbitrary wire-format byte order per access.

DataView explicitly accepts the endianness argument.

This makes:

```js
DataView
```

the preferred tool for parsing externally specified binary formats whose byte order must be explicit.

---

### 11.11 `TypedArrayCreate`

Typed-array construction has dedicated semantics for creating typed-array objects.

It distinguishes:

```text
new typed array(length)
new typed array(array)
new typed array(iterable)
new typed array(buffer, offset, length)
```

Each has a different initialization path.

---

### 11.12 `TypedArraySetElement`

Indexed writes use typed-array-specific conversion and storage behavior.

---

### 11.13 `TypedArraySpeciesCreate`

Some typed-array methods can use species-aware construction for result objects.

This connects back to Chapter 21.

---

### 11.14 `TypedArray.prototype.slice`

The typed-array `slice` operation creates a new typed-array result and copies element values.

---

### 11.15 `TypedArray.prototype.subarray`

`subarray` creates a view over existing storage rather than copying values into independent storage.

---

### 11.16 `ArrayBuffer` detachment

ArrayBuffers can have their backing data detached under defined operations.

A detached buffer no longer provides its former byte data to views.

Operations on views over detached buffers have specified behavior, commonly including failure for operations that require actual backing storage.

---

### 11.17 Transfer

Modern ECMAScript includes ArrayBuffer transfer capabilities in environments that support them.

Conceptually:

```text
old buffer
   ↓ transfer
new owner buffer
   ↓
old buffer detached
```

This enables ownership transfer without copying the entire backing store in suitable implementations.

---

### 11.18 Resizable ArrayBuffer

Resizable ArrayBuffers allow buffer byte length to change within a maximum bound under their defined semantics.

Relevant concepts include:

```js
buffer.maxByteLength
buffer.resizable
buffer.resize(newByteLength)
```

These features require careful reasoning about views whose accessible length can change.

---

### 11.19 Length-tracking views

When supported by the relevant API semantics, some views over resizable buffers can track the underlying buffer's current length.

This differs from a fixed-length view created with a fixed element count.

The view contract must therefore be documented carefully.

---

### 11.20 Growable SharedArrayBuffer

Newer ECMAScript environments can expose growable shared backing storage under defined constraints.

As with all modern binary features:

```text
language standard
vs
runtime implementation/version
```

must be distinguished.

---

### 11.21 Atomics

Atomics provide synchronization operations for shared integer typed arrays and related shared-memory scenarios.

The core distinction is:

```text
typed array
    = storage/view

Atomics
    = synchronization/atomic operations
```

Using a shared typed array without proper synchronization can produce data races at the application level.

---

## 12. Advanced Behavior

### 12.1 `Uint8Array` as raw bytes

A common binary-data abstraction:

```js
const bytes = new Uint8Array(buffer);
```

Each element represents one byte-sized value.

This is often the easiest boundary representation for:

- files,
- network payloads,
- encoded text,
- cryptographic inputs.

---

### 12.2 TextEncoder

```js
const bytes = new TextEncoder().encode("hello");
```

produces UTF-8 encoded bytes in a `Uint8Array`.

---

### 12.3 TextDecoder

```js
const text = new TextDecoder().decode(bytes);
```

turns bytes into a String according to the selected encoding.

This establishes a correct boundary:

```text
String
 ↓ encode
bytes
 ↓ decode
String
```

---

### 12.4 Parsing a 16-bit field

```js
function readUint16(bytes, littleEndian = false) {
  const view = new DataView(
    bytes.buffer,
    bytes.byteOffset,
    bytes.byteLength,
  );

  return view.getUint16(0, littleEndian);
}
```

This avoids accidentally ignoring a typed-array view's offset.

---

### 12.5 Respecting `byteOffset`

This is a common bug:

```js
const bytes = new Uint8Array(buffer, 10, 4);

const view = new DataView(bytes.buffer);

view.getUint16(0);
```

This reads from the beginning of the underlying buffer, not from:

```text
bytes[0]
```

Correct:

```js
const view = new DataView(
  bytes.buffer,
  bytes.byteOffset,
  bytes.byteLength,
);
```

This is an important production rule.

---

### 12.6 `subarray` is ideal for zero-copy views

```js
const packet = bytes.subarray(20, 40);
```

No copied storage is required.

This can be useful for high-throughput network processing.

---

### 12.7 Zero-copy is not always free

A subarray can keep the entire original buffer reachable.

```text
small subarray
   ↓
large backing buffer
```

If the large buffer is retained solely through the small view, memory usage can remain high.

---

### 12.8 `slice` can intentionally break retention

Copy:

```js
const packet = bytes.slice(20, 40);
```

The new storage contains only the selected bytes.

This can increase allocation/copy cost while reducing backing-buffer retention.

---

### 12.9 Binary packet parser

Example format:

```text
bytes 0..1 → version (uint16, big-endian)
byte 2     → flags
bytes 3..6 → payload length (uint32, big-endian)
remaining  → payload
```

Implementation:

```js
function parsePacket(bytes) {
  if (!(bytes instanceof Uint8Array)) {
    throw new TypeError("Expected Uint8Array");
  }

  const view = new DataView(
    bytes.buffer,
    bytes.byteOffset,
    bytes.byteLength,
  );

  if (bytes.byteLength < 7) {
    throw new RangeError("Packet too short");
  }

  const version = view.getUint16(0, false);
  const flags = view.getUint8(2);
  const payloadLength = view.getUint32(3, false);

  const payloadStart = 7;
  const payloadEnd = payloadStart + payloadLength;

  if (payloadEnd > bytes.byteLength) {
    throw new RangeError("Invalid payload length");
  }

  return {
    version,
    flags,
    payload: bytes.subarray(payloadStart, payloadEnd),
  };
}
```

This demonstrates:

```text
validation
+
offset tracking
+
explicit endianness
+
zero-copy payload view
```

---

### 12.10 BigInt64 parsing

```js
const view = new DataView(buffer);

const value = view.getBigInt64(0, true);
```

This returns a BigInt:

```text
123n
```

not:

```text
123
```

---

### 12.11 Float16

Modern ECMAScript environments increasingly expose half-precision floating-point typed-array/DataView facilities.

Use:

```js
Float16Array
```

or:

```js
DataView.prototype.getFloat16()
```

only when the target runtime supports them.

Binary formats that require IEEE-like half precision should not be approximated casually with Float32.

---

### 12.12 Node.js Buffer relationship

In Node.js:

```js
const buffer = Buffer.from([1, 2, 3]);
```

Buffer is a specialized byte container built on top of Uint8Array semantics.

Therefore:

```js
Buffer.isBuffer(buffer); // true
buffer instanceof Uint8Array; // true
```

in normal Node environments.

Buffer adds APIs for Node's I/O ecosystem.

---

### 12.13 Buffer views

Node APIs often accept:

```text
Buffer
Uint8Array
ArrayBuffer
DataView
```

depending on the API.

Design code around the narrowest required binary abstraction.

---

### 12.14 Browser Fetch

Fetch responses can be read as:

```js
const buffer = await response.arrayBuffer();
```

and then viewed:

```js
const bytes = new Uint8Array(buffer);
```

---

### 12.15 Blob

A Blob represents immutable binary-like data with MIME metadata.

It can often be converted to:

- ArrayBuffer,
- text,
- streams.

Blob and ArrayBuffer solve related but different problems.

---

### 12.16 File

Browser `File` extends Blob semantics with file metadata.

It is host-platform functionality rather than ECMAScript core language.

---

### 12.17 Streams and typed arrays

Web Streams commonly transport:

```text
Uint8Array
```

chunks for byte-oriented streams.

This connects typed arrays to:

- Fetch,
- compression,
- file APIs,
- streaming parsers.

---

## 13. Edge Cases

### Edge Case 1 — Wrong byte offset

```js
const bytes = new Uint8Array(buffer, 8, 4);
```

Always remember:

```js
bytes.buffer
```

refers to the entire buffer.

Use:

```js
bytes.byteOffset
```

and:

```js
bytes.byteLength
```

when constructing DataView over the exact view range.

---

### Edge Case 2 — Misaligned typed-array offset

```js
new Uint32Array(buffer, 1);
```

is invalid because the offset does not satisfy the required element alignment.

---

### Edge Case 3 — Wrong length multiple

A `Uint32Array` view requires a byte length compatible with 4-byte elements.

Do not assume arbitrary byte counts can form a `Uint32Array`.

---

### Edge Case 4 — Signed interpretation

```js
const bytes = new Uint8Array([0xff]);
```

Reading the same underlying byte through:

```js
new Int8Array(bytes.buffer)[0]
```

produces signed interpretation rather than `255`.

---

### Edge Case 5 — Endianness mismatch

A protocol encoded as big-endian can be silently corrupted when parsed as little-endian.

Always make protocol byte order explicit.

---

### Edge Case 6 — `subarray` aliasing

```js
const a = new Uint8Array([1, 2, 3]);
const b = a.subarray(1);

b[0] = 9;
```

`a` changes.

---

### Edge Case 7 — `slice` copy

```js
const a = new Uint8Array([1, 2, 3]);
const b = a.slice(1);

b[0] = 9;
```

`a` does not change.

---

### Edge Case 8 — Detached buffer

A transferred/detached buffer no longer behaves like an ordinary live backing store.

Code that assumes:

```text
buffer.byteLength unchanged forever
```

can break across transfer operations.

---

### Edge Case 9 — Resizable buffer

Views over resizable buffers can have semantics that differ from fixed-size buffers.

Do not cache:

```js
byteLength
length
```

indefinitely if the underlying storage can change.

---

### Edge Case 10 — Shared memory race

```js
sharedView[0] += 1;
```

is not automatically an atomic increment in the concurrency sense.

Use appropriate Atomics operations when atomicity is required.

---

### Edge Case 11 — `Atomics` type restrictions

Atomics operations do not operate on every typed-array type.

They are designed for specific integer typed arrays and shared-memory cases.

---

### Edge Case 12 — BigInt/Number mixing

```js
const a = 1n;
const b = 1;

a + b;
```

throws because BigInt and Number arithmetic are not implicitly mixed.

The same conceptual distinction applies when working with BigInt typed arrays.

---

### Edge Case 13 — `Uint8ClampedArray`

Assignments outside the supported byte range are clamped according to its defined numeric conversion semantics rather than wrapping exactly like `Uint8Array`.

---

### Edge Case 14 — `NaN` in floating typed arrays

Floating typed arrays can store:

```js
NaN
```

but exact bit patterns and NaN payload preservation should not be assumed to match arbitrary native floating-point bit-level expectations unless the specific semantics guarantee it.

---

### Edge Case 15 — `-0`

Floating-point typed arrays can represent:

```text
+0
-0
```

and code that depends on sign-of-zero must use appropriate observability methods.

---

### Edge Case 16 — Buffer and ArrayBuffer sharing

Node Buffer creation from an existing ArrayBuffer can produce shared views rather than copies depending on the API used.

Always check whether the API:

```text
shares
or
copies
```

before relying on mutation independence.

---

### Edge Case 17 — Large binary lengths

A parser that reads a length field from untrusted input must validate it before slicing/allocating.

Never trust:

```js
payloadLength
```

as an allocation instruction without bounds checking.

---

### Edge Case 18 — Truncated packet

Calling:

```js
view.getUint32(offset)
```

past the available bytes throws.

Validate lengths before reading multi-byte fields.

---

## 14. Common Misconceptions

### Misconception 1 — "Typed arrays are just faster Arrays."

False.

They represent fixed-width typed binary storage and have different semantics.

---

### Misconception 2 — "ArrayBuffer contains Numbers."

False.

It contains raw bytes; views interpret those bytes.

---

### Misconception 3 — "Uint8Array is a String."

False.

It is byte-oriented binary storage.

---

### Misconception 4 — "All byte orders are the same."

False.

Binary protocols can specify little- or big-endian order.

---

### Misconception 5 — "Typed-array view means copied data."

False.

Views generally share backing storage.

---

### Misconception 6 — "`subarray` copies."

False.

It creates another view over the same storage.

---

### Misconception 7 — "`slice` shares storage."

For typed arrays, `slice` creates copied storage.

---

### Misconception 8 — "DataView is slower because it is less typed."

DataView provides flexibility that typed arrays intentionally do not.

Performance must be measured against the actual parsing workload.

---

### Misconception 9 — "Same bytes always produce the same number."

False.

Interpretation depends on numeric type, signedness, and byte order.

---

### Misconception 10 — "SharedArrayBuffer automatically synchronizes threads."

False.

Shared storage and synchronization are separate concepts.

---

### Misconception 11 — "Atomics are required for all SharedArrayBuffer reads."

Not every read requires an atomic operation, but correctness under concurrent mutation requires a deliberate memory-model design.

---

### Misconception 12 — "Buffer is part of JavaScript itself."

Node's Buffer is a Node.js host/runtime abstraction.

---

### Misconception 13 — "Converting bytes to text is automatic."

It is not.

Use explicit encodings such as:

```js
TextEncoder
TextDecoder
```

or host/runtime APIs.

---

### Misconception 14 — "Zero-copy always uses less memory."

A zero-copy view can retain a huge backing buffer through a tiny subview.

---

### Misconception 15 — "Resizable buffers remove allocation concerns."

Resizing changes storage capacity semantics but does not make arbitrary growth free or operationally safe.

---

## 15. Common Mistakes

### Mistake 1 — Ignoring byteOffset

Wrong:

```js
new DataView(bytes.buffer)
```

Correct for an exact view:

```js
new DataView(
  bytes.buffer,
  bytes.byteOffset,
  bytes.byteLength,
)
```

---

### Mistake 2 — Assuming native byte order matches protocol byte order

Use explicit DataView endianness for wire formats.

---

### Mistake 3 — Mixing BigInt and Number

Use:

```js
1n
```

with BigInt typed arrays and arithmetic.

---

### Mistake 4 — Parsing before validating length

Always verify enough bytes exist before reading each field.

---

### Mistake 5 — Allocating attacker-controlled lengths

Validate maximum payload sizes.

---

### Mistake 6 — Accidental aliasing

Know when APIs return:

```text
view
```

versus:

```text
copy
```

---

### Mistake 7 — Retaining tiny views of huge buffers

Use copies when lifecycle requires independent storage.

---

### Mistake 8 — Treating Buffer and Uint8Array as completely interchangeable

They share important semantics but Node Buffer provides additional behavior.

---

### Mistake 9 — Using strings for arbitrary binary

Do not route arbitrary bytes through Unicode text conversions unless encoding is intentional.

---

### Mistake 10 — Reading shared state non-atomically under concurrency

Define synchronization semantics before writing lock-free shared-memory code.

---

### Mistake 11 — Assuming browser and Node binary APIs are identical

Standard ECMAScript, browser APIs, and Node APIs have different layers.

---

### Mistake 12 — Treating binary parsing as trusted input

Network/file/device data can be malformed or adversarial.

---

## 16. Comparison With Related Concepts

| Structure | Storage model | Element typing | Holes | Shared backing possible | Main use |
|---|---|---|---|---|---|
| Array | JS object elements | Dynamic JS values | Yes | Through views, not directly | General sequence |
| Uint8Array | Binary view | 8-bit unsigned | No | Yes | Bytes |
| Int32Array | Binary view | 32-bit signed | No | Yes | Fixed-width integers |
| Float64Array | Binary view | 64-bit float | No | Yes | Numeric binary data |
| BigInt64Array | Binary view | 64-bit signed BigInt | No | Yes | Large integer binary values |
| DataView | Binary view | Per-access type | N/A | Yes | Protocol parsing |
| ArrayBuffer | Raw storage | None | N/A | No, not itself shared | Backing bytes |
| SharedArrayBuffer | Shared raw storage | None | N/A | Yes | Shared-memory concurrency |
| Node Buffer | Node byte view | 8-bit byte-oriented | No | Yes | Node I/O/binary |

### Array versus TypedArray

Array:

```text
dynamic JS values
```

TypedArray:

```text
fixed-width binary values
```

---

### TypedArray versus DataView

TypedArray:

```text
one element type per view
```

DataView:

```text
choose type per access
choose byte order per access
```

---

### ArrayBuffer versus Uint8Array

```text
ArrayBuffer = storage
Uint8Array = view
```

A buffer is not directly indexed like:

```js
buffer[0]
```

---

### ArrayBuffer versus SharedArrayBuffer

ArrayBuffer:

```text
ordinary private backing storage
```

SharedArrayBuffer:

```text
shared backing storage for cooperating agents
```

---

### Uint8Array versus Buffer

Buffer:

```text
Node-specific Uint8Array-derived byte API
```

Uint8Array:

```text
standard ECMAScript typed array
```

---

## 17. Performance Considerations

### 17.1 Memory density

Typed arrays use fixed-width element storage.

For large numeric datasets, this can be substantially more memory-efficient than storing full JavaScript Numbers/objects in normal Arrays.

---

### 17.2 Cache locality

Engines and runtimes can often process typed-array backing storage efficiently.

But do not make universal hardware-level claims without benchmarking.

---

### 17.3 Copy versus view

`subarray` avoids copying:

```text
lower allocation
lower immediate memory cost
shared lifetime
```

`slice` copies:

```text
higher allocation/copy cost
independent lifecycle
```

Choose based on ownership.

---

### 17.4 DataView flexibility

DataView is especially useful for parsing heterogeneous binary formats.

The extra abstraction should be measured in hot loops.

---

### 17.5 Parsing strategy

For very large binary formats:

```text
parse header
validate lengths
create views
process without unnecessary copying
```

can reduce allocation pressure.

---

### 17.6 Bounds checking

Repeated boundary validation has a cost, but removing checks from untrusted input can be a correctness/security bug.

Optimize only after profiling.

---

### 17.7 Transfer versus copy

Transferable ArrayBuffers can move ownership without copying the backing bytes in suitable environments.

This can be valuable for worker communication.

---

### 17.8 Shared memory

SharedArrayBuffer can avoid copying between workers/agents, but synchronization complexity can be much larger than copying.

Do not choose shared memory merely because:

```text
zero-copy sounds faster.
```

---

## 18. Memory Considerations

### Backing-store sharing

Multiple views can all retain the same buffer:

```text
view A
view B
view C
  ↓
large ArrayBuffer
```

The buffer remains reachable while the views remain reachable.

---

### Tiny-view retention

```js
const large = new Uint8Array(100_000_000);
const tiny = large.subarray(0, 10);
```

Keeping `tiny` alive can keep the large backing store relevant to the memory graph.

A copied slice may be appropriate if the original large buffer should be released.

---

### Resizable buffers

Resizable buffers add another lifecycle dimension:

```text
current byte length
maximum byte length
view range
```

Memory allocation can change during the buffer's lifetime.

---

### Shared memory

Shared backing storage is not collectible while actively referenced by agents/objects according to host/runtime lifetime rules.

Design ownership explicitly.

---

### Node Buffer pooling

Node may use internal pooling strategies for Buffer allocations.

Treat this as an implementation detail, not an ECMAScript guarantee.

---

## 19. Security Considerations

### Malformed binary lengths

An attacker can supply:

```text
length = enormous
```

leading to:

- huge allocation,
- excessive slicing,
- CPU exhaustion,
- memory exhaustion.

Always validate maximum sizes.

---

### Integer overflow in parser calculations

Even JavaScript Numbers can represent large values approximately.

When calculating:

```js
payloadEnd = offset + payloadLength;
```

validate ranges and safe-integer assumptions as appropriate.

---

### Endianness confusion

A byte-order bug can change:

```text
length
flags
authorization fields
timestamps
```

into attacker-controlled interpretations.

---

### Out-of-bounds reads

Use explicit bounds checks before:

```js
getUint32()
```

and related reads.

---

### Resource amplification

A small network packet can instruct the application to expand into:

```text
huge decompression
huge object graph
huge allocation
```

Binary parsing must be paired with resource quotas.

---

### Shared-memory side channels

Shared memory has historically been associated with high-resolution timing/side-channel considerations in browser environments.

The platform may restrict shared-memory availability based on isolation/security conditions.

Do not assume:

```js
new SharedArrayBuffer()
```

is equally available in every browser deployment context.

---

### Binary injection

Binary data may eventually become:

```text
text
HTML
SQL
shell commands
file paths
```

Decode and validate before crossing into another domain.

---

### Cryptographic data

Avoid accidental:

```text
text conversion
normalization
encoding changes
```

when handling cryptographic byte sequences.

Cryptographic protocols generally define exact byte-level inputs.

---

## 20. Production Usage

### Use case 1 — Network packet parsing

```js
function parseHeader(bytes) {
  const view = new DataView(
    bytes.buffer,
    bytes.byteOffset,
    bytes.byteLength,
  );

  return {
    version: view.getUint16(0, false),
    flags: view.getUint8(2),
    length: view.getUint32(3, false),
  };
}
```

Always validate minimum length first.

---

### Use case 2 — UTF-8 encoding

```js
const encoder = new TextEncoder();

const bytes = encoder.encode(text);
```

This gives a standard byte representation.

---

### Use case 3 — Decode network text

```js
const decoder = new TextDecoder();

const text = decoder.decode(bytes);
```

---

### Use case 4 — Image/pixel processing

```js
const pixels = new Uint8ClampedArray(pixelBuffer);
```

Useful for byte-like channel values requiring clamping semantics.

---

### Use case 5 — WebAssembly boundary

WebAssembly linear memory is exposed through typed-array views.

Conceptually:

```text
Wasm memory
   ↓
ArrayBuffer-like memory
   ↓
TypedArray/DataView
   ↓
JavaScript
```

This is why typed arrays are essential for JS/Wasm interoperability.

---

### Use case 6 — Worker transfer

Use transferable ArrayBuffers when moving ownership between execution contexts is more appropriate than copying.

---

### Use case 7 — Shared memory

Use SharedArrayBuffer and Atomics only when the architecture genuinely benefits from shared memory.

Document:

```text
ownership
synchronization
memory ordering
lifecycle
termination
```

---

### Use case 8 — Binary file parser

Recommended structure:

```text
input bytes
    ↓
validate overall size
    ↓
read fixed header
    ↓
validate offsets/lengths
    ↓
create views
    ↓
parse fields
    ↓
decode payload
```

---

### Production rule

For every binary format define:

```text
byte order
field widths
signedness
offset rules
maximum sizes
versioning
alignment
encoding
ownership
copy/view policy
error behavior
resource limits
```

---

## 21. Implementation From Scratch

### 21.1 Write a uint16 encoder

```js
function writeUint16(value, littleEndian = false) {
  const buffer = new ArrayBuffer(2);
  const view = new DataView(buffer);

  view.setUint16(0, value, littleEndian);

  return new Uint8Array(buffer);
}
```

---

### 21.2 Read a uint16

```js
function readUint16(bytes, littleEndian = false) {
  if (!(bytes instanceof Uint8Array) || bytes.byteLength < 2) {
    throw new RangeError("Need at least 2 bytes");
  }

  const view = new DataView(
    bytes.buffer,
    bytes.byteOffset,
    bytes.byteLength,
  );

  return view.getUint16(0, littleEndian);
}
```

---

### 21.3 Packet encoder

```js
function encodePacket(version, flags, payload) {
  if (!(payload instanceof Uint8Array)) {
    throw new TypeError("payload must be Uint8Array");
  }

  const headerSize = 7;
  const totalSize = headerSize + payload.byteLength;

  const bytes = new Uint8Array(totalSize);
  const view = new DataView(bytes.buffer);

  view.setUint16(0, version, false);
  view.setUint8(2, flags);
  view.setUint32(3, payload.byteLength, false);

  bytes.set(payload, headerSize);

  return bytes;
}
```

---

### 21.4 Packet decoder

```js
function decodePacket(bytes) {
  if (!(bytes instanceof Uint8Array)) {
    throw new TypeError("Expected Uint8Array");
  }

  if (bytes.byteLength < 7) {
    throw new RangeError("Packet too short");
  }

  const view = new DataView(
    bytes.buffer,
    bytes.byteOffset,
    bytes.byteLength,
  );

  const version = view.getUint16(0, false);
  const flags = view.getUint8(2);
  const payloadLength = view.getUint32(3, false);

  const start = 7;
  const end = start + payloadLength;

  if (end > bytes.byteLength) {
    throw new RangeError("Invalid payload length");
  }

  return {
    version,
    flags,
    payload: bytes.subarray(start, end),
  };
}
```

---

### 21.5 Binary cursor

Create a reusable binary reader:

```js
class BinaryReader {
  #view;
  #offset = 0;

  constructor(bytes) {
    if (!(bytes instanceof Uint8Array)) {
      throw new TypeError("Expected Uint8Array");
    }

    this.#view = new DataView(
      bytes.buffer,
      bytes.byteOffset,
      bytes.byteLength,
    );
  }

  get offset() {
    return this.#offset;
  }

  #require(size) {
    if (this.#offset + size > this.#view.byteLength) {
      throw new RangeError("Unexpected end of input");
    }
  }

  uint8() {
    this.#require(1);

    const value = this.#view.getUint8(this.#offset);

    this.#offset += 1;

    return value;
  }

  uint16LE() {
    this.#require(2);

    const value = this.#view.getUint16(
      this.#offset,
      true,
    );

    this.#offset += 2;

    return value;
  }

  uint32LE() {
    this.#require(4);

    const value = this.#view.getUint32(
      this.#offset,
      true,
    );

    this.#offset += 4;

    return value;
  }
}
```

---

### 21.6 Binary writer

Create a writer abstraction that tracks:

```text
capacity
offset
endianness
```

and expands or rejects when necessary.

---

### 21.7 Conformance test matrix

Test:

```text
empty input
minimum length
maximum valid length
truncated input
invalid length
big-endian
little-endian
signed values
unsigned values
zero values
maximum values
negative values
NaN/infinity for floats
BigInt boundaries
unaligned DataView offsets
subarray offsets
shared backing buffers
detached/transfer cases
```

---

### Implementation progression

**Guided**

- byte reader,
- byte writer,
- uint16/uint32 encoding.

**Partially Guided**

- packet parser,
- binary cursor,
- payload slicing.

**No Reference**

- implement a versioned binary protocol.

**Edge-Case Hardened**

Support:

- malformed offsets,
- integer overflow,
- truncated packets,
- unknown versions,
- maximum payload limits,
- endian modes.

**Production-Grade**

Add:

- fuzz testing,
- benchmarks,
- resource quotas,
- compatibility tests,
- structured errors,
- observability,
- zero-copy/copy policy.

---

## 22. Debugging Exercises

### Exercise 1 — Offset bug

```js
const bytes = new Uint8Array(buffer, 10, 4);
const view = new DataView(bytes.buffer);

console.log(view.getUint16(0));
```

Explain why the result may read the wrong bytes.

---

### Exercise 2 — Endian bug

Encode:

```text
0x1234
```

as big-endian and decode it as little-endian.

Explain the resulting corruption.

---

### Exercise 3 — Signedness bug

```js
const bytes = new Uint8Array([0xff]);
```

Interpret as:

```text
Uint8
Int8
```

and explain the different results.

---

### Exercise 4 — Aliasing bug

```js
const values = new Uint8Array([1, 2, 3]);

const header = values.subarray(0, 2);

header[0] = 9;

console.log(values);
```

Explain the mutation.

---

### Exercise 5 — Memory retention

Create a 100 MB Uint8Array, keep a 1-byte subarray, and reason about why the backing buffer can remain relevant.

---

### Exercise 6 — Truncated input

```js
const bytes = new Uint8Array([1, 2]);

const view = new DataView(bytes.buffer);

view.getUint32(0);
```

Diagnose.

---

### Exercise 7 — Malicious length

Construct a packet whose declared payload length is larger than the actual input.

Ensure the parser rejects it before allocation/slicing.

---

### Exercise 8 — BigInt mismatch

```js
const values = new BigInt64Array(1);

values[0] = 10;
```

Explain the type mismatch.

---

### Exercise 9 — Shared memory

Build a simple worker/shared-memory example and identify where Atomics are needed.

---

### Exercise 10 — Buffer sharing

Create Node Buffer and Uint8Array views over shared backing memory and verify mutation behavior for the specific constructor/API used.

---

## 23. Code Review Exercise

Review:

```js
function parseMessage(buffer) {
  const view = new DataView(buffer);

  const type = view.getUint8(0);
  const length = view.getUint32(1);

  return {
    type,
    payload: new Uint8Array(buffer, 5, length),
  };
}
```

### Questions

1. Is byte order explicit?
2. Is the minimum buffer length validated?
3. Is `length` bounded?
4. Can `5 + length` exceed available data?
5. Is the returned payload copied or shared?
6. Could the small payload retain a huge backing buffer?
7. What happens if `buffer` is a view rather than the original ArrayBuffer?
8. Should the API accept Uint8Array instead?
9. What happens under malformed input?
10. How should versioning work?
11. What observability belongs in parser failures?
12. Should zero-copy be the default?

Propose a production-quality alternative.

---

## 24. Interview Questions

### Fundamentals

1. What is ArrayBuffer?
2. What is a typed array?
3. What is DataView?
4. What is SharedArrayBuffer?
5. What is a view?
6. Why are typed arrays different from Arrays?
7. What does `byteLength` mean?
8. What does `byteOffset` mean?
9. What does typed-array `length` mean?
10. Why do typed arrays have no holes?

### Typed-array types

11. Difference between Int8Array and Uint8Array?
12. Difference between Uint8Array and Uint8ClampedArray?
13. Difference between Number typed arrays and BigInt typed arrays?
14. When would you use Float32Array versus Float64Array?
15. What is Float16Array?
16. Why does BigInt64Array require BigInt values?

### Views

17. What is the difference between `slice` and `subarray`?
18. What happens when two views share the same buffer?
19. Why does `byteOffset` matter?
20. What alignment constraints do typed-array views have?
21. Why is DataView useful for unaligned binary formats?

### Binary protocols

22. What is endianness?
23. Why should wire-format endianness be explicit?
24. Why is DataView appropriate for protocol parsing?
25. How would you parse a packet containing mixed field sizes?
26. How should length fields be validated?
27. What is a common offset bug involving typed-array views?

### Runtime/specification

28. What is an ArrayBufferView?
29. What are integer-indexed exotic objects?
30. What are `GetValueFromBuffer` and `SetValueInBuffer` conceptually?
31. What is ArrayBuffer detachment?
32. What does transfer mean?
33. What are resizable ArrayBuffers?
34. What is a length-tracking view?
35. What is SharedArrayBuffer's relationship to the memory model?
36. What do Atomics provide?

### Node/browser

37. How is Node Buffer related to Uint8Array?
38. How does Fetch expose binary responses?
39. What is the relationship between Blob and ArrayBuffer?
40. Why are Uint8Array chunks common in byte streams?
41. How does TextEncoder bridge String to binary?
42. How does TextDecoder bridge binary to String?

### Performance

43. When can zero-copy improve performance?
44. When can zero-copy increase memory retention?
45. When might copying be preferable?
46. What are the risks of excessive DataView parsing?
47. What are the trade-offs of SharedArrayBuffer versus message passing?

### Security

48. How can malformed length fields create memory exhaustion?
49. How can integer arithmetic create parser bugs?
50. How can endianness errors become security issues?
51. Why is binary input validation essential?
52. What are shared-memory side-channel considerations?
53. Why should cryptographic bytes remain byte-exact?

### Principal-level

54. Design a production binary protocol parser.
55. Decide between zero-copy views and defensive copies.
56. Decide between ArrayBuffer, Uint8Array, DataView, and Buffer.
57. Decide whether SharedArrayBuffer is justified over message passing.
58. Design parser resource limits.
59. Design versioning and forward compatibility.
60. Design observability for malformed binary input.
61. How would you fuzz-test the parser?
62. How would you prove that a parser cannot read past bounds?
63. How would you benchmark binary parsing fairly?
64. How would you design a binary API usable in both browser and Node environments?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
const a = new Uint8Array(4);

console.log(a.length);
console.log(a.byteLength);
```

---

### Exercise B

```js
const a = new Uint32Array(4);

console.log(a.length);
console.log(a.byteLength);
```

---

### Exercise C

```js
const values = new Uint8Array([1, 2, 3]);

const sub = values.subarray(1);

sub[0] = 99;

console.log(values);
```

---

### Exercise D

```js
const values = new Uint8Array([1, 2, 3]);

const copy = values.slice(1);

copy[0] = 99;

console.log(values);
console.log(copy);
```

---

### Exercise E

```js
const buffer = new ArrayBuffer(4);

const bytes = new Uint8Array(buffer);

bytes[0] = 255;

const signed = new Int8Array(buffer);

console.log(signed[0]);
```

---

### Exercise F

```js
const values = new Uint8Array([1, 2, 3, 4]);

console.log(values.byteOffset);
console.log(values.byteLength);
console.log(values.length);
```

---

### Exercise G

```js
const values = new Uint8Array([1, 2, 3]);

console.log(values instanceof Array);
console.log(ArrayBuffer.isView(values));
```

---

### Exercise H

```js
const a = new BigInt64Array(1);

a[0] = 10n;

console.log(a[0]);
```

---

### Exercise I

```js
const bytes = new Uint8Array([0x12, 0x34]);

const view = new DataView(bytes.buffer);

console.log(view.getUint16(0, false));
console.log(view.getUint16(0, true));
```

Predict both values.

---

### Exercise J

```js
const bytes = new Uint8Array([255]);

const signed = new Int8Array(bytes.buffer);

console.log(bytes[0]);
console.log(signed[0]);
```

---

### Exercise K

```js
const bytes = new Uint8Array([1, 2, 3]);

const view = new DataView(
  bytes.buffer,
  bytes.byteOffset,
  bytes.byteLength,
);

console.log(view.getUint8(1));
```

---

### Exercise L

```js
const buffer = new ArrayBuffer(8);

const a = new Uint8Array(buffer);
const b = new Uint32Array(buffer);

a[0] = 1;

console.log(b[0]);
```

Explain why the result should not be memorized as a portable constant without considering byte-order semantics.

---

## 26. Mastery Exercises

### Level 1 — Understand

Explain:

```text
ArrayBuffer
TypedArray
DataView
SharedArrayBuffer
Buffer
```

without treating them as interchangeable.

### Level 2 — Explain

Explain:

```text
storage
vs
view
vs
interpretation
```

### Level 3 — Predict

Predict:

- signedness,
- widths,
- offsets,
- endianness,
- aliasing.

### Level 4 — Implement

Build:

```text
BinaryReader
BinaryWriter
packet encoder
packet decoder
```

### Level 5 — Debug

Fix:

- offset bugs,
- endian bugs,
- truncation bugs,
- signedness bugs,
- aliasing bugs.

### Level 6 — Compare

Compare:

```text
Array
Uint8Array
DataView
ArrayBuffer
Buffer
SharedArrayBuffer
```

using:

- storage,
- typing,
- copying,
- sharing,
- performance,
- security.

### Level 7 — Apply

Implement a versioned binary protocol with:

```text
header
flags
payload length
payload
checksum
```

### Level 8 — Defend

Choose:

```text
zero-copy
vs
defensive copy
```

for a production network parser.

### Level 9 — Principal Judgment

Design a cross-runtime binary-data architecture supporting:

```text
browser
Node
workers
streaming
large payloads
security boundaries
versioning
observability
```

and justify every representation choice.

---

## 27. Key Takeaways

1. ArrayBuffer provides raw binary storage.
2. Typed arrays provide typed indexed views over binary storage.
3. DataView provides flexible per-access binary interpretation.
4. SharedArrayBuffer provides shared backing storage for concurrency scenarios.
5. Node Buffer is a Node-specific byte-oriented abstraction built around Uint8Array semantics.
6. Typed arrays are not ordinary Arrays.
7. Typed arrays have fixed-width element types.
8. Typed arrays do not have holes.
9. `length` counts elements.
10. `byteLength` counts bytes.
11. `byteOffset` identifies the view's starting byte position.
12. Multiple views can share the same backing buffer.
13. `subarray` creates a shared-storage view.
14. `slice` copies values into independent storage.
15. Signed and unsigned views can interpret identical bytes differently.
16. BigInt typed arrays use BigInt values.
17. DataView supports mixed field widths and explicit byte order.
18. Endianness is essential for binary protocol correctness.
19. Typed-array alignment differs from DataView's flexible byte offsets.
20. Binary data must be explicitly encoded/decoded when crossing text boundaries.
21. `byteOffset` must be respected when wrapping an existing typed-array view in DataView.
22. Zero-copy views can reduce copies but can retain large backing buffers.
23. ArrayBuffer transfer can change ownership and detach the original buffer.
24. Resizable ArrayBuffers introduce dynamic storage-size semantics.
25. Shared memory and synchronization are separate concerns.
26. Atomics provide atomic/synchronization operations for supported shared-memory cases.
27. Browser and Node binary APIs overlap but are not identical.
28. Malformed binary length fields are a common resource-exhaustion risk.
29. Principal-level binary engineering requires exact format, ownership, bounds, encoding, and synchronization contracts.

---

## 28. Concept Connections

### Depends On

- Chapter 02 — Values / Types
- Chapter 03 — Numbers / Floating Point / BigInt
- Chapter 07 — Coercion / Equality
- Chapter 15 — Objects / Property Semantics
- Chapter 22 — Arrays
- Chapter 23 — Strings
- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators

### Builds Toward

- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 31 — Async Fundamentals
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 44 — Realms / Agents / Execution Isolation
- Chapter 45 — Memory / GC
- Chapter 47 — JS Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 67 — Dependency / Supply Chain
- Chapter 73 — Core Algorithms
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 96 — WebAssembly / Native Interoperability

### Related Concepts

- bytes
- binary representation
- views
- aliasing
- endianness
- signedness
- IEEE floating point
- BigInt
- memory layout
- buffer transfer
- shared memory
- Atomics
- encoding
- streams
- WebAssembly

### Concepts Revisited

**Chapter 03:**  
Floating-point and BigInt semantics become concrete through Float and BigInt typed arrays.

**Chapter 22:**  
Typed arrays look Array-like but have fixed-width binary semantics rather than ordinary Array object semantics.

**Chapter 23:**  
Text encoding demonstrates the boundary between Unicode Strings and byte sequences.

**Chapter 24:**  
Typed arrays provide data structures specialized for binary/numeric workloads rather than arbitrary collection semantics.

**Chapter 25:**  
Typed arrays are iterable and can participate in generic iteration.

**Chapter 26:**  
Generators and async generators can lazily produce binary chunks for streaming pipelines.

**Chapter 21:**  
Typed-array result-producing methods can interact with subclass/species semantics.

### Why This Chapter Matters Later

Modern JavaScript is not limited to application-level Objects and Arrays.

Production systems increasingly cross:

```text
HTTP
files
streams
workers
WebAssembly
cryptography
images/audio
databases
native modules
```

At these boundaries, exact bytes matter.

A strong engineer must understand:

```text
what bytes exist
how they are viewed
how they are interpreted
who owns the storage
whether data is copied
whether data is shared
how byte order is defined
how malformed data is rejected
```

That is the foundation for reliable binary systems.

---

## Track A — Core Theory

### Level 1 — Intuition

> ArrayBuffer stores bytes; views interpret those bytes.

### Level 2 — Syntax

Know:

```js
ArrayBuffer
SharedArrayBuffer
Uint8Array
Int32Array
Float64Array
BigInt64Array
DataView
```

### Level 3 — Practical

Build:

- byte readers,
- binary writers,
- packet parsers.

### Level 4 — Edge Cases

Understand:

- byte offsets,
- alignment,
- signedness,
- endian differences,
- aliases,
- detached buffers,
- resizable buffers,
- BigInt mismatch.

### Level 5 — Runtime/Internal

Understand:

- ArrayBufferView,
- integer-indexed exotic objects,
- backing stores,
- GetValueFromBuffer,
- SetValueInBuffer,
- shared storage.

### Level 6 — Specification Semantics

Be comfortable with:

- ArrayBuffer construction,
- typed-array construction,
- DataView accessors,
- typed-array element conversion,
- buffer detachment,
- transfer,
- resizing,
- shared memory,
- Atomics.

### Level 7 — Performance/Security

Reason about:

- zero-copy,
- copying,
- backing-buffer retention,
- memory bandwidth,
- malformed lengths,
- parser bounds,
- shared-memory side channels.

### Level 8 — Production Engineering

Design:

- binary protocols,
- safe parsers,
- browser/Node abstractions,
- streaming binary pipelines.

### Level 9 — Interview/Reasoning

Answer:

> Why does DataView exist when typed arrays already provide access to binary data?

### Level 10 — Principal Judgment

Evaluate:

> Should this parser use a copied payload or a zero-copy subarray?

Defend using:

```text
throughput
lifetime
memory retention
ownership
mutation risk
security
API contract
```

---

## Track B — Implementation

The implementation ladder is:

```text
1. Uint8Array byte model
        ↓
2. DataView integer reads
        ↓
3. DataView writes
        ↓
4. BinaryReader
        ↓
5. BinaryWriter
        ↓
6. Packet format
        ↓
7. Versioning
        ↓
8. Validation/fuzz hardening
        ↓
9. Streaming parser
        ↓
10. Production binary protocol stack
```

---

## Track C — Interview / Reasoning

### Drill 1

Explain:

```text
ArrayBuffer
vs
Uint8Array
vs
DataView
```

### Drill 2

Explain why:

```js
new DataView(bytes.buffer)
```

can be wrong for a sliced/subarrayed Uint8Array.

### Drill 3

Explain:

```text
subarray → same backing storage
slice → copied storage
```

### Drill 4

Explain why wire-format endianness must be explicit.

### Drill 5

Choose between:

```text
DataView
TypedArray
```

for:

```text
mixed binary packet
```

### Drill 6

Choose between:

```text
zero-copy
defensive copy
```

for an untrusted network parser.

### Drill 7

Explain why SharedArrayBuffer without synchronization is not a safe lock-free design.

### Drill 8

Design a safe binary parser with:

```text
bounds checks
length limits
version checks
observability
fuzz testing
```

---

## 29. Completion Criteria

Mark Chapter 27 `[+] Completed` only when the learner can:

- [ ] Explain ArrayBuffer.
- [ ] Explain SharedArrayBuffer.
- [ ] Explain typed arrays.
- [ ] Explain DataView.
- [ ] Explain Node Buffer.
- [ ] Distinguish storage from view.
- [ ] Distinguish view from interpretation.
- [ ] Explain typed-array element widths.
- [ ] Explain signedness.
- [ ] Explain BigInt typed arrays.
- [ ] Explain Uint8ClampedArray.
- [ ] Explain byteLength.
- [ ] Explain byteOffset.
- [ ] Explain typed-array length.
- [ ] Explain shared backing storage.
- [ ] Explain aliasing.
- [ ] Explain subarray versus slice.
- [ ] Explain DataView.
- [ ] Explain endianness.
- [ ] Explain explicit little/big-endian access.
- [ ] Explain alignment.
- [ ] Explain byte-order protocol design.
- [ ] Explain `set()`.
- [ ] Explain ArrayBuffer transfer/detachment.
- [ ] Explain resizable ArrayBuffers.
- [ ] Explain shared-memory semantics.
- [ ] Explain Atomics conceptually.
- [ ] Explain text/binary encoding boundaries.
- [ ] Explain browser binary APIs.
- [ ] Explain Node Buffer semantics.
- [ ] Implement BinaryReader.
- [ ] Implement BinaryWriter.
- [ ] Implement packet encoding/decoding.
- [ ] Debug offset/endian/signedness bugs.
- [ ] Handle malformed lengths.
- [ ] Analyze zero-copy memory retention.
- [ ] Compare copy versus view strategies.
- [ ] Design binary parser resource limits.
- [ ] Explain fuzzing strategy.
- [ ] Defend a production binary-data architecture.

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

> How should a production JavaScript system represent, parse, share, copy, validate, encode, and transmit binary data while preserving exact byte semantics, safe memory ownership, and predictable resource usage?

---

## Chapter 27 Retrieval Set

### Retrieval 1

What is the difference between:

```text
ArrayBuffer
Uint8Array
DataView
```

### Retrieval 2

What does:

```js
byteOffset
```

mean?

### Retrieval 3

What is the difference between:

```js
subarray()
```

and:

```js
slice()
```

### Retrieval 4

Why does signedness change the meaning of the same byte?

### Retrieval 5

Why is endianness important?

### Retrieval 6

Why is DataView useful for network packet parsing?

### Retrieval 7

What is the most common bug when creating DataView from a typed-array view?

### Retrieval 8

What is the difference between a copy and a zero-copy view?

### Retrieval 9

Why can a tiny subarray keep a huge buffer relevant?

### Retrieval 10

What does ArrayBuffer transfer conceptually do?

### Retrieval 11

Why does SharedArrayBuffer require synchronization reasoning?

### Retrieval 12

Why must untrusted binary length fields be bounded?

---

## Chapter 27 Final Mental Model

Remember:

```text
                     BINARY DATA
                          |
            +-------------+-------------+
            |                           |
         STORAGE                       VIEW
            |                           |
     +------+-------+            +------+-------+
     |              |            |              |
 ArrayBuffer   SharedArrayBuffer TypedArray   DataView
                                      |
                              typed interpretation
```

And:

```text
Uint8Array
    = one byte-sized element per position

Uint32Array
    = four-byte integer elements

Float64Array
    = eight-byte floating elements

DataView
    = choose type + offset + endianness per access
```

Ownership:

```text
subarray
   ↓
same storage

slice
   ↓
new copied storage
```

Binary protocol:

```text
raw bytes
   ↓
validate size
   ↓
interpret fields
   ↓
validate semantic values
   ↓
decode payload
   ↓
application data
```

Shared memory:

```text
SharedArrayBuffer
       ↓
shared bytes
       ↓
multiple agents
       ↓
Atomics/synchronization when required
```

Finally:

> Binary-data engineering begins where ordinary JavaScript values stop being precise enough. Once bytes cross process, network, file, WebAssembly, or cryptographic boundaries, correctness depends on exact widths, offsets, signedness, endianness, ownership, and validation—not merely on whether the data "looks like an Array."
