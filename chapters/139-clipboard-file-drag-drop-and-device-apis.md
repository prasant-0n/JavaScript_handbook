# Chapter 139 — Clipboard, File, Drag/Drop & Device APIs

> **JavaScript Mastery — Part XXIII: Browser Platform & Human Interface Engineering**
>
> **Mission:** Master browser capabilities that connect JavaScript applications to user-controlled system data, files, drag-and-drop, media devices, sensors, screen capture, sharing, vibration, wake locks, orientation, and related device-facing APIs. Learn their security boundaries, permission models, user activation requirements, lifecycle behavior, browser differences, graceful fallbacks, performance implications, privacy risks, and production architecture.
>
> **Role perspective:** Principal JavaScript Engineer · Browser Platform Engineer · Frontend Platform Engineer · Security Engineer · PWA Engineer · Media Engineer · UX Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A browser device API is not “just another JavaScript API.” It is a capability boundary between a page and the user's operating system, hardware, data, or display. Treat permission, user intent, privacy, lifecycle, and failure as first-class design constraints.**

---

# 1. Learning Objectives

```text
[ ] explain the Clipboard API
[ ] explain readText()
[ ] explain writeText()
[ ] explain read()
[ ] explain write()
[ ] explain ClipboardItem
[ ] explain clipboard events
[ ] understand secure contexts
[ ] understand transient user activation
[ ] understand permission/UA policy around clipboard
[ ] distinguish clipboard read from clipboard write
[ ] understand why clipboard access can fail
[ ] understand clipboard MIME types
[ ] write text to clipboard
[ ] read text from clipboard
[ ] copy rich content
[ ] paste rich content safely
[ ] sanitize pasted HTML
[ ] prevent clipboard injection
[ ] normalize pasted text
[ ] preserve user intent
[ ] implement copy buttons
[ ] implement paste flows
[ ] provide fallback strategies
[ ] explain execCommand historical status
[ ] explain File API
[ ] explain Blob
[ ] explain File
[ ] explain FileList
[ ] explain file input
[ ] explain accept
[ ] explain multiple
[ ] explain directory selection constraints
[ ] read files with arrayBuffer()
[ ] read files with text()
[ ] read files with stream()
[ ] use FileReader
[ ] explain FileReader vs Blob methods
[ ] create object URLs
[ ] revoke object URLs
[ ] preview images
[ ] inspect file metadata
[ ] understand filename trust boundaries
[ ] validate file type safely
[ ] understand MIME type spoofing
[ ] validate file size
[ ] validate content
[ ] stream large files
[ ] upload files
[ ] use File in workers
[ ] understand DataTransfer
[ ] understand dragenter
[ ] understand dragover
[ ] understand dragleave
[ ] understand drop
[ ] distinguish drag-and-drop use cases
[ ] accept OS file drops
[ ] provide file-input fallback
[ ] understand drag data store
[ ] inspect DataTransferItem
[ ] use DataTransfer.files
[ ] understand DataTransferItemList
[ ] set drag data
[ ] read drag data
[ ] prevent accidental browser navigation on file drop
[ ] implement accessible drop zones
[ ] make drag/drop keyboard-accessible
[ ] provide non-drag alternatives
[ ] manage drag visual state
[ ] handle nested dragenter/dragleave
[ ] understand directories in drag/drop
[ ] explain getAsFile()
[ ] understand webkitGetAsEntry historical usage
[ ] understand directory/file-system picker evolution
[ ] explain File System Access concepts
[ ] understand user-selected handles
[ ] understand permissions for file access
[ ] distinguish reading a File from filesystem persistence
[ ] understand device capability APIs
[ ] explain MediaDevices
[ ] explain getUserMedia()
[ ] explain getDisplayMedia()
[ ] explain enumerateDevices()
[ ] explain MediaDeviceInfo
[ ] understand camera permissions
[ ] understand microphone permissions
[ ] understand screen sharing permissions
[ ] understand device labels and permission state
[ ] understand MediaStream
[ ] understand MediaStreamTrack
[ ] stop media tracks
[ ] switch cameras
[ ] select microphone
[ ] use constraints
[ ] understand facingMode
[ ] understand deviceId
[ ] use applyConstraints()
[ ] inspect getCapabilities()
[ ] inspect getSettings()
[ ] handle OverconstrainedError
[ ] handle NotAllowedError
[ ] handle NotFoundError
[ ] handle AbortError
[ ] understand autoplay interactions
[ ] manage camera/mic lifecycle
[ ] avoid leaving cameras active
[ ] understand screen-share lifecycle
[ ] respond to track ended
[ ] explain Image Capture concepts
[ ] explain device orientation
[ ] explain screen orientation
[ ] understand orientation lock constraints
[ ] explain vibration
[ ] explain Web Share API
[ ] explain Web Share target behavior conceptually
[ ] understand sharing permissions
[ ] understand share cancellation
[ ] explain Screen Wake Lock
[ ] request a screen wake lock
[ ] release a wake lock
[ ] handle visibility changes
[ ] handle lock revocation
[ ] understand battery implications
[ ] understand geolocation/device location as a distinct capability class
[ ] understand permission prompts
[ ] design permission UX
[ ] avoid permission spam
[ ] request capabilities in context
[ ] explain secure-context requirements
[ ] explain top-level/embedding restrictions
[ ] explain Permissions Policy concepts
[ ] explain privacy/fingerprinting risks
[ ] understand browser availability differences
[ ] feature-detect device APIs
[ ] design fallback behavior
[ ] test permission-denied states
[ ] test cancellation states
[ ] test unavailable hardware
[ ] test disconnected devices
[ ] test background/hidden-document behavior
[ ] test mobile browser behavior
[ ] test desktop browser behavior
[ ] stream files efficiently
[ ] avoid memory explosions from large blobs
[ ] revoke object URLs
[ ] avoid leaking device streams
[ ] avoid unsafe HTML paste
[ ] protect upload endpoints
[ ] design capability abstraction layers
[ ] build production-grade device adapters
[ ] reason about capability-first UI
[ ] distinguish standardized APIs from browser-specific APIs
[ ] distinguish browser APIs from OS-native APIs
[ ] understand production observability for device features


# 2. Prerequisites

You should already understand:

```text
Chapter 33 — Browser Event Loop
Chapter 49 — DOM Architecture
Chapter 50 — Browser Events
Chapter 51 — Browser Web APIs
Chapter 52 — Workers / Concurrency
Chapter 53 — Streams
Chapter 55 — Fetch / HTTP
Chapter 56 — Browser Security
Chapter 63 — Diagnostics
Chapter 70 — Production Debugging
Chapter 71 — Browser Security
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 101 — Production Scenarios
Chapter 127 — Shared Memory / Security Model
Chapter 132 — Browser Storage Architecture
Chapter 133 — Service Workers
Chapter 135 — WebRTC / Peer-to-Peer JavaScript
Chapter 136 — WebTransport
Chapter 137 — Browser Performance APIs
Chapter 138 — Accessibility Engineering
```

Core mental prerequisites:

```text
Promise
async/await
Blob
ArrayBuffer
Streams
DOM events
permissions
HTTPS
CORS
content security
```

---

# 3. What Are These APIs?

This chapter covers browser capabilities that cross the boundary between:

```text
web application
        ↓
browser
        ↓
operating system / hardware / user data
```

Major capability families:

```text
Clipboard
Files
Drag & Drop
Media devices
Screen capture
Orientation
Wake Lock
Vibration
Web Share
Related device-facing APIs
```

These interfaces share a common property:

```text
the browser must protect the user.
```

Therefore:

```text
permission
user activation
secure context
visibility
sandboxing
privacy policy
```

are part of the programming model.

---

# 4. Capability Mental Model

Treat every device-facing API as:

```text
CAPABILITY REQUEST
        ↓
BROWSER POLICY
        ↓
USER INTENT / PERMISSION
        ↓
RESOURCE GRANTED
        ↓
ACTIVE LIFECYCLE
        ↓
RESOURCE RELEASE
```

The engineering contract includes:

```text
success
denial
cancellation
unavailability
revocation
cleanup.
```

---

# 5. Secure Contexts

Many powerful browser APIs require:

```text
secure context
```

which generally means:

```text
HTTPS
```

with browser-defined trusted local-development exceptions.

Clipboard APIs, for example, require secure contexts. citeturn595815search0turn595815search1

Do not design a production capability around:

```text
http://
```

and expect:

```text
camera
clipboard
wake lock
```

to behave the same.

---

# 6. User Activation

Some powerful actions require a recent:

```text
user gesture / transient user activation.
```

Examples can include:

```text
clipboard operations
screen sharing
sharing
```

This prevents:

```text
silent background code
```

from freely accessing sensitive capabilities.

---

# 7. Permission Is Part of API Semantics

For a device API, the call:

```js
await capability.request();
```

may fail because:

```text
permission denied
permission not yet granted
browser policy
OS policy
user cancellation
device unavailable
document not active
```

Treat these outcomes differently.

---

# 8. Permission UX

Bad:

```text
open site
→ immediately ask camera
→ immediately ask microphone
→ immediately ask notifications
```

Better:

```text
user chooses video call
→ explain why camera is needed
→ request camera
```

Contextual permission requests improve:

```text
trust
clarity
conversion
```

and reduce unnecessary denial.

---

# 9. Feature Detection

Never assume:

```js
navigator.someCapability
```

exists.

Prefer:

```js
if ("clipboard" in navigator) {
  // ...
}
```

and:

```js
if ("mediaDevices" in navigator) {
  // ...
}
```

Use:

```text
feature detection
```

rather than:

```text
browser sniffing.
```

---

# 10. Clipboard API

The Clipboard API provides access to the system clipboard for:

```text
cut
copy
paste
read
write
```

and supports asynchronous operations with Promises. citeturn595815search0turn595815search1

Modern applications should prefer:

```js
navigator.clipboard
```

over historical:

```js
document.execCommand(...)
```

for clipboard access. citeturn595815search0

---

# 11. Clipboard Entry Points

Typical:

```js
navigator.clipboard.writeText(text);

navigator.clipboard.readText();
```

For richer content:

```js
navigator.clipboard.write([...]);

