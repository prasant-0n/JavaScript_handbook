# Chapter 123 — ECMAScript Grammar, Parsing & Early Errors

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Understand how JavaScript source text becomes a valid ECMAScript program, how grammar rules determine syntax, how contextual grammar works, how static semantics produce early errors, and why parsing is a distinct phase from runtime execution.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Compiler/Parser Engineer · Runtime Engineer · Tooling Engineer · Language Design Reviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A JavaScript program does not begin executing merely because its text exists; source text must first satisfy the language's syntactic and static-semantic requirements.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain lexical grammar vs syntactic grammar
[ ] explain lexical goals
[ ] distinguish tokens from source text
[ ] explain whitespace, line terminators, comments, and lexical boundaries
[ ] explain identifiers, keywords, punctuators, literals, and tokenization
[ ] explain syntactic grammar and parse structure
[ ] explain grammar parameters and contextual parsing
[ ] explain automatic semicolon insertion at grammar level
[ ] explain why "/" can participate in very different parses
[ ] explain cover grammars at a conceptual and practical level
[ ] explain static semantics
[ ] distinguish syntax errors from early errors
[ ] distinguish parse-time failure from runtime failure
[ ] reason about declaration restrictions
[ ] reason about duplicate bindings and strict-mode restrictions
[ ] understand lexical goal changes such as RegExp vs division
[ ] understand why tooling must model ECMAScript grammar correctly
[ ] debug syntax/parser failures systematically
[ ] relate grammar rules to specification algorithms
[ ] explain where host parsing differs from ECMAScript parsing
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 01 — JavaScript, ECMAScript, Runtime Landscape
Chapter 05 — Variables, Declarations, Assignment
Chapter 06 — Operators, Expressions
Chapter 07 — Type Conversion, Coercion, Equality
Chapter 09 — Functions / First-Class Behavior
Chapter 10 — Scope / Lexical Environments / Identifier Resolution
Chapter 11 — Hoisting / TDZ
Chapter 12 — Execution Contexts / Execution Model
Chapter 18 — Classes / OOP
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 68 — Transpilation / Compilation
Chapter 69 — Bundlers / Build Systems
Chapter 90 — Modern ECMAScript Features
Chapter 91 — TC39 Proposal Tracking
```

Useful supporting knowledge:

```text
tokens
ASTs
parsers
compilers
source maps
module systems
strict mode
```

---

# 3. What Is ECMAScript Grammar?

ECMAScript grammar defines which sequences of source characters can form valid language constructs.

At a high level:

```text
source text
    ↓
lexical interpretation
    ↓
tokens / lexical structures
    ↓
syntactic parsing
    ↓
parse structure
    ↓
static semantics / early-error checks
    ↓
runtime evaluation
```

This is a conceptual model.

A specific engine may combine phases internally for performance.

Do not confuse:

```text
specification phases
```

with:

```text
literal implementation phases inside a particular engine.
```

---

# 4. Why Does Grammar Exist?

Grammar provides a formal language for expressing:

```text
what source programs look like
```

Without grammar, implementations could disagree about:

```text
what is valid
what a construct means
where one token ends
where another begins
```

Grammar also enables:

```text
parsers
linters
formatters
syntax highlighters
IDEs
transpilers
bundlers
static analyzers
```

A language implementation needs an unambiguous way to determine structure.

---

# 5. Mental Model

Use:

```text
Characters
    ↓
Lexical Grammar
    ↓
Tokens / lexical structures
    ↓
Syntactic Grammar
    ↓
Parse structure
    ↓
Static Semantics
    ↓
Early-error checks
    ↓
Runtime semantics
```

Another useful model:

```text
"What characters are allowed?"
        ↓
"What tokens can those characters form?"
        ↓
"What syntactic structure can those tokens form?"
        ↓
"Is that structure statically valid?"
        ↓
"How does that valid structure execute?"
```

---

# 6. Source Text Is Not Yet JavaScript Execution

Consider:

```js
const value = 1 + 2;
```

Before evaluating:

```text
1 + 2
```

the implementation must determine that the source text can be interpreted as a valid declaration containing an initializer expression.

Compare:

```js
const value = ;
```

The second program fails before normal execution of the declaration.

That distinction matters:

```text
source invalid
≠
runtime exception
```

---

# 7. Core Grammar Layers

ECMAScript specification grammar is commonly discussed through several cooperating layers.

Conceptually:

```text
lexical grammar
syntactic grammar
numeric/string/template lexical details
static semantics
runtime semantics
```

The important distinction is:

### Lexical grammar

Determines how source characters form lexical elements.

### Syntactic grammar

Determines how lexical elements form larger language structures.

### Static semantics

Defines checks or derived information that can be determined from source structure without normal runtime evaluation.

### Runtime semantics

Defines what valid constructs do when executed.

---

# 8. Lexical Grammar

Lexical grammar answers questions such as:

```text
Where does an identifier end?

Is this character part of a numeric literal?

Is this token a keyword?

Is this slash beginning a comment?

Is this slash part of a regular expression literal?

Is this sequence a punctuator?
```

Examples:

```js
const total = 10_000;
```

contains lexical structures for:

```text
const
total
=
10_000
;
```

The exact implementation representation may differ.

---

# 9. Tokens

Think of tokens as meaningful lexical units.

Examples include:

```text
identifiers
keywords
numeric literals
string literals
template literals
punctuators
private identifiers
regular expression literals
```

Examples:

```js
user
const
123
"hello"
`
=> 
?. 
```

Not every internal parser necessarily exposes a token stream in exactly this form.

The token model is a reasoning tool and a specification-level abstraction.

---

# 10. Whitespace

Whitespace often separates lexical elements without becoming a meaningful runtime operation.

Example:

```js
const x = 1;
```

and:

```js
const     x     =     1;
```

can have equivalent syntactic structure.

But whitespace can matter when it changes token boundaries.

Example:

```js
a + +b
```

is not lexically the same as:

```js
a ++b
```

The first contains separate `+` punctuators.

The second may be interpreted differently based on grammar context.

The lesson:

```text
"whitespace does nothing"
```

is too simplistic.

---

# 11. Comments

JavaScript supports comments that participate in lexical processing rather than runtime computation.

Examples:

```js
// line comment

/*
  block comment
*/
```

Comments can still matter indirectly because line terminators around comments can affect parsing.

Example:

```js
return
{
  value: 1
};
```

is affected by the line terminator between:

```text
return
```

and:

```text
{
```

Do not analyze comments only as “ignored text.”

Analyze whether they contain or separate syntactically meaningful boundaries.

---

# 12. Line Terminators

Line terminators can influence:

```text
Automatic Semicolon Insertion
restricted productions
```

Important conceptual distinction:

```text
all whitespace
```

is not equivalent to:

```text
line terminator
```

Examples of line terminator characters include:

```text
LF
CR
CRLF
Unicode line separator
Unicode paragraph separator
```

The specification treats line terminators specially in several places.

---

# 13. Identifiers

Identifiers name things such as:

```js
user
total
processOrder
```

Modern ECMAScript identifiers support a broad range of Unicode characters under specified rules.

Examples:

```js
const café = 1;
const π = 3.14;
```

Identifier syntax is not equivalent to:

```text
"any Unicode character"
```

The grammar places restrictions on:

```text
identifier start
identifier continuation
escape forms
reserved words
context
```

---

# 14. Reserved Words vs Identifiers

These are not always interchangeable:

```text
identifier
keyword
reserved word
contextually restricted name
```

Examples include language words associated with:

```text
if
for
class
return
yield
await
```

Whether a spelling can be used in a particular position can depend on:

```text
grammar context
strictness
module/script goal
```

Do not memorize a single flat “reserved words list” and assume it explains every case.

---

# 15. Private Identifiers

Class private fields use a distinct lexical form:

```js
class User {
  #name = "A";

  getName() {
    return this.#name;
  }
}
```

The:

```text
#name
```

token is not equivalent to:

```js
"name"
```

or:

```js
name
```

Private identifiers participate in grammar and static semantics.

This is a useful example of:

```text
lexical form
+
grammar context
+
static validation
```

---

# 16. Punctuators

JavaScript uses punctuation heavily:

```text
.
?.
...
===
=>
**
??
&&
||
++
--
```

Lexical recognition matters because several punctuators share prefixes.

For example:

```text
=
==
===
=>
```

A language implementation must tokenize these according to its lexical rules and context.

Do not assume tokenization is equivalent to:

```text
take the shortest possible token
```

or:

```text
take the longest possible token
```

without considering the grammar model.

---

# 17. Numeric Literals

JavaScript supports multiple numeric literal forms.

Examples:

```js
42
0b1010
0o755
0xFF
1_000_000
1.5
1e3
```

Numeric literal grammar determines which spellings are legal.

Compare:

```js
1_000
```

with malformed forms that violate numeric-literal grammar.

A syntax failure here is fundamentally different from:

```js
Number("not-a-number")
```

because the latter is runtime conversion.

---

# 18. String Literal Grammar

String literals have lexical constraints.

Examples:

```js
"hello"
'hello'
"line\nbreak"
```

Escape processing is defined separately from merely saying:

```text
"a string"
```

The parser must understand:

```text
quote type
escape sequence
line terminator restrictions
unicode escapes
termination
```

An unterminated literal is a source-level failure.

---

# 19. Template Literal Grammar

Template literals are especially important because their lexical and syntactic behavior interacts.

Example:

```js
const message = `Hello ${name}`;
```

There is:

```text
template text
expression interpolation
closing template delimiter
```

Template parsing can alternate between:

```text
template characters
```

and:

```text
JavaScript expression grammar
```

This is one reason template literals cannot be modeled as ordinary strings with a few extra characters.

---

# 20. Regular Expression Literal Ambiguity

Consider:

```js
a / b / c
```

versus:

```js
/foo+/g
```

The slash character can participate in different constructs.

Conceptually:

```text
division operator
```

or:

```text
regular expression literal
```

This is a classic example of lexical-goal sensitivity.

A parser cannot always classify `/` in isolation.

It must use syntactic context to determine what lexical interpretation is appropriate.

---

# 21. Lexical Goals

A lexical goal is a specification mechanism that tells the grammar which lexical interpretation should be used in a context.

This helps resolve cases involving constructs such as:

```text
RegularExpressionLiteral
Template
division
```

The important insight is:

```text
lexing and parsing are not always cleanly separable as two context-free passes
```

The specification uses grammar machinery to express the necessary contextual behavior.

---

# 22. Automatic Semicolon Insertion

JavaScript has rules for semicolon insertion.

Do not model ASI as:

```text
"JavaScript automatically inserts semicolons at the end of lines."
```

That is inaccurate.

A better model is:

```text
When token sequences cannot continue to form a permitted structure,
certain grammar rules allow a semicolon-like boundary to be inserted
under specified conditions.
```

This is why:

```js
return
{
  value: 1
}
```

does not mean:

```js
return { value: 1 };
```

---

# 23. Restricted Productions

Some grammar productions are sensitive to line terminators.

Examples commonly encountered include constructs involving:

```text
return
throw
break
continue
yield
async function/certain async forms
```

The exact rule depends on the production.

When debugging a newline-sensitive parse issue, ask:

```text
Is this a restricted production?
Is a line terminator forbidden here?
Can ASI apply?
```

---

