# Chapter 131 — URI, URL, Encoding & URL Parsing Semantics

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Master URI/URL semantics in JavaScript well enough to reason about parsing, serialization, percent-encoding, Unicode, query strings, origins, paths, credentials, fragments, relative references, canonicalization, redirects, SSRF, open redirects, signature construction, and browser/Node interoperability.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript/Platform Specialist · Browser Engineer · Node.js Engineer · Security Engineer · Networking Engineer · API Architect
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A URL is structured data, not just a string. Encode according to the component and parse before comparing, validating, or modifying it.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] distinguish URI, URL, URN, and URL-reference concepts
[ ] explain the WHATWG URL model at a practical level
[ ] distinguish URL parsing from string manipulation
[ ] explain scheme, authority, userinfo, host, hostname, port, origin, path, query, fragment
[ ] explain absolute vs relative references and base resolution
[ ] explain dot-segment processing
[ ] explain percent-encoding and UTF-8 byte representation
[ ] distinguish encoding from escaping
[ ] choose encodeURI vs encodeURIComponent correctly
[ ] explain decodeURI/decodeURIComponent and URIError
[ ] explain application/x-www-form-urlencoded and URLSearchParams
[ ] handle duplicate query parameters correctly
[ ] understand query ordering and canonicalization
[ ] explain default ports and origin semantics
[ ] understand Unicode hostnames / IDN / Punycode concepts
[ ] identify Unicode confusable and normalization risks
[ ] explain URL serialization and mutation
[ ] distinguish path, query, and fragment encoding contexts
[ ] explain protocol-specific URL behavior for http/https/ws/wss/file/data/blob
[ ] explain browser vs Node URL handling
[ ] identify open redirect vulnerabilities
[ ] identify SSRF and redirect-chain risks
[ ] identify parser confusion, double-encoding, and double-decoding bugs
[ ] explain canonicalization/signing problems
[ ] design explicit URL allowlists
[ ] design safe redirect and fetch policies
[ ] test malformed, Unicode, encoded, and adversarial URLs
[ ] use URLPattern appropriately when supported
[ ] know when to use URL APIs instead of RegExp or manual string parsing
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 04 — Strings / Unicode / Text Semantics
Chapter 07 — Type Conversion / Coercion / Equality
Chapter 23 — String APIs
Chapter 28 — JSON / Serialization / Structured Clone
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 49 — DOM Architecture
Chapter 51 — Browser Web APIs
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JavaScript Security Engineering
Chapter 64 — ES Modules
Chapter 79 — API Design
Chapter 83 — Observability
Chapter 85 — Performance
Chapter 88 — Debugging Methodology
Chapter 98 — Anti-Patterns / Failure Modes
Chapter 123 — ECMAScript Grammar / Parsing
Chapter 128 — Intl Deep Dive
Chapter 129 — Regular Expressions
Chapter 130 — Legacy Date / Time Zones
```

Supporting concepts:

```text
UTF-8
Unicode
HTTP
DNS
TCP
origin
authentication
canonicalization
serialization
```

---

# 3. What Is a URL?

A URL is a structured reference to a resource or processing target.

Example:

```text
https://user:pass@example.com:8443/docs/page.html?mode=dark#overview
```

Conceptual structure:

```text
scheme      https
username    user
password    pass
host        example.com:8443
hostname    example.com
port        8443
path        /docs/page.html
query       ?mode=dark
fragment    #overview
```

The practical JavaScript parser is:

```js
new URL(input, base)
```

The browser/platform URL model is primarily specified by the WHATWG URL Standard; ECMAScript separately specifies URI encoding/decoding functions.

---

# 4. URI, URL, URN

Use the standards vocabulary carefully:

```text
URI
├── URL
└── URN
```

A URL provides location/access semantics. A URN identifies by name. Modern JavaScript web development most often deals with URL references and WHATWG URL processing.

Do not collapse every identifier string into “a URL.”

---

# 5. Why URL Parsing Matters

Dangerous:

```js
if (input.startsWith("https://example.com")) {
  // allow
}
```

A prefix check proves only a text property.

Security decisions need parsed structure:

```js
const url = new URL(input);
```

Then validate the actual policy-relevant component:

```js
url.protocol
url.hostname
url.port
url.origin
url.pathname
```

---

# 6. URL as Structured Data

Think:

```text
URL reference
   ↓
parser
   ↓
structured URL record
   ↓
component semantics
   ↓
serializer
```

Do not think:

```text
URL = arbitrary string containing :, /, ?, #
```

That model leads to incorrect parsing and security validation.

---

# 7. URL Components

Common components:

```text
scheme
userinfo
host
port
path
query
fragment
```

Example:

```text
https://alice:secret@example.com:8443/a/b?x=1&x=2#top
```

The authority-like region contains userinfo, host, and port.

---

# 8. Scheme

Example:

```text
https
```

JavaScript:

```js
url.protocol
// "https:"
```

Do not compare scheme strings casually across APIs that may include or omit the colon.

---

# 9. Host vs Hostname vs Port

```js
url.host
url.hostname
url.port
```

Conceptually:

```text
host      → hostname + explicit port when applicable
hostname  → host name without port
port      → explicit port representation
```

Example:

```js
const u = new URL("https://example.com:8443/a");