navigator.clipboard.read();
```

where supported. citeturn595815search9turn595815search12

---

# 12. `writeText()`

Example:

```js
async function copy(text) {
  await navigator.clipboard.writeText(text);
}
```

Potential failures:

```text
NotAllowedError
security policy
missing activation
browser restriction
```

Always handle rejection.

---

# 13. Copy Button

Accessible UX:

```html
<button id="copy">Copy</button>
```

```js
copyButton.addEventListener("click", async () => {
  try {
    await navigator.clipboard.writeText(value);
    status.textContent = "Copied.";
  } catch {
    status.textContent = "Copy failed.";
  }
});
```

Do not treat:

```text
clipboard success
```

as guaranteed.

---

# 14. Clipboard Read

```js
const text =
  await navigator.clipboard.readText();
```

Reading is more sensitive than writing.

Modern browser policies may require:

```text
secure context
user activation
permission/prompt
```

depending on browser and context. citeturn595815search0

---

# 15. Clipboard `read()`

For arbitrary clipboard data:

```js
const items =
  await navigator.clipboard.read();

for (const item of items) {
  console.log(item.types);
}
```

A `ClipboardItem` can represent:

```text
multiple MIME types
```

for one clipboard item. citeturn595815search9

---

# 16. Clipboard `write()`

Example:

```js
const item = new ClipboardItem({
  "text/plain":
    new Blob(["Hello"], { type: "text/plain" })
});

await navigator.clipboard.write([item]);
```

Modern browsers can support text, HTML, image formats, and other data according to implementation/support. citeturn595815search12

---

# 17. Clipboard MIME Types

Conceptually:

```text
text/plain
text/html
image/png
```

The clipboard can carry:

```text
representations
```

rather than one universal string.

Choose:

```text
safe primary representation
+
optional richer representation.
```

---

# 18. Rich Copy

A rich editor may write:

```text
text/plain
text/html
```

so destination applications can select:

```text
HTML
```

when they support it, or:

```text
plain text
```

otherwise.

---

# 19. Rich Paste Is Untrusted

Never trust:

```text
clipboard HTML
```

as safe application HTML.

Pasted content can contain:

```text
malicious markup
attributes
URLs
styles
embedded data.
```

Sanitize before inserting into:

```js
innerHTML
```

or a rich-text model.

---

# 20. Clipboard Injection

A malicious clipboard payload can attempt:

```text
script injection
javascript: URLs
event handlers
dangerous CSS
tracking content
```

Use:

```text
HTML sanitizer
allowlist
trusted types/CSP where appropriate
```

rather than:

```js
element.innerHTML = clipboardHtml;
```

---

# 21. Paste as Plain Text

Safe baseline:

```js
const text =
  await navigator.clipboard.readText();

target.textContent = text;
```

This treats clipboard text as:

```text
data
```

rather than:

```text
markup.
```

---

# 22. Clipboard Events

Traditional events include:

```text
copy
cut
paste
```

Example:

```js
element.addEventListener("paste", event => {
  const text =
    event.clipboardData.getData("text/plain");

  console.log(text);
});
```

Use event-level clipboard access when:

```text
the browser's native editing workflow
```

is part of the feature.

---

# 23. Clipboard Event Data

`ClipboardEvent` provides:

```text
clipboardData
```

through a:

```text
DataTransfer
```

interface.

This connects:

```text
clipboard
+
drag/drop
```

through related data-transfer primitives.

---

# 24. Don't Hijack Paste

Bad:

```js
document.addEventListener("paste", e => {
  e.preventDefault();
  // custom global behavior
});
```

This can break:

```text
text editing
password managers
assistive technology
user expectations.
```

Only intercept paste when:

```text
the feature actually requires it.
```

---

# 25. Copy Feedback

After a successful copy:

```text
visual confirmation
+
optional status announcement
```

Use the accessibility principles from:

```text
Chapter 138.
```

Do not:

```text
steal focus.
```

---

# 26. Clipboard Privacy

Clipboard content can contain:

```text
passwords
API keys
documents
personal text
financial data.
```

Therefore:

```text
don't read clipboard continuously
don't upload clipboard contents without clear purpose
don't log raw clipboard contents
```

---

# 27. Clipboard Monitoring

Avoid:

```text
polling clipboard every second.
```

Even when technically possible in some contexts, it is:

```text
privacy-hostile
battery-expensive
user-unfriendly.
```

Build:

```text
explicit user action
```

around reads.

---

# 28. File API

The File API gives web applications access to user-provided files and their contents.

Files can come from:

```text
<input type="file">
drag and drop
other browser APIs
```

and File objects provide information such as:

```text
name
size
type
lastModified
```

File objects can also be used where Blob objects are accepted. citeturn595815search6turn595815search11

---

# 29. Blob

A `Blob` represents:

```text
immutable binary-like data
```

with:

```text
size
type
```

and methods such as:

```text
text()
arrayBuffer()
stream()
slice()
```

---

# 30. File Extends Blob Semantics

A `File` is a specialized:

```text
Blob
```

with additional metadata such as:

```text
name
lastModified
```

This means:

```js
file instanceof Blob
```

is conceptually useful for understanding API compatibility.

---

# 31. File Input

Native:

```html
<input
  id="files"
  type="file"
  multiple
>
```

is the baseline mechanism for:

```text
user chooses files.
```

It also provides:

```text
keyboard access
native OS file picker
browser security model.
```

---

# 32. `accept`

Example:

```html
<input
  type="file"
  accept="image/*"
>
```

This can guide selection.

But:

```text
accept
```

is not:

```text
security validation.
```

The server must still validate uploaded content.

---

# 33. `multiple`

```html
<input
  type="file"
  multiple
>
```

allows multiple file selection.

The resulting:

```js
input.files
```

is a:

```text
FileList.
```

---

# 34. FileList

```js
const files = input.files;

for (const file of files) {
  console.log(file.name);
}
```

Do not assume:

```text
files
```

is a normal mutable Array.

Convert if array methods are needed:

```js
const array = [...files];
```

---

# 35. File Metadata Is Untrusted

A filename:

```text
invoice.pdf
```

does not prove:

```text
valid PDF
```

Similarly:

```text
file.type
```

is not a cryptographic truth about content.

---

# 36. MIME Type Spoofing

An attacker can provide:

```text
malicious.exe
```

with a misleading:

```text
type
```

or:

```text
filename
```

.

Server-side processing should inspect:

```text
actual content
```

according to the application's threat model.

---

# 37. File Size Validation

Client:

```js
if (file.size > MAX_SIZE) {
  throw new Error("Too large");
}
```

is good UX.

But:

```text
server
```

must enforce:

```text
real upload limit.
```

Never trust:

```text
client validation alone.
```

---

# 38. `file.text()`

```js
const content =
  await file.text();
```

Useful for:

```text
small text files
CSV
JSON
configuration
```

where loading the whole file into memory is acceptable.

---

# 39. `file.arrayBuffer()`

```js
const bytes =
  await file.arrayBuffer();
```

Useful for:

```text
binary parsing
image processing
cryptographic hashing
protocol decoding
```

but:

```text
whole-file memory
```

may be expensive for huge files.

---

# 40. `file.stream()`

For large data:

```js
const stream = file.stream();
```

This provides a stream-based path.

Prefer streaming when:

```text
file size is large
processing is incremental
upload can be chunked/streamed.
```

---

# 41. FileReader

Historical asynchronous API:

```js
const reader = new FileReader();

reader.onload = () => {
  console.log(reader.result);
};

reader.readAsText(file);
```

Still useful in environments/codebases that use it, but modern Blob methods such as:

```text
text()
arrayBuffer()
stream()
```

often produce simpler Promise-based code.

---

# 42. Worker File Processing

The File API is available in workers, and FileReaderSync exists specifically in workers. citeturn595815search6

Use workers for:

```text
CPU-heavy parsing
image processing
large-file transformation
```

without blocking the main thread.

---

# 43. Large File Architecture

Bad:

```text
10 GB file
→ arrayBuffer()
→ duplicate
→ JSON parse
→ upload
```

This can cause:

```text
memory explosion
GC pressure
tab crash.
```

Better:

```text
File stream
→ incremental parser
→ bounded chunks
→ upload/processing.
```

---

# 44. Object URLs

Preview:

```js
const url =
  URL.createObjectURL(file);

img.src = url;
```

This allows:

```text
Blob/File
→ browser-consumable URL.
```

---

# 45. Revoke Object URLs

After use:

```js
URL.revokeObjectURL(url);
```

Failure to revoke long-lived object URLs can retain resources longer than necessary.

Use:

```text
create
→ consume
→ revoke
```

as a lifecycle.

---

# 46. Image Preview Security

Do not assume:

```text
image file
```

is harmless.

Potential concerns:

```text
malformed decoder input
large decompression bombs
metadata privacy
unexpected format
server-side parser vulnerabilities
```

Apply:

```text
size limits
dimension limits
server validation
safe processing pipeline.
```

---

# 47. File Upload Architecture

```text
file picker
    ↓
client validation
    ↓
preview
    ↓
upload
    ↓
server validation
    ↓
virus/malware scanning where needed
    ↓
storage
    ↓
processing
```

Never make:

```text
client validation
```

the security boundary.

---

# 48. Upload Progress

For upload progress, choose an API that provides reliable progress semantics for your target browsers and transport.

Conceptually:

```text
bytes sent
÷
total bytes
```

gives:

```text
approximate progress.
```

But streaming/chunked uploads may not have a simple known total.

---

# 49. Drag and Drop API

The Drag and Drop API supports multiple use cases:

```text
drag within page
drag out of page
drag data into page
```

including file drops from the operating system. citeturn595815search10

---

# 50. File Drop Zone

A production file drop zone should have:

```text
drop
+
click
+
keyboard
```

The drop zone should be backed by:

```text
<input type="file">
```

when practical, providing an alternative to dragging. citeturn595815search10

---

# 51. Basic Drop Events

Relevant events:

```text
dragenter
dragover
dragleave
drop
```

Often:

```js
dragover
```

must call:

```js
event.preventDefault();
```

to allow dropping.

---

# 52. File Drop Example

```js
zone.addEventListener("dragover", event => {
  event.preventDefault();
});

zone.addEventListener("drop", event => {
  event.preventDefault();

  const files = event.dataTransfer.files;

  for (const file of files) {
    process(file);
  }
});
```

---

# 53. Prevent Browser Navigation

A common production bug:

```text
user drops PDF onto page
→ browser navigates to PDF.
```

Prevent default handling on the intended:

```text
drop zone
```

and consider a page-level defensive strategy when the application context makes accidental navigation especially dangerous.

---

# 54. `DataTransfer`

`DataTransfer` is the central data object for:

```text
drag/drop
clipboard events
```

and can expose:

```text
files
items
types
```

---

# 55. DataTransfer Items

Use:

```js
for (const item of event.dataTransfer.items) {
  console.log(item.kind, item.type);
}
```

Possible kinds include:

```text
file
string
```

This allows a drop target to distinguish:

```text
file drop
```

from:

```text
text/HTML drag.
```

---

# 56. `getAsFile()`

```js
const file =
  item.getAsFile();