# 24. Syntax Error vs Early Error

These are related but conceptually distinct.

### Syntax failure

The source structure cannot be parsed as the required grammar.

Example:

```js
if (
```

### Early error

A syntactically structured program violates a static restriction that must be rejected before runtime evaluation.

Examples can involve:

```text
duplicate lexical declarations
invalid private-name usage
certain strict-mode restrictions
invalid assignment targets
duplicate parameter restrictions in relevant contexts
```

The exact mechanism depends on the grammar/static-semantics rules involved.

---

# 25. Why Early Errors Exist

Early errors prevent programs from entering runtime states that the language defines as invalid.

Benefits include:

```text
deterministic language rules
clear developer feedback
simpler runtime semantics
static validation
tooling support
security boundaries
```

The important distinction is:

```text
"Can the source be structurally parsed?"
```

versus:

```text
"Is this structurally valid according to all required static rules?"
```

---

# 26. Example — Duplicate Lexical Binding

```js
{
  let value = 1;
  let value = 2;
}
```

This is not merely:

```text
"value gets overwritten"
```

The source violates the declaration rules for lexical bindings.

No runtime assignment race occurs.

The program is rejected before normal evaluation of that block.

---

# 27. Example — Conflicting Declarations

Consider:

```js
var x;
let x;
```

The interaction between declaration forms is constrained by the grammar and static semantics.

You should reason in terms of:

```text
declaration instantiation
binding rules
static restrictions
```

rather than assuming that all declarations are interchangeable storage operations.

---

# 28. Example — Invalid Assignment Target

This:

```js
1 = 2;
```

is not:

```text
runtime assignment to an immutable value
```

It fails because the left-hand side does not satisfy the required syntactic/semantic conditions for assignment.

Compare with:

```js
const x = 1;
x = 2;
```

which reaches a different failure mechanism involving a binding whose assignment is not permitted.

These examples are useful because similar-looking source failures can arise at different semantic layers.

---

# 29. Example — `const` Is Not “Immutable Value”

Consider:

```js
const user = {
  name: "A"
};

user.name = "B";
```

The declaration grammar permits the binding.

Runtime semantics then allow mutation of the referenced object.

Compare:

```js
const user = {};
user = {};
```

The second operation attempts to update the binding itself.

Grammar and runtime binding semantics must be kept separate.

---

# 30. Static Semantics

Static semantics are specification rules that compute properties or validate restrictions based on program structure.

Examples conceptually include:

```text
Is this a valid assignment target?
Does this declaration conflict with another declaration?
Does this class reference a valid private name?
Does this construct contain a forbidden form?
What lexical names are declared here?
```

Static semantics are not necessarily runtime algorithms.

They often appear in specification sections associated with grammar productions.

---

# 31. Parse Trees vs ASTs

A specification grammar can describe a richer syntactic structure than the simplified AST exposed by tooling.

Think:

```text
grammar derivation
        ↓
syntactic structure
        ↓
implementation parse tree / AST representation
```

Tools frequently normalize or compress grammar details.

Therefore:

```text
AST node names
```

are not the same thing as:

```text
ECMAScript grammar productions
```

A Babel/Acorn/Espree node model should not be treated as the ECMAScript specification itself.

---

# 32. Cover Grammars

Some JavaScript syntax is difficult to express using a simple one-pass grammar because a sequence of tokens may initially be compatible with more than one interpretation.

ECMAScript uses grammar techniques often described as:

```text
cover grammars
```

The core idea is:

```text
parse a broad form first
→ later determine whether that structure is valid in the required context
```

This can support syntax such as ambiguous-looking expression/arrow-function forms.

The important lesson:

```text
a successful intermediate parse does not always mean
the final program is statically valid.
```

---

# 33. Why Cover Grammars Matter

Without contextual grammar techniques, language syntax could become unnecessarily complicated.

They allow the specification to express cases where:

```text
the same token sequence can participate in multiple syntactic interpretations
```

and later semantic checks determine:

```text
which interpretations remain valid
```

This is especially relevant when implementing:

```text
JavaScript parsers
linters
formatters
transpilers
codemods
IDEs
```

---

# 34. Arrow Functions and Contextual Parsing

Consider:

```js
(a) => a;
```

versus:

```js
(a);
```

The parenthesized token sequence can participate in different constructs.

A parser needs enough context to recognize:

```text
parenthesized expression
```

versus:

```text
arrow-function parameter list
```

This is a practical example of why JavaScript grammar is richer than:

```text
"read left to right and tokenize independently."
```

---

# 35. Destructuring Context

Consider:

```js
const { a } = value;
```

and:

```js
({ a });
```

The same punctuation participates in different grammatical contexts.

Context influences whether braces represent:

```text
block
object literal
destructuring pattern
class body
```

This is one reason:

```text
"{} means object"
```

is not a valid complete mental model.

---

# 36. Statement vs Expression Ambiguity

JavaScript has many contexts in which the same punctuation can represent different structures.

Examples:

```text
{
  ...
}
```

could be:

```text
block statement
object literal in expression position
destructuring pattern in an appropriate binding context
```

Likewise:

```js
function f() {}
```

and:

```js
({
  f() {}
})
```

contain similar punctuation with very different grammatical roles.

Always identify the surrounding production.

---

# 37. Scripts vs Modules as Parse Goals

JavaScript source can be interpreted under different top-level goals.

The most important common distinction is:

```text
Script
Module
```

Module source supports constructs such as:

```js
import ...
export ...
```

that are not ordinary Script syntax.

This is why:

```text
"the file contains valid JavaScript"
```

is incomplete.

You also need to know:

```text
What parse goal is being used?
```

---

# 38. Module Syntax and Static Structure

Module syntax has stronger static structure than many ordinary script constructs.