u.host;
// "example.com:8443"

u.hostname;
// "example.com"

u.port;
// "8443"
```

---

# 10. Default Ports

Examples of conventional defaults:

```text
http  → 80
https → 443
ws    → 80
wss   → 443
```

An omitted default port and an explicit default port can be semantically equivalent for an endpoint while having different input strings.

Do not use raw string equality as your only endpoint-equivalence test.

---

# 11. Origin

For ordinary network schemes, origin conceptually consists of:

```text
scheme + host + port
```

Example:

```text
https://example.com:8443
```

Origin is central to:

```text
same-origin policy
CORS
cookies
storage
window messaging
browser isolation
```

---

# 12. Origin Is Not the Full URL

These can share an origin:

```text
https://example.com/a
https://example.com/b
```

while differing in:

```text
path
query
fragment
```

So:

```text
origin equality ≠ URL equality
```

---

# 13. Path

Example:

```text
/docs/tutorial/index.html
```

JavaScript:

```js
url.pathname
```

Paths have their own parsing and encoding rules. Encoded separators such as `%2F` should not automatically be treated as ordinary `/` during routing/security checks.

---

# 14. Query

Example:

```text
?category=books&sort=price
```

JavaScript:

```js
url.search
url.searchParams
```

`search` is the serialized query component; `searchParams` provides structured name/value operations.

---

# 15. Fragment

Example:

```text
#section-2
```

JavaScript:

```js
url.hash
```

A fragment is generally client-side state and is not sent as part of a normal HTTP request target.

This matters for:

```text
SPA routing
analytics
security assumptions
server logs
HTTP behavior
```

---

# 16. Absolute URL

Example:

```text
https://example.com/docs/page
```

It contains a scheme and enough URL context to represent a complete absolute URL for the scheme.

---

# 17. Relative Reference

Example:

```text
../images/logo.svg
```

It requires a base URL.

```js
new URL("../images/logo.svg", "https://example.com/docs/a/");
```

Result:

```text
https://example.com/docs/images/logo.svg
```

---

# 18. Base URL Resolution

Never implement general URL resolution with:

```js
base + relative
```

A proper resolver considers:

```text
scheme
authority
path
query
fragment
relative path
absolute path
protocol-relative reference
```

---

# 19. Dot Segments

Paths can include:

```text
.
..
```

For example:

```text
/a/b/../c
```

can resolve to:

```text
/a/c
```

Do not replace strings like `../` blindly: URL parsing, path normalization, decoding, and filesystem semantics are separate layers.

---

# 20. Protocol-Relative Reference

Example:

```text
//example.com/path
```

This is not merely an ordinary local path. Its scheme is inherited from the base context.

Therefore:

```js
input.startsWith("/")
```

does not by itself prove “same-origin local path.”

---

# 21. URL Serialization

Useful properties/methods:

```js
url.href
url.toString()
url.toJSON()
```

They expose a serialized representation of the structured URL.

Parsing may normalize representations, so:

```text
raw input string
```

need not equal:

```text
serialized URL string
```

byte-for-byte.

---

# 22. Component Access and Mutation

Example:

```js
const url = new URL("https://example.com/a?x=1");

url.pathname = "/b";
url.searchParams.set("x", "2");

console.log(url.href);
```

Structured mutation is generally safer than hand-built string slicing.

---

# 23. Percent-Encoding

Percent encoding represents bytes using:

```text
%
+
hexadecimal digits
```

Example:

```text
space → %20
```

For Unicode text, the usual conceptual pipeline is:

```text
Unicode scalar values
→ UTF-8 bytes
→ %HH sequences
```

---

# 24. UTF-8 and Unicode

Conceptually:

```text
é
→ UTF-8 bytes C3 A9
→ %C3%A9
```

One Unicode character can therefore become several encoded bytes.

This connects directly to:

```text
Chapter 04 — Strings / Unicode
Chapter 128 — Intl
```

---

# 25. Encoding Is Context-Specific

There is no single universal:

```text
encode URL
```

operation.

You must know whether the data is being inserted into:

```text
whole URI reference
path segment
query parameter
form field
fragment
userinfo
```

The character set that is safe to leave literal depends on the context.

---

# 26. `encodeURI`

```js
encodeURI(value)
```

is intended for URI-like strings while preserving characters that function as URI syntax.

It is therefore not the correct general-purpose encoder for a single query parameter.

---

# 27. `encodeURIComponent`

```js
encodeURIComponent(value)
```

is intended for encoding one URI component.

Example:

```js
encodeURIComponent("hello world");
```

The resulting component contains percent-encoded data for characters that cannot remain literal in that context.

---

# 28. `decodeURI` vs `decodeURIComponent`

Conceptually:

```text
decodeURI
→ decode URI-like strings while respecting URI delimiters

