# Chapter 102 — Production JavaScript CLI

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build a production-grade JavaScript command-line application from first principles.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Architect · API Designer · Library Author · Security Engineer · Performance Engineer · DX Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **A CLI is not “a script that reads argv.” It is a user-facing product with a protocol, lifecycle, configuration model, error contract, security boundary, observability strategy, test strategy, and distribution surface.**

---

# 1. Project Mission

Build a real command-line application that demonstrates advanced JavaScript and Node.js engineering.

The project should progress through:

```text
single-file script
→ structured CLI
→ validated commands
→ configuration
→ filesystem/workspace support
→ async workflows
→ tests
→ packaging
→ observability
→ security hardening
→ production distribution
```

The project is intentionally designed to force the learner to use:

```text
language fundamentals
Node.js APIs
filesystem
streams
async/await
errors
modules
package.json
dependency management
testing
performance
security
observability
architecture
```

---

# 2. Recommended Project

Build a CLI named:

```text
jsforge
```

Purpose:

```text
inspect
validate
transform
analyze
and package JavaScript project artifacts
```

Example:

```bash
jsforge init
jsforge inspect src/
jsforge check src/
jsforge stats src/
jsforge graph src/
jsforge format-report src/
```

The exact feature set can evolve.

The architectural discipline should not.

---

# 3. Learning Objectives

By completing this project you should be able to:

- Design a CLI command model.
- Parse arguments safely.
- Separate command parsing from business logic.
- Implement subcommands.
- Implement flags/options.
- Validate CLI input.
- Handle positional arguments.
- Handle unknown options.
- Design help output.
- Design version output.
- Design exit codes.
- Design stdout/stderr semantics.
- Handle interactive input.
- Handle non-interactive execution.
- Handle environment variables.
- Handle configuration files.
- Define precedence rules.
- Resolve paths safely.
- Handle filesystem errors.
- Stream large files.
- Control concurrency.
- Implement cancellation.
- Handle SIGINT/SIGTERM appropriately.
- Design error taxonomy.
- Produce machine-friendly output.
- Produce human-friendly output.
- Implement JSON output.
- Implement quiet mode.
- Implement verbose/debug mode.
- Add structured logging.
- Design a plugin or adapter boundary.
- Use ES modules.
- Package the CLI.
- Define executable bin metadata.
- Test command behavior.
- Test failure paths.
- Test subprocess behavior.
- Test deterministic output.
- Benchmark expensive operations.
- Prevent accidental secret leakage.
- Avoid unsafe shell execution.
- Avoid path traversal.
- Prevent command injection.
- Handle symlinks deliberately.
- Handle partial failure.
- Implement atomic file writes where needed.
- Build a useful dependency graph.
- Create release artifacts.
- Document installation and usage.
- Design a maintainable CLI architecture.

---

# 4. Project Constraints

The project must satisfy:

```text
Node.js runtime
ES modules
strict input validation
no shell interpolation for untrusted input
clear exit status
deterministic tests
structured errors
machine-readable output mode
bounded concurrency
cancellation
documentation
```

Avoid:

```text
global mutable state
huge command handlers
process.exit() scattered through business logic
silent errors
unsafe exec string construction
unbounded Promise.all()
```

---

# 5. Project Structure

Target architecture:

```text
jsforge/
├── package.json
├── README.md
├── LICENSE
├── bin/
│   └── jsforge.js
├── src/
│   ├── cli.js
│   ├── main.js
│   ├── commands/
│   │   ├── init.js
│   │   ├── inspect.js
│   │   ├── check.js
│   │   ├── stats.js
│   │   ├── graph.js
│   │   └── report.js
│   ├── config/
│   │   ├── load-config.js
│   │   ├── defaults.js
│   │   └── validate-config.js
│   ├── fs/
│   │   ├── walk.js
│   │   ├── read-text.js
│   │   └── atomic-write.js
│   ├── analysis/
│   │   ├── file-stats.js
│   │   ├── imports.js
│   │   └── dependency-graph.js
│   ├── output/
│   │   ├── human.js
│   │   ├── json.js
│   │   └── errors.js
│   ├── runtime/
│   │   ├── signals.js
│   │   └── context.js
│   └── errors/
│       └── cli-errors.js
├── test/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
└── docs/
    ├── architecture.md
    ├── commands.md
    └── release.md
```

This is a target, not a demand to create every file on day one.

---

# 6. Architecture

Use:

```text
CLI boundary
   ↓
command parsing
   ↓
validated command model
   ↓
application/service layer
   ↓
domain operations
   ↓
Node/platform adapters
```

Do not let every command directly manipulate:

```text
process.argv
process.env
console
fs
```

---

# 7. CLI Boundary

Only the outer layer should understand:

```text
argv
stdin
stdout
stderr
exit code
signals
environment
```

The application layer should receive ordinary JavaScript values.

---

# 8. Command Model

Define a command as:

```text
name
description
arguments
options
validation
handler
```

Example conceptual model:

```js
{
  name: "stats",
  description: "Analyze project files",
  options: {
    json: { type: "boolean" },
    path: { type: "string" }
  },
  run: runStats
}
```

---

# 9. Parsing Strategy

You may initially parse manually.

Then decide whether a parser library is justified.

Manual parsing teaches:

```text
argv
option boundaries
values
errors
```

A mature library can later reduce parsing maintenance.

---

# 10. Manual Parser Exercise

Given:

```bash
jsforge stats src --json --max-files 100
```

produce:

```js
{
  command: "stats",
  args: ["src"],
  options: {
    json: true,
    maxFiles: 100
  }
}
```

---

# 11. Unknown Option Handling

Input:

```bash
jsforge stats --jsoon
```

Do not silently ignore the typo.

Return:

```text
unknown option: --jsoon
```

with a non-zero exit status.

---

# 12. Missing Value Handling

Input:

```bash
jsforge stats --max-files
```

should fail clearly.

Avoid:

```text
undefined
```

propagating into deep business logic.

---

# 13. Boolean Parsing

Support explicit semantics:

```text
--json
--no-color
--verbose
```

Avoid surprising coercion such as:

```text
--json=false
```

unless documented.

---

# 14. Short Options

Optional support:

```text
-v
-q
-h
```

Ensure collisions are handled.

---

# 15. Combined Short Options

If supporting:

```bash
-qv
```

define whether it means:

```text
-q -v
```

Do not implement ambiguous parsing accidentally.

---

# 16. Positional Arguments

Define:

```text
zero
one
many
```

allowed positionals per command.

Example:

```bash
jsforge inspect src/index.js
```

versus:

```bash
jsforge inspect src/a.js src/b.js
```

---

# 17. `--` Separator

Understand:

```bash
jsforge command -- --literal-name
```

This can be useful when positional data begins with `-`.

---

# 18. Help Design

Every command should provide:

```text
description
usage
arguments
options
examples
exit statuses
```

Example:

```text
Usage:
  jsforge stats <path> [options]

Options:
  --json
  --max-files <n>
  --verbose
  --help
```

---

# 19. Version Design

```bash
jsforge --version
```

should provide a deterministic version.

The value should come from the package release metadata rather than duplicated source constants where practical.

---

# 20. Exit Codes

Define an application-level policy.

Example:

```text
0 — success
1 — expected command failure
2 — usage/input error
3 — configuration error
4 — operational/environment error
```

The exact mapping is a design choice.

Document it.

---

# 21. stdout vs stderr

Use:

```text
stdout
→ successful program output

stderr
→ diagnostics/errors
```

This enables:

```bash
jsforge stats src > report.txt
```

without mixing diagnostics into the report.

---

# 22. Machine-Readable Output

Support:

```bash
jsforge stats src --json
```

Output stable structured JSON.

Avoid mixing:

```text
progress text
```

into JSON stdout.

Send progress to stderr if necessary.

---

# 23. Human Output

Default output should optimize:

```text
readability
scanability
useful summaries
```

Do not make terminal output impossible to pipe.

---

# 24. Quiet Mode

Optional:

```bash
--quiet
```

should suppress nonessential diagnostics while preserving meaningful failure information.

---

# 25. Verbose Mode

Optional:

```bash
--verbose
```

should add useful diagnostics.

Do not dump secrets.

---

# 26. Debug Mode

A debug mode can include:

```text
timing
resolved paths
command decisions
configuration source
```

Avoid:

```text
tokens
passwords
private URLs
```

unless explicitly redacted.

---

# 27. Configuration Sources

Support a clear precedence:

```text
built-in defaults
↓
configuration file
↓
environment variables
↓
CLI flags
```

Document the exact order.

---

# 28. Configuration Example

```json
{
  "include": ["src"],
  "exclude": ["node_modules", "dist"],
  "extensions": [".js", ".mjs", ".cjs"],
  "maxConcurrency": 8
}
```

---

# 29. Configuration Validation

Reject:

```json
{
  "maxConcurrency": -100
}
```

before execution.

Validate structure and ranges.

---

# 30. Configuration Discovery

Decide whether to support:

```text
jsforge.config.js
jsforge.config.json
package.json field
```

Do not silently load arbitrary config locations.

---

# 31. Environment Variables

Use namespaced variables:

```text
JSFORGE_MAX_CONCURRENCY
JSFORGE_LOG_LEVEL
```

Document conversions.

---

# 32. Configuration Precedence Test

Test:

```text
default = 4
file = 8
env = 16
CLI = 32
```

Expected:

```text
32
```

---

# 33. Path Handling

Resolve paths explicitly.

Distinguish:

```text
current working directory
CLI-provided relative path
absolute path
config-relative path
```

Document which base directory applies.

---

# 34. Path Traversal

If the CLI writes files under a controlled root, do not accept:

```text
../../outside-root
```

without deliberate semantics.

Validate normalized paths where trust boundaries require it.

---

# 35. Symlink Policy

Decide whether file traversal:

```text
follows symlinks
```

or:

```text
does not follow symlinks
```

Following symlinks can escape an intended workspace.

---

# 36. Recursive Directory Walk

Implement:

```js
async function* walk(root) {
  // yield files
}
```

Prefer async iteration for large trees.

---

# 37. Directory Traversal Requirements

Support:

```text
ignore directories
file extension filter
maximum depth
symlink policy
permission errors
cancellation
```

---

# 38. Node Directory API

Use modern filesystem APIs intentionally.

Choose between:

```text
fs.promises
streams
Dirent
```

according to scale.

---

# 39. Large Directory Trees

Do not eagerly build:

```js
const allFiles = [];
```

for arbitrarily large trees if a streaming traversal can process incrementally.

---

# 40. Bounded Concurrency

Bad:

```js
await Promise.all(
  files.map(analyze)
);
```

for enormous repositories.

Build:

```text
worker pool
```

or bounded mapper.

---

# 41. Concurrency Configuration

Example:

```bash
jsforge stats . --max-concurrency 8
```

The default should be conservative.

---

# 42. Backpressure

If analysis produces results faster than reporting/storage can consume them, use a bounded pipeline.

Avoid unlimited in-memory result accumulation.

---

# 43. Cancellation

Create an `AbortController`.

Pass:

```text
signal
```

through relevant operations.

---

# 44. Ctrl-C