```

returns a File when the item represents a file and the browser exposes it as such.

Handle:

```text
null
```

because:

```text
not every item is a file.
```

---

# 57. Drag Data Store Lifecycle

The browser controls when drag data is:

```text
writable
readable
```

Do not assume:

```text
event.dataTransfer
```

is a persistent unrestricted data store.

Its behavior is tied to the drag lifecycle.

---

# 58. Nested `dragleave`

A common UX bug:

```text
drag enters zone
→ enters child element
→ dragleave fires on zone
→ highlight disappears.
```

Use:

```text
depth counter
```

or:

```text
relatedTarget containment checks
```

to avoid flicker.

---

# 59. Drag Highlight State Machine

```text
idle
 ↓ dragenter
active
 ↓ nested dragenter
active
 ↓ nested dragleave
active
 ↓ final dragleave/drop
idle
```

Do not implement highlight with:

```text
one naive boolean
```

without understanding event nesting.

---

# 60. Accessible Drop Zone

A drop zone should still support:

```text
keyboard focus
Enter/Space activation
file picker
```

through:

```text
native input
```

rather than:

```text
drag as only path.
```

See:

```text
Chapter 138.
```

---

# 61. Dragging Elements

Dragging application elements is different from:

```text
file drop.
```

For internal reordering, prefer an interaction model that has:

```text
keyboard alternative
pointer alternative
clear state
```

and does not require:

```text
HTML drag events
```

if another model is simpler.

---

# 62. Directory Drops

Browsers have supported directory/file-system entry mechanisms in varying forms.

Historical APIs such as:

```text
webkitGetAsEntry()
```

exist in some implementations.

Treat such APIs as:

```text
browser-specific/legacy compatibility mechanisms
```

rather than universal standards.

MDN documents File and Directory Entries as an advanced user-provided directory-reading mechanism with browser-specific history. citeturn595815search14

---

# 63. File System Access Concepts

Modern browsers may provide user-mediated filesystem capabilities such as:

```text
open file
save file
directory handle
```

through newer file-system APIs.

The key model:

```text
user explicitly grants access
→ application receives a handle/capability
```

This differs from:

```text
arbitrary filesystem access.
```

---

# 64. Capability Handles

A file-system handle can represent:

```text
user-approved capability
```

to access a selected file/directory according to browser permissions and the API.

Treat the handle as:

```text
sensitive capability
```

not as:

```text
ordinary JSON data.
```

---

# 65. Persisted Permissions

Some browser APIs can retain permission state.

However:

```text
previous permission
```

does not mean:

```text
future access is guaranteed.
```

Permissions may change due to:

```text
browser policy
site settings
OS state
user revocation
```

---

# 66. Media Devices

Media Capture and Streams APIs provide:

```text
camera
microphone
audio/video streams
tracks
constraints
```

through:

```js
navigator.mediaDevices
```

The system is based on:

```text
MediaStream
+
MediaStreamTrack.
```

citeturn595815search7

---

# 67. `getUserMedia()`

Conceptually:

```js
const stream =
  await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
  });
```

This requests:

```text
camera
+
microphone
```

access.

Use:

```text
only when needed
```

and:

```text
stop tracks when finished.
```

---

# 68. Media Permissions

Camera/microphone access is sensitive.

Users should understand:

```text
which capability
why
when
```

is being requested.

A permission prompt without context can feel:

```text
suspicious.
```

---

# 69. `MediaStream`

A stream contains:

```text
MediaStreamTrack[]
```

For example:

```text
audio track
video track.
```

Inspect:

```js
stream.getTracks();
stream.getVideoTracks();
stream.getAudioTracks();
```

---

# 70. Stop Tracks

When done:

```js
for (const track of stream.getTracks()) {
  track.stop();
}
```

This matters because:

```text
camera hardware
microphone hardware
```

may remain active otherwise.

---

# 71. Media Lifecycle

```text
request
 ↓
permission
 ↓
stream acquired
 ↓
attach
 ↓
use
 ↓
stop
```

A production component needs:

```text
cleanup on unmount
cleanup on navigation
cleanup on call end
```

---

# 72. Device Enumeration

```js
const devices =
  await navigator.mediaDevices.enumerateDevices();
```

This can provide information about:

```text
audio input
audio output
video input
```

subject to:

```text
browser privacy rules
permissions
visibility
```

---

# 73. Device Labels

Before permission is granted, device labels may be:

```text
restricted or unavailable
```

for privacy reasons.

Do not assume:

```text
enumerateDevices()
```

always returns human-readable names.

---

# 74. Device Selection

A device may be targeted with:

```js
{
  video: {
    deviceId: { exact: selectedId }
  }
}
```

But:

```text
device IDs
```

must be treated as browser-scoped opaque identifiers, not stable global hardware IDs.

---

# 75. Camera Facing Mode

For mobile:

```js
{
  video: {
    facingMode: "environment"
  }
}
```

can request:

```text
rear-facing camera
```

while:

```js
facingMode: "user"
```

commonly requests:

```text
front-facing camera.
```

Availability is browser/device dependent.

---

# 76. Media Constraints

Constraints let the application request:

```text
width
height
frameRate
facingMode
deviceId
audio settings
```

They are:

```text
requests/requirements
```

not guarantees.

---

# 77. `ideal` vs `exact`

Conceptually:

```js
{
  width: { ideal: 1280 }
}
```

means:

```text
prefer this value.
```

while:

```js
{
  width: { exact: 1280 }
}
```

means:

```text
require this value.
```

Overly strict constraints can cause:

```text
OverconstrainedError.
```

---

# 78. `getSettings()`

```js
track.getSettings();
```

tells you:

```text
what is actually being used.
```

This is different from:

```text
requested constraints.
```

---

# 79. `getCapabilities()`

```js
track.getCapabilities();
```

can describe:

```text
supported ranges/options
```

where exposed.

Use:

```text
capabilities
→ choose viable constraints
→ apply
```

for adaptive device selection.

---

# 80. `applyConstraints()`

A running track can sometimes be reconfigured:

```js
await track.applyConstraints({
  width: { ideal: 1280 }
});
```

This is useful for:

```text
camera switching
resolution adaptation
frame-rate adjustment
```

---

# 81. Camera Switching

A robust camera switch can:

```text
enumerate
→ select device
→ request new stream/track
→ replace track
→ stop old track.
```

For WebRTC calls:

```text
replaceTrack()
```

may avoid renegotiation in appropriate cases.

See:

```text
Chapter 135.
```

---

# 82. Media Errors

Important classes include:

```text
NotAllowedError
NotFoundError
OverconstrainedError
AbortError
NotReadableError
SecurityError
```

Do not show:

```text
“Camera failed.”
```

for every error.

Map errors to:

```text
user action
```

and:

```text
engineering diagnostics.
```

---

# 83. `NotAllowedError`

Can indicate:

```text
permission denied
policy restriction
```

depending on API/context.

User-facing message:

```text
“Camera permission is required. Check site settings.”
```

may be more useful than:

```text
“NotAllowedError.”
```

---

# 84. `NotFoundError`

Often indicates:

```text
no matching device
```

or:

```text
device unavailable.
```

Explain:

```text
no camera/microphone matching the request.
```

---

# 85. `OverconstrainedError`

This usually means:

```text
requested constraints cannot be satisfied.
```

Recover by:

```text
relaxing constraints
```

instead of:

```text
retrying the same impossible request.
```

---

# 86. Device Disconnect

A camera can disappear because:

```text
USB unplugged
Bluetooth device lost
OS changed default
mobile hardware changed.
```

Monitor:

```js
navigator.mediaDevices.addEventListener(
  "devicechange",
  handler
);
```

where supported.

---

# 87. Track `ended`

Media tracks can end.

Listen:

```js
track.addEventListener("ended", () => {
  // recover
});
```

This is important for:

```text
screen sharing
device removal
permission revocation
```

---

# 88. Screen Capture

`getDisplayMedia()` allows the user to select a:

```text
screen
window
tab
```

or other supported display surface for capture.

The API returns:

```text
MediaStream.
```

Screen Capture is security-sensitive and its browser availability varies; MDN currently classifies the feature as not Baseline. citeturn595815search2turn595815search16

---

# 89. Screen Share Requires User Choice

The application should not silently choose:

```text
arbitrary screen pixels.
```

The browser presents a user-mediated selection flow.

This protects:

```text
private windows
other applications
other browser tabs
```

from silent capture.

---

# 90. Screen Share Lifecycle

```text
request
 ↓
user selects surface
 ↓
stream
 ↓
render/send
 ↓
user stops sharing
 ↓
track ends
```

Listen for:

```text
track ended.
```

---

# 91. Screen Share Indicators

Browser/OS may provide:

```text
visible indicators
```

that screen sharing is active.

Do not design a UI that implies:

```text
sharing ended
```

before the actual track ends.

---

# 92. Screen Share + WebRTC

Typical:

```text
getDisplayMedia()
        ↓
video track
        ↓
RTCPeerConnection
        ↓