decodeURIComponent
→ decode one component
```

Do not use a decoder merely because a string “looks encoded.” First establish:

```text
who encoded it
what context they encoded
how many decoding steps are expected
```

---

# 29. Invalid Percent Escapes

Example:

```js
decodeURIComponent("%");
```

throws:

```text
URIError
```

Malformed input therefore belongs in your validation/error-handling design.

---

# 30. Double-Encoding

Conceptual progression:

```text
/
→ %2F
→ %252F
```

A second encoding converts the `%` character itself into `%25`.

Double-encoding often causes:

```text
routing mismatch
signature mismatch
security-policy bypass
```

---

# 31. Double-Decoding

Conceptual attack string:

```text
%252e%252e
```

One decode can produce:

```text
%2e%2e
```

Two can produce:

```text
..
```

The application must define exactly where decoding occurs. Never let multiple layers independently “helpfully decode.”

---

# 32. Canonicalization

Canonicalization means transforming multiple equivalent representations into one defined representation.

Potential canonicalization dimensions:

```text
scheme
hostname
default port
path
query ordering
percent encoding
```

There is no universal “canonical URL” independent of a protocol/application contract.

---

# 33. Canonicalization Is Not Validation

Bad design:

```text
normalize everything
→ assume safe
```

Better:

```text
parse
→ normalize only according to a defined equivalence rule
→ validate policy
→ pass canonical representation to downstream consumer
```

Canonicalization can create bugs if it changes what a downstream system interprets.

---

# 34. Case Rules

Do not blindly lowercase the whole URL:

```js
urlString.toLowerCase()
```

Some components are case-insensitive by protocol/URL rules, while application paths, query values, and fragments can be case-sensitive.

Canonicalize only the component for which equivalence is defined.

---

# 35. Trailing Slash

These can be distinct application resources:

```text
https://example.com/api
https://example.com/api/
```

Do not normalize away trailing slashes unless the routing/API contract defines them as equivalent.

---

# 36. Query Parameters

Example:

```text
?tag=javascript&tag=security
```

There are two `tag` values.

Use:

```js
url.searchParams.get("tag");
```

for one value, and:

```js
url.searchParams.getAll("tag");
```

for all values.

---

# 37. Duplicate Parameter Semantics

An API must define whether duplicates mean:

```text
array
first wins
last wins
invalid
all values
```

Never silently assume:

```text
one name = one value
```

---

# 38. `URLSearchParams`

Example:

```js
const params = new URLSearchParams();

