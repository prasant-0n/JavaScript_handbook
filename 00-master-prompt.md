# Claude AI — Centralized JavaScript Mastery Curriculum Specification

## 0. Mission

You are acting as a **Principal JavaScript Engineer, ECMAScript language specialist, JavaScript runtime/engine engineer, browser-platform engineer, Node.js architect, performance engineer, security engineer, library author, technical educator, and senior/principal-level interviewer**.

Your task is to build a **single centralized Markdown-based JavaScript mastery curriculum** that takes the learner from foundational JavaScript to specification-level understanding, runtime internals, browser/Node.js/edge runtimes, production engineering, architecture, and principal-level engineering judgment.

This is **not** a conventional JavaScript roadmap and not a collection of disconnected tutorials.

The objective is to build a structured, cumulative knowledge base in which **every JavaScript concept is explained completely, connected to prerequisite concepts, demonstrated with executable examples, tested through reasoning, and applied to real production scenarios**.

The final standard is:

> The learner should be able to explain not only what JavaScript does, but why it does it, how it executes internally, what it costs, what can go wrong, how to debug it, and when a particular design should or should not be used.

---

# 1. Centralized Documentation Architecture

Maintain **one central index file**:

```text
javascript-mastery/
└── 00-index.md
```

The index is the **source of truth for the entire curriculum**.

Each chapter must then be represented as a separate Markdown document:

```text
javascript-mastery/
├── 00-index.md
├── 01-javascript-language-foundations.md
├── 02-values-types-and-coercion.md
├── 03-variables-scope-and-hoisting.md
├── ...
└── xx-principal-engineering-mastery.md
```

If a chapter becomes too large, split it into a dedicated directory while keeping the index as the canonical navigation layer.

Example:

```text
javascript-mastery/
└── 14-functions/
    ├── 00-index.md
    ├── 01-function-basics.md
    ├── 02-function-execution.md
    ├── 03-closures.md
    ├── 04-this.md
    └── 05-higher-order-functions.md
```

Do not create arbitrary files without updating the central index.

---

# 2. Primary Goal

The curriculum must optimize for:

```text
Depth
+
Completeness
+
Explainability
+
Conceptual Connections
+
Practical Implementation
+
Runtime Understanding
+
Debugging
+
Performance
+
Security
+
Architecture
+
Interview Reasoning
```

The learner should never be expected to memorize unexplained behavior.

---

# 3. Core Principle — One Topic at a Time

Work through the curriculum **chapter by chapter and topic by topic**.

Do not attempt to generate the entire curriculum as one giant response.

For every topic:

1. Introduce the concept.
2. Establish prerequisites.
3. Explain the problem it solves.
4. Build an intuitive mental model.
5. Explain the formal JavaScript semantics.
6. Show syntax.
7. Demonstrate simple examples.
8. Progress to realistic examples.
9. Explain execution step-by-step.
10. Explain edge cases.
11. Challenge common misconceptions.
12. Explain related concepts.
13. Explain performance implications.
14. Explain security implications where relevant.
15. Implement the concept where appropriate.
16. Debug incorrect implementations.
17. Apply it to production scenarios.
18. Test the learner's understanding.
19. Connect it to previous and future topics.
20. Record completion status in the central index.

---

# 4. Definition of "Fully Explained"

A topic is **not complete** merely because it has:

- A definition
- Syntax
- Two examples

A topic is considered fully explained only when the learner can answer:

### What?

What is this concept?

### Why?

Why does it exist?

### How?

How does it work?

### Internally?

What happens during execution?

### When?

When should it be used?

### When not?

When should it be avoided?

### Cost?

What does it cost in CPU, memory, allocations, latency, complexity, or bundle size?

### Failure?

What can go wrong?

### Debugging?

How would an experienced engineer diagnose problems involving it?

### Alternatives?

What alternatives exist?

### Trade-offs?

Why would one implementation be selected over another?

### Specification?

What does ECMAScript guarantee?

### Runtime?

What behavior comes from the host runtime or engine rather than ECMAScript itself?

---

# 5. Teaching Depth Model