For example:

```js
import { x } from "./x.js";
```

is not an ordinary runtime function call.

The language defines module structure through grammar and static semantics.

This is essential for tooling because:

```text
dependency discovery
```

can be performed from module syntax.

---

# 39. Strict Mode and Grammar

Strict mode affects more than runtime behavior.

It also changes which source forms are permitted.

Examples include restrictions around:

```text
certain identifier names
duplicate parameters in relevant forms
legacy syntax
```

Therefore:

```text
strict mode
```

is partly a parsing/static-validity concern and partly a runtime semantic concern.

---

# 40. `yield` and `await` as Context-Sensitive Keywords

Words such as:

```text
yield
await
```

cannot be modeled purely as:

```text
always keyword
```

or:

```text
always identifier
```

Their interpretation depends on grammatical context such as:

```text
generator
async function
module
other language context
```

This is another example of contextual grammar.

---

# 41. `async` and Newline Sensitivity

Consider:

```js
async function f() {}
```

versus:

```js
async
function f() {}
```

The newline can affect how the source is parsed.

This demonstrates:

```text
identifier-like token
+
line terminator
+
grammar context
```

can materially change the structure.

Never teach newline behavior as merely stylistic.

---

# 42. Private Name Static Validation

Consider:

```js
class User {
  method() {
    return this.#name;
  }
}
```

but no `#name` declaration exists.

The issue is not:

```text
"property doesn't exist at runtime."
```

Private-name usage is subject to static class semantics.

This is intentionally stronger than ordinary property access:

```js
this.name
```

The language can reject the private-name reference before normal execution.

---

# 43. Grammar Parameters

ECMAScript grammar sometimes uses parameters to express contextual restrictions.

Conceptually:

```text
Production[Parameter]
```

lets the grammar say:

```text
this form is permitted under one context
but restricted under another
```

Examples of concepts that can influence grammar include:

```text
Yield
Await
In
Return
```

The exact parameter names and productions are specification details.

The mental model is:

```text
same local syntax
+
different surrounding grammar state
=
different validity/interpretation
```

---

# 44. Grammar Goal Symbols

A parser needs an appropriate starting production.

Conceptually:

```text
Script
Module
FunctionBody
StatementList
Expression
```

A goal symbol determines the syntactic universe being parsed.

For tooling:

```text
parse as script
```

and:

```text
parse as module
```

can produce different validity outcomes for the same text.

---

# 45. Example — Module-Only Syntax

```js
export const value = 1;
```

As module source:

```text
valid
```

As ordinary script source:

```text
invalid
```

The source characters did not change.

The parse goal did.

---

# 46. Example — Top-Level `await`

Top-level `await` is another example where context matters.

The ability to write:

```js
const result = await fetch(url);
```

depends on the surrounding parsing/evaluation context.

Do not reduce this to:

```text
"await is only allowed inside async functions."
```

That rule is incomplete for modern ECMAScript modules.

---

# 47. Early Errors and Tooling

Editors often highlight errors before execution.

Examples:

```text
duplicate binding
invalid syntax
unknown private name
invalid declaration
```

Tooling must implement or approximate specification static semantics.

Differences between tools can arise because:

```text
parser version
language mode
proposal support
configuration
```

may differ.

---

# 48. Parser Version Mismatch

Suppose a project uses syntax introduced by a newer ECMAScript edition.

One parser may accept:

```js
newerSyntax
```

while another parser rejects it.

This can produce:

```text
IDE error
but runtime works
```

or:

```text
build succeeds
but another tool fails
```

The debugging process should identify:

```text
which parser
which ECMAScript grammar version
which language mode
which tool configuration
```

---

# 49. Proposal Syntax

When syntax comes from a proposal, distinguish:

```text
current ECMAScript standard
```

from:

```text
proposal/stage syntax
```

and:

```text
tool-specific extension
```

A parser accepting syntax does not prove that:

```text
all JavaScript runtimes accept it.
```

---

# 50. Host Parsing vs ECMAScript Parsing

Browsers and Node.js parse more than just ECMAScript.

Examples:

```text
HTML
CSS
JSON/resource formats
shell/configuration files
```

Similarly, build tools may parse:

```text
TypeScript
JSX
decorator syntax variants
custom macros
```

A syntax failure may originate in:

```text
ECMAScript parser
TypeScript parser
JSX parser
bundler parser
HTML parser
```

Identify the layer.

---

# 51. Parse Error Debugging Workflow

When source fails to parse:

```text
1. Capture exact source text.
2. Identify parser/tool.
3. Identify language mode.
4. Identify parse goal.
5. Identify ECMAScript/tooling version.
6. Reduce to smallest failing input.
7. Locate first unexpected token.
8. Inspect surrounding grammar context.
9. Check line terminators.
10. Check contextual keywords.
11. Check proposal/tooling support.
12. Verify with an independent parser/runtime.
```

Do not start by changing unrelated runtime code.

---

# 52. First Unexpected Token Is Not Always Root Cause

Suppose the parser points at:

```text
}
```

The real mistake may be several lines earlier:

```js
const value = {
  name: "A"
  active: true
};
```

The parser only reports where the grammar finally becomes impossible.

Therefore:

```text
error location
```

is not always:

```text
cause location
```

This principle applies broadly to parser diagnostics.

---

# 53. Prediction Exercise

Predict whether each is:

```text
valid source
syntax failure
early error
runtime failure
```

### A

```js
const x = ;
```

### B

```js
{
  let x;
  let x;
}
```

### C

```js
const x = {};
x.value = 1;
```

### D

```js
const x = 1;
x = 2;
```

### E

```js
class A {
  test() {
    return this.#missing;
  }
}
```

Do not answer by memorization.

Classify the failure layer.