Handle:

```text
SIGINT
```

gracefully.

Possible policy:

```text
first Ctrl-C
→ request shutdown

second Ctrl-C
→ force termination
```

Document behavior.

---

# 45. SIGTERM

Handle termination in CI/containers/server execution.

Stop creating new work and clean up safely.

---

# 46. Error Taxonomy

Define errors such as:

```js
class CliUsageError extends Error {}
class ConfigError extends Error {}
class FileOperationError extends Error {}
class AnalysisError extends Error {}
class CancelledError extends Error {}
```

Do not use:

```text
string messages
```

as the only classification mechanism.

---

# 47. Error Ownership

The application layer can throw.

The CLI boundary decides:

```text
message
exit code
format
```

---

# 48. Avoid process.exit Everywhere

Do not write:

```js
function analyze() {
  if (...) process.exit(1);
}
```

Prefer:

```text
throw structured error
```

and let the outer layer translate it.

---

# 49. Error Formatting

Human:

```text
Error: directory not found: src/
```

JSON:

```json
{
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "Directory not found",
    "path": "src/"
  }
}
```

---

# 50. Error Stability

Machine-readable error codes should be more stable than human wording.

Users may script against:

```text
error.code
```

not message sentences.

---

# 51. File Reading

For small files:

```js
await fs.readFile(path, "utf8");
```

For huge files, prefer streaming.

---

# 52. Atomic Writes

When transforming files, avoid leaving partially written output after failure.

Possible pattern:

```text
write temporary file
→ fsync where required by durability policy
→ rename
```

Exact durability needs depend on platform and use case.

---

# 53. Dry Run

Support:

```bash
jsforge transform src --dry-run
```

A dry run should report intended changes without mutating files.

---

# 54. Backup Policy

Do not automatically create unlimited backups.

If backups are supported, define:

```text
location
naming
retention
cleanup
```

---

# 55. Idempotency

Running:

```bash
jsforge init
```

twice should either:

```text
safely do nothing
```

or:

```text
explicitly report existing state
```

Avoid destructive repetition.

---

# 56. Deterministic Output

For graph/stats commands, sort output where order has no semantic meaning.

Stable output simplifies:

```text
tests
diffs
CI
automation
```

---

# 57. File Ordering

Do not rely on filesystem enumeration order if output is expected to be deterministic.

Explicitly sort.

---

# 58. Line/Byte Counting

Define:

```text
line
byte
UTF-16 code unit
Unicode code point
```

For CLI statistics, specify which metric you report.

---

# 59. Text Encoding

Do not silently assume every file is UTF-8 if the CLI can encounter arbitrary binary data.

Use extension/type policies and explicit decode behavior.

---

# 60. Binary Files

A source-analysis CLI should normally skip:

```text
images
archives
executables
```

unless explicitly requested.

---

# 61. Ignore Rules

Support at least:

```text
node_modules
.git
dist
coverage
```

as defaults if appropriate.

Allow configuration overrides.

---

# 62. Gitignore Integration

Decide whether to honor `.gitignore`.

If yes, document:

```text
whether it is default
whether config overrides
whether negation patterns work
```

Do not claim Git compatibility unless actually implemented.

---

# 63. Command — init

Purpose:

```text
create jsforge configuration
```

Example:

```bash
jsforge init
```

Should be:

```text
non-destructive
interactive or non-interactive
repeatable
```

---

# 64. Command — inspect

Purpose:

```text
show files and metadata
```

Example:

```bash
jsforge inspect src/
```

Potential fields:

```text
path
size
extension
line count
hash
```

---

# 65. Command — stats

Purpose:

```text
aggregate source statistics
```

Example:

```bash
jsforge stats src/
```

Possible output:

```text
Files
Lines
Bytes
Extensions
Largest files
```

---

# 66. Command — check

Purpose:

```text
validate project rules
```

Example:

```bash
jsforge check .
```

Return non-zero on violations.

---

# 67. Command — graph

Purpose:

```text
analyze imports and dependency relationships
```

Example:

```bash
jsforge graph src/
```

Output could support:

```text
text
json
dot
```

---

# 68. Command — report

Purpose:

```text
combine multiple analyses
```

Example:

```bash
jsforge report . --json > report.json
```

Ensure the report has a versioned schema if automation depends on it.

---

# 69. Dependency Graph Model

Represent:

```js
{
  nodes: [...],
  edges: [...]
}
```

Keep domain representation separate from rendering formats.

---

# 70. Import Extraction

Start with straightforward JavaScript module syntax:

```text
import
export from
dynamic import
```

Be explicit about unsupported syntax/heuristics.

Do not pretend a regex is a complete JavaScript parser.

---

# 71. Parser Decision

For serious syntax analysis, choose:

```text
AST parser
```

rather than increasingly complex regular expressions.

This is an architecture exercise.

---

# 72. Parser Abstraction

Define:

```js
parseModule(source, options)
```

Then the command layer does not care which parser implementation is used.

---

# 73. Plugin Boundary

Potential future:

```text
analyzer
formatter
reporter
```

plugins.

Do not create a plugin system prematurely.

First stabilize the internal contracts.

---

# 74. Output Adapter

Define:

```text
HumanReporter
JsonReporter
DotReporter
```

They should consume the same domain results.

---

# 75. Reporting Contract

Example:

```js
{
  version: 1,
  command: "stats",
  generatedAt: "...",
  summary: {...},
  files: [...]
}
```

Decide whether timestamps make deterministic output harder.

If deterministic CI output matters, omit generated timestamps unless explicitly requested.

---

# 76. Time Injection

For testable reports:

```js
createReport({
  clock
});
```