remote peers
```

Combine with:

```text
camera/microphone
```

and:

```text
replaceTrack()
```

where appropriate.

See:

```text
Chapter 135.
```

---

# 93. Image Capture

Image Capture can work with:

```text
MediaStreamTrack
```

for camera-specific still-image operations where supported.

Potential workflows:

```text
camera stream
→ ImageCapture
→ photo
```

Treat it as:

```text
browser/device capability
```

with compatibility checks.

---

# 94. Orientation Concepts

Two related concepts can appear:

```text
device/screen orientation information
```

and:

```text
screen orientation locking.
```

The Screen Orientation API exposes current orientation and, in supported contexts, can request an orientation lock. W3C published updated Screen Orientation material in 2026. citeturn595815search15

---

# 95. Orientation Lock

Useful for:

```text
games
camera UI
kiosks
immersive viewers
```

But do not lock orientation without a:

```text
clear product reason.
```

Unexpected locking can:

```text
frustrate users
```

and interfere with:

```text
accessibility
device workflows.
```

---

# 96. Device Orientation vs Screen Orientation

Do not confuse:

```text
where the physical device is facing/tilting
```

with:

```text
how the browser screen is oriented.
```

They are different concepts and APIs.

---

# 97. Vibration

Where supported:

```js
navigator.vibrate(100);
```

can request:

```text
device vibration.
```

Treat it as:

```text
optional enhancement
```

because:

```text
not all devices support it
```

and:

```text
user/device policies vary.
```

---

# 98. Haptic UX

Do not vibrate:

```text
every click
```

.

Reserve haptics for:

```text
important feedback
```

such as:

```text
error
confirmation
game event
```

when suitable.

---

# 99. Web Share API

Web Share allows supported browsers/devices to invoke the platform's:

```text
native sharing UI.
```

Conceptually:

```js
await navigator.share({
  title: "Report",
  text: "Read this report.",
  url: location.href
});
```

Treat it as:

```text
best-effort platform integration.
```

---

# 100. Share Cancellation

A user can:

```text
open share sheet
→ cancel.
```

Cancellation is not necessarily:

```text
application failure.
```

Distinguish:

```text
user cancellation
```

from:

```text
unsupported/failure.
```

---

# 101. Share Files

Where supported:

```js
navigator.canShare({
  files: [file]
});
```

can be used before:

```js
navigator.share({
  files: [file]
});
```

Support can vary by:

```text
browser
OS
file type
file size
```

---

# 102. Wake Lock

The Screen Wake Lock API can request a lock that prevents the display from:

```text
dimming
locking
```

while an active visible document needs to remain awake.

The API is exposed through:

```js
navigator.wakeLock
```

and returns:

```text
WakeLockSentinel
```

when granted. citeturn595815search3turn595815search4

---

# 103. Wake Lock Use Cases

Examples:

```text
recipe display
presentation
navigation
QR scanner
reading
voice/gesture interaction
```

MDN notes these as practical cases for keeping a screen awake. citeturn595815search4

---

# 104. Wake Lock Is Not Guaranteed

A request may fail because of:

```text
low battery
power-saving mode
document not visible
browser policy
system conditions.
```

A granted lock can later be:

```text
released by the system.
```

citeturn595815search4

---

# 105. Wake Lock Lifecycle

```js
let sentinel;

sentinel =
  await navigator.wakeLock.request("screen");

sentinel.addEventListener("release", () => {
  // lock no longer active
});
```

Store the sentinel when the application needs to:

```text
release
observe
reacquire.
```

---

# 106. Reacquire After Visibility

A screen wake lock may be released when the page becomes hidden.

A robust application can:

```text
listen to visibilitychange
→ when visible again
→ request wake lock if product state still requires it.
```

But do not:

```text
reacquire forever
```

when the user no longer needs the feature.

---

# 107. Battery Trade-Off

Keeping a display awake costs:

```text
energy
```

.

Therefore:

```text
wake lock
```

is a:

```text
resource reservation
```

and should have an explicit:

```text
lifecycle.
```

---

# 108. Permissions Policy

Some powerful browser features can be restricted by:

```text
Permissions Policy
```

especially in:

```text
iframes
embedded applications
multi-tenant pages.
```

A feature may work in:

```text
top-level page
```

but fail in:

```text
embedded iframe.
```

---

# 109. Embedding Security

When a parent page embeds your application:

```text
iframe
```

you must account for:

```text
sandbox
Permissions Policy
origin
user activation
```

before promising:

```text
camera
microphone
clipboard
```

behavior.

---

# 110. `allow` on Iframes

Certain capabilities may require iframe permission declarations such as:

```html
<iframe
  src="..."
  allow="camera; microphone">
</iframe>
```

Exact policy depends on:

```text
feature
browser
embedding architecture.
```

---

# 111. Capability Gateway

Build one application-level abstraction:

```js
const capabilities = {
  clipboard,
  files,
  media,
  screenShare,
  share,
  wakeLock
};
```

Each adapter exposes:

```text
isSupported()
request()
release()
status()
```

This keeps components from depending directly on:

```text
browser quirks.
```

---

# 112. Capability Adapter Example

```js
const clipboard = {
  supported() {
    return Boolean(
      navigator.clipboard
    );
  },

  async writeText(text) {
    if (!this.supported()) {
      throw new Error("Clipboard unavailable");
    }

    await navigator.clipboard.writeText(text);
  }
};
```

Later you can add:

```text
fallback
telemetry
policy checks
testing mocks.
```

---

# 113. Permission State Machine

Model:

```text
UNKNOWN
   ↓
REQUESTING
   ↓
GRANTED
   ↓
ACTIVE
   ↓
RELEASED
```

with alternate paths:

```text
DENIED
CANCELLED
UNAVAILABLE
REVOKED
```

This is more robust than:

```text
boolean permission = true;
```

---

# 114. Device Feature State

For camera:

```text
UNAVAILABLE
IDLE
REQUESTING
ACTIVE
STOPPING
ENDED
ERROR
```

Use explicit state transitions.

---

# 115. Device Resources Need Cleanup

Every capability should define:

```text
acquire
use
release
```

Examples:

```text
camera → track.stop()
wake lock → sentinel.release()
object URL → revoke
observer → disconnect
file handle → close/release where applicable
```

---

# 116. Page Lifecycle

Device features must respond to:

```text
visibilitychange
pagehide
beforeunload where relevant
route change
component unmount
```

Do not depend only on:

```text
window.onunload
```

for cleanup.

---

# 117. Hidden Documents

A page becoming hidden can change behavior for:

```text
camera
screen capture
wake lock
timers
media playback
```

The application should define:

```text
pause
continue
release
reacquire
```

behavior intentionally.

---

# 118. Background Behavior

Never assume:

```text
browser continues identical execution
```

when the page is:

```text
backgrounded
locked
suspended
discarded.
```

Build:

```text
resume/recovery paths.
```

---

# 119. Device Enumeration Is Not Inventory

`enumerateDevices()` should not be treated as:

```text
complete hardware inventory.
```

Privacy controls may:

```text
hide devices
reduce labels
change identifiers.
```

Think:

```text
available browser-visible capabilities
```

not:

```text
all physical hardware.
```

---

# 120. Fingerprinting

Device APIs can expose:

```text
number of devices
supported capabilities
camera/mic types
screen characteristics
```

which can contribute to:

```text
fingerprinting.
```

Therefore browsers may:

```text
reduce detail
gate information on permission
```

or otherwise constrain exposure.

---

# 121. Capability Detection vs Fingerprinting

Good:

```text
“Can this user use screen sharing?”
```

Bad:

```text
“Let's collect every exposed device property
and build a hardware profile.”
```

Only collect:

```text
minimum information needed
```

for:

```text
product function
or diagnostics.
```

---

# 122. Media Privacy

Never:

```text
upload raw camera frames
```

unless:

```text
explicitly required and expected.
```

Do not log:

```text
microphone content
```

to analytics.

Telemetry should report:

```text
device type/class
error code
duration
state
```

not:

```text
content.
```

---

# 123. Camera Indicator Alignment

The UI should align with browser reality:

```text
camera active
```

when:

```text
MediaStreamTrack.readyState === "live"
```

and:

```text
track.enabled
```

is not necessarily the same thing as:

```text
hardware capture stopped.
```

Use:

```text
stop()
```

to release the source.

---

# 124. Mute vs Stop

These are different:

```text
track.enabled = false
```

often means:

```text
temporarily disable media contribution
```

while:

```text
track.stop()
```

ends the track and releases its source when no other track requires it.

Choose based on:

```text
pause vs full release.
```

---

# 125. Microphone Mute

For WebRTC:

```js
audioTrack.enabled = false;
```

can commonly represent:

```text
mute.
```

But the microphone device may still be:

```text
open.
```

if the track remains active.

If privacy requires:

```text
hardware release
```

stop the track.

---

# 126. Camera Preview

A typical preview:

```html
<video autoplay playsinline></video>
```

then:

```js
video.srcObject = stream;
```

For mobile, understand:

```text
playsInline
```

and browser-specific autoplay requirements.

---

# 127. Media Autoplay

Autoplay policies can restrict:

```text
audio playback
```

especially when:

```text
unmuted.
```

Design:

```text
user-initiated start
```

for media workflows.

---

# 128. Local Media Preview Security

Do not assume:

```text
local camera preview
```

is network-free forever.

Application code may still:

```text
send frames
record
upload
```

if programmed to do so.

Make the data flow:

```text
explicit
```

and:

```text
observable.
```

---

# 129. Recording Architecture

For browser recording:

```text
MediaStream
↓
MediaRecorder
↓
Blob chunks
↓
bounded queue
↓
upload/storage
```

Do not accumulate:

```text
hours of Blob chunks
```

in memory.

---

# 130. File + Media Integration

A video workflow may combine:

```text
camera capture
+
MediaRecorder
+
Blob
+
File
+
upload.
```

Understand the relationship:

```text
Blob = data container
File = Blob + file metadata
MediaStream = live tracks
```

---

# 131. Clipboard + File Integration

Rich applications may:

```text
paste image
→ ClipboardItem
→ Blob
→ File-like application model
→ preview/upload.
```

The data should still pass:

```text
validation
sanitization
size limits
```

before use.

---

# 132. Drag + File + Upload Integration

A production uploader can support:

```text
click picker
+
drag/drop
+
paste image
```

all into one normalization pipeline:

```text
input source
→ File
→ validation
→ preview
→ upload.
```

---

# 133. Unified File Intake

Normalize:

```text
<input>
DataTransfer.files
ClipboardItem
recorded Blob
```

into:

```text
application FileInput model
```

Example:

```js
{
  source: "drop",
  name,
  type,
  size,
  data
}
```

---

# 134. File Intake Security Pipeline

```text
INPUT
 ↓
extension hint
 ↓
MIME hint
 ↓
size limit
 ↓
signature/content validation
 ↓
malware scan if required
 ↓
storage
```

Never trust:

```text
extension alone
```

or:

```text
browser-provided MIME type alone.
```

---

# 135. Directory Uploads

Large directory uploads can produce:

```text
thousands of files
```

.

Do not:

```text
materialize every file into memory.
```

Prefer:

```text
lazy enumeration
bounded concurrency
streaming uploads
```

and:

```text
progress aggregation.
```

---

# 136. Concurrency-Limited Uploading

Instead of:

```js
await Promise.all(
  files.map(upload)
);
```

for thousands of files, implement:

```text
worker pool
```

with:

```text
concurrency = 3
```

or another measured limit.

This controls:

```text
memory
connections
server load.
```

---

# 137. Hashing Large Files

For integrity/deduplication:

```text
chunk
→ digest
→ combine/protocol-defined hash
```

Use workers when:

```text
CPU-heavy hashing
```

would block the UI.

Do not:

```text
arrayBuffer()
```

huge files unnecessarily.

---

# 138. Streaming Upload Mental Model

```text
File
 ↓