params.set("page", "2");
params.append("tag", "javascript");
params.append("tag", "security");
```

Then:

```js
params.toString();
```

serializes the parameter list using application/x-www-form-urlencoded-style rules.

---

# 39. `set` vs `append`

```js
params.set("tag", "js");
```

replaces existing values for that name.

```js
params.append("tag", "security");
```

adds another entry.

This difference is critical for multi-valued APIs.

---

# 40. Query Ordering

`URLSearchParams` models parameters as an ordered list.

Ordering can matter for:

```text
signatures
cache keys
snapshots/tests
debugging
canonical output
```

If the application considers ordering semantically irrelevant, define a separate canonical-sort policy rather than assuming it occurs automatically.

---

# 41. Space and `+`

Form-style query serialization commonly represents spaces with:

```text
+
```

rather than:

```text
%20
```

Therefore:

```text
URL percent encoding intuition
```

and:

```text
application/x-www-form-urlencoded serialization
```

are not identical concepts.

---

# 42. Query Values Are Strings

`URLSearchParams` does not recursively encode arbitrary JavaScript objects as JSON.

For structured payloads, use an explicit serialization format such as:

```text
JSON
```

rather than expecting:

```js
new URLSearchParams({ filters: { status: "open" } });
```

to preserve object structure.

---

# 43. Query Injection

Bad:

```js
const url = "/search?q=" + value;
```

Better:

```js
const url = new URL("/search", base);
url.searchParams.set("q", value);
```

Structured construction prevents user data from accidentally becoming query syntax.

---

# 44. Path Segment Construction

Bad:

```js
const path = "/users/" + userId;
```

when `userId` can contain URL syntax such as:

```text
/
?
#
%
```

Encode according to the path-segment contract rather than assuming one universal encoder exists.

---

# 45. Path vs Query Encoding

Do not treat:

```js
encodeURIComponent(value)
```

as a complete URL-construction API.

Use:

```text
URL
URLSearchParams
context-appropriate encoding
```

instead.

---

# 46. URL Equality

Possible meanings include:

```text
same raw string
same serialized URL
same parsed components
same origin
same effective network endpoint
same application resource
same canonical signature input
```

Choose one definition for each feature.

---

# 47. URL Objects and Equality

```js
new URL("https://example.com") ===
new URL("https://example.com");
```

is:

```text
false
```

because object identity differs.

For semantic comparison, compare the representation appropriate to the contract.

---

# 48. URL Credentials

URLs can contain:

```text
https://user:password@example.com/
```

JavaScript exposes:

```js
url.username
url.password
```

But credentials in URLs can leak through:

```text
logs
history
telemetry
analytics
error reports
```

Avoid them for application authentication when headers/body mechanisms are available.

---

# 49. URL Credentials Are Not Authorization

The presence of:

```text
url.username
```

does not authenticate the user and must not be treated as an authorization proof.

Authentication is a protocol/security concern, not a string-property check.

---

# 50. Unicode Hostnames / IDN

Internationalized domain names can involve:

```text
Unicode display form
ASCII-compatible encoding
Punycode
DNS representation
```

A visually meaningful hostname is not necessarily the same byte/string representation used on the network.

---

# 51. IDN Security

Risks include:

```text
homoglyphs
confusable characters
mixed scripts
lookalike domains
visual spoofing
```

Security decisions should be based on parsed host/origin policy, not visual resemblance.

---

# 52. Unicode Normalization

Do not confuse:

```text
Unicode normalization
percent-decoding
URL normalization
hostname processing
```

They are different transformations.

Applying NFC/NFKC or another normalization arbitrarily to security-sensitive URL material can change semantics.

---

# 53. Fragments and Server Requests

For:

```text
https://example.com/page#settings
```

the fragment generally does not form part of the HTTP request target.

The browser may use it for:

```text
scrolling
client routing
document state
```

Server logic should not assume it receives the fragment.

---

# 54. Hash Routing

A client application can use:

```text
/app#settings
```

while the server sees a request for the path without the fragment.

This creates a useful host-boundary example:

```text
browser URL
→ navigation
→ HTTP request
```

are related but not identical representations.

---

# 55. `http` / `https`

These usually include:

```text
scheme
authority
path
query
fragment
```

and are the most common domain for:

```text
Fetch
HTTP
navigation
cookies
CORS
```

---

# 56. `ws` / `wss`

WebSocket URLs resemble:

```text
ws://
wss://
```

but they participate in WebSocket connection semantics rather than being simple string aliases for ordinary HTTP.

---

# 57. `file:` URLs

`file:` URLs are especially platform-sensitive.

Differences can involve:

```text
POSIX paths
Windows drive letters
UNC paths
permissions
host handling
sandboxing
```

Do not convert filesystem paths to file URLs with string concatenation.

In Node, use the filesystem URL utilities designed for this boundary.

---

# 58. `data:` URLs

A `data:` URL embeds content:

```text
data:[media-type][;base64],payload
```

It is not a normal remote network destination.

Treat it as an embedded-content scheme with its own security and size implications.

---

# 59. `blob:` URLs

A browser `blob:` URL references browser-managed Blob/Media state.

It is not equivalent to:

```text
https://...
```

and its lifecycle/origin behavior is governed by the browser platform.

---

# 60. HTTP Request Target vs URL

A URL object is not identical to raw HTTP request-target syntax.

HTTP can use forms such as:

```text
origin-form
absolute-form
authority-form
asterisk-form
```

A high-level API such as Fetch performs additional processing between:

```text
URL object
→ request creation
→ HTTP request
```

---

# 61. URL Parsing vs String Parsing

Bad:

```js
const [base, query] = input.split("?");
```

This fails to model:

```text
fragment
userinfo
relative references
encoded delimiters
scheme rules
query semantics
```

Use the platform parser.

---

# 62. Regex vs URL Parsing

Bad general approach:

```js
/[?&]id=([^&]+)/.exec(input);
```

for URL query handling.

Prefer:

```js
const url = new URL(input);
const id = url.searchParams.get("id");
```

Regex is excellent for generic text patterns; URL APIs understand URL structure.

---

# 63. URL and `URLPattern`

`URLPattern`, where supported, provides URL-aware pattern matching across URL components.

Conceptually:

```text
URL
→ structured components
→ component-aware pattern
```

Use it when the problem is routing/matching URL structure rather than generic text pattern matching.

Always verify target runtime support.

---

# 64. `URL.canParse`

Modern runtimes can expose:

```js
URL.canParse(input, base)
```

for a boolean parseability check.

This can be useful when malformed user input is expected and exceptions would otherwise be used for routine control flow.

Check target compatibility before adopting it universally.

---

# 65. `URL.parse`

Some modern environments also expose parse-without-throw semantics through:

```js
URL.parse(input, base)
```

Support is runtime-dependent.

The important design lesson is:

```text
parseability
vs
exception-driven validation
```

should be an intentional API choice.

---

# 66. Browser vs Node

Modern browsers and Node.js expose the WHATWG-style:

```text
URL
URLSearchParams
```

for common URL manipulation.

Node also contains historical URL APIs with different behavior. Avoid mixing old and WHATWG URL mental models inside security-sensitive code.

---

# 67. Legacy Node URL Parsing

Historical Node code may use:

```text
require("url").parse(...)
```

This belongs to a legacy API family with different parsing semantics.

For new code, start with:

```js
new URL(...)
```

unless a compatibility requirement explicitly demands otherwise.

---

# 68. Open Redirects

Dangerous:

```js
redirect(req.query.next);
```

An attacker can provide an external destination.

Safer contracts may allow only:

```text
relative local paths
allowlisted origins
signed destinations
```

The correct policy depends on product requirements.

---

# 69. Safe Redirect Validation

Do not rely on:

```js
next.startsWith("/")
```

alone.

Protocol-relative references such as:

```text
//evil.example
```

must be considered.

A safe implementation resolves against an explicit base and validates the resulting structure.

---

# 70. SSRF

Server-side fetching of a user-controlled URL can create SSRF risk:

```js
fetch(userProvidedUrl);
```

Potential targets include:

```text
loopback
private networks
cloud metadata endpoints
internal DNS names
unexpected ports
unexpected protocols
```

---

# 71. SSRF Defense Is Multi-Layered

A production policy can include:

```text
allowed schemes
hostname policy
origin policy
port policy
DNS resolution policy
IPv4/IPv6 policy
private/loopback filtering
redirect policy
egress network controls
```

URL parsing is necessary but not sufficient.

---

# 72. Redirect-Based SSRF

A trusted initial URL may redirect to an internal destination.

Therefore:

```text
validate first URL
```

is not enough when the client follows redirects.

Validate every redirect target according to the same fetch policy.

---

# 73. DNS Rebinding

Even if:

```text
hostname → allowed IP
```

at validation time, DNS can later produce a different address.

This shows the difference between:

```text
URL policy
```

and:

```text
network enforcement.
```

Use network-level egress controls when the threat model requires strong guarantees.

---

# 74. Hostname Allowlisting

Avoid:

```js
hostname.endsWith("example.com")
```

as the complete policy.

It could permit:

```text
evil-example.com
```

A subdomain policy needs an explicit boundary such as:

```text
hostname === "example.com"
OR
hostname.endsWith(".example.com")
```

plus normalization/IDN rules where applicable.

---

# 75. Origin Allowlists

For many browser/security cases, an exact origin allowlist is clearer:

```js
const allowedOrigins = new Set([
  "https://app.example.com",
  "https://admin.example.com",
]);