Avoid reading current time deep inside domain code.

---

# 77. Randomness Injection

If IDs are needed, inject an ID generator for deterministic tests.

---

# 78. Environment Injection

Instead of:

```js
process.env.X
```

everywhere:

```js
createRuntimeContext({
  env
});
```

This improves testability.

---

# 79. Filesystem Injection

You may abstract filesystem operations for unit tests:

```js
createFsAdapter(...)
```

Do not abstract every `fs` function without purpose.

Use boundaries where isolation matters.

---

# 80. Clock Injection

Useful for:

```text
reports
cache TTL
temporary file names
```

---

# 81. Logging Interface

A small interface:

```js
logger.info(...)
logger.warn(...)
logger.error(...)
logger.debug(...)
```

Keep logging separate from business decisions.

---

# 82. Structured Logging

Conceptual event:

```js
logger.info("analysis_complete", {
  files: 120,
  durationMs: 34
});
```

Choose a stable schema if machine ingestion matters.

---

# 83. Redaction

Centralize redaction for:

```text
tokens
passwords
API keys
authorization headers
```

Do not rely on every caller remembering to redact.

---

# 84. Security Boundary — Shell Execution

Avoid:

```js
exec(`git ${userInput}`);
```

Prefer argument-array APIs or dedicated libraries when invoking external commands.

---

# 85. Security Boundary — Path Input

Validate paths according to the allowed workspace.

Normalize before policy decisions.

---

# 86. Security Boundary — File Permissions

When writing files, use explicit permissions appropriate to the content.

Do not accidentally create secrets as world-readable files.

---

# 87. Security Boundary — Environment

Environment variables can contain secrets.

Do not dump all of `process.env` in debug output.

---

# 88. Security Boundary — Config Files

Treat configuration files as potentially sensitive.

Do not automatically print every setting.

---

# 89. Security Boundary — Temporary Files

Use safe temporary paths and cleanup policies.

Avoid predictable shared filenames.

---

# 90. Security Boundary — Symlinks

Symlink traversal can cross workspace boundaries.

Make policy explicit.

---

# 91. Security Boundary — Resource Exhaustion

Input can intentionally contain:

```text
millions of files
huge file
deep tree
large line
pathological syntax
```

Use:

```text
limits
streaming
bounded concurrency
```

---

# 92. Resource Limits

Support configurable limits such as:

```text
max files
max file bytes
max depth
max total bytes
max concurrency
```

---

# 93. Failure Isolation

One invalid file should not necessarily terminate a whole batch.

Define per-command policy:

```text
fail-fast
best-effort
collect-all-errors
```

---

# 94. Partial Failure Output

JSON example:

```json
{
  "succeeded": 98,
  "failed": 2,
  "errors": [...]
}
```

Define exit status for partial success.

---

# 95. Exit Status for Partial Failure

Possible policy:

```text
0 = all succeeded
1 = one or more checks failed
2 = CLI misuse
```

Document it.

---

# 96. Progress Reporting

For large jobs:

```text
processed 500 / 10000
```

Progress should go to:

```text
stderr
```

when stdout is machine-readable.

---

# 97. TTY Detection

Interactive formatting can depend on:

```js
process.stdout.isTTY
```

But always provide explicit override options if automation matters.

---

# 98. Color

Do not force color into piped output.

Support:

```text
--color
--no-color
```

or a documented environment policy.

---

# 99. Terminal Width

Interactive tables should respect terminal width where practical.

Avoid making machine output depend on terminal width.

---

# 100. Interactive Prompts

If `init` prompts:

```text
Is stdin a TTY?
```

Non-interactive CI should have:

```text
defaults
or explicit flags
```

so it does not hang waiting for input.

---

# 101. CI Mode

Example:

```bash
CI=1 jsforge check .
```

Do not automatically infer all semantics from CI environment variables.

Document behavior.

---

# 102. Configuration in CI

Prefer explicit:

```bash
jsforge check . --json
```

over relying on hidden interactive behavior.

---

# 103. Shell Completion

Optional advanced feature:

```text
bash
zsh
fish
PowerShell
```

Completion definitions should be generated from command metadata when practical.

---

# 104. Help Generation

Store command definitions in structured form so help can be generated rather than duplicated manually.

---

# 105. Package Metadata

`package.json` should define:

```json
{
  "name": "jsforge",
  "type": "module",
  "bin": {
    "jsforge": "./bin/jsforge.js"
  }
}
```

Add version, description, license, engines, files, and scripts appropriate to the project.

---

# 106. Executable Entry Point

The executable should have an appropriate Node shebang:

```js
#!/usr/bin/env node
```

Keep the entry point thin.

---

# 107. Thin Entry Point

Ideal:

```text
parse environment
→ invoke main
→ map result/error to process outcome
```

Avoid putting the full application in `bin/jsforge.js`.

---

# 108. Package Files

Choose package contents explicitly.

Avoid shipping:

```text
private notes
tests
local artifacts
secrets
```

unless intentionally included.

---

# 109. `npm pack` Inspection

Before release:

```bash
npm pack --dry-run
```

Review the file list.

Package contents are part of your security and compatibility surface.

---

# 110. Engine Compatibility

Define the Node.js version range you actually test.

Do not claim support for versions you do not test.

---

# 111. Dependency Policy

Keep dependencies minimal but do not reinvent mature parsing, argument, or security-sensitive functionality without justification.

---

# 112. Lockfile

Commit the lockfile for the application.

Do not assume it eliminates all supply-chain risk.

---

# 113. Scripts

Example:

```json
{
  "scripts": {
    "test": "node --test",
    "lint": "...",
    "format": "...",
    "check": "node ./bin/jsforge.js check .",
    "build": "..."
  }
}
```