ReadableStream
 ↓
transform
 ↓
request body
 ↓
server
```

When transport and browser support align, this can avoid:

```text
whole-file buffering.
```

---

# 139. Progress Is a Product Contract

For file uploads, distinguish:

```text
queued
uploading
server processing
complete
failed
```

A `100% uploaded` file can still be:

```text
server-processing
```

.

Do not represent:

```text
upload bytes
```

as:

```text
total workflow completion.
```

---

# 140. Offline File Intake

For PWAs:

```text
file selected
→ local queue
→ retry
→ upload later.
```

Possible building blocks:

```text
IndexedDB
Service Worker
Background synchronization patterns
```

where supported.

See:

```text
Chapter 132/133.
```

---

# 141. File Persistence Privacy

A user's selected file may contain:

```text
personal documents
photos
contracts
credentials.
```

Do not persist a file indefinitely just because:

```text
“offline support”
```

exists.

Define:

```text
retention
encryption
deletion
user controls.
```

---

# 142. Object URL vs Data URL

### Object URL

```text
small string reference
→ Blob/File managed by browser
```

### Data URL

```text
content encoded into string
```

Large Data URLs can create:

```text
memory overhead
string duplication
```

Prefer object URLs for:

```text
large previews.
```

---

# 143. Clipboard HTML Sanitization

A robust paste pipeline:

```text
read clipboard
 ↓
identify MIME
 ↓
prefer text/html only when needed
 ↓
sanitize
 ↓
normalize
 ↓
application model
```

Never:

```text
raw HTML
→ trusted DOM.
```

---

# 144. Plain Text Normalization

Pasted text can include:

```text
CRLF
CR
non-breaking spaces
zero-width characters
Unicode normalization differences.
```

Normalize according to:

```text
application semantics.
```

Do not blindly strip:

```text
all Unicode formatting.
```

when text meaning depends on it.

---

# 145. Filenames

Filenames may include:

```text
Unicode
spaces
path-like strings
reserved characters
right-to-left control characters.
```

Display safely and do not interpret the filename as:

```text
trusted HTML
```

or:

```text
filesystem path.
```

---

# 146. Path Traversal

Never construct server filesystem paths directly from:

```text
user filename
```

such as:

```text
uploads/${file.name}
```

.

Normalize and assign:

```text
server-generated storage IDs.
```

---

# 147. File Downloads

When creating downloadable files:

```js
const url =
  URL.createObjectURL(blob);
```

then:

```html
<a
  href="..."
  download="report.csv"
>
```

Treat:

```text
download filename
```

as user-facing data, not trusted filesystem naming.

---

# 148. Downloaded Data Safety

Before opening:

```text
blob URLs
```

in application-controlled contexts, consider:

```text
origin
content type
XSS
sandbox
```

especially when the Blob contains:

```text
HTML
SVG
script-capable content.
```

---

# 149. SVG File Risk

SVG can be:

```text
vector graphics
+
active content
```

depending on context.

Do not automatically inject uploaded SVG into:

```text
innerHTML
```

without:

```text
sanitization
```

and:

```text
trusted rendering strategy.
```

---

# 150. PDF Preview

A PDF preview may be:

```text
browser-native
third-party viewer
download
server conversion
```

.

Do not assume:

```text
File.type === "application/pdf"
```

proves content safety.

---

# 151. Device API Observability

Log:

```text
feature
browser family
support state
request state
error category
duration
release/version
```

Do not log:

```text
raw media
raw clipboard
file contents
private URLs
```

---

# 152. Error Taxonomy

Normalize browser-specific errors into:

```text
UNSUPPORTED
PERMISSION_DENIED
USER_CANCELLED
NO_DEVICE
CONSTRAINT_FAILED
NOT_VISIBLE
POLICY_BLOCKED
RESOURCE_ENDED
UNKNOWN
```

Then expose:

```text
user message
diagnostic code
```

separately.

---

# 153. Support Matrix

Maintain:

```text
feature
desktop
mobile
Safari
Firefox
Chromium
embedded iframe
secure context
permissions
```

because:

```text
“supported in Chrome”
```

is not:

```text
“supported everywhere.”
```

---

# 154. Browser Support Documentation

For current API support, use:

```text
MDN compatibility data
W3C specifications
browser vendor documentation
```

Current MDN material shows:

```text
Clipboard: widely available baseline
File API: widely available
Screen Wake Lock: Baseline 2025
Screen Capture: not Baseline
```

and notes varying support for some subfeatures. citeturn595815search0turn595815search6turn595815search4turn595815search16

---

# 155. Accessibility

Device features must also remain:

```text
keyboard-accessible
screen-reader understandable
```

Examples:

```text
file input
start/stop camera button
share button
copy button
screen share button
wake lock control
```

See:

```text
Chapter 138.
```

---

# 156. Permission UI Accessibility

When requesting permission:

```text
explain why
```

before:

```text
prompt.
```

Don't hide:

```text
permission-dependent action
```

behind inaccessible:

```text
tooltip
```

or:

```text
hover-only explanation.
```

---

# 157. Focus After Permission Prompt

Browser permission dialogs are outside your page's normal DOM focus model.

After the user returns:

```text
page focus state may differ.
```

Design the page so:

```text
the user can continue naturally.
```

---

# 158. User Cancellation

Cancellation is often:

```text
normal user behavior.
```

Examples:

```text
share canceled
file picker canceled
screen-share canceled
permission denied
```

Avoid:

```text
error toast
```

for every expected cancellation.

---

# 159. Retry Strategy

Retry:

```text
transient device failures
```

but do not automatically re-prompt:

```text
permission denied
```

over and over.

Retry should depend on:

```text
failure class
```

not:

```text
catch → retry.
```

---

# 160. Testing Permission States

Test:

```text
granted
denied
prompt
revoked
```

where your automation environment permits.

Also test:

```text
no device
device disconnected
browser unsupported
iframe blocked
HTTP context
hidden tab
```

---

# 161. Mocking Device APIs

Create adapters so unit tests can inject:

```js
fakeClipboard
fakeMediaDevices
fakeWakeLock
fakeShare
```

rather than making components know:

```text
navigator.*
```

directly.

---

# 162. Fake Media Device

Example interface:

```js
class MediaAdapter {
  async requestCamera() {}
  stop() {}
}
```

Production:

```text
browser implementation
```

Test:

```text
deterministic fake.
```

---

# 163. E2E Device Testing

Run browser-level tests for:

```text
file chooser
drag/drop
clipboard
camera/mic
screen sharing
```

where CI environment supports them.

Do not pretend:

```text
unit tests
```

prove:

```text
real device behavior.
```

---

# 164. File Test Fixtures

Create fixtures for:

```text
tiny text
large text
valid image
invalid image
large image
valid PDF
malformed PDF
Unicode filename
path-like filename
zero-byte file
```

---

# 165. Security Test Fixtures

Include:

```text
HTML with script
SVG with active content
malicious filename
oversized file
wrong MIME type
zip bomb
polyglot file where relevant
```

and verify:

```text
client handling
server rejection
```

.

---

# 166. Accessibility Test Fixtures

Verify:

```text
file input labeled
copy button named
drop zone has keyboard path
camera state announced
recording state visible
screen-share stop button accessible
```

---

# 167. Performance Test Fixtures

Measure:

```text
large file preview
image decode
file hashing
directory upload
clipboard parsing
camera startup
screen-share startup
```

and monitor:

```text
main-thread blocking
memory
network
```

.

---

# 168. File Memory Experiment

Compare:

```js
await file.arrayBuffer();
```

against:

```js
const reader =
  file.stream().getReader();
```

for:

```text
1 GB file.
```

Observe:

```text
peak memory
GC
latency
```

---

# 169. Object URL Leak Experiment

Create:

```text
10,000 object URLs
```

then:

```text
revoke
```

and compare memory behavior using:

```text
DevTools memory tooling.
```

---

# 170. Clipboard Benchmark

Measure:

```text
small text
large text
HTML
image
```

and compare:

```text
readText
read
event-driven paste.
```

Do not benchmark with:

```text
real private clipboard data.
```

---

# 171. Media Startup Measurement

Instrument:

```text
request-start
permission-return
stream-ready
first-frame
```

with:

```text
User Timing.
```

Then segment by:

```text
device
browser
camera
network
```

where useful.

---

# 172. Device Startup Critical Path

For camera:

```text
click
→ permission
→ device selection
→ hardware startup
→ track
→ video element
→ first rendered frame
```

This is more useful than:

```text
“getUserMedia took 500 ms”
```

alone.

---

# 173. Screen Share Timing

Measure:

```text
button click
→ picker
→ selection
→ track active
→ first remote frame
```

For WebRTC:

```text
add/replace track
→ signaling
→ remote playback.
```

---

# 174. Wake Lock Timing

Measure:

```text
request
→ granted
→ active
→ release.
```

Failures should be classified by:

```text
visibility
battery/system policy
permission
unsupported.
```

---

# 175. Web Share UX

A share button should degrade to:

```text
copy link
download
open known share URL
```

only when these alternatives actually make sense.

Do not show:

```text
“Share”
```

when:

```text
the platform does not support sharing
```

without:

```text
fallback.
```

---

# 176. Capability-First Rendering

Instead of:

```js
renderCameraButton();
then fail on click
```

prefer:

```text
detect capability
→ render appropriate action
```

while still:

```text
handling runtime failure
```

because detection is not a guarantee.

---

# 177. Capability Detection Is Advisory

This is a critical distinction:

```text
supported = possible to request
```

not:

```text
supported = guaranteed to succeed.
```

A camera can be:

```text
supported
```

but:

```text
permission denied
device absent
OS blocked
```

---

# 178. Device API Abstraction

A robust adapter:

```js
{
  supported,
  request,
  status,
  release,
  onChange
}
```

allows product code to remain:

```text
browser-neutral.
```

---

# 179. Capability Registry

Example:

```js
const capabilityRegistry = {
  clipboard: clipboardAdapter,
  media: mediaAdapter,
  share: shareAdapter,
  wakeLock: wakeLockAdapter
};
```

Consumers call:

```js
capabilities.media.requestCamera();
```

rather than:

```js
navigator.mediaDevices.getUserMedia(...);
```

everywhere.

---

# 180. State + Capability Architecture

Use:

```text
application state
+
capability adapter
+
UI
```

rather than:

```text
UI directly controls raw browser resource.
```

This improves:

```text
testing
cleanup
browser compatibility
observability.
```

---

# 181. Resource Ownership

Every resource should have one owner:

```text
CameraController owns stream.
UploadController owns upload queue.
WakeLockController owns sentinel.
PreviewController owns object URLs.
```

This avoids:

```text
double-stop
leaks
races
```

---

# 182. Race Conditions

Example:

```text
user clicks Start Camera
→ request pending