if (!allowedOrigins.has(url.origin)) {
  throw new Error("Untrusted origin");
}
```

This makes the policy directly comparable to:

```text
scheme + host + port
```

rather than a substring.

---

# 76. URL Injection

If user data changes URL structure, you have a data/code boundary problem.

Example:

```js
new URL("https://example.com/search?q=" + userInput);
```

A safer approach is:

```js
const u = new URL("https://example.com/search");
u.searchParams.set("q", userInput);
```

---

# 77. URL Signature Canonicalization

Signed URLs often require a canonical representation:

```text
scheme
host
path
query
```

Potential mismatch sources:

```text
parameter ordering
duplicate parameters
percent-encoding
default ports
path normalization
case handling
```

Sign and verify the same defined canonical form.

---

# 78. Canonical Signing Pipeline

Use a pipeline like:

```text
raw input
→ parse
→ policy validation
→ canonicalization
→ canonical serialization
→ sign
```

Verification must perform:

```text
same parse
→ same validation
→ same canonicalization
→ same serialization
→ verify
```

Do not rely on ad-hoc string normalization on one side only.

---

# 79. URL Secrets and Logs

URLs can contain:

```text
tokens
codes
passwords
PII
user IDs
signed parameters
```

Avoid blindly logging:

```js
url.href
```

Build redaction rules for sensitive query keys and credentials.

---

# 80. URL Secrets and Referrers

Secrets placed in URLs can propagate through:

```text
logs
browser history
telemetry
screenshots
analytics
referrer-related mechanisms
```

Prefer:

```text
Authorization headers
request bodies
secure cookie mechanisms
```

where protocol semantics support them.

---

# 81. URL Length

Limits can arise from:

```text
browser
reverse proxy
load balancer
web server
framework
upstream service
```

There is no single universal safe maximum for every deployment.

Do not use massive query strings for large application payloads when a request body is appropriate.

---

# 82. Performance

`new URL(...)` performs parsing and object construction.

For hot paths:

```text
avoid unnecessary reparsing
reuse trusted bases when lifecycle is clear
benchmark before replacing standard APIs
```

Do not “optimize” URL handling into unsafe manual parsing without evidence.

---

# 83. URL Object Reuse

Example pattern:

```js
const base = new URL("https://example.com");