---

# 54. Prediction Exercise — Parse Goal

Given:

```js
export const x = 1;
```

Predict:

```text
Script parse?
Module parse?
Runtime execution?
```

Then explain which answer depends on the selected top-level goal.

---

# 55. Prediction Exercise — Newline

Predict the structural difference between:

```js
async function load() {
  return await value;
}
```

and:

```js
async
function load() {
  return await value;
}
```

Your reasoning must reference:

```text
line terminator
grammar context
```

rather than saying simply:

```text
"JavaScript doesn't like newlines there."
```

---

# 56. Prediction Exercise — Slash

Determine whether the following slash is more plausibly part of:

```text
division
```

or:

```text
RegExpLiteral
```

and explain the contextual reasoning.

```js
const a = 10 / 2;
```

```js
if (/^user/.test(name)) {
  console.log(name);
}
```

Then create one example where inserting/removing parentheses changes the parser's interpretation.

---

# 57. Syntax vs Static Semantics Drill

Classify each statement:

```text
"The token sequence cannot form an expression."
```

```text
"The parse structure exists, but a required static restriction is violated."
```

```text
"The program parses and passes static checks, but fails during execution."
```

Map them to:

```text
syntax
early error
runtime failure
```

---

# 58. Implementation Exercise — Minimal Tokenizer

Implement a simplified tokenizer for a deliberately restricted JavaScript subset:

```text
identifiers
integers
+
-
*
/
=
;
(
)
{
}
```

Requirements:

```text
input string
→ token list
```

Handle:

```text
whitespace
unexpected characters
token boundaries
line/column tracking
```

Do not attempt to implement the full ECMAScript lexical grammar.

The purpose is to understand:

```text
characters
→ tokens
```

---

# 59. Implementation Exercise — Tiny Parser

Extend the tokenizer into a tiny parser for:

```text
variable declarations
arithmetic expressions
blocks
```

Support:

```js
let x = 1 + 2;
```

and:

```js
{
  let x = 1;
  let y = x + 2;
}
```

Produce a simplified AST.

Required stages:

```text
source
→ tokens
→ parser
→ AST
```

---

# 60. Implementation Exercise — Static Checker

Add static checks to the tiny parser.

At minimum detect:

```text
duplicate declarations in the same local scope
unknown private-like names if you model them
assignment to unsupported targets
```

Produce diagnostics:

```text
message
line
column
category
```

This demonstrates:

```text
parse structure
+
static validation
```

---

# 61. Implementation Progression

Use:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

### Guided

Implement tokenization from provided interfaces.

### Partially Guided

Design parser precedence yourself.

### No Reference

Build the restricted language parser from a specification.

### Edge-Case Hardened

Add:

```text
line/column tracking
malformed input
unterminated structures
ambiguous operators
```

### Production-Grade

Add:

```text
diagnostic codes
error recovery
incremental parsing discussion
benchmark
fuzzing
test corpus
```

---

# 62. Parser Error Recovery

Production tooling often needs more than:

```text
parse or fail
```

Editors may need to continue operating while code is temporarily invalid.

Examples:

```text
missing )
missing }
unfinished string
half-written expression
```

Error-recovery strategies may include:

```text
synchronization tokens
panic recovery
error nodes
partial parse trees
```

This is especially important for:

```text
IDEs
formatters
language servers
interactive tooling
```

---

# 63. Incremental Parsing

Interactive tools may reparse source repeatedly as users type.

A useful architecture can avoid reparsing everything when only a small region changed.

Conceptually:

```text
old source
+
small edit
↓
reuse unaffected structure
↓
reparse affected region
```

This introduces trade-offs involving:

```text
complexity
memory
correctness
incremental state
```

Do not implement incremental parsing merely because it sounds faster.

Measure the workload.

---

# 64. Parser Performance

Parser performance can depend on:

```text
source size
token count
grammar complexity
error recovery
AST creation
allocation rate
incremental reuse
```

For tooling, important metrics include:

```text
initial parse latency
incremental parse latency
memory per file
throughput
diagnostic latency
```

---

# 65. Parser Security

Parsers process potentially untrusted source.

Security considerations include:

```text
resource exhaustion
pathological input
deep nesting
huge literals
unexpected Unicode
invalid UTF-16 sequences at boundaries
```

For developer tools, also consider:

```text
malicious repository contents
untrusted generated code
editor-triggered analysis
```

Use resource limits where appropriate.

---

# 66. Unicode and Parsing

JavaScript source uses Unicode-aware identifier rules and a UTF-16-oriented string model.

Tooling must carefully distinguish:

```text
byte offset
Unicode code point
UTF-16 code unit
grapheme cluster
```

An editor displaying:

```text
one visible character
```

does not guarantee:

```text
one code unit
```

This becomes important for diagnostics:

```text
line
column
offset
highlight range
```

---

# 67. Source Positions

A parser commonly tracks:

```text
start offset
end offset
line
column
```

Different tools may define columns using:

```text
code units
code points
display columns
```

Do not assume all diagnostics use the same coordinate system.

For production tooling, document the position model.

---

# 68. AST Fidelity

Different tools may represent equivalent syntax differently.

Examples:

```text
ParenthesizedExpression
```

may:

```text
exist explicitly
```

or:

```text
be represented through source ranges/metadata
```

Likewise, parser libraries may differ in:

```text
node names
field names
token retention
comments
literal representation
experimental syntax
```

Treat AST format as a tooling contract, not a universal ECMAScript standard.

---

# 69. Parser Testing

A parser should test:

```text
valid syntax
invalid syntax
boundary cases
Unicode
line terminators
comments
nested structures
ambiguous constructs
contextual keywords
modules/scripts
strict mode
new language features
```

Also test:

```text
error position
diagnostic message/category
recovery behavior
AST output
```

---

# 70. Differential Testing

A powerful parser-testing technique is comparing independent implementations.

Conceptually:

```text
input
→ parser A
→ parser B
→ compare acceptance/structure
```

Differences can reveal:

```text
implementation bugs
version mismatch
proposal support mismatch
grammar misunderstandings
```

Be careful when the parsers intentionally support different syntax sets.

---

# 71. Grammar Conformance Testing

For language tooling, build a corpus containing:

```text
known-valid programs
known-invalid programs
edge cases
regression cases
```

Track:

```text
accepted/rejected
AST shape
diagnostic category
version support
```

This turns grammar support into a testable engineering artifact.

---

# 72. Tooling Architecture

A production parser-based tool may look like:

```text
source
  ↓
encoding / source loader
  ↓
lexer/tokenizer
  ↓
parser
  ↓
AST / CST
  ↓
static semantics
  ↓
analysis
  ↓
diagnostics / transform / code generation
```

Different tools may combine or reorder internal stages.

The architectural principle remains:

```text
make assumptions explicit
```

---

# 73. ECMAScript vs TypeScript vs JSX

These are not interchangeable grammars.

TypeScript adds syntax such as:

```text
type annotations
interfaces
type parameters
```

JSX adds syntax such as:

```text
<Component />
```

A toolchain may therefore perform:

```text
extended parsing
→ transform
→ ECMAScript output
```

Do not conclude:

```text
"JavaScript parser rejected it, so the code is invalid."
```

It may be valid TypeScript/JSX source but invalid plain ECMAScript source.

---

# 74. Transpiler Boundary

A transpiler may accept:

```text
source language
```

that differs from the target runtime language.

Therefore distinguish:

```text
source grammar
target grammar
transformation
runtime grammar
```

For example:

```text
TypeScript
→ TypeScript parser
→ transform
→ JavaScript
→ ECMAScript runtime
```

The runtime never executes TypeScript type syntax as JavaScript.

---

# 75. Build-System Debugging

When a project reports:

```text
Unexpected token
```

ask:

```text
Which stage emitted it?

Which file did it parse?

Which language mode?

Which parser?

Which transformation happened first?

Was the source transformed before another tool saw it?
```

This prevents debugging the wrong layer.

---

# 76. Common Misconceptions

### Misconception 1

> “The parser executes the code.”

Correction:

```text
Parsing determines structure.
Execution evaluates valid program semantics.
```

### Misconception 2

> “Every newline means a semicolon.”

Correction:

```text
ASI follows specified grammar conditions.
```

### Misconception 3

> “SyntaxError means only malformed punctuation.”

Correction:

```text
Source-level rejection can involve grammar/static semantic restrictions.
```

### Misconception 4

> “AST = ECMAScript specification.”

Correction:

```text
ASTs are implementation/tool representations.
```

### Misconception 5

> “A parser accepting syntax proves browser/Node support.”

Correction:

```text
Parser support and runtime support are separate compatibility questions.
```

---

# 77. Common Mistakes

```text
[ ] treating whitespace and line terminators as identical
[ ] assuming slash always means division
[ ] treating ASI as line-based text rewriting
[ ] treating all keywords as always reserved
[ ] ignoring parse goal
[ ] ignoring strict/module context
[ ] assuming parser error location is exact root cause
[ ] treating early errors as runtime exceptions
[ ] assuming tooling ASTs are standardized identically
[ ] mixing TypeScript/JSX grammar with ECMAScript grammar
[ ] ignoring parser version
[ ] ignoring contextual grammar
```

---

# 78. Comparison With Related Concepts

| Concept | Main Question |
|---|---|
| Lexical grammar | How is source text split/interpreted lexically? |
| Syntactic grammar | How do lexical structures form language constructs? |
| Static semantics | Which structural restrictions/properties can be determined before runtime? |
| Runtime semantics | What does valid code do when evaluated? |
| AST | How does a tool represent parsed structure? |
| Compiler/transpiler | How is source transformed or compiled? |
| Type checker | Which type constraints hold? |
| Linter | Which policy/style/semantic patterns should be reported? |

---

# 79. Performance Considerations

Parser performance can be affected by:

```text
source size
token density
AST allocation
comment handling
error recovery
deep nesting
Unicode processing
incremental state
```

Measure:

```text
parse latency
memory
throughput
incremental latency
diagnostic latency
```

Avoid broad claims such as:

```text
"parser X is faster."
```

without controlling:

```text
input corpus
features enabled
AST options
hardware
runtime
version
```

---

# 80. Memory Considerations

Parsing can allocate:

```text
tokens
AST nodes
source ranges
comments
diagnostics
symbol tables
scope structures
```

For large repositories, memory can be dominated by:

```text
AST retention
multiple source copies
parser caches
cross-file indexes
```

Design for:

```text
bounded caches
disposable parse state
incremental reuse
file-level isolation
```

where justified.

---

# 81. Security Considerations

Security-sensitive parser systems must consider:

```text
resource exhaustion
deeply nested input
huge source files
malformed Unicode
ambiguous parser behavior
prototype pollution in AST consumers
unsafe code generation
untrusted transformations
```

A parser itself does not make downstream generated code safe.

The full flow matters:

```text
source
→ parse
→ transform
→ generate
→ execute
```

---

# 82. Production Usage

Grammar knowledge is useful when building:

```text
linters
formatters
codemods
transpilers
bundlers
IDEs
language servers
static analyzers
syntax highlighters
code search
dependency analyzers
security scanners
```

A principal engineer should know where to place checks:

```text
parser
static analysis
type system
build system
runtime
```

and why.

---

# 83. Implementation From Scratch — Milestone Plan

### Milestone 1