user clicks Stop
→ before request resolves

permission resolves
→ camera unexpectedly starts.
```

Solve with:

```text
request generation token
AbortSignal where supported/applicable
explicit state machine.
```

---

# 183. Stale Async Result

Pattern:

```js
const requestId = ++currentRequest;

const stream = await request();

if (requestId !== currentRequest) {
  stopStream(stream);
  return;
}
```

This prevents:

```text
late request
→ resurrects dead state.
```

---

# 184. Permission Race

Similar race:

```text
request A
request B
A resolves last.
```

Make capability requests:

```text
serialized
or generation-aware.
```

---

# 185. File Intake Race

User selects:

```text
file A
```

then immediately:

```text
file B.
```

If image preview A finishes after B:

```text
A can overwrite B.
```

Use:

```text
request identity
```

to reject stale results.

---

# 186. Device Change Race

Device list changes while:

```text
user selects camera.
```

Revalidate:

```text
device still available
```

before committing.

---

# 187. Upload Cancellation

Large file uploads should support:

```text
cancel
```

where the transport allows it.

Use:

```text
AbortController
```

to cancel:

```text
fetch
parsing
processing
```

where applicable.

---

# 188. File Processing Cancellation

Worker-based parsing can support:

```text
worker termination
```

or:

```text
application cancellation protocol.
```

Do not continue expensive processing after:

```text
user deleted the file.
```

---

# 189. Directory Traversal Safety

When recursively processing directory contents:

```text
limit depth
limit file count
limit total size
limit concurrency
```

to avoid:

```text
resource exhaustion.
```

---

# 190. Quotas and Limits

Device-facing applications can hit:

```text
memory
storage
upload limits
browser limits
OS limits
```

Define:

```text
bounded behavior
```

for:

```text
file count
file size
queue length
recording duration
```

---

# 191. Recording Duration Limits

A browser recording application should not permit:

```text
unbounded recording in RAM.
```

Use:

```text
chunked recording
periodic upload
storage quotas
maximum session length.
```

---

# 192. Media Resource Limits

Video resolution affects:

```text
CPU
GPU
memory
network bandwidth
battery.
```

Do not request:

```text
4K
60fps
```

just because:

```text
the camera supports it.
```

Choose based on:

```text
actual product requirements.
```

---

# 193. Constraint Strategy

Prefer:

```text
ideal
```

for:

```text
preferred quality
```

and:

```text
exact
```

only when:

```text
the feature fundamentally requires it.
```

---

# 194. Adaptive Video

For a call:

```text
good device
→ 720p/1080p
```

for:

```text
low-end device
→ lower resolution/frame rate.
```

Use:

```text
actual track settings
+
network conditions
```

to adapt.

---

# 195. Clipboard UX and Accessibility

After:

```text
copy
```

announce:

```text
Copied.
```

through appropriate status UI.

After:

```text
paste
```

avoid:

```text
duplicate announcements
```

from:

```text
native editing
+
custom live region.
```

---

# 196. File Picker UX and Accessibility

Use a real:

```html
<input type="file">
```

whenever possible.

A custom “Upload” button can:

```js
input.click();
```

but the input should remain:

```text
semantically integrated
```

and:

```text
labelled.
```

---

# 197. Drag Zone UX

Show:

```text
Drop files here
or choose files
```

with:

```text
real button/input.
```

Don't communicate:

```text
drag only.
```

---

# 198. Screen Share UX

Before requesting:

```text
screen sharing
```

show a clear action:

```text
Share screen
```

and explain:

```text
what will be shared
```

before the browser picker.

---

# 199. Camera UX

Show:

```text
Start camera
```

then:

```text
camera active indicator
Stop camera
```

so the user always understands:

```text
capture state.
```

---

# 200. Microphone UX

Use:

```text
Mute
Unmute
```

and make the difference clear:

```text
muted ≠ hardware stopped.
```

When appropriate, also offer:

```text
Stop microphone.
```

---

# 201. Privacy-by-Default

Design:

```text
no capability
→ until needed
```

and:

```text
release immediately
→ when feature ends.
```

This minimizes:

```text
risk
battery
memory
user concern.
```

---

# 202. Security Architecture

For every capability ask:

```text
What data can enter?
What data can leave?
Who controls permission?
What can an iframe do?
What happens without HTTPS?
What can malicious input contain?
What is logged?
What persists?
What is uploaded?
```

---

# 203. Principal Threat Model

### Clipboard

```text
secret leakage
malicious HTML
cross-application data exposure
```

### Files

```text
malware
parser exploits
oversized uploads
path traversal
PII
```

### Media

```text
camera/mic surveillance
recording leakage
permission abuse
```

### Screen capture

```text
confidential screen disclosure
```

### Device APIs

```text
fingerprinting
privacy leakage
resource abuse
```

---

# 204. Server-Side File Security

The backend should enforce:

```text
size
content type/content signature
storage isolation
filename normalization
virus/malware scanning where required
authorization
access control
download policy
retention
```

Client checks improve UX but do not create the security boundary.

---

# 205. File Storage Isolation

Avoid storing uploads directly under:

```text
public web root
```

unless the security model explicitly supports it.

Prefer:

```text
opaque object key
authorized retrieval
correct response headers
```

---

# 206. Content-Disposition

For downloads, the server should carefully control:

```http
Content-Disposition
Content-Type
Content-Security-Policy
```

according to file type and application context.

---

# 207. Clipboard Content Security

If supporting rich text:

```text
sanitize
normalize
validate
```

before:

```text
persist
render
execute.
```

The browser should never be treated as:

```text
trusted editor boundary.
```

---

# 208. Device Permission Audit

For each device feature document:

```text
permission prompt
required secure context
embedding restrictions
user activation
release path
denial path
logging policy
fallback
```

---

# 209. Production Support Matrix

Maintain a living table:

```text
Feature
API
Secure Context
User Activation
Permission
Desktop
Mobile
Iframe
Fallback
```

Example:

```text
Clipboard Write
Clipboard API
HTTPS
often gesture-sensitive
browser policy
supported
supported
policy-dependent
copy fallback
```

---

# 210. Production Decision Framework

For every new device API:

```text
1. Is native browser capability necessary?
2. What user problem does it solve?
3. What data does it expose?
4. What permission is needed?
5. What happens when denied?
6. What happens when unsupported?
7. What happens when hardware disappears?
8. How is the resource released?
9. What is the battery cost?
10. What is the memory cost?
11. What is the privacy risk?
12. What is the security risk?
13. What is the accessibility path?
14. What is the fallback?
15. How is it tested?
16. How is it observed?
```

---

# 211. Implementation From Scratch — Capability Layer

Build:

```text
CapabilityRegistry
ClipboardAdapter
FileAdapter
DropAdapter
MediaAdapter
ScreenShareAdapter
ShareAdapter
WakeLockAdapter
DeviceStateMachine
```

Each adapter should expose:

```text
supported()
request()
status()
release()
```

and:

```text
error normalization.
```

---

# 212. Implementation Milestone 1 — Clipboard Adapter

```js
class ClipboardAdapter {
  supported() {
    return !!navigator.clipboard;
  }

  async writeText(text) {
    if (!this.supported()) {
      throw new Error("UNSUPPORTED");
    }

    await navigator.clipboard.writeText(text);
  }
}
```

Add:

```text
fallback
telemetry
permission error mapping
tests.
```

---

# 213. Implementation Milestone 2 — File Intake Adapter

Normalize:

```text
input.files
DataTransfer.files
ClipboardItem
```

into:

```js
{
  source,
  file
}
```

Then run:

```text
size validation
type validation
deduplication
preview.
```

---

# 214. Implementation Milestone 3 — Drop Adapter

Implement:

```text
drag depth
drop
file extraction
keyboard fallback
```

and test:

```text
nested children
multiple files
text drag
unsupported file.
```

---

# 215. Implementation Milestone 4 — Media Adapter

Expose:

```js
await media.startCamera();
await media.startMicrophone();
await media.stop();
```

Internally manage:

```text
stream
tracks
state
generation
error normalization.
```

---

# 216. Implementation Milestone 5 — Device State Machine

Implement:

```text
IDLE
REQUESTING
ACTIVE
STOPPING
ENDED
ERROR
```

Transitions must be:

```text
explicit
observable
testable.
```

---

# 217. Implementation Milestone 6 — Screen Share

Implement:

```text
start
stop
ended
```

and:

```text
track ended
```

handling.

---

# 218. Implementation Milestone 7 — Wake Lock

Implement:

```text
acquire
release
visibility recovery
system release
```

and expose:

```text
active
inactive
unsupported.
```

---

# 219. Implementation Milestone 8 — Capability Registry

```js
const registry = {
  clipboard: new ClipboardAdapter(),
  media: new MediaAdapter(),
  drop: new DropAdapter(),
  wakeLock: new WakeLockAdapter()
};
```

Consumers should depend on:

```text
interfaces
```

rather than:

```text
browser globals.
```

---

# 220. Implementation Milestone 9 — Error Normalizer

Map:

```text
NotAllowedError
NotFoundError
OverconstrainedError
AbortError
NotSupportedError
```

into:

```text
PermissionDenied
DeviceMissing
ConstraintFailure
Cancelled
Unsupported
```

---

# 221. Implementation Milestone 10 — Capability Telemetry

Record:

```text
feature
state
errorCode
browserClass
appVersion
duration
```

Never record:

```text
raw file content
clipboard content
camera frames
microphone audio
```

---

# 222. Implementation Milestone 11 — Resource Ownership

Every adapter gets:

```text
acquire
release
dispose
```

and tests:

```text
double release
release before acquire
acquire twice
dispose while active.
```

---

# 223. Implementation Milestone 12 — Chaos Testing

Simulate:

```text
permission denied
user cancels
device disappears
browser blocks
page hidden
route changes
request races
network fails
large input
```

and ensure:

```text
state returns to safe condition.
```

---

# 224. Debugging Exercises

## Exercise A — Clipboard Failure

```text
Copy button works in dev,
fails in production.
```

Investigate:

```text
HTTPS
user activation
browser policy
iframe
Permissions Policy.
```

---

## Exercise B — Pasted HTML XSS

A rich editor directly assigns:

```js
editor.innerHTML = clipboardHtml;
```

Find:

```text
injection path
```

and redesign:

```text
clipboard
→ sanitizer
→ editor model.
```

---

## Exercise C — File Preview Crash

A user uploads:

```text
3 GB video.
```

The browser tab crashes.

Find:

```text
whole-file buffering
image/video decode
duplicate copies
```

and replace with:

```text
bounded/streaming processing.
```

---

## Exercise D — Drop Zone Flicker

Highlight repeatedly changes when:

```text
drag enters children.
```

Implement:

```text
depth tracking.
```

---

## Exercise E — Camera Doesn't Stop

User closes camera panel, but browser indicator remains active.

Find:

```text
stream retained
track.stop() missing
```

and fix ownership.

---

## Exercise F — Camera Race

User:

```text
start
stop
start
```

quickly.

A stale request activates the wrong stream.

Implement:

```text
request generation.
```

---

## Exercise G — Wake Lock Disappears

The lock works, then disappears after:

```text
screen/tab visibility change.
```

Implement:

```text
visibility-aware reacquisition.
```

---

## Exercise H — Screen Share Ends

User presses browser's:

```text
Stop sharing
```

but application still displays:

```text
“Sharing.”
```

Fix:

```text
MediaStreamTrack ended
```

state synchronization.

---

# 225. Code Review Exercise — File Upload

Review:

```js
async function upload(file) {
  if (!file.name.endsWith(".jpg")) {
    throw new Error("Invalid");
  }

  const body = await file.arrayBuffer();

  await fetch("/upload", {
    method: "POST",
    body
  });
}
```

Identify:

```text
extension-only validation
whole-file buffering
no size limit
no server validation
no cancellation
no progress model
no content sniffing
no retry classification
no auth discussion
```

---

# 226. Code Review Exercise — Camera

Review:

```js
async function start() {
  const stream =
    await navigator.mediaDevices.getUserMedia({
      video: true
    });

  video.srcObject = stream;
}
```

Find:

```text
no feature detection
no error normalization
no cleanup
no stop path
no stale-request protection
no track ended handling
no UI state
```

---

# 227. Code Review Exercise — Clipboard

Review:

```js
document.addEventListener("paste", event => {
  const html =
    event.clipboardData.getData("text/html");

  editor.innerHTML = html;
});
```

Find:

```text
XSS risk
global paste interception
no sanitization
no fallback
no MIME strategy
no accessibility consideration.
```

---

# 228. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
await navigator.clipboard.writeText("hello");
```

