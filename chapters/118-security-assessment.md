# Chapter 118 — Security Assessment — 10 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 10 progressive security problems covering trust boundaries, XSS, prototype pollution, code injection, unsafe deserialization, authorization, secrets, dependency/supply-chain risk, browser security, Node.js security, and principal-level threat modeling.
>
> **Role perspective:** Principal JavaScript Engineer · Application Security Engineer · Browser Security Engineer · Node.js Security Engineer · Platform Architect · Security Reviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Treat every boundary crossing as a trust decision.**

---

# 1. Assessment Mission

This assessment evaluates whether you can identify security failures by reasoning about:

```text
attacker-controlled input
→ trust boundary
→ interpretation
→ dangerous capability
→ security invariant violation
→ exploit impact
→ mitigation
→ regression protection
```

The goal is not to memorize:

```text
"never use X"
```

The goal is to understand:

```text
what is trusted
what is untrusted
where it is interpreted
what authority it gains
what invariant must hold
how the system enforces that invariant
```

---

# 2. Scope Discipline

Separate:

```text
ECMAScript language behavior
browser security model
Node.js/runtime security
web platform policies
application authorization
dependency/supply-chain security
deployment/infrastructure controls
```

Do not present:

```text
browser security behavior
```

as an ECMAScript language guarantee.

Likewise, do not assume:

```text
framework defaults
```

replace application-level authorization.

---

# 3. Assessment Scoring

Each question:

```text
0–10 points
```

Total:

```text
100 points
```

Recommended interpretation:

```text
90–100 → Principal-level security reasoning
80–89  → Strong security engineering
70–79  → Good foundation with targeted gaps
60–69  → Significant security gaps
<60     → Rebuild security fundamentals
```

---

# 4. Full-Credit Standard

A full-credit response should include:

```text
1. attacker capability
2. trust boundary
3. vulnerable operation
4. exploit path
5. impact
6. minimal mitigation
7. defense-in-depth mitigation
8. regression test
9. operational control
```

For principal-level answers also include:

```text
abuse cases
blast radius
monitoring
incident response
migration risk
residual risk
```

---

# 5. Question 1 — DOM XSS

Browser code:

```js
const message = new URLSearchParams(location.search).get("message");

document.querySelector("#output").innerHTML = message;
```

An attacker sends a specially crafted URL.

### Task

Identify:

```text
source
sink
trust boundary
security impact
```

Explain why:

```text
URL input
```

must be treated as attacker-controlled.

Provide:

```text
minimal safe fix
defense-in-depth controls
regression test
```

Compare:

```text
textContent
innerHTML
sanitized HTML
trusted static markup
```

---

# 6. Question 2 — Prototype Pollution

```js
function assign(target, source) {
  for (const key in source) {
    target[key] = source[key];
  }

  return target;
}

const input = JSON.parse(userInput);

assign({}, input);
```

### Symptom

A security test demonstrates that attacker-controlled properties can influence inherited object behavior.

### Task

Identify the dangerous path.

Explain:

```text
property key
prototype chain
special properties
inherited behavior
```

Then provide a hardened design.

Your answer should discuss:

```text
own-property checks
allowlists
safe object creation
schema validation
Map
```

Do not assume that:

```text
"JSON.parse is safe"
```

means:

```text
"the parsed object is safe to merge into arbitrary application objects."
```

---

# 7. Question 3 — Code Injection

```js
function evaluateFilter(filter) {
  return Function(`return (${filter})`)();
}
```

The filter is supplied by an authenticated user.

### Task

Explain why authentication does not make dynamic code execution safe.

Identify:

```text
data vs code boundary
execution authority
attacker-controlled input
blast radius
```

Design a safe replacement.

For example, define a constrained representation:

```js
{
  field: "status",
  operator: "equals",
  value: "active"
}
```

Then validate and interpret that data without evaluating JavaScript.

---

# 8. Question 4 — Command Injection in Node.js

```js
import { exec } from "node:child_process";

exec(`grep "${req.query.term}" data.txt`, (error, stdout) => {
  if (error) {
    return res.status(500).end();
  }

  res.send(stdout);
});
```

### Task

Identify:

```text
attacker input
shell interpretation
command boundary
```

Explain why escaping one character is not a sufficiently general security strategy.

Compare:

```text
exec
execFile
spawn
```

and design the safer implementation around:

```text
fixed executable
structured arguments
input validation
least privilege
```

---

# 9. Question 5 — Authorization vs Authentication

```js
app.get("/accounts/:id", requireLogin, async (req, res) => {
  const account = await db.accounts.findById(req.params.id);

  if (!account) {
    return res.status(404).end();
  }

  res.json(account);
});
```

The authenticated user can request another user's account by changing:

```text
:id
```

### Task

Identify the security failure.

Explain the difference between:

```text
authentication
authorization
object-level authorization
identity
ownership
```

Design the authorization check.

Then discuss:

```text
horizontal privilege escalation
```

and how you would test for it.

---

# 10. Question 6 — Secret Exposure

```js
const config = {
  apiKey: process.env.API_KEY
};

res.json(config);
```

The endpoint is accidentally reachable from the frontend.

### Task

Identify:

```text
secret
trust boundary
exposure path
```

Explain why merely storing a secret in:

```text
process.env
```

does not make it safe.

Design a safer configuration model.

Also consider:

```text
logs
error reports
source maps
client bundles
browser storage
CI/CD output
```

---

# 11. Question 7 — Unsafe Object Deserialization

```js
function loadPreferences(raw) {
  const preferences = JSON.parse(raw);

  if (preferences.theme) {
    applyTheme(preferences.theme);
  }

  if (preferences.callback) {
    preferences.callback();
  }

  return preferences;
}
```

### Symptom

A developer assumes JSON input is “just data,” but the application later invokes values based on attacker-controlled structure.

### Task

Explain the difference between:

```text
parsing data
```

and:

```text
trusting semantics
```

Define a validation boundary using:

```text
schema
allowed fields
allowed types
allowed enum values
```

Then explain why:

```text
JSON.parse
```

does not validate business meaning.

---

# 12. Question 8 — Dependency / Supply-Chain Incident

A Node.js application contains:

```text
dependencies: 800+
direct dependencies: 70+
lockfile committed
```

A newly introduced transitive package is later found to contain malicious code.

### Task

You are the security owner.

Design a supply-chain strategy covering:

```text
dependency inventory
lockfiles
review
update policy
advisories
provenance
CI checks
package scripts
least privilege
runtime isolation
secret exposure
incident response
```

Explain why:

```text
"we have a lockfile"
```

does not mean:

```text
"the dependency supply chain is risk-free."
```

---

# 13. Question 9 — Browser Storage Security

An application stores:

```js
localStorage.setItem("sessionToken", token);
```

### Task

Evaluate the design.

Explain the threat model involving:

```text
XSS
same-origin JavaScript
token theft
session lifetime
browser storage
cookies
HttpOnly
Secure
SameSite
```

Do not present one mechanism as universally correct.

Instead state:

```text
what threat is being reduced
what threat remains
```

Then propose a safer session architecture.

---

# 14. Question 10 — Principal-Level Security Incident

A production web application has:

```js
app.post("/admin/export", requireLogin, async (req, res) => {
  const format = req.body.format;
  const filter = req.body.filter;

  const code = `exportData(${JSON.stringify(filter)}, "${format}")`;

  const result = eval(code);

  res.json(result);
});
```

The application also:

```text
1. accepts rich HTML in some user profiles
2. uses a shared admin database credential
3. exposes verbose errors in production
4. installs dependencies directly from public registries
5. logs request bodies for debugging
6. stores long-lived tokens in browser localStorage
7. has no centralized audit trail
```

### Task

Treat this as a principal-level security review.

Identify at least:

```text
8 distinct security risks
```

For each provide:

```text
Risk
Attacker capability
Trust boundary
Exploit path
Impact
Likelihood
Immediate mitigation
Durable fix
Regression test
Monitoring
```

Then prioritize remediation using:

```text
exploitability
impact
blast radius
attack preconditions
ease of mitigation
operational risk
```

Do not simply list vulnerabilities.

Construct a remediation sequence.

---

# 15. Trust-Boundary Worksheet

For every security-sensitive feature write:

```md
## Trust Boundary

### Source
-

### Data Controlled By
-

### Validation
-

### Interpretation
-

### Privilege / Authority
-

### Dangerous Sink
-

### Expected Invariant
-

### Mitigation
-

### Regression Test
-
```

---

# 16. Source → Sink Thinking

Common source categories:

```text
URL
query parameter
form input
JSON body
cookies
headers
WebSocket messages
postMessage
database content
queue payloads
environment variables
uploaded files
third-party API responses
dependency code
```

Common dangerous sinks:

```text
innerHTML
eval
Function
shell execution
SQL construction
HTML rendering
dynamic module loading
filesystem paths
template interpretation
privileged API calls
```

The security question is:

```text
Can attacker-controlled data reach a powerful sink without an enforced safety boundary?
```

---

# 17. Validate at the Boundary

Prefer:

```text
parse
→ validate
→ normalize
→ authorize
→ perform
```

not:

```text
accept
→ use
→ hope
```

Validation should answer:

```text
Is the type correct?

Is the shape correct?

Is the value allowed?

Is the size bounded?

Is the operation authorized?
```

---

# 18. Authentication vs Authorization

Remember:

```text
Authentication:
Who are you?

Authorization:
What are you allowed to do?

Object-level authorization:
Are you allowed to access this specific object?

Action authorization:
Are you allowed to perform this specific operation?
```

A valid session does not grant unrestricted access.

---

# 19. Code vs Data Boundary

High-risk operations include:

```text
eval
new Function
dynamic script construction
shell commands
HTML interpretation
SQL construction
template evaluation
dynamic imports from attacker-controlled paths
```

The safest general strategy is:

```text
represent intent as data
validate the data
interpret only allowed operations
```

---

# 20. Prototype Safety Checklist

For untrusted keys/objects inspect:

```text
prototype mutation
special property names
inherited properties
for...in behavior
merging functions
deep merge utilities
configuration objects
authorization maps
```

Prefer explicit ownership and allowlists for security-sensitive state.

---

# 21. Browser Security Checklist

Review:

```text
XSS
CSRF
CORS assumptions
cookie attributes
CSP
iframe embedding
postMessage origin validation
DOM sinks
storage
token exposure
dependency scripts
third-party resources
```

Do not use:

```text
CORS
```

as an authorization mechanism.

---

# 22. Node.js Security Checklist

Review:

```text
command injection
path traversal
SSRF
prototype pollution
unsafe deserialization
secrets
dependency risk
filesystem permissions
process privileges
environment exposure
request limits
body limits
rate limiting
error exposure
logging
```

---

# 23. Security Regression Testing

For every fixed issue add the appropriate control:

```text
unit test
integration test
authorization test
fuzz/property test
payload validation test
security scanner
dependency audit
dynamic test
penetration test
runtime alert
```

Examples:

```text
XSS fix
→ malicious payload regression test

Authorization fix
→ access-other-user-object test

Command injection fix
→ metacharacter/argument-boundary test

Prototype pollution fix
→ malicious-key regression test
```

---

# 24. Threat Modeling Exercise

For a feature, document:

```text
Assets
Actors
Trust boundaries
Entry points
Privileges
Attack surfaces
Abuse cases
Security invariants
Controls
Detection
Response
```

Ask:

```text
What does the attacker control?

What can they reach?

What authority does the reached component have?

What happens if validation fails?

What happens if authorization fails?

What happens if a dependency is compromised?
```

---

# 25. Defense-in-Depth

Strong systems combine:

```text
input validation
output encoding
authorization
least privilege
sandboxing/isolation
secure defaults
short-lived credentials
monitoring
audit trails
dependency controls
rate limits
resource limits
```

Defense-in-depth is not permission to neglect the primary fix.

---

# 26. Least Privilege

Reduce:

```text
database permissions
filesystem permissions
cloud permissions
process privileges
API scopes
service-account access
dependency capabilities
```

A vulnerability's blast radius is strongly influenced by the authority available after compromise.

---

# 27. Security Observability

Monitor:

```text
authentication failures
authorization failures
suspicious input patterns
admin actions
token anomalies
dependency changes
unexpected process behavior
command execution
resource exhaustion
spikes in error rates
```

Logs should avoid unnecessary sensitive data.

Before logging ask:

```text
Do we need it?

Is it sensitive?

Who can access it?

How long is it retained?

Could an attacker use it?
```

---

# 28. Security Misdiagnosis Taxonomy

Classify failures as:

```text
A — input trusted too early
B — validation confused with authorization
C — authentication confused with authorization
D — data/code boundary violation
E — dangerous sink overlooked
F — prototype/inheritance trust error
G — secret exposure
H — dependency/supply-chain risk ignored
I — browser/Node boundary confusion
J — insufficient privilege isolation
K — no regression test
L — no observability
M — defense-in-depth mistaken for primary control
N — vulnerability listed without exploit reasoning
```

---

# 29. Principal Security Decision Framework

Evaluate remediation using:

```text
Security impact
Exploitability
Blast radius
Correctness
Performance
Memory
Reliability
Maintainability
Scalability
Observability
Operational complexity
Developer experience
Migration risk
Future change
```

Security fixes should reduce risk without introducing a larger operational failure.

---

# 30. Retrieval Record

```md
# Chapter 118 — Security Assessment — Retrieval Record

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

## Strongest Areas
-

## Weakest Areas
-

## XSS / Browser Gaps
-

## Injection Gaps
-

## Authorization Gaps
-

## Prototype / Object-Safety Gaps
-

## Supply-Chain Gaps
-

## Threat-Modeling Gaps
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 31. Spaced Retrieval Schedule

### Day 0

Complete all 10.

### Day 1

Redo every question where:

```text
confidence < 4
```

### Day 3

Rework:

```text
Q1–Q4
```

with explicit source → sink traces.

### Day 7

Rework:

```text
Q5–Q7
```

with authorization and validation boundaries.

### Day 14

Rework:

```text
Q8–Q10
```

as incident-response exercises.

### Day 21

Threat-model one real application.

### Day 30

Perform a complete security review without notes.

---

# 32. Dependency Graph

```text
Chapters 01–20
        ↓