Use tooling appropriate to the repository.

---

# 114. Unit Tests

Test:

```text
parser
validators
path rules
configuration precedence
analysis functions
report rendering
```

---

# 115. Integration Tests

Run:

```text
real CLI process
```

against temporary fixture directories.

Test:

```text
argv
stdout
stderr
exit code
```

---

# 116. Fixture Design

Create fixtures for:

```text
small project
large project
invalid files
symlinks
permission failure
nested directories
binary files
empty directory
```

---

# 117. Temporary Directories

Each test should own its temporary workspace.

Do not reuse global test directories.

---

# 118. Process Isolation

Subprocess tests are useful for:

```text
signal handling
exit codes
environment
real module loading
```

---

# 119. Deterministic Tests

Avoid tests that depend on:

```text
real clock
real random IDs
host-specific path assumptions
network
user home directory
```

unless the test specifically validates those behaviors.

---

# 120. Snapshot Testing

Useful for:

```text
help
human reports
```

but keep snapshots focused.

---

# 121. JSON Contract Testing

Validate:

```text
schema
types
required fields
exit status
```

rather than only string equality when appropriate.

---

# 122. Error Testing

For every command, test:

```text
missing input
bad input
missing file
permission issue
cancellation
unexpected internal failure
```

---

# 123. Cancellation Testing

Start a large operation.

Abort it.

Assert:

```text
no new work starts
temporary resources clean up
process terminates within expected bounds
```

---

# 124. Concurrency Testing

Create:

```text
1000 files
```

and a concurrency limit:

```text
8
```

Instrument active operations.

Assert:

```text
max active <= 8
```

---

# 125. Performance Testing

Measure:

```text
files/sec
MB/sec
peak memory
CPU
startup time
```

---

# 126. Startup Performance

A CLI is often invoked frequently.

Startup cost matters.

Avoid loading expensive modules before the command requires them.

---

# 127. Lazy Loading

Potential design:

```text
parse command
→ dynamically import command implementation
```

Use only when the complexity is justified.

---

# 128. Large Repository Performance

Test:

```text
1k files
10k files
100k files
```

The exact scale depends on machine resources.

Observe:

```text
memory
time
GC
```

---

# 129. Streaming Performance

Compare:

```text
read all files into memory
```

against:

```text
stream/bounded processing
```

for large inputs.

---

# 130. Algorithmic Complexity

For:

```text
N files
```

avoid accidental:

```text
N²
```

dependency comparisons if a map/index can reduce it.

---

# 131. Hashing Cost

File hashes can be expensive.

Make hashing:

```text
optional
```

unless required by the command.

---

# 132. Parse Cost

Parsing every file into an AST is more expensive than counting bytes/lines.

Choose analysis depth per command.

---

# 133. Cache Trade-Off

Optional project cache can improve repeated runs.

Cost:

```text
invalidation
disk space
staleness
complexity
```

Do not add it until profiling shows benefit.

---

# 134. Progress vs Performance

Frequent progress updates can increase overhead.

Batch updates.

---

# 135. Memory Budget

Define:

```text
maximum expected repository
maximum file
maximum output
```

Then test near the boundary.

---

# 136. Security Testing

Test:

```text
path traversal
symlink escape
shell injection
oversized input
malformed configuration
secret logging
unsafe temporary files
```

---

# 137. Dependency Security

Audit:

```text
direct
transitive
native
install scripts
```

according to your ecosystem tooling.

---

# 138. Fuzzing Targets

Good fuzz candidates:

```text
argument parser
config parser
path normalization
source parser adapter
```

---

# 139. Property Tests

Useful properties:

```text
normalizing an already-normalized path is stable
sorting output twice does not change it
JSON report parses as valid JSON
```

---

# 140. Documentation

README should include:

```text
install
quick start
commands
options
examples
exit codes
configuration
security notes
supported Node versions
development
release
```

---

# 141. Architecture Documentation

Document:

```text
layers
command lifecycle
configuration precedence
error flow
cancellation
output contracts
```

---

# 142. Operational Documentation

Include:

```text
troubleshooting
debug mode
common failures
performance limits
```

---

# 143. Release Checklist

```text
[ ] tests pass
[ ] lint passes
[ ] package contents reviewed
[ ] version correct
[ ] changelog updated
[ ] supported runtime verified
[ ] security checks complete
[ ] npm pack reviewed
[ ] README examples tested
```

---

# 144. Failure Modes

Watch for:

```text
global mutable CLI state
unclear argument parsing
silent unknown flags
mixed stdout/stderr
unstable JSON
unbounded concurrency
unbounded memory
unsafe shell execution
unsafe paths
symlink escape
partial writes
uncancelable work
hanging interactive prompts
incorrect exit codes
hidden configuration precedence
dependency bloat
```

---

# 145. Production Architecture Review

Ask:

```text
Can I test commands without launching a process?
Can I reuse analysis functions as a library?
Can CI consume output safely?
Can the process cancel cleanly?
Can a huge repository be processed?
Can malformed input fail safely?
Can errors be classified?
Can secrets leak?
Can the package be reproduced?
Can users understand failures?
```

---

# 146. Implementation Progression

## Stage 1 — Guided

Build:

```text
--help
--version
one command
simple parser
```

## Stage 2 — Partially Guided

Add:

```text
config
multiple commands
JSON output
errors
```

## Stage 3 — No Reference

Implement:

```text
directory traversal
stats
bounded concurrency
```

## Stage 4 — Edge-Case Hardened

Add:

```text
symlink policy
limits
cancellation
partial failures
atomic writes
```