Question:

```text
Will this always succeed from arbitrary JavaScript execution?
```

Explain:

```text
secure context
user activation
browser policy.
```

---

### Exercise 2

```js
const url =
  URL.createObjectURL(file);

img.src = url;
```

Question:

```text
What happens if the URL is never revoked
in a long-lived application?
```

---

### Exercise 3

```js
const stream =
  await navigator.mediaDevices.getUserMedia({
    video: true
  });

video.srcObject = stream;
video.srcObject = null;
```

Question:

```text
Does removing srcObject necessarily stop the camera?
```

Explain:

```text
track lifecycle
```

---

### Exercise 4

```js
track.enabled = false;
```

Question:

```text
Does this necessarily release the camera device?
```

Explain:

```text
mute/disable vs stop.
```

---

### Exercise 5

```js
const files =
  [...input.files];

files.forEach(file => {
  file.arrayBuffer();
});
```

Question:

```text
Why can this be dangerous for hundreds of large files?
```

---

### Exercise 6

A user drops a file onto a webpage without:

```js
event.preventDefault();
```

Predict a possible browser-level consequence.

---

### Exercise 7

A screen-share track receives:

```text
ended
```

but application state remains:

```text
ACTIVE
```

Predict:

```text
what the user sees.
```

---

### Exercise 8

A wake lock is granted, then document visibility becomes:

```text
hidden
```

Predict:

```text
whether the application can rely on the original sentinel
remaining active forever.
```

---

# 229. Interview Questions

### Clipboard

```text
1. Why does the Clipboard API require stronger security controls than ordinary string APIs?
2. What is the difference between readText() and read()?
3. What is ClipboardItem?
4. Why is pasted HTML untrusted?
5. Why shouldn't an application continuously read the clipboard?
```

### Files

```text
6. What is the difference between Blob and File?
7. What is FileList?
8. Why is file.type not a security boundary?
9. When would you use file.stream()?
10. Why must uploaded files be validated on the server?
```

### Drag/Drop

```text
11. What is DataTransfer?
12. Why does dragover often need preventDefault()?
13. How do you build a keyboard-accessible drop zone?
14. How do you avoid dragenter/dragleave flicker?
15. How do you process directories safely?
```

### Media

```text
16. What does getUserMedia() return?
17. What is the difference between MediaStream and MediaStreamTrack?
18. Why call track.stop()?
19. What is the difference between enabled=false and stop()?
20. What are media constraints?
```

### Screen Share

```text
21. How does getDisplayMedia() differ from getUserMedia()?
22. How do you detect when screen sharing ends?
23. Why can't a page silently choose any display surface?
```

### Device / Platform

```text
24. What is a secure context?
25. What is transient user activation?
26. What is Permissions Policy?
27. Why does device capability detection not guarantee success?
28. How would you design a capability abstraction layer?
29. How would you test permission-denied and hardware-disconnected states?
30. How would you prevent device APIs from becoming resource leaks?
```

### Principal

```text
31. Design a browser capability platform for a large SPA.
32. How would you unify file picker, drag/drop, and paste into one intake pipeline?
33. How would you design a production camera component?
34. How would you prevent clipboard HTML injection?
35. How would you handle capability differences across browsers?
36. How would you instrument device feature failures without collecting sensitive content?
37. How would you handle races between permission requests and UI state?
```

---

# 230. Mastery Exercises

### Exercise 1 — Universal File Intake

Build:

```text
file picker
drag/drop
paste image
```

into:

```text
one FileInput pipeline.
```

### Exercise 2 — Production Uploader

Build:

```text
validation
preview
concurrency limit
progress
cancellation
retry
server correlation
```

### Exercise 3 — Camera Component

Build:

```text
start
stop
device selection
constraints
switch camera
error states
track ended
```

### Exercise 4 — Screen Sharing

Build:

```text
start
stop
track ended
WebRTC integration
```

### Exercise 5 — Clipboard Editor

Build:

```text
copy
plain paste
rich paste
HTML sanitization
status announcements.
```

### Exercise 6 — Wake Lock Controller

Build:

```text
request
release
visibility recovery
system release
```

### Exercise 7 — Capability Registry

Build:

```text
feature detection
adapter
permission state
resource ownership
telemetry
```

### Exercise 8 — Chaos Test Suite

Simulate:

```text
denied
cancelled
unsupported
device missing
device removed
page hidden
race
large input
```

and verify:

```text
safe state.
```

---

# 231. Track A — Core Theory

Master:

```text
Clipboard API
Clipboard events
File API
Blob
File
FileList
DataTransfer
Drag and Drop
File System concepts
MediaDevices
getUserMedia
getDisplayMedia
MediaStream
MediaStreamTrack
constraints
device enumeration
orientation
Web Share
Vibration
Wake Lock
Permissions Policy
secure contexts
user activation
privacy
fingerprinting
resource lifecycle
```

Deliverable:

```text
explain every capability as a browser-controlled resource boundary.
```

---

# 232. Track B — Implementation

Build:

```text
ClipboardAdapter
FileIntakePipeline
DropAdapter
UploadController
MediaAdapter
CameraController
ScreenShareController
WakeLockController
CapabilityRegistry
PermissionStateMachine
DeviceStateMachine
TelemetryAdapter
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```

---

# 233. Track C — Interview / Reasoning

Practice:

```text
“Why does this API require user activation?”

“Why can file.type not be trusted?”

“Why is track.stop() important?”

“How would you design a keyboard-accessible drop zone?”

“How would you unify files from input/drop/paste?”

“How would you handle browser support differences?”

“How would you prevent a stale camera request from reviving after stop?”

“How would you monitor device failures without collecting sensitive data?”
```

Deliverable:

```text
capability
+
permission
+
resource ownership
+
fallback
+
security
+
trade-off.
```

---

# 234. Performance Considerations

Device APIs can be expensive.

### Files

```text
decode
hashing
parsing
copying
```

### Media

```text
camera
microphone
encoding
decoding
GPU
```

### Clipboard

```text
large HTML/image conversion
```

### Uploads

```text
concurrency
compression
encryption
network
```

### Wake Lock

```text
battery.
```

Measure:

```text
CPU
memory
latency
network
battery impact
```

before optimizing.

---

# 235. Memory Considerations

Common hazards:

```text
large arrayBuffer
duplicate Blob
object URL leaks
recording chunks
unbounded upload queue
image bitmap accumulation
stale MediaStreams
```

Apply:

```text
streaming
bounded queues
revocation
cleanup
concurrency limits.
```

---

# 236. Security Considerations

Treat all user-provided capability data as:

```text
untrusted.
```

Especially:

```text
clipboard HTML
filenames
file contents
drag data
device metadata
screen-capture content.
```

Use:

```text
sanitization
validation
authorization
server-side enforcement
data minimization.
```

Never make:

```text
client-side capability access
```

equivalent to:

```text
server-side trust.
```

---

# 237. Reliability Considerations

Production device flows must handle:

```text
permission denial
permission revocation
browser changes
hardware changes
device disconnect
page visibility
iframe restrictions
request races
stale results
cleanup failures
unsupported features.
```

The desired state is:

```text
safe and recoverable
```

not:

```text
everything always succeeds.
```

---

# 238. Common Misconceptions

### Misconception 1

```text
“HTTPS means the API will always work.”
```

Reality:

```text
secure context is only one condition.
```

### Misconception 2

```text
“file.type proves the file is safe.”
```

Reality:

```text
metadata is not content validation.
```

### Misconception 3

```text
“setting video.srcObject = null turns off the camera.”
```

Reality:

```text
stream/track ownership must be released.
```

### Misconception 4

```text
“supported API means successful capability.”
```

Reality:

```text
permission/device/policy can still fail.
```

### Misconception 5

```text
“drag/drop is enough for file upload.”
```

Reality:

```text
provide picker and keyboard alternatives.
```

---

# 239. Common Mistakes