Tokenizer.

### Milestone 2

Expression parser.

### Milestone 3

Statements/declarations.

### Milestone 4

AST.

### Milestone 5

Static scope checker.

### Milestone 6

Diagnostics.

### Milestone 7

Error recovery.

### Milestone 8

Fuzz corpus.

### Milestone 9

Benchmark.

### Milestone 10

Differential comparison against a mature parser.

---

# 84. Debugging Exercises

## Exercise A — Unexpected Token

Given a parser error:

```text
Unexpected token '}'
```

Find the actual root cause in:

```js
const config = {
  host: "localhost",
  port: 3000
  secure: true
};
```

Explain:

```text
reported location
actual omission
parser state
```

---

## Exercise B — Module vs Script

A tool rejects:

```js
import fs from "node:fs";
```

but another tool accepts it.

Diagnose:

```text
parse goal
tool configuration
file classification
parser/version
```

---

## Exercise C — Async Newline

A formatter changes:

```js
async function f() {}
```

into:

```js
async
function f() {}
```

Explain why formatting JavaScript by arbitrary whitespace insertion can change semantics.

---

# 85. Code Review Exercise

Review:

```js
function isValidJavaScript(source) {
  try {
    new Function(source);
    return true;
  } catch {
    return false;
  }
}
```

Questions:

```text
What does this actually validate?

Which parse goal is used?

Does it validate modules?

Does it validate runtime behavior?

Does it prove security?

Does it fully model application/tooling syntax?

What failures can be hidden by this abstraction?
```

Then propose a more precise API contract.

---

# 86. Interview Questions

### Foundational

```text
1. What is lexical grammar?
2. What is syntactic grammar?
3. What is an early error?
4. What is the difference between syntax failure and runtime failure?
5. Why does JavaScript need contextual grammar?
```

### Advanced

```text
6. Why is "/" context-sensitive?
7. What is a lexical goal?
8. Why is ASI not simply newline insertion?
9. What is a cover grammar?
10. What is a grammar parameter?
11. How do Script and Module parse goals differ?
12. How do static semantics relate to grammar?
13. Why are ASTs not the same as the ECMAScript grammar?
14. Why can different parsers disagree?
15. How would you design parser error recovery?
```

### Principal

```text
16. How would you diagnose a parser-version mismatch across a large monorepo?
17. How would you safely support a new syntax proposal in tooling?
18. How would you design incremental parsing?
19. How would you test a parser against the ECMAScript specification?
20. How would you prevent parser-driven resource exhaustion?
```

---

# 87. Predict-the-Output / Behavior Exercises

For each, predict:

```text
valid
early error
runtime result
```

before checking.

### Exercise 1

```js
const x = ;
```

### Exercise 2

```js
{
  let a = 1;
  let a = 2;
}
```

### Exercise 3

```js
const x = {};
x.value = 1;
```

### Exercise 4

```js
const x = 1;
x = 2;
```

### Exercise 5

```js
class A {
  read() {
    return this.#x;
  }
}
```

### Exercise 6

```js
export const x = 1;
```

Classify it under:

```text
Script
Module
```

### Exercise 7

```js
return
value;
```

Explain the parse/evaluation implications only when this code appears inside an appropriate function context.

---

# 88. Mastery Exercises

### Exercise 1 — Grammar Map

Create a one-page diagram:

```text
source
→ lexical grammar
→ syntactic grammar
→ static semantics
→ runtime semantics
```

Annotate each stage with examples.

### Exercise 2 — Context Matrix

Create a table for:

```text
await
yield
return
in
/ 
{
}
```

and document how context changes their interpretation.

### Exercise 3 — Parser Differential

Take 100 syntax examples and compare two parsers.

Record:

```text
accepted/rejected
AST difference
version difference
reason
```

### Exercise 4 — Static Semantics

Write a checker for a restricted language subset that detects:

```text
duplicate lexical declarations
```

### Exercise 5 — Diagnostic Quality

Create 20 malformed snippets and produce:

```text
first unexpected token
root cause
best diagnostic
```

---

# 89. Track A — Core Theory

Master:

```text
lexical grammar
syntactic grammar
tokens
lexical goals
grammar parameters
contextual keywords
cover grammars
static semantics
early errors
parse goals
Script vs Module
ASI
restricted productions
AST boundaries
```

Deliverable:

```text
explain why a source form is valid/invalid without hand-waving
```

---

# 90. Track B — Implementation

Build:

```text
tokenizer
parser
AST
static checker
diagnostic system
error-recovery prototype
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

Deliverable:

```text
understand parsing by implementing a constrained language
```

---

# 91. Track C — Interview / Reasoning

Practice:

```text
"Why is this invalid?"

"Is this a syntax or runtime failure?"

"Why does a newline change this?"

"Why does this parser accept it but that parser reject it?"

"Which grammar goal is active?"

"Where does the language specification establish this restriction?"
```

Deliverable:

```text
precise language reasoning
```

---

# 92. Specification / Runtime Source Discipline

When answering grammar questions, prefer:

```text
1. ECMAScript specification grammar
2. ECMAScript static semantics
3. ECMAScript runtime semantics
4. parser/tool implementation
5. host behavior
```

Distinguish:

```text
standardized syntax
```

from:

```text
proposal syntax
```

and:

```text
tool extension
```

Do not use:

```text
"Chrome accepts it"
```

as proof of:

```text
ECMAScript grammar validity
```

---

# 93. Common Failure Modes

```text
Failure 1:
Treating parser implementation details as language rules.

Failure 2:
Treating AST representation as specification grammar.

Failure 3:
Treating ASI as newline substitution.

Failure 4:
Ignoring parse goal.

Failure 5:
Ignoring contextual grammar.