## Stage 5 — Production Grade

Add:

```text
tests
security
performance
packaging
observability
documentation
release pipeline
```

---

# 147. Suggested Milestones

```text
Milestone 1 — CLI shell
Milestone 2 — command parser
Milestone 3 — configuration
Milestone 4 — file traversal
Milestone 5 — statistics
Milestone 6 — dependency graph
Milestone 7 — cancellation
Milestone 8 — bounded concurrency
Milestone 9 — output contracts
Milestone 10 — test suite
Milestone 11 — security hardening
Milestone 12 — packaging
Milestone 13 — performance
Milestone 14 — release
```

---

# 148. Track A — Core Theory

Study while building:

```text
process.argv
stdin/stdout/stderr
process lifecycle
signals
filesystem
streams
async iterators
Promises
AbortSignal
ES modules
package.json
dependency resolution
errors
memory
concurrency
security
testing
```

---

# 149. Track B — Implementation

You must actually build:

```text
CLI parser
command registry
configuration resolver
filesystem walker
bounded worker pool
analyzer
reporters
error mapper
signal manager
test harness
package artifact
```

---

# 150. Track C — Interview / Reasoning

Be able to defend:

```text
Why ESM?
Why this command structure?
Why bounded concurrency?
Why stdout/stderr separation?
Why structured errors?
Why process boundary tests?
Why not a global event bus?
Why not Promise.all on every file?
Why not shell strings?
Why not parse source with regex?
Why not add caching immediately?
Why is cancellation part of correctness?
```

---

# 151. Mastery Gate

```text
Understand
→ Explain
→ Predict
→ Implement
→ Debug
→ Apply
→ Compare
→ Defend
```

---

# 152. Code Review Exercise

Review:

```js
async function stats(dir) {
  const files = await findAll(dir);

  const results = await Promise.all(
    files.map(async file => {
      const content =
        await fs.readFile(file, "utf8");

      return {
        file,
        lines: content.split("\n").length
      };
    })
  );

  console.log(JSON.stringify(results));
}
```

Identify at least:

```text
unbounded memory
unbounded concurrency
stdout contract problem
binary-file risk
cancellation gap
result retention
```

Then redesign.

---

# 153. Code Review Exercise

Review:

```js
exec(
  `git status --short ${dir}`
);
```

Identify the security problem.

Redesign using safe argument handling.

---

# 154. Code Review Exercise

Review:

```js
catch (error) {
  console.error(error);
  process.exit(1);
}
```

Discuss:

```text
where should process exit happen?
how should errors be classified?
how should JSON mode behave?
```

---

# 155. Code Review Exercise

Review:

```js
const options =
  JSON.parse(
    await fs.readFile("jsforge.json")
  );
```

Questions:

```text
what if file missing?
malformed?
wrong types?
unexpected keys?
wrong working directory?
```

---

# 156. Code Review Exercise

Review:

```js
for (const file of files) {
  await analyze(file);
}
```

Is this wrong?

Answer:

```text
not necessarily
```

It may be appropriate for small work or when strict sequencing is required.

The question is:

```text
required throughput
resource limits
dependency relationships
```

---

# 157. Code Review Exercise

Review:

```js
await Promise.all(
  files.map(analyze)
);
```

Is this wrong?

Again:

```text
not universally
```

It becomes risky when:

```text
N large
resource capacity bounded
memory budget low
downstream service limited
```

---

# 158. Interview Questions — Senior

1. How would you architect a Node CLI?
2. How would you keep the CLI testable?
3. Why separate stdout and stderr?
4. How do you implement exit codes?
5. How do you handle SIGINT?
6. How do you bound concurrency?
7. How do you avoid reading every file into memory?
8. How do you handle symlinks?
9. How do you prevent shell injection?
10. How do you design machine-readable output?

---

# 159. Interview Questions — Principal

1. How would you design a CLI that also has a reusable library API?
2. How would you handle 10 million files?
3. How would you guarantee bounded memory?
4. How would you preserve compatibility of JSON output?
5. How would you evolve commands without breaking scripts?
6. How would you design plugin APIs?
7. How would you decide whether to add a cache?
8. How would you package native dependencies?
9. How would you debug a production CLI failure?
10. How would you design the release and support policy?

---

# 160. Mastery Exercise — Build From Scratch

Starting with an empty directory, implement:

```bash
jsforge --help
jsforge --version
jsforge inspect .
jsforge stats .
jsforge check .
jsforge graph .
```

No tutorial copy-paste.

Use references only for syntax/API verification.

---

# 161. Mastery Exercise — Non-Interactive CI

Run:

```bash
jsforge check . --json
```

and ensure:

```text
no prompt
stable JSON
correct exit status
diagnostics on stderr
```

---

# 162. Mastery Exercise — Large Repository

Create or use a large fixture.

Measure:

```text
time
peak memory
files/sec
```

---

# 163. Mastery Exercise — Cancellation

Start a long analysis.

Send:

```text
Ctrl-C
```

Verify:

```text
cleanup
bounded shutdown
non-zero exit
no corruption
```

---

# 164. Mastery Exercise — Partial Failure

Include:

```text
valid file
invalid file
permission failure
binary file
```

Define behavior.

---

# 165. Mastery Exercise — Security

Attempt:

```text
../../../
symlink escape
shell metacharacters
huge input
secret in config
```

Verify defenses.

---

# 166. Mastery Exercise — Release

Produce:

```text
package
checksum
README
CHANGELOG
release notes
```

and inspect package contents.

---

# 167. Mastery Exercise — Migration

Change:

```text
JSON output version 1
```

to:

```text
version 2
```

without silently breaking users.

Design compatibility.

---