for (const id of ids) {
  const u = new URL(base);
  u.pathname = `/items/${id}`;
}
```

This can make the base structure reusable while keeping each request URL isolated.

Avoid shared mutable URL objects when concurrency or lifecycle makes state difficult to reason about.

---

# 84. URLSearchParams Performance

`URLSearchParams` provides structured correctness.

If it appears in a very hot path, benchmark:

```text
allocation
parameter count
serialization
GC effects
```

before replacing it with manual string logic.

Correctness usually dominates micro-optimization here.

---

# 85. Filesystem Mapping Security

Mapping:

```text
URL pathname
→ filesystem path
```

requires careful handling of:

```text
percent-decoding
path normalization
platform separators
encoded traversal
root escape
symbolic links
```

Never use:

```js
baseDir + url.pathname
```

as a complete filesystem security strategy.

---

# 86. Path Traversal Boundary

A secure file-serving pipeline should define:

```text
URL parsing
→ decoding policy
→ URL path normalization
→ filesystem path construction
→ filesystem canonicalization
→ root containment check
→ file access
```

Each layer should know what representation it receives.

---

# 87. URL and Cookies

Cookie matching uses structured attributes such as:

```text
domain
path
secure/scheme-related rules
```

Do not invent cookie policy with generic URL string operations.

Cookie semantics are related to URLs but are defined by their own standards.

---

# 88. URL and CORS

CORS decisions involve:

```text
origin
```

not arbitrary URL string equality.

For example:

```text
https://example.com/a
https://example.com/b
```

share an origin even though the full URLs differ.

---

# 89. URL and Web APIs

Many browser APIs accept URLs or URL-like inputs:

```text
fetch
WebSocket
URL constructors
module/resource loading
navigation
```

Learn the boundary:

```text
string
→ parsed URL
→ API-specific validation
→ network/resource semantics
```

Each API can add constraints beyond the URL parser.

---

# 90. Errors and Validation

URL construction can throw for invalid input:

```js
try {
  const u = new URL(input);
} catch (error) {
  // malformed URL/reference
}
```

For expected invalid-user-input paths, a boolean parseability API can sometimes be cleaner where supported.

---

# 91. Testing Strategy

Test:

```text
absolute URLs
relative references
protocol-relative references
empty input
Unicode
spaces
percent-encoding
duplicate parameters
fragments
default ports
explicit ports
credentials
IPv4
IPv6
invalid ports
invalid escapes
unexpected schemes
```

---

# 92. Security Testing Matrix

Include:

```text
userinfo confusion
subdomain confusion
protocol-relative URLs
encoded separators
double-encoding
double-decoding
mixed-case hostnames
IDN confusables
localhost
loopback
private addresses
metadata addresses
redirect chains
unexpected schemes
```

---

# 93. Property-Based Testing

Generate combinations of:

```text
scheme
host
port
path
query
fragment
Unicode
encoded delimiters
relative references
```

Check invariants such as:

```text
parse(serialize(parse(u)))
```

being stable according to your defined canonicalization contract.

Do not assume raw byte-for-byte round trips for every accepted input representation.

---

# 94. Differential Testing

Compare shared URL behavior across:

```text
browser
Node.js
```

for standard WHATWG URL features.

Also compare:

```text
platform URL
vs
application wrapper
```

and make sure the wrapper does not silently introduce weaker or different semantics.

---

# 95. Fuzzing

Fuzz:

```text
malformed percent escapes
very long URLs
Unicode
control characters
userinfo
ports
IPv6
encoded delimiters
mixed normalization forms
```

Look for:

```text
unexpected exceptions
parser differences
canonicalization instability
resource spikes
security-policy bypasses
```

---

# 96. Implementation From Scratch — URL Model

Build an educational restricted parser supporting:

```text
scheme
username/password
hostname
port
pathname
query
fragment
```

Then add:

```text
relative resolution
percent-encoding
query parameter list
serialization
```

The purpose is to understand structure, not to replace the platform URL implementation.

---

# 97. Implementation Milestone 1 — URL Record

Represent:

```js
{
  scheme,
  username,
  password,
  hostname,
  port,
  pathname,
  search,
  hash,
}
```

Implement:

```text
parse
→ record
→ serialize
```

Document which syntax you intentionally do not support.

---

# 98. Implementation Milestone 2 — Query Model

Represent query parameters as:

```text
ordered name/value pairs
```

Support:

```text
append
set
get
getAll
delete
sort
```

This demonstrates why a query is not naturally a plain JavaScript object.

---

# 99. Implementation Milestone 3 — Percent-Encoding

For a constrained context, implement:

```text
UTF-8 bytes
→ %HH
```

Document the exact literal-safe character set.

Do not attempt a complete WHATWG implementation in the first exercise.

---

# 100. Implementation Milestone 4 — Relative Resolution

Implement:

```text
base
+
relative reference
→
absolute URL
```

Test:

```text
/a/b
./c
../c
/c
//other.example/x
```

---

# 101. Implementation Milestone 5 — Security Wrapper

Build:

```js
parseTrustedUrl(input, policy)
```

where policy defines:

```text
allowed protocols
allowed origins
allowed ports
credential policy
relative/absolute policy
redirect policy
```

Make failures explicit and testable.

---

# 102. Debugging Exercises

## Exercise A — Host Prefix Bypass

```js
function allowed(input) {
  return input.startsWith("https://example.com");
}
```

Find an input that passes the prefix test while targeting a different host.

Then replace it with parsed policy checks.

---

## Exercise B — Query Injection

```js
const url = "/search?q=" + value;
```

Find an input that alters query structure.

Replace the construction with `URL` + `URLSearchParams`.

---

## Exercise C — Double Decode

Trace:

```text
%252e%252e
```

through zero, one, and two decoding layers.

Explain exactly where `..` becomes present.

---

## Exercise D — Open Redirect

```js
redirect("/login?next=" + next);
```

Determine how an attacker can escape the intended local destination.

---

# 103. Code Review Exercise

Review:

```js
function isTrusted(value) {
  return (
    value.startsWith("https://api.example.com") &&
    !value.includes("@")
  );
}
```

Identify:

```text
string-prefix flaws
userinfo assumptions
hostname boundaries
port issues
scheme handling
IDN handling
canonicalization gaps
redirect implications
```

Rewrite the policy around parsed URL structure.

---

# 104. Interview Questions

### Fundamentals

```text
1. What is the difference between URI and URL?
2. What is an origin?
3. What is the difference between host and hostname?
4. What is the difference between pathname, search, and hash?
5. Why use URL instead of string concatenation?
```

### Encoding

```text
6. What does percent-encoding represent?
7. What is the difference between encodeURI and encodeURIComponent?
8. Why can URLSearchParams encode spaces as +?
9. Why can decodeURIComponent throw URIError?
10. What is double-encoding?
```

### Query Strings

```text
11. Why can query parameters be duplicated?
12. What is the difference between set and append?
13. Why isn't a query naturally represented by a plain object?
```

### Security

```text
14. How do you prevent open redirects?
15. How do you prevent SSRF?
16. Why is startsWith() insufficient for hostname validation?
17. Why are credentials in URLs dangerous?
18. How can canonicalization create authorization bugs?
```

### Principal

```text
19. How would you design a URL allowlist?
20. How would you canonicalize a URL for signing?
21. How would you handle redirect chains in an SSRF-sensitive service?
22. How would you support internationalized hostnames securely?
23. How would you design URL utilities shared by browser and Node services?
```

---

# 105. Predict-the-Output Exercises

Predict before running.

### Exercise 1

```js
const url = new URL("https://example.com:443/a");

