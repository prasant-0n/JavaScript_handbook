

# Chapter 38 — Revision / Retrieval Record

### Retrieval Prompts

1. What is an async iterable?
2. What is an async iterator?
3. What is `Symbol.asyncIterator`?
4. Why does async `next()` return a Promise-like result?
5. What does `for await...of` do?
6. What is an async generator?
7. What happens on `break` from `for await...of`?
8. What is iterator closing?
9. What is async-from-sync iteration?
10. What is backpressure?
11. Why doesn't async iteration automatically guarantee bounded buffering?
12. How do you adapt an EventEmitter into an async iterator?
13. How can an async iterator own a resource?
14. Why is cancellation different from `done: true`?
15. How should cancellation affect an infinite stream?
16. What happens when the consumer throws?
17. What happens when the producer throws?
18. How would you implement bounded buffering?
19. How would you implement concurrent processing?
20. How would you preserve output ordering?
21. How would you checkpoint stream progress?
22. When is an async iterator better than an Observable?
23. When is a Web/Node Stream better than an async iterator?
24. How would you diagnose memory growth in a streaming pipeline?
25. How would you design graceful shutdown for an async stream?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit async iterator protocol
- [ ] Revisit for-await-of
- [ ] Revisit async generator cleanup
- [ ] Revisit iterator closing
- [ ] Revisit backpressure
- [ ] Revisit bounded buffering
- [ ] Revisit cancellation
- [ ] Revisit concurrency/order
- [ ] Revisit checkpointing
- [ ] Revisit stream ownership
```

### Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

### Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 38 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — async iteration protocols, `Symbol.asyncIterator`, `for await...of`, async generators, async-from-sync iteration, and iterator closing.
2. WHATWG Streams Standard — ReadableStream, queuing strategies, backpressure, cancellation, piping, readers/writers, and byte streams.
3. WHATWG Fetch Standard — streaming response bodies and cancellation integration.
4. Node.js official documentation — async iteration over Node streams, stream backpressure, pipeline APIs, and Node-specific lifecycle behavior.
5. MDN / browser documentation — developer-facing async iteration, async generators, `for await...of`, Web Streams, and browser compatibility.
6. Application architecture documentation — queue semantics, buffering, retry, checkpointing, ordering, ownership, concurrency, and shutdown.

Always distinguish:

```text
ECMAScript async-iteration semantics
vs
Web Streams semantics
vs
Node Streams semantics
vs
network/storage behavior
vs
application streaming policy
```

Do not claim that the language iterator protocol itself defines:

```text
highWaterMark
backpressure
exactly-once delivery
durability
retry
checkpointing
```

Those are properties of the surrounding stream or application architecture.

Do not assume that consuming a stream with `for await...of` automatically cancels or closes every underlying host resource; verify the specific stream/iterator contract.

---

# Chapter 38 — Completion Snapshot

```text
Chapter: 38
Title: Async Iteration and Streaming
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