Failure 6:
Calling every source rejection a runtime error.

Failure 7:
Assuming parser acceptance means runtime support.

Failure 8:
Ignoring toolchain grammar extensions.

Failure 9:
Debugging the reported token instead of the preceding grammar violation.

Failure 10:
Using regex alone to parse general JavaScript.
```

---

# 94. Completion Criteria

```text
[ ] lexical grammar explained
[ ] syntactic grammar explained
[ ] static semantics explained
[ ] early errors distinguished from runtime errors
[ ] lexical goals understood
[ ] slash ambiguity understood
[ ] ASI understood precisely
[ ] restricted productions understood
[ ] Script vs Module parse goals understood
[ ] contextual keywords understood
[ ] cover grammars understood conceptually
[ ] grammar parameters understood conceptually
[ ] parser/tool/runtime boundaries understood
[ ] tokenizer implemented
[ ] parser implemented
[ ] static checker implemented
[ ] parser diagnostics analyzed
[ ] parser testing strategy created
```

---

# 95. Concept Connections

## Depends On

```text
Chapter 41 — Spec Architecture
Chapter 42 — Abstract Operations
Chapter 11 — Hoisting / TDZ
Chapter 18 — Classes
Chapter 64 — ES Modules
Chapter 68 — Transpilation
Chapter 69 — Bundlers
Chapter 90 — Modern ECMAScript
Chapter 91 — TC39 Proposal Tracking
```

## Builds Toward

```text
Chapter 124 — Execution Records, Completion Records & References
Chapter 125 — Promise Internals
Chapter 126 — Module Linking / Async Module Evaluation
Chapter 147 — Modern Package Resolution
Chapter 148 — JavaScript Packaging / Distribution
```

## Related Concepts

```text
ASTs
compiler design
type systems
linters
formatters
source maps
language servers
code generation
```

## Concepts Revisited

```text
strict mode
modules
private fields
async/await
classes
declarations
ASI
```

## Why This Chapter Matters Later

Many advanced JavaScript bugs are easier once you know exactly where they originate:

```text
lexical layer
→ grammar layer
→ static layer
→ runtime layer
→ host layer
```

This prevents the common debugging mistake of treating every failure as:

```text
"JavaScript runtime behavior."
```

---

# 96. Principal Decision Framework

When designing or evaluating JavaScript grammar/tooling behavior, consider:

```text
Correctness
Specification conformance
Compatibility
Parser performance
Memory use
Security
Developer Experience
Diagnostics
Tool interoperability
Maintainability
Future language evolution
Migration cost
```

A parser that is fast but wrong is not a successful parser.

A parser that is correct but unusably slow can also fail its product requirements.

---

# 97. Retrieval Record

```md
# Chapter 123 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Prediction Accuracy
- Validity classification:
- Syntax vs early error:
- Parse goal:
- ASI:
- Contextual grammar:

## Strongest Areas
-

## Weakest Areas
-

## Misconceptions Found
-

## Parser Implementation Progress
-

## Tooling Debugging Gaps
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 98. Spaced Retrieval Schedule

### Day 0

Complete:

```text
concept review
prediction exercises
tokenizer milestone
```

### Day 1

Redo:

```text
syntax vs early-error classification
```

without notes.

### Day 3

Explain from memory:

```text
lexical grammar
syntactic grammar
static semantics
```

### Day 7

Implement:

```text
tiny parser
```

from the written requirements only.

### Day 14

Solve:

```text
module/script
ASI
slash ambiguity
contextual keyword
```

problems.

### Day 21

Perform parser differential testing.

### Day 30

Explain the entire source-to-runtime pipeline in ten minutes.

---

# 99. Dependency Graph

```text
Chapter 01
  ↓
JavaScript / ECMAScript / Runtime
  ↓
Chapter 05–12
  ↓
declarations + expressions + execution
  ↓
Chapter 18
  ↓
classes
  ↓
Chapter 41
  ↓
specification architecture
  ↓
Chapter 42
  ↓
abstract operations
  ↓
Chapter 68–70
  ↓
transpilation + bundling + debugging
  ↓
Chapter 90–91
  ↓
modern ECMAScript + proposals
  ↓
Chapter 123
  ↓
grammar + parsing + early errors
  ↓
Chapter 124
  ↓
execution records + completion records + references
```

---

# 100. Completion Snapshot

```md
# Chapter 123 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Primary Gaps:
-

Lexical Grammar:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Syntactic Grammar:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Static Semantics:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Early Errors:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

ASI / Contextual Grammar:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Parser Engineering:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Specification Reasoning:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 101. Final Mental Model

Use this whenever you encounter unusual JavaScript syntax:

```text
1. What characters are present?
2. How are they lexically interpreted?
3. Which lexical goal is active?
4. What tokens result?
5. Which grammar production is expected here?
6. Is the token sequence syntactically valid?
7. Does a static semantic restriction apply?
8. Is this an early error?
9. If valid, what runtime semantics execute?
10. Which parts are ECMAScript and which belong to the host/toolchain?
```

That sequence is dramatically more reliable than:

```text
"JavaScript probably interprets this like..."
```

---

# 102. Final Principal Principle

> **Parsing is the boundary where raw source text becomes language structure. Early errors are the boundary where structurally parsed source is rejected before normal execution.**

The mature mental model is:

```text
source text
→ lexical interpretation
→ grammar
→ static semantics
→ valid program
→ runtime semantics
→ observable behavior
```

Once you understand this boundary, you can reason much more precisely about:

```text
syntax errors
ASI
modules
private fields
contextual keywords
proposal syntax
parser/tooling differences
transpilers
compilers
static analysis
```

And you can debug JavaScript language behavior by asking:

```text
Which layer rejected this program?
```

rather than guessing at runtime behavior.