console.log(url.protocol);
console.log(url.hostname);
console.log(url.port);
console.log(url.origin);
```

### Exercise 2

```js
const params = new URLSearchParams();

params.append("tag", "javascript");
params.append("tag", "security");

console.log(params.get("tag"));
console.log(params.getAll("tag"));
```

### Exercise 3

```js
const url = new URL("https://example.com/path");
url.searchParams.set("q", "hello world");
console.log(url.href);
```

### Exercise 4

```js
const url = new URL("../img/logo.svg", "https://example.com/docs/a/");
console.log(url.href);
```

### Exercise 5

```js
console.log(encodeURIComponent("a/b?c=d"));
```

Explain why component encoding treats URL syntax characters differently from encoding an already-structured full URL.

---

# 106. Mastery Exercises

### Exercise 1 — URL Policy Engine

Build:

```text
allowed protocols
allowed origins
allowed ports
credential rejection
relative-only mode
```

### Exercise 2 — Safe Redirect Helper

Implement:

```js
safeRedirect(target, base, policy)
```

that supports an explicit relative-path or allowlisted-origin policy and rejects unexpected schemes and credential-bearing targets.

### Exercise 3 — SSRF URL Gate

Build a testable first-stage pipeline:

```text
parse
→ scheme check
→ host policy
→ port policy
→ DNS policy
→ redirect policy
```

Document why network egress controls are still required.

### Exercise 4 — Canonical Request

Implement:

```text
method
+
canonical origin
+
canonical path
+
canonical query
```

then sign the result with HMAC.

### Exercise 5 — URL Fuzzer

Generate:

```text
paths
Unicode
percent escapes
duplicate parameters
relative references
```

and check parser/serializer stability.

---

# 107. Track A — Core Theory

Master:

```text
URL structure
origin
base resolution
percent-encoding
URI encoding APIs
URLSearchParams
normalization
serialization
protocol-specific URLs
Unicode hostnames
canonicalization
```

Deliverable:

```text
explain every relevant URL component and its encoding/security contract.
```

---

# 108. Track B — Implementation

Build:

```text
restricted URL parser
URL serializer
query parameter model
percent encoder
relative resolver
safe redirect helper
URL allowlist engine
canonical request builder
URL fuzz harness
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

# 109. Track C — Interview / Reasoning

Practice:

```text
“Why is string prefix validation unsafe for URLs?”

“Why can %252e%252e matter?”

“Why isn't query syntax just an object?”

“Why is origin different from URL?”

“Why must every redirect target be reconsidered in SSRF-sensitive systems?”

“How would you canonicalize a signed URL?”

“What changes when URLs contain Unicode?”
```

Deliverable:

```text
URL model + encoding context + trust boundary + threat model.
```

---

# 110. Specification / Runtime Source Discipline

Use this hierarchy:

```text
1. WHATWG URL Standard
2. Fetch / HTML / relevant Web Platform standards
3. ECMAScript URI encoding/decoding semantics
4. Node.js URL documentation
5. browser/runtime compatibility data
6. application/framework behavior
```

Keep these distinctions explicit:

```text
ECMAScript language
→ encodeURI / encodeURIComponent / decodeURI / decodeURIComponent / URIError

Web Platform
→ URL / URLSearchParams / URLPattern

Host/runtime
→ Fetch, filesystem integration, network APIs, OS behavior
```

The goal is to avoid treating every URL feature as an ECMAScript language primitive.

---

# 111. Principal Decision Framework

For every URL-related requirement ask:

```text
1. What schemes are allowed?
2. Is the input absolute or relative?
3. What is the base URL?
4. Which component contains untrusted data?
5. What encoding context applies?
6. Are Unicode hostnames allowed?
7. What exact origin/hostname policy applies?
8. Are credentials allowed?
9. Are default ports relevant?
10. How are duplicate query values handled?
11. Are redirects followed?
12. Are redirect destinations revalidated?
13. Could DNS change between validation and connection?
14. Is canonicalization required?
15. Is canonicalization identical on signing and verification paths?
16. Is this browser or Node code?
17. Could the URL reach logs or telemetry?
18. Does the URL contain secrets?
19. Does downstream code interpret the same representation you validated?
20. What is the maximum accepted URL size?
```

---

# 112. Production Checklist

```text
[ ] URL parsed structurally
[ ] raw prefix/substring checks avoided for security decisions
[ ] protocol allowlist explicit
[ ] origin/hostname policy explicit
[ ] port policy explicit
[ ] relative URL policy explicit
[ ] query construction uses URLSearchParams where appropriate
[ ] encoding context documented
[ ] decoding boundaries documented
[ ] double-decoding avoided
[ ] IDN/security policy documented
[ ] credentials rejected or explicitly handled
[ ] redirects validated per hop
[ ] SSRF policy includes network controls
[ ] canonicalization defined before signing
[ ] sensitive query fields redacted from logs
[ ] URL size limits defined
[ ] browser/Node compatibility checked
[ ] malformed URLs tested
[ ] Unicode tested
[ ] encoded separators tested
```