# 168. Mastery Exercise — Library Extraction

Refactor:

```text
analysis
```

into a reusable API:

```js
import { analyzeProject }
  from "jsforge";
```

while retaining the CLI.

Defend the boundary.

---

# 169. Mastery Exercise — Benchmark

Compare:

```text
sequential
bounded concurrency 4
bounded concurrency 8
bounded concurrency 16
unbounded
```

Record:

```text
duration
peak memory
CPU
```

Choose the operating point based on the workload.

---

# 170. Mastery Exercise — Failure Injection

Simulate:

```text
missing file
slow filesystem
permission failure
abort
invalid config
broken parser
```

Verify predictable behavior.

---

# 171. Final Acceptance Test

The CLI is production-grade when:

```text
[ ] help is clear
[ ] version works
[ ] commands are composable
[ ] stdout/stderr semantics are stable
[ ] exit codes are documented
[ ] configuration precedence is defined
[ ] paths are safe
[ ] symlinks have explicit policy
[ ] input is bounded
[ ] concurrency is bounded
[ ] cancellation works
[ ] errors are classified
[ ] JSON output is stable
[ ] human output is readable
[ ] tests cover failure paths
[ ] package contents are reviewed
[ ] security tests exist
[ ] performance is measured
[ ] documentation is usable
[ ] release process is reproducible
```

---

# 172. Production Checklist

```text
[ ] executable entry point
[ ] supported Node versions
[ ] version metadata
[ ] command parser
[ ] config loader
[ ] validation
[ ] path normalization
[ ] symlink policy
[ ] resource limits
[ ] bounded concurrency
[ ] cancellation
[ ] structured errors
[ ] JSON mode
[ ] stderr diagnostics
[ ] signal handling
[ ] tests
[ ] packaging
[ ] security controls
[ ] performance profile
[ ] release documentation
```

---

# 173. Concept Connections

## Depends On

```text
Chapter 22 — Arrays
Chapter 25 — Iterators
Chapter 31–38 — Async / Promises / Streams / Cancellation
Chapter 45 — Memory
Chapter 58–63 — Node.js
Chapter 64–70 — Modules / Tooling
Chapter 71–73 — Data Structures / Complexity / Algorithms
Chapter 78–89 — Production / Testing / Debugging
Chapter 94–97 — Compatibility / Legacy / Wasm / Edge
Chapter 98–101 — Judgment
```

## Builds Toward

```text
Chapter 103 — Vanilla Browser App
Chapter 104 — Production HTTP Client
Chapter 105 — Node REST API
Chapter 106 — Real-Time WebSocket
```

## Revisited

```text
filesystem
streams
async iteration
errors
signals
modules
package metadata
dependency management
security
performance
testing
observability
architecture
```

---

# 174. Spaced Retrieval Schedule

### Day 0

```text
CLI boundary
argv
stdout/stderr
exit codes
```

### Day 1

```text
configuration
filesystem
paths
errors
```

### Day 3

```text
concurrency
streams
cancellation
signals
```

### Day 7

```text
security
testing
packaging
```

### Day 14

```text
performance
compatibility
release
```

### Day 30

Rebuild the CLI core from scratch.

### Day 60

Design a versioned command/API migration.

### Day 90

Defend the architecture as a principal engineer.

---

# 175. Revision / Retrieval Record

```md
# Chapter 102 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- CLI boundary [ ]
- argv parsing [ ]
- command model [ ]
- flags/options [ ]
- stdout/stderr [ ]
- exit codes [ ]
- configuration precedence [ ]
- filesystem traversal [ ]
- path safety [ ]
- symlink policy [ ]
- bounded concurrency [ ]
- cancellation [ ]
- signals [ ]
- error taxonomy [ ]
- machine-readable output [ ]
- packaging [ ]
- testing [ ]
- security [ ]
- performance [ ]
- observability [ ]
- release [ ]

## Build Evidence
- Repository:
- Commit:
- Benchmark:
- Security test:
- Integration test:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 102 — Canonical References and Source Discipline

Primary references:

1. Node.js Documentation  
   https://nodejs.org/docs/

2. Node.js File System  
   https://nodejs.org/api/fs.html

3. Node.js Process  
   https://nodejs.org/api/process.html

4. Node.js Child Process  
   https://nodejs.org/api/child_process.html

5. Node.js Streams  
   https://nodejs.org/api/stream.html

6. Node.js Test Runner  
   https://nodejs.org/api/test.html

7. npm package.json documentation  
   https://docs.npmjs.com/cli/configuring-npm/package-json

8. ECMAScript Language Specification  
   https://tc39.es/ecma262/

9. OWASP  
   https://owasp.org/

Source discipline:

```text
language semantics
→ ECMAScript

Node CLI/runtime behavior
→ Node.js documentation

package metadata/install behavior
→ npm documentation

security
→ authoritative security guidance + threat model

performance
→ benchmarks + profiling

project decisions
→ measured workload + explicit trade-offs
```

---

# 176. Completion Snapshot

```text
Part XX — Projects

Chapter 102 — Production JavaScript CLI
[ ] Not Started

Track A — Core Theory
[ ] CLI architecture
[ ] argv
[ ] stdout/stderr
[ ] exit codes
[ ] stdin/TTY
[ ] signals
[ ] filesystem
[ ] paths
[ ] streams
[ ] async iteration
[ ] promises
[ ] cancellation
[ ] modules
[ ] package.json
[ ] dependencies
[ ] errors
[ ] security
[ ] performance
[ ] testing
[ ] observability
[ ] release