```text
[ ] reading clipboard without user intent
[ ] injecting clipboard HTML directly
[ ] trusting file extension
[ ] trusting file.type
[ ] buffering huge files
[ ] forgetting revokeObjectURL
[ ] uploading every dropped file automatically
[ ] allowing unlimited directory uploads
[ ] forgetting preventDefault on drop
[ ] naive dragleave logic
[ ] no keyboard drop alternative
[ ] leaving MediaStream tracks alive
[ ] confusing mute with stop
[ ] using overly strict media constraints
[ ] not handling OverconstrainedError
[ ] not handling track ended
[ ] assuming device labels are always present
[ ] assuming device IDs are global stable IDs
[ ] assuming getDisplayMedia is universally supported
[ ] keeping Wake Lock forever
[ ] treating user cancellation as an error
[ ] repeatedly prompting denied permissions
[ ] logging sensitive capability content
```

---

# 240. Specification / Runtime Source Discipline

Primary sources should be:

```text
W3C / WHATWG standards
MDN Web API documentation
browser vendor documentation
Permissions Policy documentation
WebRTC specifications where media overlaps
```

Maintain distinctions:

```text
standardized
vs
experimental
vs
browser-specific
vs
historical
```

Examples:

```text
Clipboard API
→ standardized browser capability

File API
→ standardized

Screen Capture
→ standardized capability with varying browser support

webkitGetAsEntry
→ historical/browser-specific mechanism

vendor-specific device APIs
→ do not assume universal support.
```

The W3C continues publishing current File API and Clipboard API specifications, including 2026 working-draft publications. citeturn595815search5turn595815search17

---

# 241. Current Platform Notes

As of September 2026:

```text
Clipboard API:
widely available, secure-context dependent,
with varying requirements for read/write operations.

File API:
widely available and usable from workers.

Drag/drop file workflows:
widely available, but directory-specific mechanisms
have historical/browser-specific variations.

Screen Capture:
available in modern browsers but remains
not Baseline according to current MDN compatibility guidance.

Screen Wake Lock:
Baseline 2025 and broadly available on modern devices,
while still subject to system conditions.

Media Capture:
widely used but sensitive permissions and
browser/OS behavior remain part of the contract.

Screen Orientation:
standardized capability with browser/context
constraints.

Device-facing APIs should always be feature-detected
and tested against actual target browsers/devices.
```

Current MDN documents Clipboard as widely available while noting secure-context and permission/user-activation constraints; File API is widely available and worker-capable; Screen Capture remains not Baseline; and Screen Wake Lock is Baseline 2025. citeturn595815search0turn595815search6turn595815search16turn595815search4

---

# 242. Production Accessibility Checklist

```text
[ ] native file input exists
[ ] drop zone has keyboard alternative
[ ] copy/share buttons have accessible names
[ ] camera/mic state is visible
[ ] screen-share state is visible
[ ] stop controls are accessible
[ ] permission explanation is accessible
[ ] dynamic errors are announced appropriately
[ ] focus remains logical after picker closes
[ ] device state changes are communicated
[ ] no drag-only interaction
```

---

# 243. Production Security Checklist

```text
[ ] secure context enforced
[ ] Permissions Policy reviewed
[ ] iframe permissions reviewed
[ ] clipboard reads minimized
[ ] pasted HTML sanitized
[ ] filenames treated as untrusted
[ ] MIME values treated as hints
[ ] server validates files
[ ] upload limits enforced
[ ] storage isolated
[ ] object URLs revoked
[ ] raw media never logged
[ ] raw clipboard never logged
[ ] device data minimized
[ ] permission denial handled
```

---

# 244. Production Resource-Lifecycle Checklist

```text
[ ] every stream has an owner
[ ] every track has stop path
[ ] every object URL has revoke path
[ ] every wake lock has release path
[ ] every worker has termination path
[ ] every upload has cancellation path
[ ] every async request handles stale results
[ ] every capability has unsupported path
[ ] page visibility handled
[ ] route transition cleanup handled
[ ] component unmount cleanup handled
```

---

# 245. Principal Capability Architecture

Use:

```text
UI
 ↓
Capability Controller
 ↓
Capability Adapter
 ↓
Browser API
 ↓
Permission / Policy
 ↓
OS / Hardware
```

And for user data:

```text
clipboard/file/drop
        ↓
normalization
        ↓
validation
        ↓
sanitization
        ↓
application model
        ↓
storage/upload
```

This creates:

```text
clear trust boundaries
```

and:

```text
testable architecture.
```

---

# 246. Principal Resource Ownership Model

```text
One resource
        ↓
one controller
        ↓
one lifecycle
        ↓
one release path
```

Examples:

```text
CameraController → MediaStream
PreviewController → object URLs
UploadController → upload tasks
WakeLockController → WakeLockSentinel
ClipboardController → clipboard operations
```

---

# 247. Final Mental Model

```text
USER INTENT
    ↓
CAPABILITY CHECK
    ↓
SECURE CONTEXT / POLICY
    ↓
PERMISSION / USER ACTIVATION
    ↓
RESOURCE ACQUISITION
    ↓
ACTIVE WORK
    ↓
OBSERVABILITY
    ↓
RELEASE / CLEANUP
```

Every branch can fail:

```text
unsupported
denied
cancelled
blocked
unavailable
revoked
disconnected
raced
expired
```

A principal engineer designs:

```text
all of these states.
```

---

# 248. Dependency Graph

```text
Chapter 33
Browser Event Loop
        ↓
Chapter 49
DOM
        ↓
Chapter 50
Events
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 52
Workers
        ↓
Chapter 53
Streams
        ↓
Chapter 55
Fetch
        ↓
Chapter 56
Browser Security
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Production Debugging
        ↓
Chapter 83
Observability
        ↓
Chapter 85
Performance
        ↓
Chapter 86
Testing
        ↓
Chapter 127
Shared Memory / Security
        ↓
Chapter 132
Browser Storage
        ↓
Chapter 133
Service Workers
        ↓
Chapter 135
WebRTC
        ↓
Chapter 136
WebTransport
        ↓
Chapter 137
Browser Performance APIs
        ↓
Chapter 138
Accessibility
        ↓
Chapter 139
Clipboard, Files, Drag/Drop & Device APIs
```

Cross-cutting:

```text
Permissions
Security
Privacy
Accessibility
Performance
Observability
Concurrency
Lifecycle
```

---

# 249. Concept Connections

## Depends On

```text
DOM
events
Promises
Blob/File
Streams
Fetch
browser security
permissions
workers
WebRTC
accessibility
performance.
```

## Builds Toward

```text
PWA engineering
browser capability platforms
media applications
file-processing applications
collaboration tools
web editors
upload platforms
device-aware web applications.
```

## Related Concepts

```text
Clipboard
File
Blob
DataTransfer
MediaDevices
MediaStream
MediaStreamTrack
getDisplayMedia
Wake Lock
Web Share
Screen Orientation
Permissions Policy
secure contexts.
```

## Concepts Revisited

```text
Events
Async
Streams
Workers
Security
Privacy
Accessibility
Performance
Observability
State machines
Cancellation
```

## Why This Chapter Matters

Browser APIs that touch:

```text
system clipboard
files
camera
microphone
screen
device state
```

are some of the strongest examples of:

```text
host capability
```

inside JavaScript.

Mastery requires more than memorizing:

```text
navigator.mediaDevices
navigator.clipboard
URL.createObjectURL
```

.

You must understand:

```text
permission
policy
user intent
resource ownership
cleanup
security
privacy
accessibility
performance
failure
fallback.
```

---

# 250. Retrieval Record

```md
# Chapter 139 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Clipboard API
-

## Clipboard Security
-

## File API
-

## Blob / File
-

## File Validation
-

## Object URLs
-

## Drag/Drop
-

## DataTransfer
-

## File System Concepts
-

## MediaDevices
-

## getUserMedia
-

## MediaStream / Track
-

## Constraints
-

## Device Enumeration
-

## Screen Capture
-

## Orientation
-

## Web Share
-

## Wake Lock
-

## Permissions Policy
-

## Secure Contexts
-

## User Activation
-

## Privacy / Fingerprinting
-

## Resource Ownership
-

## Cancellation / Races
-

## Accessibility
-

## Performance
-

## Testing
-

## Implementation Progress
-

## Strongest Areas
-

## Weakest Areas
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 251. Spaced Retrieval Schedule

### Day 0

Study:

```text
Clipboard
File
DataTransfer
MediaDevices.
```

### Day 1

Explain:

```text
secure context
user activation
permission
policy
```

### Day 3

Build:

```text
file picker + drag/drop
```

### Day 7

Build:

```text
upload pipeline
```

with:

```text
streaming
concurrency
cancellation.
```

### Day 14

Build:

```text
camera controller
```

### Day 21

Build:

```text
screen-share + WebRTC flow
```

### Day 30

Build:

```text
capability registry
+
resource ownership
+
permission/error state machine
```

without notes.

---

# 252. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
use the APIs in normal examples.
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly:

```text
forget permissions
forget cleanup
trust file metadata
confuse stream/track lifecycle
```

Mark:

```text
[+] Completed
```

when you can:

```text
build file, clipboard, and media workflows
with fallbacks and tests.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
design
implement
debug
secure
test
observe
and defend
```

a browser capability architecture across:

```text
clipboard
files
drag/drop
media
screen sharing
wake lock
device state
```

Reading alone does not mark mastery.

---

# 253. Final Principal Principle

> **Powerful browser APIs should be engineered as capabilities with explicit acquisition, policy, ownership, lifecycle, privacy, security, accessibility, and release—not as ordinary helper functions.**

The production sequence is:

```text
identify user need
→ check capability
→ explain permission
→ acquire under browser policy
→ maintain explicit state
→ process safely
→ observe failures
→ release resources
→ provide fallback
→ test the denied/disconnected/unsupported cases
```

For user data:

```text
INPUT
→ VALIDATE
→ SANITIZE
→ NORMALIZE
→ PROCESS
→ STORE/UPLOAD
```

For device resources:

```text
REQUEST
→ GRANT
→ ACTIVATE
→ MONITOR
→ RELEASE
```

For browser portability:

```text
STANDARD
→ FEATURE DETECT
→ SUPPORT MATRIX
→ FALLBACK
→ REAL DEVICE TEST
```

The principal question is:

```text
“What capability is the user intentionally granting,
what is the smallest amount of data/resource we need,
what can fail or be revoked, who owns the resource,
how do we clean it up, what does the user experience when
the browser refuses, and how do we prove the feature is
safe, accessible, performant, and portable?”
```

That is browser capability engineering.