---

# 113. Retrieval Record

```md
# Chapter 131 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## URL Structure
-

## Origin
-

## Base Resolution
-

## Percent-Encoding
-

## encodeURI / encodeURIComponent
-

## URLSearchParams
-

## Unicode / IDN
-

## Canonicalization
-

## Redirects
-

## SSRF
-

## Security
-

## Serialization
-

## Browser / Node
-

## Testing
-

## Implementation
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

# 114. Spaced Retrieval Schedule

### Day 0

Study:

```text
URL components
URL object
URLSearchParams
encoding APIs
```

### Day 1

Trace:

```text
absolute
relative
protocol-relative
```

resolution.

### Day 3

Analyze:

```text
open redirect
SSRF
encoding/decoding ambiguity
```

### Day 7

Build:

```text
safe URL helper
```

without notes.

### Day 14

Build:

```text
canonical signed URL
```

and document the canonicalization contract.

### Day 21

Review:

```text
Unicode
IDN
credentials
redirects
```

### Day 30

Perform a complete:

```text
URL security audit
```

without notes.

---

# 115. Dependency Graph

```text
Chapter 04
Strings / Unicode
        ↓
Chapter 07
Coercion / Equality
        ↓
Chapter 23
String APIs
        ↓
Chapter 28
Serialization
        ↓
Chapter 41
Specification Architecture
        ↓
Chapter 42
Abstract Operations
        ↓
Chapter 49
DOM Architecture
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 55
Fetch / HTTP
        ↓
Chapter 56
Browser Security
        ↓
Chapter 57
Security Engineering
        ↓
Chapter 79
API Design
        ↓
Chapter 123
Grammar / Parsing
        ↓
Chapter 128
Intl
        ↓
Chapter 129
Regular Expressions
        ↓
Chapter 130
Legacy Date / Time Zones
        ↓
Chapter 131
URI / URL / Encoding / Parsing
```

Cross-cutting:

```text
Chapter 67 → supply-chain inputs
Chapter 69 → asset URLs
Chapter 70 → source-map URLs
Chapter 83 → URL/log redaction
Chapter 84 → redirect/reliability
Chapter 85 → parser/allocation performance
Chapter 98 → string-manipulation anti-patterns
```

---

# 116. Concept Connections

## Depends On

```text
Unicode
strings
encoding
HTTP
browser security
Node networking
parsing
serialization
API design
```

## Builds Toward

```text
Fetch
HTTP clients
SSR frameworks
routing
authentication
redirect systems
SSRF defenses
signed URLs
CDN/cache design
```

## Related Concepts

```text
WHATWG URL
URI
percent-encoding
UTF-8
Punycode / IDN
same-origin policy
CORS
HTTP request-targets
DNS
redirects
canonicalization
cryptographic signing
```

## Concepts Revisited

```text
Strings
Unicode
Symbols
Parsing
Serialization
Security
Performance
API design
```

## Why This Chapter Matters

URLs sit at the boundary between:

```text
text
protocol
network
browser
server
security
```

That makes them one of the highest-risk places for string-based assumptions.

The mature approach is:

```text
parse
→ understand component
→ apply component-specific rules
→ canonicalize when required
→ validate policy
→ serialize
```

not:

```text
split("?")
replace(...)
startsWith(...)
```

---

# 117. Final Principal Mental Model

Use:

```text
RAW INPUT
    ↓
parse
    ↓
URL STRUCTURE
    ├── scheme
    ├── credentials
    ├── host
    ├── port
    ├── path
    ├── query
    └── fragment
    ↓
component semantics
    ↓
encoding / decoding
    ↓
normalization / canonicalization
    ↓
security policy
    ↓
host protocol
    ↓
network / browser behavior
```

For user-controlled data:

```text
data
→ identify URL component
→ use context-specific encoder
→ construct structured URL
```

For security:

```text
untrusted URL
→ parse
→ scheme policy
→ origin/host policy
→ port policy
→ DNS/network policy
→ redirect policy
→ downstream request
```

For cryptographic signing:

```text
input
→ parse
→ validate
→ canonicalize
→ serialize
→ sign
```

---

# 118. Final Principal Principle

> **Never manipulate a security-sensitive URL as if it were an ordinary string.**

The production-grade sequence is:

```text
classify the URL/reference
→ parse structurally
→ understand the target component
→ encode/decode exactly where required
→ normalize only according to a defined contract
→ validate against the downstream consumer
→ guard redirects/DNS/network behavior
→ redact secrets
→ serialize consistently
→ test adversarial cases
```

The core distinctions to internalize are:

```text
A URL is structured data.

A URI reference can be relative.

Percent-encoding represents encoded bytes.

Encoding is context-specific.

A query is an ordered collection of name/value pairs.

An origin is not the entire URL.

A hostname is not the entire host.

A string prefix is not a security policy.

Canonicalization is not automatically validation.

Parsing is not the same as fetching.

URL safety requires parser correctness
and protocol/network policy.
```

At principal level, the skill is not:

```text
memorizing URL properties.
```

It is:

```text
knowing which representation the next system will interpret,
then validating exactly that representation at the trust boundary.
```