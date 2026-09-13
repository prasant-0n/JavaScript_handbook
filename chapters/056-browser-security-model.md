# Chapter 56 — Browser Security Engineering

> **Curriculum Position:** Part X — Networking / Security  
> **Prerequisites:** Chapters 49–55  
> **Primary Focus:** Browser security model, origins, same-origin policy, CORS, cookies, CSRF, XSS, CSP, Trusted Types, framing, sandboxing, mixed content, SRI, Fetch Metadata, COOP/COEP/CORP, Permissions Policy, storage isolation, postMessage, Web Workers, extensions, supply chain, and production threat modeling  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Browser security primitives → attack surfaces → policy controls → isolation boundaries → secure application architecture → production threat modeling  
> **Scope Rule:** Browser security is a cross-layer system. It involves HTML, Fetch, HTTP, DOM, cookies, storage, JavaScript execution, browser processes, and security policies. No single API provides complete application security.

---

# Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Browser Security Is](#3-what-browser-security-is)
- [4. Why Browser Security Exists](#4-why-browser-security-exists)
- [5. Mental Model](#5-mental-model)
- [6. Trust Boundaries](#6-trust-boundaries)
- [7. Origin](#7-origin)
- [8. Same-Origin Policy](#8-same-origin-policy)
- [9. Origin Isolation vs Resource Sharing](#9-origin-isolation-vs-resource-sharing)
- [10. CORS](#10-cors)
- [11. Cookies](#11-cookies)
- [12. Cookie Security Attributes](#12-cookie-security-attributes)
- [13. CSRF](#13-csrf)
- [14. XSS](#14-xss)
- [15. DOM XSS](#15-dom-xss)
- [16. Output Encoding and Contexts](#16-output-encoding-and-contexts)
- [17. Sanitization](#17-sanitization)
- [18. Content Security Policy](#18-content-security-policy)
- [19. CSP Directives](#19-csp-directives)
- [20. Nonces, Hashes, and Strict CSP](#20-nonces-hashes-and-strict-csp)
- [21. Trusted Types](#21-trusted-types)
- [22. Subresource Integrity](#22-subresource-integrity)
- [23. Mixed Content](#23-mixed-content)
- [24. Clickjacking and Framing](#24-clickjacking-and-framing)
- [25. iframe Sandbox](#25-iframe-sandbox)
- [26. postMessage](#26-postmessage)
- [27. Cross-Origin Isolation](#27-cross-origin-isolation)
- [28. COOP](#28-coop)
- [29. COEP](#29-coep)
- [30. CORP](#30-corp)
- [31. Origin-Agent Isolation](#31-origin-agent-isolation)
- [32. Permissions Policy](#32-permissions-policy)
- [33. Fetch Metadata](#33-fetch-metadata)
- [34. Storage and Privacy Boundaries](#34-storage-and-privacy-boundaries)
- [35. Storage Partitioning](#35-storage-partitioning)
- [36. Private Network / Local Network Concerns](#36-private-network--local-network-concerns)
- [37. Service Workers and Security](#37-service-workers-and-security)
- [38. Web Workers and Security](#38-web-workers-and-security)
- [39. Web Components and Security](#39-web-components-and-security)
- [40. Supply Chain Security](#40-supply-chain-security)
- [41. Dependency and Script Loading](#41-dependency-and-script-loading)
- [42. Authentication and Session Security](#42-authentication-and-session-security)
- [43. Authorization](#43-authorization)
- [44. Secrets in the Browser](#44-secrets-in-the-browser)
- [45. Browser Security vs Server Security](#45-browser-security-vs-server-security)
- [46. Threat Modeling](#46-threat-modeling)
- [47. Security Headers](#47-security-headers)
- [48. Error Handling and Information Leakage](#48-error-handling-and-information-leakage)
- [49. Logging and Telemetry](#49-logging-and-telemetry)
- [50. Performance and Security Trade-offs](#50-performance-and-security-trade-offs)
- [51. Memory and Security](#51-memory-and-security)
- [52. Production Security Architecture](#52-production-security-architecture)
- [53. Anti-Patterns](#53-anti-patterns)
- [54. Security Review Checklist](#54-security-review-checklist)
- [55. Debugging Methodology](#55-debugging-methodology)
- [56. Implementation From Scratch](#56-implementation-from-scratch)
- [57. Debugging Exercises](#57-debugging-exercises)
- [58. Code Review Exercise](#58-code-review-exercise)
- [59. Interview Questions](#59-interview-questions)
- [60. Predict-the-Outcome Exercises](#60-predict-the-outcome-exercises)
- [61. Mastery Exercises](#61-mastery-exercises)
- [62. Key Takeaways](#62-key-takeaways)
- [63. Concept Connections](#63-concept-connections)
- [64. Completion Criteria](#64-completion-criteria)
- [65. Revision / Retrieval Record](#65-revision--retrieval-record)
- [66. Canonical References and Source Discipline](#66-canonical-references-and-source-discipline)
- [67. Completion Snapshot](#67-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain why browsers need a security architecture.
2. Define origin precisely.
3. Explain the same-origin policy.
4. Distinguish same-origin from same-site.
5. Explain what cross-origin actions are permitted, restricted, or filtered.
6. Explain CORS as a browser/server response-sharing mechanism.
7. Explain cookies and the security effect of:
   - `Secure`
   - `HttpOnly`
   - `SameSite`
   - `Domain`
   - `Path`
8. Explain CSRF.
9. Explain XSS.
10. Distinguish reflected, stored, and DOM-based XSS.
11. Identify dangerous DOM sinks.
12. Explain context-sensitive output encoding.
13. Explain sanitization and its limits.
14. Explain Content Security Policy.
15. Design a strict CSP.
16. Explain nonces and hashes.
17. Explain Trusted Types.
18. Explain Subresource Integrity.
19. Explain mixed-content protection.
20. Explain clickjacking and framing defenses.
21. Explain iframe sandboxing.
22. Use `postMessage` safely.
23. Explain cross-origin isolation.
24. Explain COOP, COEP, and CORP.
25. Explain origin-agent isolation at a conceptual level.
26. Explain Permissions Policy.
27. Explain Fetch Metadata request headers.
28. Explain browser storage security boundaries.
29. Explain service-worker security.
30. Explain worker isolation.
31. Explain Web Component security boundaries.
32. Explain supply-chain risk in frontend applications.
33. Explain why frontend secrets are not secrets.
34. Explain authentication vs authorization.
35. Build a browser threat model.
36. Audit security headers.
37. Diagnose CSP, CORS, cookie, framing, and isolation failures.
38. Review browser code for XSS and credential leakage.
39. Design a production security architecture.
40. Defend security decisions at principal-engineer level.

### Mastery target

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

---

# 2. Prerequisites

## Required

### Chapter 49 — DOM Architecture

Needed for:

```text
DOM tree
documents
iframes
elements
shadow DOM
```

### Chapter 50 — Browser Events

Needed for:

```text
event paths
postMessage
event propagation
shadow boundaries
```

### Chapter 51 — Browser APIs

Needed for understanding host capabilities and permission boundaries.

### Chapter 54 — Web Components

Needed for:

```text
Shadow DOM
custom elements
slots
component boundaries
```

### Chapter 55 — Fetch / HTTP Networking

Needed for:

```text
CORS
credentials
cookies
headers
HTTP
service workers
```

---

# 3. What Browser Security Is

Browser security is the set of mechanisms that constrain what code loaded into a browser can:

```text
read
write
execute
embed
communicate with
persist
access
control
```

The primary problem is:

> **A browser simultaneously executes code from many mutually untrusted origins.**

For example:

```text
yourbank.example
social.example
news.example
attacker.example
```

may all be open in the same browser.

The browser must keep them from freely interacting.

---

# 4. Why Browser Security Exists

Without browser isolation:

```text
evil.example
    ↓
reads bank.example data
    ↓
reads mail
    ↓
steals session state
    ↓
modifies pages
    ↓
captures private information
```

Browser security establishes barriers.

A useful model:

```text
untrusted web code
      ↓
browser security policy
      ↓
allowed capabilities only
```

---

# 5. Mental Model

Use this security stack:

```text
                   Browser
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
       Origin                 Permissions
          │                       │
          ↓                       ↓
       SOP/CORS             feature policies
          │
          ↓
   ┌──────┼──────────┐
   ↓      ↓          ↓
 Cookies  DOM       Network
   │      │          │
   ↓      ↓          ↓
 CSRF    XSS      CORS/Fetch
   │      │          │
   └──────┼──────────┘
          ↓
     Security Policy
          │
   ┌──────┼─────────────┐
   ↓      ↓             ↓
 CSP    Trusted Types   Isolation
```

No single layer is sufficient.

---

# 6. Trust Boundaries

A security boundary exists when different trust levels meet.

Examples:

```text
browser ↔ server
page ↔ iframe
origin A ↔ origin B
application ↔ third-party script
trusted code ↔ user input
HTML parser ↔ untrusted string
frontend ↔ API
main thread ↔ worker
component ↔ page
```

---

## Principal Security Question

For every data flow ask:

```text
Who controls the input?
Who can read the output?
What authority does the receiver have?
What policy prevents misuse?
```

---

# 7. Origin

Two URLs have the same origin when their:

```text
scheme
host
port
```

match.

MDN defines origin using the scheme/host/port tuple and describes the same-origin policy as a foundational browser mechanism for restricting cross-origin interactions. citeturn635273search6

---

## Example

These are different origins:

```text
https://example.com
http://example.com
https://api.example.com
https://example.com:8443
```

because at least one origin component differs.

---

# 8. Same-Origin Policy

The same-origin policy (SOP) restricts how one origin can interact with another origin.

It is not:

```text
no cross-origin network traffic
```

It is closer to:

```text
cross-origin capabilities are restricted by context
```

MDN explains that the same-origin policy prevents malicious documents/scripts from freely reading data from other origins and also discusses the different rules for network access, DOM access, storage, and embedding. citeturn635273search6

---

## 8.1 Cross-Origin Does Not Mean Everything Is Blocked

Depending on the API, cross-origin operations may include:

```text
embedding
sending requests
loading images
loading scripts
navigating
postMessage
reading responses
```

with different restrictions.

---

# 9. Origin Isolation vs Resource Sharing

A common mistake:

```text
same-origin policy = block all cross-origin requests
```

Incorrect.

Instead:

```text
same-origin policy
→ baseline isolation

CORS
→ explicit response-sharing permission

CSP
→ resource/execution restrictions

COEP/CORP
→ embedding/resource isolation

COOP
→ browsing-context isolation
```

These solve different problems.

---

# 10. CORS

CORS allows a server to explicitly tell a browser:

```text
this origin may read this response
```

Example:

```http
Access-Control-Allow-Origin: https://app.example.com
```

CORS is part of Fetch/HTTP processing rather than JavaScript language semantics.

See Chapter 55 for request-level CORS details.

---

## Important

CORS does not make your API authenticated.

CORS does not replace:

```text
authorization
CSRF protection
server-side access control
```

---

# 11. Cookies

Cookies are browser-managed state associated with websites and requests.

Typical use:

```text
session cookie
```

Browser:

```text
request
+
Cookie: session=...
```

---

## 11.1 HttpOnly

```http
Set-Cookie: session=...; HttpOnly
```

helps prevent ordinary JavaScript from reading that cookie via `document.cookie`.

---

## 11.2 Secure

```http
Set-Cookie: session=...; Secure
```

restricts normal cookie transmission to secure contexts/HTTPS requests according to cookie rules.

---

## 11.3 SameSite

Common values:

```text
Strict
Lax
None
```

These influence cross-site cookie sending.

---

# 12. Cookie Security Attributes

## `HttpOnly`

Reduces JavaScript access to cookie contents.

It does **not** stop an attacker with XSS from causing authenticated requests using the browser's cookie.

---

## `Secure`

Protects against ordinary transmission over insecure HTTP.

It does not stop:

```text
XSS
CSRF
server compromise
```

---

## `SameSite`

Can reduce CSRF exposure by restricting cross-site cookie attachment.

Still use defense in depth.

---

## `Domain`

Controls which hosts receive the cookie.

Avoid broad domain scope when not required.

---

## `Path`

Restricts matching by URL path, but should not be treated as an authorization boundary.

---

# 13. CSRF

**Cross-Site Request Forgery** occurs when a victim's browser is induced to send an authenticated state-changing request that the victim did not intentionally initiate.

Common target:

```text
POST /transfer
```

with a session cookie automatically attached.

---

## Attack Model

```text
Victim logged into bank.example

attacker.example
    ↓
causes browser to request
    ↓
bank.example/transfer
    ↓
browser sends cookie
```

---

## Defenses

```text
SameSite cookies
CSRF tokens
Origin validation
Referer validation where appropriate
application-specific request controls
```

MDN notes CSRF tokens as a defense for preventing unauthorized cross-origin writes and emphasizes that CORS alone is not the same as a CSRF defense. citeturn635273search6

---

# 14. XSS

**Cross-Site Scripting** occurs when attacker-controlled content is interpreted as executable script in a victim's security context.

Common categories:

```text
stored
reflected
DOM-based
```

---

## Why XSS Is Serious

XSS can allow malicious script to act with the page's authority:

```text
read page data
modify DOM
issue requests
steal accessible tokens
capture input
perform user actions
```

`HttpOnly` cookies reduce direct cookie theft but do not make an XSS-infected application safe.

---

# 15. DOM XSS

DOM XSS happens when application code takes attacker-controlled data and writes it into a dangerous DOM sink.

Example:

```js
const value =
  new URL(location.href).searchParams.get("name");

target.innerHTML = value;
```

The vulnerability is the data flow:

```text
untrusted source
→ dangerous sink
```

---

## Common Dangerous Sinks

```text
innerHTML
outerHTML
insertAdjacentHTML
document.write
eval
Function
setTimeout(string)
setInterval(string)
javascript: URLs
dynamic script injection
```

MDN identifies `innerHTML` and `Document.write()` as examples of injection sinks that can become XSS vulnerabilities when attacker-controlled input reaches them. citeturn635273search9

---

# 16. Output Encoding and Contexts

There is no single universal “escape everything” function.

Context matters.

```text
HTML text
HTML attribute
JavaScript string
CSS value
URL
```

require different handling.

---

## Safer Default

Instead of:

```js
element.innerHTML = userInput;
```

prefer:

```js
element.textContent = userInput;
```

when plain text is intended.

---

# 17. Sanitization

Sometimes applications intentionally accept HTML:

```text
rich text editor
CMS content
user formatting
```

Then sanitization may be appropriate.

---

## Sanitization Pipeline

```text
untrusted HTML
    ↓
parser
    ↓
sanitizer
    ↓
trusted representation
    ↓
HTML sink
```

---

## Principal Rule

Do not write a one-line homemade regex sanitizer.

HTML parsing is not regular-expression territory.

---

# 18. Content Security Policy

**Content Security Policy (CSP)** allows developers to control what resources a page may load or execute and many security-relevant behaviors.

The W3C CSP Level 3 specification is a current Working Draft dated August 13, 2026. It defines CSP as a mechanism for controlling resources a page can fetch or execute and related security decisions. citeturn635273search0

---

## Basic Example

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  object-src 'none';
```

---

## Why CSP Exists

Even if a bug reaches:

```text
HTML injection
```

CSP can reduce the ability to execute arbitrary script.

CSP is defense in depth, not a substitute for fixing injection.

---

# 19. CSP Directives

Important directives include:

```text
default-src
script-src
style-src
img-src
font-src
connect-src
media-src
frame-src
frame-ancestors
object-src
base-uri
form-action
worker-src
manifest-src
upgrade-insecure-requests
```

---

## `script-src`

Controls script execution sources.

---

## `connect-src`

Controls sources for network connections such as:

```text
Fetch
XHR
WebSocket
EventSource
```

---

## `frame-ancestors`

Controls which pages may embed the document.

Important clickjacking defense.

---

## `object-src`

Commonly:

```http
object-src 'none'
```

to disable legacy plugin embedding.

---

# 20. Nonces, Hashes, and Strict CSP

## Nonce

Server generates unpredictable per-response value:

```http
Content-Security-Policy:
  script-src 'nonce-randomValue'
```

Then trusted script:

```html
<script nonce="randomValue">
  ...
</script>
```

The nonce must be unpredictable and fresh enough for the security model.

---

## Hash

CSP can authorize specific inline script content using a cryptographic hash.

---

## Why Nonce/Hash Beats Host Allowlists

A broad allowlist such as:

```http
script-src https://cdn.example.com
```

may permit more script-loading paths than intended.

Modern strict CSP designs often focus on:

```text
nonces
hashes
trusted execution paths
```

---

# 21. Trusted Types

Trusted Types can constrain dangerous DOM sinks to require typed trusted values rather than ordinary strings.

MDN documents `require-trusted-types-for` as a CSP directive that can control data passed to DOM XSS sinks such as `Element.innerHTML`, reducing the attack surface by requiring Trusted Type values. citeturn635273search5

---

## Example Policy

Conceptually:

```http
Content-Security-Policy:
  require-trusted-types-for 'script';
```

---

## Trusted Policy

```js
const policy = trustedTypes.createPolicy(
  "app",
  {
    createHTML(value) {
      return sanitize(value);
    }
  }
);
```

Then:

```js
target.innerHTML = policy.createHTML(input);
```

---

## Policy Allowlist

The CSP `trusted-types` directive can constrain which named policies may be created.

MDN documents this allowlisting behavior and notes that the policy implementation itself must still be safe; merely wrapping unsanitized data in a Trusted Type does not make it safe. citeturn635273search1

---

# 22. Subresource Integrity

**SRI** allows a page to specify a cryptographic integrity hash for externally loaded resources.

Example:

```html
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-..."
  crossorigin="anonymous">
</script>
```

---

## Threat Model

Without integrity checking:

```text
CDN compromised
→ malicious script
→ application compromised
```

With SRI:

```text
download
→ hash check
→ mismatch
→ resource rejected
```

Use SRI where static external resource integrity is important and operationally practical.

---

# 23. Mixed Content

A secure HTTPS page should not casually load insecure HTTP resources.

Examples:

```text
HTTPS page
  ↓
HTTP script
```

is dangerous because the insecure resource can be modified in transit.

Modern browsers actively block or upgrade/restrict mixed content according to resource type and policy.

---

# 24. Clickjacking and Framing

**Clickjacking** tricks a user into interacting with a UI hidden or framed by an attacker.

Defenses include:

```http
Content-Security-Policy:
  frame-ancestors 'none';
```

and, where appropriate:

```http
X-Frame-Options: DENY
```

Prefer CSP `frame-ancestors` for modern policy design while understanding legacy compatibility requirements.

---

# 25. iframe Sandbox

HTML supports:

```html
<iframe
  sandbox
  src="https://untrusted.example">
</iframe>
```

The HTML Standard defines sandbox tokens that restrict capabilities such as scripts, forms, popups, downloads, navigation, and other behaviors. citeturn635273search10

---

## Example

```html
<iframe
  sandbox="allow-scripts"
  src="https://untrusted.example">
</iframe>
```

---

## Principle

Start restrictive.

Add capabilities only when required:

```text
allow-scripts
allow-forms
allow-popups
allow-downloads
...
```

---

## Dangerous Combination

Be especially careful with:

```text
sandbox
+
allow-scripts
+
allow-same-origin
```

when embedding same-origin untrusted content, because the combination can undermine the intended sandbox boundary.

---

# 26. postMessage

Cross-origin windows cannot freely access each other's DOM, but they can communicate with:

```js
otherWindow.postMessage(message, targetOrigin);
```

---

## Secure Receiver

Do not simply:

```js
window.addEventListener("message", event => {
  process(event.data);
});
```

Validate:

```js
event.origin
event.source
message schema
expected channel
```

---

## Example

```js
window.addEventListener("message", event => {
  if (event.origin !== "https://trusted.example") {
    return;
  }

  if (event.source !== expectedWindow) {
    return;
  }

  if (event.data?.type !== "AUTH_RESULT") {
    return;
  }

  // Process validated message.
});
```

---

# 27. Cross-Origin Isolation

Some high-powered capabilities require a document to become cross-origin isolated.

A common deployment model uses:

```text
COOP
+
COEP
```

along with compatible resource behavior.

This can enable APIs such as:

```text
SharedArrayBuffer
high-resolution timing behavior under isolation
```

subject to browser/runtime rules.

---

# 28. COOP

**Cross-Origin-Opener-Policy** controls relationships between browsing contexts/windows.

For example:

```http
Cross-Origin-Opener-Policy: same-origin
```

can isolate a document into a separate browsing-context group.

MDN explains that COOP controls whether documents are opened into the same or a new browsing context group and is used to isolate same-origin/trusted relationships from unrelated cross-origin content. citeturn635273search3

---

## Security Goal

Reduce unwanted interaction between:

```text
window
popup
opener
cross-origin page
```

---

# 29. COEP

**Cross-Origin-Embedder-Policy** controls what cross-origin resources a document may embed.

Common values:

```text
unsafe-none
require-corp
credentialless
```

MDN documents `require-corp` and `credentialless` and explains how they require resource consent through CORS/CORP or omit credentials for specific no-CORS loads. citeturn635273search4

---

## `require-corp`

Cross-origin resources generally need an explicit permission mechanism:

```text
CORS
or
CORP
```

---

## `credentialless`

For relevant no-CORS resource loads, credentials can be omitted, changing the resource-sharing requirements.

---

# 30. CORP

**Cross-Origin-Resource-Policy** lets a resource declare who may embed it.

Example:

```http
Cross-Origin-Resource-Policy: same-origin
```

or:

```http
Cross-Origin-Resource-Policy: cross-origin
```

MDN describes CORP as an additional protection layer against cross-origin resource leaks and notes that it is relevant to no-CORS requests and interaction with COEP. citeturn635273search7

---

## Mental Model

```text
COOP
→ opener/window isolation

COEP
→ what my document may embed

CORP
→ who may embed my resource
```

---

# 31. Origin-Agent Isolation

Browsers can apply stronger process/agent isolation to an origin using mechanisms such as:

```http
Origin-Agent-Cluster: ?1
```

This is a browser isolation feature rather than an application authorization system.

---

## Important

Isolation improves security boundaries between unrelated origins, but it does not prevent:

```text
XSS within your own origin
```

from controlling your application.

---

# 32. Permissions Policy

Permissions Policy controls which browser features a document and its frames may use.

Examples of capabilities can include:

```text
camera
microphone
geolocation
fullscreen
payment
```

W3C's Web Application Security working group lists Permissions Policy as a current specification defining mechanisms to selectively enable or disable browser features and APIs. citeturn635273search2

---

## Example Concept

```http
Permissions-Policy:
  camera=(),
  microphone=()
```

This can reduce the capability available to the document and embedded frames.

---

# 33. Fetch Metadata

Browsers can send request metadata headers such as:

```text
Sec-Fetch-Site
Sec-Fetch-Mode
Sec-Fetch-Dest
Sec-Fetch-User
```

Servers can use these signals as part of request filtering and CSRF defense.

---

## Example Policy

A server may distinguish:

```text
same-origin
same-site
cross-site
none
```

and reject dangerous cross-site state-changing requests.

---

## Important

Fetch Metadata is defense in depth.

Do not use it as the only authorization mechanism.

---

# 34. Storage and Privacy Boundaries

Browser storage includes:

```text
cookies
localStorage
sessionStorage
IndexedDB
Cache API
service-worker state
```

Security questions:

```text
which origin owns it?
which contexts can access it?
is it partitioned?
is it sent automatically?
can JavaScript read it?
what happens in an embedded context?
```

---

## Critical Difference

```text
HttpOnly cookie
→ JS cannot normally read value

localStorage
→ JS can read value
```

Therefore:

```text
XSS risk
```

has different consequences for different storage models.

---

# 35. Storage Partitioning

Modern browsers increasingly partition certain state by more than just origin in embedded/cross-site contexts to reduce tracking and data leakage.

This affects:

```text
third-party cookies
storage
cache
embedded frames
```

---

## Production Rule

Do not assume:

```text
third-party iframe
→ always shares all storage with top-level site
```

Browser privacy behavior evolves and may differ across engines.

---

# 36. Private Network / Local Network Concerns

Browsers increasingly add controls around requests from public websites toward local/private network resources.

The security goal is to reduce abuse of the user's network position and local services.

---

## Architecture Lesson

Do not assume a public origin can freely reach:

```text
localhost
LAN devices
private network services
router admin panels
```

browser policy can intervene.

---

# 37. Service Workers and Security

A service worker can intercept requests for its controlled scope.

Therefore it has significant authority.

Threats include:

```text
malicious service worker
stale compromised cache
overbroad scope
unsafe response rewriting
```

---

## Registration Security

Only register service workers from trusted origins and understand:

```text
scope
update
activation
cache ownership
```

---

## Incident Response

If a malicious service-worker script has been deployed, simply changing application UI may not be sufficient because cached/intercepted resources can persist.

Design update and revocation procedures.

---

# 38. Web Workers and Security

Workers isolate:

```text
execution context
```

but do not inherently provide a security boundary against untrusted application code.

A worker still belongs to the page/origin security context subject to browser rules.

---

## Worker Capabilities

Workers generally cannot directly access normal DOM APIs.

This reduces accidental DOM manipulation but does not make worker code trusted automatically.

---

# 39. Web Components and Security

Shadow DOM provides encapsulation, not security isolation.

A closed root:

```js
attachShadow({ mode: "closed" });
```

does not protect:

```text
secrets
tokens
trusted decisions
```

from code already executing with sufficient page authority.

---

## Public Component Security

Treat component APIs as trust boundaries:

```text
properties
attributes
events
slots
parts
```

Validate:

```text
data
URLs
HTML
commands
```

---

# 40. Supply Chain Security

Frontend applications often execute third-party code:

```text
npm package
CDN script
analytics
ads
widgets
SDKs
tag managers
browser extensions
```

Any script executing in your origin can often act with major page authority.

Therefore:

```text
dependency = code trust decision
```

---

## Supply Chain Failure

```text
trusted package
   ↓
maintainer compromise
   ↓
malicious release
   ↓
build
   ↓
production
   ↓
browser
   ↓
application compromise
```

---

# 41. Dependency and Script Loading

Security decisions include:

```text
lockfiles
dependency pinning
review
SBOM
vulnerability scanning
provenance
SRI
CSP
minimal dependencies
permissions/capability review
```

---

## Do Not Assume

```text
popular package
→ safe forever
```

Security is temporal.

---

# 42. Authentication and Session Security

## Cookie Session

Common architecture:

```text
browser
 ↓
HttpOnly Secure SameSite cookie
 ↓
server session
```

Advantages:

```text
JS does not need direct access to session token
```

but CSRF defenses remain important.

---

## Token-Based Browser Auth

Tokens kept in JavaScript-accessible storage can become directly exposed through XSS.

There is no magic storage location that makes an XSS-compromised origin fully safe.

---

# 43. Authorization

The browser is not the final authority.

Do not rely on:

```js
if (user.role === "admin") {
  showDeleteButton();
}
```

for actual security.

Server must enforce:

```text
resource ownership
roles
permissions
tenant boundaries
business rules
```

---

# 44. Secrets in the Browser

Anything shipped to the browser must be considered obtainable by the user or code executing in that browser security context.

Never treat:

```text
API secret
private signing key
database password
cloud root key
```

as safe merely because it is hidden in frontend source/minified code.

---

## Public Configuration Is Different

Safe examples can include:

```text
public API base URL
public client ID
feature flags
build metadata
```

when intentionally designed as public.

---

# 45. Browser Security vs Server Security

Browser controls:

```text
SOP
CORS
CSP
cookies
framing
permissions
isolation
```

Server controls:

```text
authentication
authorization
validation
rate limiting
business rules
database access
secret protection
```

---

## Core Principle

```text
browser policy
≠
server authorization
```

You need both.

---

# 46. Threat Modeling

Use a systematic process.

## Step 1 — Identify Assets

Examples:

```text
session
money
PII
tenant data
API capability
credentials
source code
analytics
```

---

## Step 2 — Identify Actors

```text
anonymous user
authenticated user
attacker
malicious site
third-party script
compromised dependency
malicious iframe
browser extension
```

---

## Step 3 — Map Entry Points

```text
URL
query string
hash
form
postMessage
API response
WebSocket
storage
URL redirects
third-party scripts
file uploads
```

---

## Step 4 — Map Sinks

```text
innerHTML
script loading
navigation
Fetch
cookie
postMessage
eval
DOM APIs
```

---

## Step 5 — Identify Trust Boundaries

```text
origin
server
iframe
worker
third party
```

---

## Step 6 — Add Controls

```text
CSP
Trusted Types
SameSite
CSRF
CORS
COOP
COEP
CORP
Permissions Policy
validation
sanitization
authorization
```

---

# 47. Security Headers

A production application may use some combination of:

```http
Content-Security-Policy: ...
Strict-Transport-Security: ...
Cross-Origin-Opener-Policy: ...
Cross-Origin-Embedder-Policy: ...
Cross-Origin-Resource-Policy: ...
Permissions-Policy: ...
Referrer-Policy: ...
X-Content-Type-Options: nosniff
```

---

## Header Selection Principle

Do not deploy headers from a copy-paste checklist without testing.

Every policy can break:

```text
third-party integrations
CDNs
analytics
fonts
workers
iframes
payments
uploads
```

---

# 48. Error Handling and Information Leakage

Avoid returning sensitive information in frontend errors:

```text
stack traces
database messages
internal hostnames
tokens
full request headers
```

---

## Client Error

Prefer:

```json
{
  "code": "ORDER_NOT_FOUND",
  "message": "Order was not found."
}
```

over:

```json
{
  "stack": "...",
  "sql": "SELECT ...",
  "dbHost": "..."
}
```

---

# 49. Logging and Telemetry

Do not log:

```text
Authorization
Cookie
password
reset token
session token
sensitive personal data
```

unless there is a tightly controlled security/operational reason.

---

## Telemetry Threat

Your analytics provider is another data processor/trust boundary.

Ask:

```text
what data is sent?
where?
with what identifiers?
for how long?
who can access it?
```

---

# 50. Performance and Security Trade-offs

Security controls can have costs.

Examples:

```text
strict CSP
→ integration complexity

COEP
→ resource compatibility burden

SRI
→ deployment/versioning complexity

sandbox
→ capability restrictions

heavy sanitization
→ CPU cost

strong validation
→ latency/complexity
```

---

## Principal Decision

Never ask only:

```text
"Is this secure?"
```

Ask:

```text
What threat?
What control?
What assurance?
What cost?
What compatibility impact?
What residual risk?
```

---

# 51. Memory and Security

Memory management interacts with security.

Examples:

```text
sensitive data retained too long
large attacker-controlled payload
unbounded cache
DOM explosion
detached sensitive nodes
```

---

## Rule

Minimize retention of:

```text
credentials
PII
cryptographic material
large attacker-controlled strings
```

where practical.

---

# 52. Production Security Architecture

A mature browser security architecture might look like:

```text
                         HTTPS
                           │
                    ┌──────┴──────┐
                    ↓             ↓
                  Browser       Server
                    │             │
          ┌─────────┼─────────┐   │
          ↓         ↓         ↓   │
         SOP       CSP      Cookies│
          │         │         │    │
          ↓         ↓         ↓    │
        CORS   TrustedTypes  CSRF  │
          │         │         │    │
          └─────────┼─────────┘    │
                    ↓              ↓
                 UI/API       Auth/AuthZ
                    │              │
                    └──────┬───────┘
                           ↓
                     Business Rules
```

---

## Recommended Baseline

```text
HTTPS
strict cookie attributes
server authorization
CSP
XSS-safe DOM APIs
input/schema validation
CSRF protection where applicable
secure framing policy
security-aware dependency management
safe postMessage
careful third-party script review
```

---

# 53. Anti-Patterns

## Anti-Pattern 1

```js
element.innerHTML = userInput;
```

without a controlled trust/sanitization model.

---

## Anti-Pattern 2

```http
Access-Control-Allow-Origin: *
```

used as a substitute for authentication/authorization design.

---

## Anti-Pattern 3

Storing long-lived sensitive tokens in JavaScript-readable storage without a strong threat model.

---

## Anti-Pattern 4

Logging:

```js
console.log(requestHeaders);
```

in production where headers can contain credentials.

---

## Anti-Pattern 5

Using:

```text
document.domain
```

as a modern cross-origin integration mechanism.

The current HTML Standard explicitly advises avoiding the `document.domain` setter because it undermines same-origin protections. citeturn635273search8

---

## Anti-Pattern 6

Treating:

```text
closed Shadow DOM
```

as a secret container.

---

## Anti-Pattern 7

Trusting:

```text
frontend role checks
```

as authorization.

---

## Anti-Pattern 8

Allowing arbitrary:

```text
postMessage
```

without origin/schema validation.

---

## Anti-Pattern 9

Copying a CSP from another application without understanding its assets and integrations.

---

## Anti-Pattern 10

Treating a package lockfile as proof of software supply-chain security.

---

## Anti-Pattern 11

Using one global sanitizer without considering HTML vs URL vs CSS vs JavaScript contexts.

---

## Anti-Pattern 12

Allowing third-party scripts with unrestricted access to sensitive page state.

---

# 54. Security Review Checklist

## Origin

- [ ] What are the application's origins?
- [ ] Which origins are trusted?
- [ ] Which cross-origin integrations exist?

## DOM / XSS

- [ ] Where does untrusted input enter?
- [ ] Which sinks consume it?
- [ ] Is textContent sufficient?
- [ ] Is sanitization required?
- [ ] Is Trusted Types viable?

## Network

- [ ] Is HTTPS mandatory?
- [ ] Is CORS explicit?
- [ ] Are credentials intentional?
- [ ] Are redirects safe?
- [ ] Are sensitive URLs avoided?

## Cookies

- [ ] HttpOnly where appropriate?
- [ ] Secure?
- [ ] SameSite?
- [ ] Narrow Domain?
- [ ] Narrow Path?

## CSRF

- [ ] Are state-changing requests protected?
- [ ] Is Origin/Referer checked where appropriate?
- [ ] Are CSRF tokens used when required?

## CSP

- [ ] Is CSP present?
- [ ] Is `object-src 'none'` appropriate?
- [ ] Are scripts nonce/hash controlled?
- [ ] Is `frame-ancestors` configured?
- [ ] Is `connect-src` intentional?

## Isolation

- [ ] COOP?
- [ ] COEP?
- [ ] CORP?
- [ ] Permissions Policy?
- [ ] iframe sandbox?

## Messaging

- [ ] postMessage origin validation?
- [ ] source validation?
- [ ] schema validation?

## Supply Chain

- [ ] Dependency review?
- [ ] lockfile?
- [ ] vulnerability monitoring?
- [ ] third-party scripts minimized?
- [ ] SRI where appropriate?

## Server

- [ ] Authentication?
- [ ] Authorization?
- [ ] Input validation?
- [ ] Rate limiting?
- [ ] Tenant isolation?
- [ ] Audit logging?

---

# 55. Debugging Methodology

When security behavior is surprising, classify the boundary first.

## Step 1 — Identify Origin

Write exact:

```text
scheme
host
port
```

for both sides.

---

## Step 2 — Identify Context

Is this:

```text
DOM access?
Fetch?
iframe?
script?
image?
worker?
storage?
postMessage?
```

Different APIs have different security rules.

---

## Step 3 — Identify Policy

Check:

```text
SOP
CORS
CSP
cookie rules
Permissions Policy
COOP
COEP
CORP
sandbox
```

---

## Step 4 — Inspect Network

Look for:

```text
Origin
Access-Control-*
Cookie
Set-Cookie
Sec-Fetch-*
Cross-Origin-*
Content-Security-Policy
```

---

## Step 5 — Identify Trust

Ask:

```text
Was attacker-controlled data involved?
Was third-party code involved?
Was a credential attached?
```

---

## Step 6 — Reduce

Reproduce with:

```text
one page
one origin
one request
one header
one policy
```

Security bugs become easier once the boundary is isolated.

---

# 56. Implementation From Scratch

Build a **toy browser security model**.

## Stage 1 — Origin Object

```js
class Origin {
  constructor(scheme, host, port) {
    this.scheme = scheme;
    this.host = host;
    this.port = port;
  }

  equals(other) {
    return this.scheme === other.scheme &&
           this.host === other.host &&
           this.port === other.port;
  }
}
```

---

## Stage 2 — Same-Origin Policy

Implement:

```js
canRead(sourceOrigin, targetOrigin)
```

with:

```text
same origin → allow
different origin → deny
```

then layer context-specific exceptions.

---

## Stage 3 — CORS

Model:

```text
request Origin
+
response Access-Control-Allow-Origin
```

and determine whether response data is exposed.

---

## Stage 4 — Cookies

Implement:

```text
Domain
Path
Secure
HttpOnly
SameSite
```

matching logic.

---

## Stage 5 — CSRF Model

Simulate:

```text
attacker site
→ authenticated browser
→ state-changing request
```

then implement:

```text
SameSite
CSRF token
Origin validation
```

---

## Stage 6 — XSS Flow

Implement:

```text
input
→ dangerous sink
→ script execution
```

Then replace with:

```text
textContent
```

and compare.

---

## Stage 7 — CSP Evaluator

Model:

```text
script source
→ policy
→ allow/deny
```

Implement nonce matching.

---

## Stage 8 — postMessage

Model:

```text
window A
→ message
→ origin validation
→ receiver
```

---

## Stage 9 — Frame Sandbox

Represent:

```text
allow-scripts
allow-forms
allow-popups
allow-same-origin
```

as capabilities.

---

## Stage 10 — Threat Model Engine

Given:

```text
asset
actor
entry
trust boundary
```

produce:

```text
threat
control
residual risk
```

---

# 57. Debugging Exercises

## Exercise 1 — SOP

Page:

```text
https://app.example.com
```

tries to read DOM of:

```text
https://admin.example.com
```

Explain why ordinary cross-origin DOM access is restricted.

---

## Exercise 2 — CORS

API returns:

```http
Access-Control-Allow-Origin: https://app.example.com
```

Request comes from:

```text
https://evil.example
```

Predict browser behavior for response exposure.

---

## Exercise 3 — Credentials

Cross-origin request:

```js
fetch(api, {
  credentials: "include"
});
```

What additional server/browser conditions matter?

---

## Exercise 4 — CSRF

A state-changing endpoint relies entirely on a session cookie.

Design a CSRF attack and three defenses.

---

## Exercise 5 — DOM XSS

```js
target.innerHTML =
  new URL(location.href).searchParams.get("q");
```

Identify:

```text
source
sink
exploit path
safe alternative
```

---

## Exercise 6 — CSP

Policy:

```http
script-src 'self'
```

Page contains:

```html
<script>
  console.log("hello");
</script>
```

Does this automatically allow every inline script? Explain the role of CSP inline-script rules.

---

## Exercise 7 — Trusted Types

Enable:

```http
require-trusted-types-for 'script'
```

Then execute:

```js
element.innerHTML = "<b>hello</b>";
```

What security behavior should you expect in supporting browsers?

---

## Exercise 8 — postMessage

Receiver:

```js
window.addEventListener("message", e => {
  if (e.data.type === "DELETE") {
    deleteAccount();
  }
});
```

Find the security flaws.

---

## Exercise 9 — iframe Sandbox

Explain the risks and capability differences among:

```text
sandbox
sandbox="allow-scripts"
sandbox="allow-scripts allow-same-origin"
```

---

## Exercise 10 — COOP/COEP

A document needs:

```text
SharedArrayBuffer
```

but is not cross-origin isolated.

Design the isolation configuration and identify resource compatibility concerns.

---

## Exercise 11 — CORP

A cross-origin image/resource is blocked after COEP is enabled.

Determine what headers or CORS behavior must be investigated.

---

## Exercise 12 — Third-Party Script

An analytics vendor asks for:

```text
full DOM access
all form events
localStorage
```

Perform a security review.

---

# 58. Code Review Exercise

Review:

```js
const token =
  localStorage.getItem("token");

async function loadProfile() {
  const response = await fetch(
    "https://api.example.com/profile",
    {
      headers: {
        Authorization: `Bearer ${token}`
      }
    }
  );

  const data = await response.json();

  document.querySelector("#name").innerHTML =
    data.name;

  window.addEventListener("message", e => {
    if (e.data === "REFRESH") {
      loadProfile();
    }
  });

  console.log({
    headers: response.headers,
    token
  });
}
```

Developer says:

> “It uses HTTPS, so it is secure.”

It is not production-safe.

## Problems

1. Sensitive token is stored in JavaScript-readable storage.
2. XSS can potentially access the token.
3. Network response is trusted without schema validation.
4. `innerHTML` creates an XSS sink.
5. `postMessage` lacks origin validation.
6. Message listener is registered every time the function runs.
7. Token is logged.
8. Response metadata may be over-logged.
9. No timeout.
10. No cancellation.
11. No HTTP status classification.
12. No authorization reasoning.
13. Third-party API origin trust is implicit.
14. No CSP is visible.
15. No Trusted Types strategy is visible.
16. No CSRF analysis if cookie auth is later introduced.
17. HTTPS alone does not address client-side injection or confused-deputy problems.

A stronger architecture should define:

```text
authentication
+
authorization
+
XSS defense
+
CSP
+
Trusted Types where practical
+
message validation
+
schema validation
+
safe telemetry
+
request lifecycle
```

---

# 59. Interview Questions

## Foundations

1. What is browser security?
2. Why does the browser need a same-origin policy?
3. What is an origin?
4. What makes two URLs same-origin?
5. Same-origin vs same-site?
6. Can cross-origin requests ever be sent?

## SOP / CORS

7. What does SOP protect?
8. What is CORS?
9. What does Access-Control-Allow-Origin mean?
10. Why is CORS not authorization?
11. Why does a request work in curl but fail in browser JavaScript?
12. What is preflight?
13. What does `credentials: include` change?

## Cookies / CSRF

14. What does HttpOnly do?
15. What does Secure do?
16. What does SameSite do?
17. Can HttpOnly prevent CSRF?
18. Can HttpOnly prevent XSS?
19. How do you prevent CSRF?

## XSS

20. What is reflected XSS?
21. Stored XSS?
22. DOM XSS?
23. What are common dangerous sinks?
24. Why is context-sensitive encoding required?
25. When is sanitization required?

## CSP / Trusted Types

26. What is CSP?
27. Nonce vs hash?
28. What is strict CSP?
29. What is Trusted Types?
30. What does `require-trusted-types-for` do?
31. Why can Trusted Types policies still be unsafe?

## Isolation

32. What is clickjacking?
33. What does `frame-ancestors` do?
34. What is iframe sandbox?
35. What is `postMessage`?
36. What must be validated in a message handler?
37. What is COOP?
38. What is COEP?
39. What is CORP?
40. What is cross-origin isolation?

## Architecture

41. Why aren't frontend secrets secret?
42. Why is server-side authorization mandatory?
43. How do third-party scripts affect your trust boundary?
44. How would you secure a payment page?
45. How would you secure a multi-tenant SPA?
46. How would you isolate untrusted customer HTML?
47. When would you use iframe sandboxing?
48. How would you design a CSP for a large application?

## Principal-Level

49. Threat-model a banking SPA.
50. Threat-model a design-system component library.
51. Threat-model a SaaS application embedding customer content.
52. Design browser isolation for sensitive workflows.
53. Design a supply-chain security strategy for frontend dependencies.
54. Design a security posture for a microfrontend architecture.
55. Explain how an XSS bug can bypass otherwise strong network/security controls.
56. Explain why browser security must be layered rather than header-based.

---

# 60. Predict-the-Outcome Exercises

## Exercise 1 — Origin

Compare:

```text
https://example.com
https://example.com:443
```

Determine whether they represent the same origin under standard URL/origin normalization.

---

## Exercise 2 — Cross-Origin DOM

Page A tries:

```js
frame.contentWindow.document.body
```

against a cross-origin frame.

Predict access behavior.

---

## Exercise 3 — Cookie

Cookie:

```http
Set-Cookie:
  session=abc;
  HttpOnly;
  Secure;
  SameSite=Lax
```

Can ordinary page JavaScript read `session` using `document.cookie`?

---

## Exercise 4 — XSS

```js
element.textContent = "<img src=x onerror=alert(1)>";
```

Predict whether the string is interpreted as HTML.

---

## Exercise 5 — CSP

A policy does not permit unsafe inline scripts.

Page contains an ordinary inline:

```html
<script>alert(1)</script>
```

Predict the security outcome.

---

## Exercise 6 — Trusted Types

With Trusted Types enforcement active:

```js
element.innerHTML = "<img>";
```

Predict the type/security behavior in a supporting browser.

---

## Exercise 7 — postMessage

Sender:

```js
target.postMessage(
  { type: "AUTH", token },
  "*"
);
```

Identify the security concern with `"*"`.

---

## Exercise 8 — iframe

An iframe has:

```html
sandbox
```

Predict whether arbitrary script execution and other capabilities are available by default.

---

## Exercise 9 — CORP

A resource sends:

```http
Cross-Origin-Resource-Policy: same-origin
```

and another origin attempts a relevant cross-origin no-CORS embedding scenario.

What policy should be investigated?

---

## Exercise 10 — COEP

A document uses:

```http
Cross-Origin-Embedder-Policy: require-corp
```

A cross-origin subresource has no CORS permission and no appropriate CORP policy.

Predict the likely result.

---

# 61. Mastery Exercises

## Level 1 — Secure DOM

Build a page that renders hostile strings using only safe text APIs.

---

## Level 2 — XSS Lab

Create:

```text
reflected XSS
stored XSS
DOM XSS
```

in an isolated local lab.

Then fix each.

---

## Level 3 — CSRF Lab

Build a toy authenticated app vulnerable to CSRF.

Add:

```text
SameSite
CSRF token
Origin checking
```

and measure the difference.

---

## Level 4 — Cookie Lab

Test combinations of:

```text
HttpOnly
Secure
SameSite
Domain
Path
```

---

## Level 5 — CORS Lab

Build:

```text
frontend origin
API origin
```

and test:

```text
simple request
preflight
credentials
exposed headers
```

---

## Level 6 — CSP Lab

Create increasingly strict policies:

```text
reporting
basic allowlist
nonce-based
strict
```

Document what breaks.

---

## Level 7 — Trusted Types Lab

Enable Trusted Types and identify every HTML sink that fails.

Create a single audited sanitization policy.

---

## Level 8 — postMessage Lab

Build two origins that communicate securely.

Require:

```text
origin
source
schema
```

validation.

---

## Level 9 — Isolation Lab

Build a pair of documents and experiment with:

```text
COOP
COEP
CORP
```

until the application becomes cross-origin isolated.

Document every resource that needs changes.

---

## Level 10 — Supply Chain Lab

Audit:

```text
npm dependencies
CDN scripts
analytics
tag manager
```

and produce a trust-boundary map.

---

## Level 11 — Secure Multi-Tenant SPA

Design:

```text
tenant A
tenant B
admin
anonymous user
third-party integrations
```

with:

```text
authentication
authorization
CSP
CSRF
CORS
cookie strategy
```

---

## Level 12 — Principal Security Architecture

Design a production security architecture for:

```text
financial SaaS
```

with:

```text
multi-tenant data
payments
PII
customer-uploaded HTML
third-party analytics
embedded reports
microfrontends
Web Workers
service workers
SSR
```

Deliver:

```text
threat model
trust boundaries
security controls
security headers
cookie model
XSS strategy
CSRF strategy
CORS strategy
isolation strategy
supply-chain policy
incident response
residual risk
```

---

# 62. Key Takeaways

1. Browser security exists because many mutually untrusted origins execute in the same browser.
2. Origin is fundamentally based on scheme, host, and port.
3. Same-origin policy is a foundational browser isolation mechanism.
4. SOP does not mean all cross-origin activity is blocked.
5. Different Web APIs apply different cross-origin rules.
6. CORS allows controlled response sharing; it is not authorization.
7. Cookies are powerful authenticated state and require careful security attributes.
8. `HttpOnly` reduces JavaScript cookie access but does not stop XSS.
9. `Secure` protects cookie transmission over insecure HTTP.
10. `SameSite` can reduce cross-site request exposure and CSRF risk.
11. CSRF targets authenticated state-changing actions.
12. XSS executes attacker-controlled code in the page's security context.
13. DOM XSS is fundamentally a source-to-sink data-flow problem.
14. `textContent` is often safer than `innerHTML` for plain text.
15. HTML/URL/CSS/JavaScript contexts require different handling.
16. Sanitization is needed when untrusted rich HTML must be accepted.
17. CSP provides defense in depth against content injection and resource/execution abuse.
18. Strict nonce/hash-based CSP can provide stronger script control than broad host allowlists.
19. Trusted Types can enforce safer DOM sink usage.
20. Trusted Type policy code itself must be trusted.
21. SRI helps verify integrity of static external resources.
22. HTTPS does not solve XSS, CSRF, or authorization failures.
23. Clickjacking is a framing problem.
24. CSP `frame-ancestors` is a key framing defense.
25. iframe sandboxing can reduce embedded-content capabilities.
26. `postMessage` requires origin/source/schema validation.
27. COOP isolates browsing-context relationships.
28. COEP controls cross-origin embedding requirements.
29. CORP allows resource owners to restrict cross-origin embedding.
30. Cross-origin isolation combines browser policies to enable stronger isolation/capabilities.
31. Permissions Policy limits browser feature availability.
32. Fetch Metadata can support server-side cross-site request filtering.
33. Storage mechanisms have different JavaScript visibility and privacy properties.
34. Service workers have substantial authority and must be treated as security-sensitive code.
35. Web Workers provide execution isolation, not universal trust isolation.
36. Shadow DOM is encapsulation, not a security boundary.
37. Third-party frontend dependencies become part of the application's trusted code base.
38. Secrets shipped to the browser are not truly secret.
39. The server must enforce authorization.
40. Security is a layered system, not one header or one API.
41. Every security decision should begin with a threat model and trust boundary.
42. Principal-level browser security requires reasoning across DOM, HTTP, cookies, JavaScript, policies, isolation, and server authorization.

---

# 63. Concept Connections

## Depends On

```text
Chapter 49 — DOM Architecture
        ↓
Chapter 50 — Browser Events
        ↓
Chapter 51 — Browser Web APIs
        ↓
Chapter 54 — Web Components
        ↓
Chapter 55 — Fetch / HTTP Networking
        ↓
Chapter 56 — Browser Security
```

## Builds Toward

```text
Chapter 57 — JS Security Engineering
Chapter 58 — Node Architecture
Chapter 59 — Node Core APIs
Chapter 60 — Node Streams
Chapter 61 — Worker Threads / Processes
Chapter 62 — Process Lifecycle
Chapter 63 — Async Context / Diagnostics
Chapter 67 — Dependency Management / Supply Chain
Chapter 78 — Production Architecture
Chapter 79 — API Design
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

```text
SOP
CORS
cookies
CSRF
XSS
CSP
Trusted Types
SRI
COOP
COEP
CORP
Permissions Policy
Fetch Metadata
sandbox
postMessage
TLS
authentication
authorization
supply chain
```

## Concepts Revisited

### Chapter 49 — DOM

XSS becomes a consequence of DOM data flowing into executable/injectable sinks.

### Chapter 50 — Events

`postMessage`, event boundaries, and event-driven attacks require precise event reasoning.

### Chapter 54 — Web Components

Shadow DOM encapsulates implementation but does not establish security isolation.

### Chapter 55 — Fetch

CORS, credentials, cookies, and cross-origin network requests become security-policy problems.

### Chapter 45 — Memory

Sensitive data retention and lifecycle leaks can increase security impact.

---

## Why This Chapter Matters Later

Chapter 57 turns browser-security concepts into broader JavaScript security engineering.

Later Node/backend chapters will expose a crucial boundary:

```text
browser security
≠
server security
```

A production engineer must design:

```text
browser
+
network
+
server
+
identity
+
data
```

as one security system.

---

# 64. Completion Criteria

## Fundamentals

- [ ] Define browser security.
- [ ] Explain origin.
- [ ] Explain same-origin policy.
- [ ] Explain same-site vs same-origin.
- [ ] Explain trust boundaries.

## CORS / Networking

- [ ] Explain CORS.
- [ ] Explain preflight.
- [ ] Explain credentialed CORS.
- [ ] Explain Fetch Metadata.
- [ ] Explain mixed-content policy.

## Cookies / CSRF

- [ ] Explain HttpOnly.
- [ ] Explain Secure.
- [ ] Explain SameSite.
- [ ] Explain Domain.
- [ ] Explain Path.
- [ ] Explain CSRF.
- [ ] Implement CSRF defenses.

## XSS

- [ ] Explain reflected XSS.
- [ ] Explain stored XSS.
- [ ] Explain DOM XSS.
- [ ] Identify dangerous sinks.
- [ ] Use context-safe output.
- [ ] Explain sanitization.
- [ ] Explain Trusted Types.

## CSP / Integrity

- [ ] Explain CSP.
- [ ] Configure script restrictions.
- [ ] Explain nonces.
- [ ] Explain hashes.
- [ ] Explain frame-ancestors.
- [ ] Explain SRI.

## Isolation

- [ ] Explain clickjacking.
- [ ] Explain iframe sandbox.
- [ ] Explain postMessage.
- [ ] Validate message origin.
- [ ] Explain COOP.
- [ ] Explain COEP.
- [ ] Explain CORP.
- [ ] Explain cross-origin isolation.
- [ ] Explain Permissions Policy.

## Storage / Workers

- [ ] Compare cookies/localStorage/IndexedDB/Cache API.
- [ ] Explain storage privacy boundaries.
- [ ] Explain service-worker risks.
- [ ] Explain worker isolation.

## Application Security

- [ ] Explain authentication.
- [ ] Explain authorization.
- [ ] Explain frontend secret exposure.
- [ ] Threat-model third-party scripts.
- [ ] Threat-model dependencies.

## Production

- [ ] Review security headers.
- [ ] Build a browser threat model.
- [ ] Design XSS defenses.
- [ ] Design CSRF defenses.
- [ ] Design CORS correctly.
- [ ] Design cookie policy.
- [ ] Design isolation policy.
- [ ] Design supply-chain controls.
- [ ] Design safe telemetry.
- [ ] Design incident response.

## Principal Judgment

- [ ] Defend a layered security architecture.
- [ ] Explain residual risk.
- [ ] Identify false security boundaries.
- [ ] Prioritize security controls by threat.
- [ ] Balance security, performance, compatibility, and operational complexity.

---

# 65. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What is an origin?
2. What is SOP?
3. Does SOP block all cross-origin requests?
4. What does CORS do?
5. What does CORS not do?
6. What does HttpOnly do?
7. What does Secure do?
8. What does SameSite do?
9. What is CSRF?
10. What is XSS?
11. What is DOM XSS?
12. Name common dangerous DOM sinks.
13. Why is context-sensitive encoding required?
14. What is sanitization?
15. What is CSP?
16. What is a nonce?
17. What is a CSP hash?
18. What is Trusted Types?
19. What is SRI?
20. What is clickjacking?
21. What does frame-ancestors do?
22. What does iframe sandbox do?
23. What is postMessage?
24. What must a message receiver validate?
25. What is COOP?
26. What is COEP?
27. What is CORP?
28. What is cross-origin isolation?
29. What is Permissions Policy?
30. What are Fetch Metadata headers?
31. Why are frontend secrets not secret?
32. Why is server authorization mandatory?
33. Why are third-party scripts dangerous?
34. How would you threat-model a browser application?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Origin | [ ] | [ ] | [ ] | [ ] |
| SOP | [ ] | [ ] | [ ] | [ ] |
| Same-site | [ ] | [ ] | [ ] | [ ] |
| CORS | [ ] | [ ] | [ ] | [ ] |
| Cookies | [ ] | [ ] | [ ] | [ ] |
| HttpOnly | [ ] | [ ] | [ ] | [ ] |
| Secure | [ ] | [ ] | [ ] | [ ] |
| SameSite | [ ] | [ ] | [ ] | [ ] |
| CSRF | [ ] | [ ] | [ ] | [ ] |
| XSS | [ ] | [ ] | [ ] | [ ] |
| DOM XSS | [ ] | [ ] | [ ] | [ ] |
| Encoding | [ ] | [ ] | [ ] | [ ] |
| Sanitization | [ ] | [ ] | [ ] | [ ] |
| CSP | [ ] | [ ] | [ ] | [ ] |
| Nonces | [ ] | [ ] | [ ] | [ ] |
| Hashes | [ ] | [ ] | [ ] | [ ] |
| Trusted Types | [ ] | [ ] | [ ] | [ ] |
| SRI | [ ] | [ ] | [ ] | [ ] |
| Mixed Content | [ ] | [ ] | [ ] | [ ] |
| Clickjacking | [ ] | [ ] | [ ] | [ ] |
| iframe sandbox | [ ] | [ ] | [ ] | [ ] |
| postMessage | [ ] | [ ] | [ ] | [ ] |
| COOP | [ ] | [ ] | [ ] | [ ] |
| COEP | [ ] | [ ] | [ ] | [ ] |
| CORP | [ ] | [ ] | [ ] | [ ] |
| Isolation | [ ] | [ ] | [ ] | [ ] |
| Permissions Policy | [ ] | [ ] | [ ] | [ ] |
| Fetch Metadata | [ ] | [ ] | [ ] | [ ] |
| Storage security | [ ] | [ ] | [ ] | [ ] |
| Service-worker security | [ ] | [ ] | [ ] | [ ] |
| Worker security | [ ] | [ ] | [ ] | [ ] |
| Supply chain | [ ] | [ ] | [ ] | [ ] |
| Authentication | [ ] | [ ] | [ ] | [ ] |
| Authorization | [ ] | [ ] | [ ] | [ ] |
| Threat modeling | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw:

```text
origin
 ↓
SOP
 ↓
cross-origin policy
 ↓
CORS / cookies / CSP
 ↓
application
```

### Day 2

Explain the difference between:

```text
XSS
CSRF
CORS
CSP
```

without notes.

### Day 7

Build an XSS lab and secure it with:

```text
safe DOM APIs
CSP
Trusted Types
```

where supported.

### Day 14

Threat-model a cookie-authenticated SPA.

### Day 30

Design a complete browser security architecture for a multi-tenant SaaS platform.

---

# 66. Canonical References and Source Discipline

## WHATWG HTML Standard

https://html.spec.whatwg.org/

Use for:

```text
origin
sandboxing
browsing contexts
iframe
postMessage
document.domain
browser security architecture
```

The current HTML Standard explicitly advises avoiding `document.domain` because it weakens same-origin protections. citeturn635273search8

---

## WHATWG Fetch Standard

https://fetch.spec.whatwg.org/

Use for:

```text
CORS
credentials
request modes
response filtering
Fetch Metadata-related concepts
```

Chapter 55 contains the detailed Fetch networking model.

---

## W3C Content Security Policy Level 3

https://www.w3.org/TR/CSP/

The current W3C CSP Level 3 publication is a Working Draft dated August 13, 2026. It defines mechanisms for controlling resources a page can fetch/execute and other security-relevant decisions. citeturn635273search0

---

## W3C Web Application Security

https://www.w3.org/groups/wg/webappsec/publications/

Use for current security specifications such as:

```text
Trusted Types
Permissions Policy
CSP
Subresource Integrity
```

W3C's current Web Application Security publications include Trusted Types, Permissions Policy, and related browser security specifications. citeturn635273search2

---

## MDN Same-Origin Policy

https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy

Use for practical browser-facing SOP behavior. MDN describes the origin tuple and distinguishes cross-origin reads/writes/embeds and CORS-related behaviors. citeturn635273search6

---

## MDN CSP

https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP

Use for practical CSP and Trusted Types explanations. MDN documents dangerous injection sinks, sanitization, and Trusted Types as part of CSP hardening. citeturn635273search9

---

## MDN Trusted Types

https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/require-trusted-types-for

Use for current Trusted Types enforcement behavior. MDN notes that `require-trusted-types-for` controls values passed to DOM XSS sinks. citeturn635273search5

---

## MDN COOP

https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cross-Origin-Opener-Policy

Use for current browsing-context isolation behavior. citeturn635273search3

---

## MDN COEP

https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cross-Origin-Embedder-Policy

Use for current cross-origin embedding and isolation behavior. citeturn635273search4

---

## MDN CORP

https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cross-Origin_Resource_Policy

Use for resource-owner cross-origin embedding policy. citeturn635273search7

---

## Source Classification

Classify every security claim as:

```text
[HTML Standard]
[Fetch Standard]
[W3C Security Spec]
[HTTP Specification]
[Browser Implementation]
[Measured]
[Historical]
```

---

## Security Discipline

Never state:

```text
CORS = authorization
CSP = XSS prevention
HttpOnly = XSS prevention
HTTPS = application security
Shadow DOM = security isolation
frontend role check = authorization
```

because each is only one layer.

---

## Compatibility Discipline

Security features evolve.

Verify support for:

```text
Trusted Types
COEP: credentialless
Permissions Policy features
Fetch Metadata
Origin-Agent-Cluster
advanced isolation behavior
```

against your target browser matrix.

MDN currently labels Trusted Types enforcement/allowlisting features as newly broadly available in 2026, but older browsers may still lack them. citeturn635273search1turn635273search5

---

# 67. Completion Snapshot

## Chapter Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current status:

```text
[ ] Not Started
```

## Knowledge Snapshot

### I can explain

- [ ] browser security
- [ ] trust boundaries
- [ ] origin
- [ ] SOP
- [ ] same-site
- [ ] CORS
- [ ] cookies
- [ ] HttpOnly
- [ ] Secure
- [ ] SameSite
- [ ] CSRF
- [ ] XSS
- [ ] DOM XSS
- [ ] sinks
- [ ] encoding
- [ ] sanitization
- [ ] CSP
- [ ] nonces
- [ ] hashes
- [ ] Trusted Types
- [ ] SRI
- [ ] mixed content
- [ ] clickjacking
- [ ] framing
- [ ] iframe sandbox
- [ ] postMessage
- [ ] COOP
- [ ] COEP
- [ ] CORP
- [ ] cross-origin isolation
- [ ] Permissions Policy
- [ ] Fetch Metadata
- [ ] storage security
- [ ] storage partitioning
- [ ] service-worker security
- [ ] worker security
- [ ] Web Component security
- [ ] supply-chain security
- [ ] authentication
- [ ] authorization
- [ ] threat modeling
- [ ] security headers

### I can predict

- [ ] SOP access
- [ ] CORS response exposure
- [ ] cookie transmission
- [ ] CSRF scenarios
- [ ] XSS sinks
- [ ] CSP outcomes
- [ ] Trusted Types enforcement
- [ ] postMessage exposure
- [ ] iframe sandbox behavior
- [ ] COOP isolation
- [ ] COEP blocking
- [ ] CORP restrictions
- [ ] security-header effects

### I can implement

- [ ] secure DOM rendering
- [ ] XSS defenses
- [ ] CSRF defenses
- [ ] cookie strategy
- [ ] CORS configuration
- [ ] CSP
- [ ] Trusted Types
- [ ] SRI
- [ ] secure postMessage
- [ ] iframe sandbox
- [ ] security headers
- [ ] threat model
- [ ] security review

### I can debug

- [ ] SOP failures
- [ ] CORS failures
- [ ] cookie issues
- [ ] CSRF issues
- [ ] XSS
- [ ] CSP violations
- [ ] Trusted Types violations
- [ ] framing failures
- [ ] postMessage failures
- [ ] COOP/COEP/CORP failures
- [ ] worker/service-worker security behavior

### I can defend

- [ ] cookie vs token auth
- [ ] CSP strategy
- [ ] Trusted Types adoption
- [ ] iframe sandboxing
- [ ] postMessage protocol
- [ ] cross-origin isolation
- [ ] third-party script policy
- [ ] supply-chain controls
- [ ] frontend/backend security boundary
- [ ] security/performance trade-offs
- [ ] principal-level threat model

---

## Final Principal-Level Test

Explain this system without notes:

```text
User
  ↓
Browser
  ↓
Origin
  ↓
Same-Origin Policy
  ↓
DOM / JavaScript
  ↓
untrusted input
  ↓
safe DOM APIs / sanitization / Trusted Types
  ↓
CSP
  ↓
Fetch / CORS / credentials
  ↓
cookies / CSRF
  ↓
TLS
  ↓
server authentication
  ↓
server authorization
  ↓
tenant/business rules
```

Then answer:

```text
What is the trust boundary?
What happens if the page has XSS?
What happens if a third-party script is compromised?
What happens if an API is CORS-enabled incorrectly?
What happens if cookies are misconfigured?
What happens if the browser is framed?
What happens if postMessage is trusted blindly?
What happens if a dependency is compromised?
What does CSP stop?
What does CSP not stop?
What does HttpOnly stop?
What does HttpOnly not stop?
What does CORS stop?
What does CORS not stop?
What does server authorization stop?
What can still happen after all browser controls are correctly configured?
```

Your mastery is complete only when you can reason about browser security as a **layered capability-control system** rather than memorizing individual headers.

The central Chapter 56 lesson is:

> **A secure browser application is built by composing isolation, safe data flow, explicit trust boundaries, strong browser policies, secure credentials, server-side authorization, and disciplined supply-chain controls. No individual browser security feature is a substitute for the complete system.**