Track B — Implementation
[ ] CLI shell
[ ] parser
[ ] command registry
[ ] init
[ ] inspect
[ ] stats
[ ] check
[ ] graph
[ ] report
[ ] config
[ ] walker
[ ] bounded concurrency
[ ] cancellation
[ ] reporters
[ ] errors
[ ] test harness
[ ] security tests
[ ] benchmarks
[ ] package release

Track C — Interview / Reasoning
[ ] Defend CLI boundary
[ ] Defend stdout/stderr
[ ] Defend exit codes
[ ] Defend concurrency choice
[ ] Defend cancellation
[ ] Defend parser choice
[ ] Defend dependency choices
[ ] Defend security controls
[ ] Defend performance design
[ ] Defend packaging strategy

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# 177. Completion Criteria

Do not mark this project mastered because the command runs locally.

You are ready to move forward when you can independently:

1. Explain the CLI architecture.
2. Parse and validate commands.
3. Design stable options and exit codes.
4. Separate stdout and stderr.
5. Implement configuration precedence.
6. Traverse large directories safely.
7. Bound concurrency.
8. Implement cancellation.
9. Handle SIGINT/SIGTERM.
10. Classify errors.
11. Produce stable JSON output.
12. Produce useful human output.
13. Protect against path traversal.
14. Define symlink behavior.
15. Avoid shell injection.
16. Bound memory/resource use.
17. Write integration tests around the real executable.
18. Test failure and cancellation behavior.
19. Benchmark realistic workloads.
20. Package the CLI correctly.
21. Review package contents.
22. Define runtime compatibility.
23. Document user and automation contracts.
24. Explain every major architectural trade-off.
25. Defend the project in a principal-level design review.

---

# Final Mental Model

```text
argv
 ↓
parse
 ↓
validate
 ↓
command model
 ↓
application logic
 ↓
filesystem / runtime adapters
 ↓
domain result
 ↓
renderer
 ↓
stdout / stderr
 ↓
exit status
```

Cross-cutting:

```text
config
security
limits
cancellation
observability
testing
packaging
```

The production CLI is successful when it behaves predictably for:

```text
happy path
bad input
large input
slow input
partial failure
cancellation
CI
interactive users
automation
upgrades
```

> **Mastery reminder:** A production CLI is a protocol. Treat its arguments, output, exit codes, configuration, errors, and filesystem behavior as public contracts.


---

# Appendix A — CLI Contract Matrix

| Area | Human User | Shell Script | CI | Library Consumer |
|---|---|---|---|---|
| Help | rich | available | available | API docs |
| stdout | readable | parseable if requested | stable | not primary |
| stderr | diagnostics | diagnostics | diagnostics | not primary |
| exit code | meaningful | critical | critical | not primary |
| config | convenient | explicit | explicit | programmatic |
| cancellation | Ctrl-C | signal | signal | AbortSignal |
| errors | friendly | code/schema | stable | Error classes |

---

# Appendix B — Command Contract Template

```md
# Command: stats

## Purpose

-

## Usage

-

## Positional Arguments

-

## Options

-

## Configuration

-

## stdout

-

## stderr

-

## Exit Codes

-

## JSON Schema

-

## Performance Notes

-

## Failure Modes

-

## Examples

-
```

---

# Appendix C — Architecture Decision Record

```md
# ADR: Bounded Concurrency

## Context
Repositories may contain very large numbers of files.

## Problem
Unbounded Promise.all can create excessive memory and downstream pressure.

## Options
1. Sequential
2. Unbounded Promise.all
3. Bounded worker pool

## Decision
Bounded worker pool.

## Trade-Off
More implementation complexity for predictable resource use.

## Revisit Trigger
Measured workload shows a materially different optimal concurrency.
```

---

# Appendix D — Test Matrix

```text
Command
├── success
├── missing input
├── invalid option
├── invalid config
├── permission failure
├── malformed source
├── empty input
├── huge input
├── cancellation
├── partial failure
└── JSON mode

Filesystem
├── normal path
├── relative path
├── absolute path
├── symlink
├── broken symlink
├── nested tree
├── large file
└── inaccessible directory

Process
├── exit 0
├── exit non-zero
├── SIGINT
├── SIGTERM
└── non-TTY
```

---

# Appendix E — Release Verification Commands

Use commands appropriate to your package manager and release process.

Conceptual verification:

```bash
node ./bin/jsforge.js --help
node ./bin/jsforge.js --version
node ./bin/jsforge.js stats . --json
npm test
npm pack --dry-run
```

Then test the produced package in a clean temporary project.

---

# Appendix F — Principal Review Questions

```text
What contracts are public?
What breaks if stdout changes?
What breaks if an exit code changes?
What happens with 10 million files?
What happens with one 20 GB file?
What happens on Ctrl-C?
What happens if the config is malformed?
What happens if a symlink points outside the workspace?
What happens if a dependency is compromised?
What happens if one file cannot be read?
What happens if output is piped?
What happens in a CI environment?
What happens when the runtime is upgraded?
What happens when JSON schema evolves?
What behavior is intentionally unsupported?
```

---

# Appendix G — Weekly Project Journal

```md
## Week

### Feature
-

### Design Decision
-

### Evidence
-

### Failure
-

### Root Cause
-

### Fix
-

### Trade-Off
-

### Benchmark
-

### Security Finding
-

### Test Added
-

### Documentation Updated
-

### Next Risk
-
```

---

# Appendix H — Project Quality Bar

```text
Level 1 — Runs
Level 2 — Works on normal input
Level 3 — Handles expected failures
Level 4 — Tested
Level 5 — Bounded
Level 6 — Observable
Level 7 — Secure
Level 8 — Performant
Level 9 — Packaged
Level 10 — Defensible at principal review
```