language + objects + prototypes + functions
        ↓
Chapters 29–40
        ↓
errors + async + browser/Node runtime
        ↓
Chapters 49–70
        ↓
browser APIs + networking + security + Node/tooling
        ↓
Chapters 71–101
        ↓
algorithms + production architecture + reliability
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
Chapter 115 — Async / Event Loop Assessment
        ↓
Chapter 116 — Memory Assessment
        ↓
Chapter 117 — Performance Assessment
        ↓
Chapter 118 — Security Assessment
        ↓
Chapter 119 — Architecture Assessment
```

---

# 33. Concept Connections

## Depends On

```text
objects
prototypes
errors
browser APIs
Fetch/HTTP
Node.js
modules
dependencies
authentication concepts
authorization concepts
observability
```

## Builds Toward

```text
architecture assessment
system design
secure platform engineering
principal-level risk management
```

## Concepts Revisited

```text
prototype chain
property keys
JSON
Fetch
browser storage
Node child processes
dependencies
logging
caching
API architecture
```

## Why This Chapter Matters

Security is not a single library or framework feature.

It is the discipline of controlling:

```text
trust
authority
interpretation
identity
access
data flow
failure impact
```

---

# 34. Track A — Core Theory

Master:

```text
trust boundaries
threat modeling
XSS
CSRF
CORS boundaries
prototype pollution
injection
authentication
authorization
least privilege
secrets
supply chain
secure data handling
```

Deliverable:

```text
identify and explain attack paths
```

---

# 35. Track B — Implementation

Build:

```text
schema validation boundary
safe HTML rendering path
authorization middleware
safe object merge
command execution wrapper
secret handling abstraction
security regression suite
audit logging
```

Every implementation must include:

```text
attacker-controlled input
expected invariant
safe behavior
failure behavior
regression test
```

---

# 36. Track C — Interview / Reasoning

Practice:

```text
"Where is the trust boundary?"

"What can the attacker control?"

"What is the dangerous sink?"

"Is this authentication or authorization?"

"What is the blast radius?"

"How would you prove the vulnerability is fixed?"

"What defense-in-depth control would you add?"
```

Deliverable:

```text
reason from attacker capability to system impact
```

---

# 37. Security Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] all 10 questions attempted
[ ] source and sink identified
[ ] trust boundaries documented
[ ] exploit paths explained
[ ] mitigations proposed
[ ] regression tests proposed
[ ] incident priorities justified
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] identify trust boundaries
[ ] distinguish authentication and authorization
[ ] recognize common injection paths
[ ] reason about XSS
[ ] reason about prototype pollution
[ ] protect command boundaries
[ ] protect secrets
[ ] evaluate dependencies as a supply-chain risk
[ ] threat-model a feature
[ ] prioritize security remediation at principal level
```

---

# 38. Assessment Completion Snapshot

```md
# Chapter 118 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Score:
____ / 100

Primary Gaps:
-

Trust Boundaries:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Injection:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Authentication / Authorization:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Browser Security:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Node.js Security:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Supply Chain:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Threat Modeling:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 39. Completion Criteria

```text
[ ] 10 questions completed
[ ] 100 points scored
[ ] attacker capability is identified
[ ] trust boundaries are explicit
[ ] source → sink paths are understood
[ ] validation is distinguished from authorization
[ ] authentication is distinguished from authorization
[ ] data/code boundaries are protected
[ ] prototype and inheritance risks are understood
[ ] browser and Node security are distinguished
[ ] secrets and dependencies are treated as security boundaries
[ ] regression tests are designed
[ ] threat models are produced
[ ] remediation is prioritized by risk and blast radius
```

---

# 40. Canonical Security Mental Model

Use:

```text
attacker
→ controlled input
→ trust boundary
→ interpretation / privilege
→ dangerous capability
→ violated invariant
→ impact
```

Then design:

```text
validate
→ authorize
→ constrain
→ execute with least privilege
→ observe
→ detect
→ recover
```

Always ask:

```text
What can the attacker control?

What can they cause the system to interpret?

What authority exists at that point?

What limits the blast radius?

How will we know an attack happened?
```

---

# 41. Final Principal Principle

> **Security is the engineering discipline of ensuring that untrusted inputs never acquire unintended authority.**

The mature model is:

```text
trust boundaries
+
validation
+
authorization
+
least privilege
+
safe interpretation
+
isolation
+
observability
+
regression protection
```

A secure JavaScript system does not merely avoid obvious dangerous APIs.

It deliberately controls:

```text
who can supply data
what that data can influence
what the system is allowed to interpret
what authority exists after compromise
how failures are contained
how abuse is detected
```