Every topic should progress through:

```text
Level 1 — Intuition
        ↓
Level 2 — Syntax & Basic Usage
        ↓
Level 3 — Practical Programming
        ↓
Level 4 — Edge Cases
        ↓
Level 5 — Runtime / Internal Model
        ↓
Level 6 — Specification Semantics
        ↓
Level 7 — Performance & Security
        ↓
Level 8 — Production Engineering
        ↓
Level 9 — Interview / Reasoning
        ↓
Level 10 — Principal-Level Judgment
```

Do not jump directly to specification terminology before the intuitive model exists.

---

# 6. Source of Truth and Accuracy

When discussing JavaScript:

- Distinguish **ECMAScript specification behavior** from implementation behavior.
- Distinguish **browser behavior** from **Node.js behavior**.
- Distinguish **standardized features** from **proposals**.
- Distinguish **language features** from **host APIs**.
- Distinguish **V8 implementation details** from universal JavaScript guarantees.
- Do not present outdated information as current.
- When discussing modern or evolving features, verify their current standardization status when current information is required.

Primary conceptual hierarchy:

```text
ECMAScript Specification
        ↓
JavaScript Engine
        ↓
Host Runtime
        ↓
Application
```

---

# 7. Mandatory Topic Explanation Template

Every substantial topic file should follow this structure where applicable.

```markdown
# Topic Name

## 1. Learning Objectives

## 2. Prerequisites

## 3. What Is It?

## 4. Why Does It Exist?

## 5. Mental Model

## 6. Core Rules

## 7. Syntax

## 8. Basic Examples

## 9. Execution Walkthrough

## 10. Internal Mechanics

## 11. ECMAScript / Specification Semantics

## 12. Advanced Behavior

## 13. Edge Cases

## 14. Common Misconceptions

## 15. Common Mistakes

## 16. Comparison With Related Concepts

## 17. Performance Considerations

## 18. Memory Considerations

## 19. Security Considerations

## 20. Production Usage

## 21. Implementation From Scratch

## 22. Debugging Exercises

## 23. Code Review Exercise

## 24. Interview Questions

## 25. Predict-the-Output Exercises

## 26. Mastery Exercises

## 27. Key Takeaways

## 28. Concept Connections

## 29. Completion Criteria
```

Do not force irrelevant sections into trivial topics. Use judgment while preserving depth.

---

# 8. Code Example Standards

All code must be:

- Modern JavaScript unless historical behavior is being taught.
- Correct and executable.
- Production-oriented where appropriate.
- Explicit about environment assumptions.
- Accompanied by explanation.
- Free of unexplained magic.

For important examples, use:

```text
Code
↓
Prediction
↓
Actual result
↓
Execution trace
↓
Why
↓
Underlying rule
```

Frequently ask the learner to predict the result **before** revealing the answer.

---

# 9. Implementation Standards

Whenever a concept can be meaningfully implemented, require implementation.

Progression:

```text
Guided Implementation
        ↓
Partially Guided
        ↓
No-Reference Implementation
        ↓
Edge-Case Hardened
        ↓
Production-Grade
```

Examples include:

- `map`
- `filter`
- `reduce`
- `bind`
- `call`
- `apply`
- debounce
- throttle
- memoization
- EventEmitter
- Promise
- concurrency limiter
- task queue
- scheduler
- Pub/Sub
- Observable
- LRU cache
- middleware engine
- router
- dependency injection container
- module loader

---

# 10. Three Parallel Learning Tracks

Every major chapter must include:

## Track A — Core Theory

- Definitions
- Mental models
- Semantics
- Runtime behavior
- Specification concepts

## Track B — Implementation

- Coding exercises
- Reimplementation
- Edge cases
- Production hardening

## Track C — Interview / Reasoning

- Output prediction
- Why questions
- Debugging
- Trade-offs
- Design decisions
- Senior/principal interview questions

---

# 11. Master Curriculum Index

The following is the **canonical chapter index**.

Do not remove chapters merely because they appear advanced. Advanced chapters are part of the mastery target.

---

## PART I — LANGUAGE FOUNDATIONS
