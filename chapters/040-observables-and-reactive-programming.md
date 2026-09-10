# Chapter 40 — Observables and Reactive Programming

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Define reactive programming precisely.
- Define an Observable as a conceptual abstraction.
- Distinguish Observables from Promises.
- Distinguish Observables from async iterators.
- Distinguish Observables from EventEmitters.
- Distinguish push-based and pull-based data flow.
- Explain why Promises represent one eventual result while Observables can represent sequences over time.
- Explain cold and hot Observables.
- Explain unicast and multicast behavior.
- Understand subscriptions and subscription lifetime.
- Understand `next`, `error`, and `complete` notification channels.
- Explain why Observable contracts are different from Promise settlement.
- Understand lazy versus eager execution.
- Understand operators and stream composition.
- Understand transformation, filtering, combination, flattening, buffering, throttling, debouncing, and windowing.
- Explain higher-order Observables.
- Understand flattening strategies conceptually: merge, concat, switch, and exhaust behavior.
- Understand cancellation and unsubscription.
- Understand teardown and resource cleanup.
- Explain backpressure challenges in push-based systems.
- Understand buffering policies.
- Understand event storms and rate-control strategies.
- Understand multicasting and replay.
- Understand subject-like constructs and their trade-offs.
- Understand synchronous versus asynchronous emission.
- Understand reentrancy hazards.
- Understand error propagation through reactive pipelines.
- Understand completion semantics.
- Understand subscription leaks.
- Understand shared-resource ownership.
- Understand operator fusion as a conceptual optimization.
- Understand scheduler abstractions conceptually.
- Distinguish language-standard JavaScript features from library-provided Observable APIs.
- Understand that Observable behavior is not a universal ECMAScript language primitive.
- Implement a minimal Observable abstraction from scratch.
- Implement unsubscribe/teardown correctly.
- Implement basic operators.
- Implement subscription composition.
- Implement a bounded buffering strategy.
- Build a cancellation-aware reactive pipeline.
- Diagnose memory leaks caused by long-lived subscriptions.
- Diagnose event storms, dropped events, duplicated subscriptions, and stale subscriptions.
- Reason about ordering guarantees.
- Reason about concurrency inside reactive pipelines.
- Compare reactive approaches with Promises, async iterators, EventEmitters, Web Streams, and queues.
- Design production reactive flows for UI events, server events, telemetry, messaging, and live data.
- Choose reactive programming only when its temporal/compositional model adds value.
- Defend reactive architecture at senior/principal level.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

Required:

- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism

Strongly related:

- Chapter 22 — Arrays
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 45 — Memory / GC
- Chapter 53 — Web Streams / Data Flow
- Chapter 60 — Node Streams
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing

---

## 3. What Is It?

Reactive programming is a programming model organized around values, events, and state changes arriving over time.

An Observable is a commonly used abstraction for a potentially unbounded sequence of notifications:

```text
next(value)
next(value)
next(value)
...
error(error)
```

or:

```text
next(value)
next(value)
next(value)
...
complete()
```

A conceptual Observable has:

```text
producer
   ↓
Observable
   ↓
subscription
   ↓
observer
```

The observer can conceptually receive:

```text
next
error
complete
```

Unlike a Promise:

```text
Promise
   ↓
one eventual result
```

an Observable can represent:

```text
many values over time
```

Examples:

```text
mouse movements
keyboard input
WebSocket messages
market data
telemetry
application state changes
filesystem events
server-sent events
```

The abstraction is especially useful when the core problem is:

> “How should a program compose values that arrive over time?”

---

## 4. Why Does It Exist?

Callbacks become difficult to compose when many temporal operations interact:

```text
event
→ filter
→ debounce
→ map
→ asynchronous request
→ cancel stale request
→ combine with another stream
→ recover from failure
→ cleanup
```

Reactive abstractions provide a vocabulary for these operations.

Instead of manually wiring:

```js
source.addEventListener(...);
timer(...);
unsubscribe(...);
cancel(...);
```

a reactive pipeline can conceptually express:

```text
source
→ filter
→ debounce
→ map
→ switch-latest
→ observe
```

The central value is not that Observables make code “more async”.

The value is that they model **time-varying streams compositionally**.

---

## 5. Mental Model

Think of an Observable as:

```text
a recipe for producing a sequence
```

and a subscription as:

```text
an activation of that recipe
```

The distinction matters.

```text
Observable creation
        ↓
       idle
        ↓
   subscribe()
        ↓
    execution starts
        ↓
 next / error / complete
        ↓
     teardown
```

This commonly leads to the concept of a **cold Observable**.

A hot source is different:

```text
source exists independently
        ↓
events already happen
        ↓
subscriber attaches
```

Example:

```text
WebSocket
   ↓
shared event source
   ├── subscriber A
   ├── subscriber B
   └── subscriber C
```

A late subscriber may miss earlier values.

---

## 6. Core Rules

### Rule 1 — Observable is not a universal ECMAScript language primitive

Observable behavior is usually supplied by a library or host/application abstraction.

### Rule 2 — Promise and Observable model different temporal cardinalities

Promise:

```text
zero/one eventual settlement
```

Observable:

```text
zero/many notifications over time
```

### Rule 3 — Subscription activates the computation in lazy designs

Creating an Observable need not start execution.

### Rule 4 — Unsubscription is a lifetime boundary

A subscription should stop unnecessary work and release resources when possible.

### Rule 5 — Errors are part of the stream contract

Reactive pipelines need explicit error semantics.

### Rule 6 — Completion is different from cancellation

Completion:

```text
producer finished normally
```

Cancellation/unsubscription:

```text
subscriber no longer wants the work
```

### Rule 7 — A stream can be infinite

For example:

```text
clicks
WebSocket messages
telemetry
```

must not depend on natural completion.

### Rule 8 — Push sources can overwhelm consumers

Without buffering, dropping, sampling, throttling, or flow control, event rates can exceed processing capacity.

### Rule 9 — Multicasting changes ownership semantics

One producer shared by many subscribers differs from independent producer executions.

### Rule 10 — Replay changes what late subscribers observe

A replaying source may emit historical values to new subscribers.

### Rule 11 — Ordering must be explicit

Concurrent inner operations may complete out of order.

### Rule 12 — Flattening operators encode concurrency policy

Merge-like behavior permits overlap.

Concat-like behavior serializes.

Switch-like behavior favors the latest.

Exhaust-like behavior ignores new work while busy.

### Rule 13 — Operators should preserve understandable contracts

A custom operator that leaks subscriptions or changes error semantics silently is production-dangerous.

### Rule 14 — Reactive systems are lifecycle systems

Every long-lived subscription needs an ownership and teardown story.

### Rule 15 — More abstraction does not remove resource costs

Every subscription, buffer, timer, closure, and inner operation still consumes resources.

---

## 7. Syntax

Because Observables are generally library-level abstractions rather than one universal ECMAScript syntax, exact APIs vary.

A conceptual API:

```js
const source = new Observable(observer => {
  observer.next(1);
  observer.next(2);
  observer.complete();

  return () => {
    // teardown
  };
});

const subscription = source.subscribe({
  next(value) {
    console.log(value);
  },
  error(error) {
    console.error(error);
  },
  complete() {
    console.log("done");
  }
});

subscription.unsubscribe();
```

Conceptual operator composition:

```js
const result = source
  .pipe(
    map(value => transform(value)),
    filter(value => isValid(value)),
    debounce(...)
  );
```

Exact operator names and signatures depend on the Observable implementation.

---

## 8. Basic Examples

### Example 1 — Simple finite Observable

```js
const numbers = new Observable(observer => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
});

numbers.subscribe({
  next: value => console.log(value),
  complete: () => console.log("done")
});
```

Expected sequence:

```text
1
2
3
done
```

### Example 2 — Infinite event source

```js
const clicks = fromEvent(button, "click");

const subscription = clicks.subscribe({
  next: event => console.log(event)
});
```

Teardown:

```js
subscription.unsubscribe();
```

### Example 3 — Transformation

```text
source
  → map(x => x * 2)
  → filter(x => x > 10)
  → subscriber
```

### Example 4 — Debouncing

Input:

```text
a
ab
abc
abcd
```

Rapid emissions can be collapsed so that only a stabilized value is processed.

### Example 5 — Throttling

Input:

```text
1 2 3 4 5 6 7
```

A throttle can limit how frequently downstream work executes.

### Example 6 — Combining streams

Conceptually:

```text
user
+
settings
+
permissions
→ derived state
```

### Example 7 — Higher-order stream

```text
search query stream
       ↓
request Observable for each query
       ↓
flatten
```

The flattening policy determines whether:

- all requests remain active;
- requests queue;
- previous request is cancelled;
- new requests are ignored while busy.

---

## 9. Execution Walkthrough

Consider:

```text
clicks
→ map(extractId)
→ switchLatest(fetchData)
→ render
```

### Step 1

A click arrives.

```text
click #1
```

### Step 2

The ID is transformed.

```text
id = 10
```

### Step 3

A request Observable is created.

```text
request(10)
```

### Step 4

The inner request becomes active.

### Step 5

Another click arrives before request #1 finishes.

```text
click #2
id = 20
```

### Step 6

The latest-value switching policy unsubscribes the old inner subscription where the abstraction supports cancellation.

### Step 7

Request #2 becomes active.

### Step 8

Request #2 emits its response.

### Step 9

The response reaches:

```text
render
```

The important concept is:

> The flattening policy expresses concurrency and cancellation semantics.

A merge policy would allow both requests to remain active.

A concat policy would wait for request #1.

An exhaust policy could ignore click #2 while request #1 is active.

---

## 10. Internal Mechanics

### 10.1 Subscription

Conceptually:

```text
subscribe(observer)
```

creates an execution relationship:

```text
producer ↔ observer
```

and returns a handle that can terminate that relationship.

### 10.2 Observer

Conceptually:

```js
{
  next(value) {},
  error(error) {},
  complete() {}
}
```

### 10.3 Teardown

The producer may allocate:

```text
event listener
timer
socket
worker
resource
```

and return cleanup logic.

### 10.4 Subscription state

A robust subscription needs states such as:

```text
active
closed
```

and sometimes additional internal state.

### 10.5 Terminal notifications

A stream typically treats:

```text
error
complete
```

as terminal.

After termination:

```text
next
```

should no longer be delivered through that subscription.

### 10.6 Unsubscription

Unsubscription should prevent future downstream delivery and attempt to stop underlying work where the source supports it.

### 10.7 Cold execution

A simple model:

```text
subscribe A → producer instance A
subscribe B → producer instance B
```

Each subscriber may get an independent producer execution.

### 10.8 Hot execution

A shared producer:

```text
producer
   ↓
shared source
   ├── A
   ├── B
   └── C
```

### 10.9 Multicast

The producer executes once while many observers receive values.

### 10.10 Replay

A replay-capable subject/source may retain:

```text
latest value
or
N historical values
```

for future subscribers.

This creates memory and lifecycle implications.

### 10.11 Operator composition

An operator typically creates a new Observable:

```text
source
 ↓
operator
 ↓
new Observable
```

The operator subscribes upstream and forwards transformed notifications downstream.

### 10.12 Operator chains

For:

```text
A → map → filter → debounce → B
```

one can model:

```text
B subscribes
→ debounce subscribes
→ filter subscribes
→ map subscribes
→ source subscribes
```

The subscription often propagates backwards through the chain.

### 10.13 Teardown propagation

Unsubscribing downstream can propagate upstream:

```text
subscriber unsubscribes
       ↓
operator teardown
       ↓
upstream unsubscribe
       ↓
producer cleanup
```

This is essential for resource safety.

### 10.14 Push model

Producer decides when values arrive:

```text
producer → consumer
```

### 10.15 Pull model

Consumer requests the next value:

```text
consumer → producer
```

Async iterators are more naturally pull-oriented.

### 10.16 Push-pull hybrids

A reactive system can combine:

```text
push notifications
+
buffer
+
demand/capacity control
```

to approximate backpressure.

### 10.17 Buffer

A buffer stores values temporarily:

```text
producer → [queue] → consumer
```

### 10.18 Buffer overflow policies

Possible policies:

```text
block producer
drop newest
drop oldest
drop all but latest
fail
sample
```

The correct policy depends on semantics.

### 10.19 Rate control

Operators such as debounce/throttle/sample conceptually alter the temporal shape of a stream.

### 10.20 Debounce

Keep waiting for inactivity:

```text
A
AB
ABC
ABCD
   ↓ quiet period
ABCD
```

### 10.21 Throttle

Allow at most one event within a time window.

### 10.22 Sample

Observe only the latest value at selected intervals.

### 10.23 Windowing

Group values by:

```text
count
time
other boundaries
```

### 10.24 Flattening

Suppose:

```text
source emits A, B, C
```

and each creates an inner Observable.

Flattening determines how those inner streams coexist.

### 10.25 Merge-style

```text
A ───────
B ────
C ──────────
```

all can overlap.

### 10.26 Concat-style

```text
A complete
    ↓
B complete
    ↓
C
```

strict ordering.

### 10.27 Switch-style

```text
A starts
B arrives
→ stop/ignore A where cancellation is supported
B becomes current
```

### 10.28 Exhaust-style

```text
A active
B arrives → ignored
C arrives → ignored
A completes
D arrives → accepted
```

### 10.29 Synchronous emission

An Observable may emit immediately during subscription:

```js
subscribe();
```

and call:

```text
next
```

before `subscribe()` returns.

This creates reentrancy considerations.

### 10.30 Asynchronous emission

Other sources schedule work:

```text
timer
network
event
queue
worker
```

### 10.31 Scheduler abstraction

Libraries may provide scheduler concepts to control when work is executed.

Possible categories include:

```text
immediate
queued
microtask-like
timer-like
animation-like
```

Exact behavior is library/host-specific.

### 10.32 Reentrancy

A subscriber may synchronously trigger another event:

```text
next(A)
 → subscriber
 → source emits B
 → subscriber
```

Without careful design, state assumptions may break.

### 10.33 Error propagation

A pipeline may choose:

```text
error → terminate stream
```

or recover using an alternate source.

Recovery must be explicit.

### 10.34 Completion propagation

Finite upstream completion often propagates downstream after pending work according to operator semantics.

### 10.35 Subscription leaks

Long-lived sources can retain:

```text
observer
closures
DOM nodes
application state
```

if subscriptions are never torn down.

### 10.36 Shared source lifecycle

A shared connection may need:

```text
first subscriber → connect
last subscriber  → disconnect
```

This is often called reference-count-style lifecycle management conceptually.

### 10.37 Resource ownership

The application should know:

```text
who owns this subscription?
when should it end?
what resource does it hold?
```

### 10.38 Idempotent teardown

Calling cleanup more than once should ideally be safe.

### 10.39 Operator implementation invariants

A correct operator should preserve:

- terminal semantics;
- teardown;
- ordering where promised;
- error behavior;
- subscription lifetime.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Observable is not part of core historical Promise semantics

Do not confuse a library's Observable abstraction with:

```text
Promise
async function
await
```

which have ECMAScript language-level semantics.

### 11.2 No universal Observable execution model in ECMAScript

There is no single language-defined rule that says:

```text
all Observables are lazy
```

or:

```text
all Observables are asynchronous
```

or:

```text
all Observables have a standard scheduler
```

These are characteristics of particular libraries/designs.

### 11.3 Events and hosts

Browsers and Node expose event-oriented mechanisms that can be adapted into reactive abstractions.

### 11.4 Async iterators

Async iterators provide a language-level protocol for asynchronous pull-style iteration:

```text
next() → Promise<IteratorResult>
```

Observables are conceptually more push-oriented.

### 11.5 Jobs and scheduling

Observable callbacks ultimately execute through the host/language scheduling mechanisms used by the implementation.

Do not assume a particular library operator always maps to one ECMAScript Job.

### 11.6 Cancellation

Unsubscription is an Observable abstraction.

It can be connected to:

```text
AbortSignal
```

or other cancellation systems, but there is no universal language rule making every Observable automatically abortable.

---

## 12. Advanced Behavior

### 12.1 Cold vs hot is not binary in all systems

A source may be:

```text
cold by default
shared after transformation
```

### 12.2 Unicast vs multicast

Unicast:

```text
subscriber A → own execution
subscriber B → own execution
```

Multicast:

```text
one execution → A + B
```

### 12.3 State versus events

A stream of events:

```text
increment
increment
decrement
```

is different from a stream of state snapshots:

```text
1
2
3
```

Both can be modeled reactively, but semantics differ.

### 12.4 Replay versus current state

Replay buffers historical values.

A current-state abstraction typically needs a clear definition of what a new subscriber should receive.

### 12.5 Hot source loss

A late subscriber may miss events:

```text
event 1
event 2
subscribe
event 3
```

### 12.6 Multicast race

Subscriber A and B may attach at slightly different times and observe different event sets.

### 12.7 Shared errors

A shared source may terminate or recover in ways that affect every subscriber.

### 12.8 Sharing and side effects

Consider:

```js
source.map(saveToDatabase)
```

With cold independent subscriptions:

```text
subscribe A → save
subscribe B → save
```

the side effect may happen twice.

### 12.9 Referential transparency

Reactive composition is easier to reason about when operators are pure transformations.

### 12.10 Side-effect isolation

Keep side effects at clear boundaries:

```text
source
→ pure transformations
→ side effect boundary
```

### 12.11 Flattening and resource ownership

If each outer value creates a network request:

```text
outer stream
→ request stream
```

flattening controls which requests remain alive.

### 12.12 Merge concurrency limit

Merge-style behavior can be bounded:

```text
max inner streams = N
```

This connects directly to Chapter 39.

### 12.13 Concat queue growth

Concat semantics preserve ordering but can accumulate waiting inner work if outer arrivals are faster than completions.

### 12.14 Switch cancellation race

A previous request may still finish at the physical/network layer even after downstream unsubscription.

The important invariant is:

```text
stale result must not update current state
```

### 12.15 Exhaust starvation

An endless or very slow active stream can prevent later work from ever being accepted.

### 12.16 Backpressure gap

A push producer may not honor consumer demand.

Therefore:

```text
producer rate > consumer capacity
```

requires a policy.

### 12.17 Buffer memory

An unbounded buffer converts overload into memory growth.

### 12.18 Sampling versus correctness

Dropping values is correct only when intermediate observations are nonessential.

For financial ledger events:

```text
drop events
```

may be unacceptable.

For pointer movement:

```text
sampling
```

may be entirely appropriate.

### 12.19 Debounce semantic loss

Debounce intentionally discards intermediate activity.

### 12.20 Ordering

A reactive pipeline should state:

```text
Does downstream observe source order?
```

Some concurrency operators preserve order, some do not.

### 12.21 Time semantics

Timers and scheduler timing are host-dependent.

Do not treat a debounce duration as a precise real-time guarantee.

### 12.22 Reentrancy versus serialization

A synchronous source may cause nested subscriber execution.

A queued scheduler can flatten that behavior.

### 12.23 Recursive emissions

```text
subscriber
→ causes source emission
→ subscriber
→ causes source emission
```

can create deep recursion or infinite feedback loops.

### 12.24 Feedback systems

Reactive state can form:

```text
output
 ↓
feeds source
 ↓
new output
```

Feedback requires loop-breaking or convergence conditions.

### 12.25 Infinite streams

Operators like:

```text
buffer
reduce
toArray
```

can never complete on an infinite source unless explicitly bounded.

### 12.26 Memory retention through replay

Replay size and lifetime must be controlled.

### 12.27 Error recovery loops

A recovery operator can accidentally create:

```text
error
→ fallback
→ same failing source
→ error
→ ...
```

### 12.28 Retry storms

Retrying every subscriber independently can multiply downstream load.

### 12.29 Per-subscriber side effects

A cold source with expensive setup can multiply work as subscriptions increase.

### 12.30 Shared connection

A multicast WebSocket can reduce duplicate network connections.

But now:

```text
one connection
shared ownership
shared failure semantics
```

must be designed.

### 12.31 Reference counting

Conceptually:

```text
0 subscribers → disconnected
1 subscriber  → connected
2 subscribers → same connection
1 subscriber  → remain connected
0 subscribers → disconnected
```

### 12.32 Buffering during reconnect

A reconnecting stream must define whether messages are:

```text
dropped
buffered
replayed
recovered from durable source
```

### 12.33 Duplicate delivery

Reconnects can cause duplicate events.

Consumers may need idempotency.

### 12.34 Ordering across reconnects

Network reconnect may break global ordering guarantees.

### 12.35 Exactly-once misconceptions

Reactive libraries cannot automatically create distributed exactly-once semantics.

---

## 13. Edge Cases

### 13.1 Subscriber throws

An implementation must define how a thrown observer callback is handled.

### 13.2 Producer throws during subscription

The subscription should transition to a safe terminal/failure state.

### 13.3 `next` after complete

Should not be delivered to a closed subscription.

### 13.4 `next` after error

Should not be delivered.

### 13.5 Complete twice

Terminal notifications should be effectively one-shot per subscription.

### 13.6 Error twice

Second terminal notification should not create another terminal state.

### 13.7 Unsubscribe before first emission

No post-unsubscribe values should reach the subscriber.

### 13.8 Unsubscribe during `next`

The pipeline must safely transition without leaking further notifications.

### 13.9 Synchronous completion

A stream may emit all values and complete before `subscribe()` returns.

### 13.10 Synchronous error

Errors may also occur immediately during subscription.

### 13.11 Reentrant unsubscribe

A subscriber may unsubscribe itself inside `next`.

### 13.12 Reentrant subscribe

A subscriber may subscribe another observer from inside a callback.

### 13.13 Infinite replay buffer

A replay implementation without size/time limits can leak memory indefinitely.

### 13.14 Infinite stream + accumulation

A global array collecting every event becomes an unbounded memory consumer.

### 13.15 Multiple subscriptions

One source may accidentally execute expensive side effects once per subscriber.

### 13.16 Duplicate event handling

A component may subscribe repeatedly without disposing older subscriptions.

### 13.17 Stale subscription

A UI view can continue receiving events after the view is gone.

### 13.18 Async race

Two inner streams may complete out of order.

### 13.19 Slow consumer

A push source can outpace downstream processing.

### 13.20 Slow producer

Combining or scheduling logic must not assume constant producer rate.

### 13.21 Clock/timer drift

Time-based operators rely on runtime scheduling behavior.

### 13.22 Cancellation race

Unsubscribe may occur immediately before a producer emits.

### 13.23 Cleanup race

Resource cleanup may overlap with completion/error paths.

### 13.24 Network reconnect

Reconnect logic may create duplicate connections if old connections are not fully torn down.

### 13.25 Shared error

One subscriber's assumptions may conflict with global source termination.

---

## 14. Common Misconceptions

### Misconception 1 — “Observable is just a Promise with multiple values.”

No.

The subscription, cancellation, completion, hot/cold, push, and multicasting models are materially different.

### Misconception 2 — “Every Observable is asynchronous.”

No.

An Observable can emit synchronously.

### Misconception 3 — “Every Observable is lazy.”

No.

That depends on the abstraction.

### Misconception 4 — “Unsubscribe physically stops every underlying operation.”

Not necessarily.

It depends on whether the source can cancel the underlying work.

### Misconception 5 — “All reactive libraries implement operators identically.”

No.

API names and subtle semantics vary.

### Misconception 6 — “Reactive automatically means better architecture.”

No.

It can increase conceptual and debugging complexity.

### Misconception 7 — “Push means no backpressure problem.”

The opposite may be true.

### Misconception 8 — “Replay is free.”

Replay retains memory and may increase startup work.

### Misconception 9 — “Switch cancellation guarantees the HTTP request is physically cancelled.”

Not automatically.

It guarantees the downstream subscription stops receiving the old result according to the implementation's semantics.

### Misconception 10 — “Merge means parallel CPU execution.”

No.

It means concurrent inner streams at the abstraction level.

### Misconception 11 — “Completion and cancellation are the same.”

No.

### Misconception 12 — “A shared Observable always executes once globally.”

Sharing scope and lifecycle are implementation-specific.

### Misconception 13 — “Reactive code has no races.”

Reactive code can contain concurrency and ordering races.

### Misconception 14 — “A stream can always be converted to an array.”

Infinite streams make this impossible.

### Misconception 15 — “Debounce only changes performance.”

It changes semantics by discarding intermediate events.

---

## 15. Common Mistakes

### Mistake 1 — Forgetting to unsubscribe long-lived sources

### Mistake 2 — Creating duplicate subscriptions

### Mistake 3 — Using merge-style flattening for work that must remain ordered

### Mistake 4 — Using concat-style flattening when latency matters more than strict order

### Mistake 5 — Using switch-style cancellation where every event must be processed

### Mistake 6 — Using exhaust-style behavior where events cannot be dropped

### Mistake 7 — Unbounded buffering

### Mistake 8 — Unbounded retry

### Mistake 9 — Ignoring downstream capacity

### Mistake 10 — Assuming cancellation undoes side effects

### Mistake 11 — Hiding side effects inside reusable operators

### Mistake 12 — Sharing a source without documenting lifecycle

### Mistake 13 — Replay buffers without bounds

### Mistake 14 — Ignoring reentrancy

### Mistake 15 — Treating timing operators as precise clocks

### Mistake 16 — Converting infinite streams into finite collections without bounds

### Mistake 17 — Building custom operators without teardown propagation

### Mistake 18 — Mixing push and pull semantics without a clear contract

### Mistake 19 — Ignoring error ownership in shared streams

### Mistake 20 — Choosing reactive programming merely because it is fashionable

---

## 16. Comparison With Related Concepts

| Concept | Cardinality | Direction | Cancellation | Typical shape |
|---|---|---|---|---|
| Promise | One settlement | Push/eventual | External/cooperative | One result |
| Observable | Zero/many over time | Push-oriented | Unsubscribe | Stream |
| Async Iterator | Zero/many | Pull-oriented | `return()`/external signal patterns | Sequential consumption |
| EventEmitter | Many events | Push | Listener removal | Event notifications |
| Web Stream | Chunks | Pull/push hybrid | Abort/cancel | Data flow |
| Queue | Many work items | Producer/consumer | Policy-dependent | Work buffering |
| Callback | Usually one invocation | Push | Ad hoc | Function notification |

### Observable vs Promise

Use Promise when:

```text
one result
one completion
```

Use Observable when:

```text
many values
ongoing lifetime
temporal operators
cancellation
```

### Observable vs Async Iterator

Observable:

```text
producer pushes
```

Async Iterator:

```text
consumer pulls
```

### Observable vs EventEmitter

EventEmitter provides event dispatch.

An Observable abstraction adds compositional semantics such as:

```text
map
filter
flatten
combine
buffer
retry
```

### Observable vs Web Streams

Web Streams are designed around streaming data and explicit stream lifecycle/backpressure concerns.

Observables are often more general event/value composition abstractions.

### Observable vs Queue

Queue semantics focus on work transfer and buffering.

Reactive semantics focus on value/event composition over time.

### Hot vs Cold

```text
cold:
each subscriber activates its own producer

hot:
producer exists independently of subscriber
```

---

## 17. Performance Considerations

### 17.1 Operator overhead

Each abstraction layer may introduce:

- closures;
- subscriptions;
- allocations;
- function calls;
- scheduling.

### 17.2 Operator fusion

Some implementations can combine operations to reduce intermediate overhead.

Conceptually:

```text
map
+
filter
+
map
```

may be optimized into fewer execution steps.

Do not assume fusion exists unless documented/measured.

### 17.3 Allocation pressure

High-frequency streams can allocate per event:

```text
event object
closure
wrapper
operator state
```

### 17.4 Event storms

A source emitting:

```text
100,000 events/sec
```

can overwhelm downstream processing.

### 17.5 Debounce/throttle/sample

These operators can reduce downstream work but intentionally change event semantics.

### 17.6 Buffering

Buffering absorbs bursts but trades:

```text
memory
+
latency
```

for stability.

### 17.7 Merge concurrency

Unbounded inner concurrency can recreate the problems from Chapter 39.

### 17.8 Scheduler overhead

Repeated task scheduling can itself become significant at very high event rates.

### 17.9 Multicast efficiency

Sharing one producer can eliminate duplicate work:

```text
one network connection
instead of
N connections
```

### 17.10 Replay cost

Replay increases memory usage and may create bursts when subscribers join.

### 17.11 Context switching

Frequent scheduling can increase overhead.

### 17.12 Latency

Additional buffering and scheduling can increase end-to-end latency.

### 17.13 CPU-heavy operators

Reactive composition does not make CPU-heavy transformations parallel.

Use workers/parallel execution where justified.

### 17.14 Batching

Batching events can reduce per-event overhead.

### 17.15 Backpressure

Unbounded push should be treated as a potential performance hazard.

### 17.16 Measure

Useful metrics:

```text
events/sec
subscription count
active inner streams
buffer depth
dropped events
processing latency
P95/P99 latency
CPU
memory
retry rate
error rate
```

---

## 18. Memory Considerations

### 18.1 Subscription retention

A subscription may retain:

```text
observer
closure
DOM nodes
application state
```

### 18.2 Replay

Replay buffers retain prior values.

### 18.3 Buffers

Queues retain every pending event until consumed/dropped.

### 18.4 Infinite accumulation

Avoid:

```js
const values = [];

source.subscribe(value => {
  values.push(value);
});
```

for unbounded streams.

### 18.5 Shared state retention

A shared stream can accidentally retain a large graph through its observers.

### 18.6 Closure capture

Operators can capture:

```text
request
component
configuration
cache
```

for the lifetime of the subscription.

### 18.7 Inner stream retention

Flattening operators can hold active inner subscriptions.

### 18.8 Reconnect loops

Reconnect timers can accumulate if old attempts are not cleaned up.

### 18.9 Resource lifetime

Always connect:

```text
subscription lifetime
→ resource lifetime
```

### 18.10 Memory budget

A useful model:

```text
replay memory
+
buffer memory
+
active subscription state
+
inner task memory
+
runtime overhead
≤ safe budget
```

---

## 19. Security Considerations

### 19.1 Event-flood attacks

Untrusted event sources can overwhelm processing.

### 19.2 Memory exhaustion

Unbounded buffering/replay can become denial-of-service vectors.

### 19.3 Subscription amplification

One malicious input may create many subscriptions or downstream operations.

### 19.4 Fan-out amplification

A single event can trigger:

```text
N downstream requests
```

Bound it.

### 19.5 Retry amplification

Reactive retry logic can intensify dependency overload.

### 19.6 Shared state leaks

A multicast stream can accidentally expose values to subscribers that should not receive them.

### 19.7 Authorization changes

Long-lived subscriptions should respond correctly when permissions change.

### 19.8 Stale data

A subscription may continue showing data after its authorization scope expires.

### 19.9 Cross-tenant mixing

Shared streams must preserve tenant isolation.

### 19.10 Resource exhaustion through subscriptions

Attackers may create many long-lived subscriptions.

Apply:

```text
connection limits
subscription limits
timeouts
quotas
```

### 19.11 Error information

Shared error channels should avoid exposing sensitive internal details.

### 19.12 Untrusted operators

Dynamic or user-provided reactive definitions require strong validation.

---

## 20. Production Usage

### 20.1 UI search

A common pattern:

```text
keystrokes
→ debounce
→ validate
→ request
→ switch latest
→ render
```

This is appropriate when stale requests should no longer update the UI.

### 20.2 Live dashboards

```text
WebSocket
→ parse
→ validate
→ aggregate
→ throttle
→ render
```

### 20.3 Telemetry

```text
events
→ batch
→ enrich
→ send
```

### 20.4 Server events

```text
client connections
→ event stream
→ authorization
→ filtering
→ broadcast
```

### 20.5 Real-time collaboration

Reactive streams can represent:

```text
remote edits
presence
cursor movement
notifications
```

### 20.6 Event-driven backend

```text
message source
→ validation
→ transformation
→ routing
→ side effects
```

### 20.7 Monitoring

Metrics can themselves be reactive:

```text
metric events
→ windows
→ aggregation
→ alerts
```

### 20.8 UI lifecycle

Associate subscriptions with component/page lifecycle:

```text
mount
→ subscribe

unmount
→ unsubscribe
```

### 20.9 Database/change streams

Reactive abstractions can model change notifications from database or messaging sources, but durability and delivery guarantees belong to the underlying system.

### 20.10 WebSocket management

Define:

```text
connect
reconnect
backoff
heartbeat
disconnect
authorization
duplicate handling
```

### 20.11 Production retry

Prefer:

```text
bounded retry
+
backoff
+
jitter
+
cancellation
```

### 20.12 Production buffering

Always define:

```text
maximum buffer
overflow policy
drop semantics
shutdown behavior
```

### 20.13 Observability

Expose:

```text
active subscriptions
subscription age
source rates
buffer depth
dropped events
inner concurrency
errors
completion counts
```

### 20.14 Testing

Test:

```text
ordering
cancellation
completion
errors
timeouts
retries
buffer overflow
subscription cleanup
```

### 20.15 Architecture boundary

Use reactive abstractions at temporal/event boundaries.

Do not force every internal business function into stream form.

---

## 21. Implementation From Scratch

### Stage 1 — Guided minimal Observable

Implement:

```js
class Observable {
  constructor(subscribe) {
    this._subscribe = subscribe;
  }

  subscribe(observer) {
    // normalize observer
    // call producer
    // return subscription
  }
}
```

Subscription requirements:

```js
{
  unsubscribe(),
  get closed() {}
}
```

### Stage 2 — Terminal-state handling

Enforce:

```text
active
→ complete

active
→ error

active
→ unsubscribe
```

and reject repeated terminal delivery.

### Stage 3 — Teardown composition

Support:

```text
function teardown
+
subscription teardown
+
nested teardown
```

Ensure cleanup runs exactly once.

### Stage 4 — Operators

Implement:

```text
map
filter
take
tap
```

Requirements:

- correct forwarding;
- correct error propagation;
- correct completion;
- correct teardown.

### Stage 5 — Time operators

Implement conceptually:

```text
debounce
throttle
```

Use explicit timer cleanup.

### Stage 6 — Flattening

Implement:

```text
mergeMap-like
concatMap-like
switchMap-like
exhaustMap-like
```

Start with one active inner stream, then generalize.

### Stage 7 — Bounded concurrency

Extend merge-style flattening:

```text
maxConcurrency = N
```

This connects directly to Chapter 39.

### Stage 8 — Multicast

Implement a basic shared source:

```text
one producer
many observers
```

### Stage 9 — Replay

Implement bounded replay:

```text
bufferSize = N
```

Never allow unbounded history accidentally.

### Stage 10 — Production Grade

Add:

- cancellation;
- error isolation;
- teardown composition;
- bounded buffers;
- metrics;
- concurrency limits;
- reconnect;
- backoff;
- fairness;
- lifecycle management;
- deterministic tests.

---

## 22. Debugging Exercises

### Exercise 1 — Subscription leak

A UI subscribes every time it mounts but never unsubscribes.

Find:

```text
duplicate updates
memory retention
duplicate network calls
```

### Exercise 2 — Wrong flattening

A search input uses merge-style request handling and displays stale responses.

Replace it with a latest-value cancellation strategy.

### Exercise 3 — Dropped events

A system uses exhaust-style behavior for payment events.

Determine why this is unsafe.

### Exercise 4 — Queue explosion

A push source emits faster than the consumer can process.

Measure:

```text
producer rate
consumer rate
buffer depth
memory
```

### Exercise 5 — Replay leak

A replay buffer retains every event forever.

Add a bounded policy.

### Exercise 6 — Shared side effect duplication

A cold source sends a notification every time it is subscribed.

Find why three subscribers create three network operations.

### Exercise 7 — Reentrancy

Create a synchronous Observable whose subscriber causes another emission.

Observe nested execution and make the behavior safe.

### Exercise 8 — Error swallowing

An operator catches an error and returns an empty source.

Determine whether critical failures are being hidden.

### Exercise 9 — Retry storm

Several subscribers independently retry a failing service.

Design shared retry policy with bounded concurrency.

### Exercise 10 — Cleanup race

Unsubscribe during a timer callback and verify no leaked timer remains.

---

## 23. Code Review Exercise

Review conceptually:

```js
function searchStream(input) {
  return fromEvent(input, "input").pipe(
    map(event => event.target.value),
    debounce(300),
    map(query => fetch(`/search?q=${query}`)),
    mergeAll()
  );
}
```

Identify:

- lack of validation;
- potential request overlap;
- stale-response risk;
- unbounded inner concurrency;
- cancellation policy;
- error handling;
- timeout;
- resource cleanup;
- observability;
- rate limiting.

Redesign the pipeline for:

```text
latest-query semantics
+
bounded input rate
+
cancellation
+
timeout
+
error handling
```

Then explain why a different flattening strategy would be required if every submitted query must be processed.

---

## 24. Interview Questions

### Foundational

1. What is reactive programming?
2. What is an Observable?
3. How is an Observable different from a Promise?
4. How is an Observable different from an async iterator?
5. What is push versus pull?
6. What are `next`, `error`, and `complete`?
7. What is a subscription?
8. What is teardown?
9. What is a cold Observable?
10. What is a hot Observable?

### Intermediate

11. What is multicasting?
12. What is replay?
13. What is backpressure?
14. What does debounce do?
15. What does throttle do?
16. What is flattening?
17. Compare merge, concat, switch, and exhaust semantics.
18. Why are subscription leaks dangerous?
19. What is reentrancy?
20. Why does cancellation matter?

### Advanced

21. How would you implement Observable from scratch?
22. How do you propagate teardown through operators?
23. How do you bound inner concurrency?
24. How do you handle a slow consumer?
25. How do you prevent replay-buffer memory leaks?
26. How do you reason about hot versus cold sources?
27. How do you design reconnect logic?
28. How do retries amplify load?
29. How do shared streams affect error semantics?
30. How do synchronous emissions create reentrancy hazards?

### Principal-Level

31. When should a system use Observables rather than async iterators?
32. When should a system use Promises instead?
33. How would you model millions of event notifications safely?
34. How would you design per-tenant reactive isolation?
35. How would you design bounded backpressure for a push source?
36. How would you guarantee stale UI requests cannot mutate state?
37. How would you design a shared WebSocket stream?
38. How would you define replay policy for a production system?
39. How would you debug a subscription leak in production?
40. How would you decide whether reactive abstraction reduces or increases system complexity?

---

## 25. Predict-the-Output Exercises

### Exercise A — Synchronous emission

```js
console.log("before");

const source = new Observable(observer => {
  observer.next(1);
  observer.next(2);
  observer.complete();
});

source.subscribe({
  next(value) {
    console.log(value);
  },
  complete() {
    console.log("complete");
  }
});

console.log("after");
```

Predict the ordering.

### Exercise B — Unsubscribe

```js
const subscription = source.subscribe({
  next(value) {
    console.log(value);
  }
});

subscription.unsubscribe();
```

Reason about which future notifications should be observed.

### Exercise C — Cold source

```js
const source = new Observable(observer => {
  console.log("producer");
  observer.next(Math.random());
});

source.subscribe(log);
source.subscribe(log);
```

How many producer executions occur in a simple cold design?

### Exercise D — Hot source

Conceptually compare:

```text
one producer
+
two subscribers
```

against two independent subscriptions.

### Exercise E — Merge semantics

```text
outer: A, B

A inner: --A--
B inner: -B-
```

Predict the possible merged sequence.

### Exercise F — Concat semantics

Same streams as Exercise E.

Predict the ordering.

### Exercise G — Switch semantics

```text
A starts
B starts before A completes
```

Determine which result can reach the downstream subscriber under latest-value cancellation semantics.

### Exercise H — Exhaust semantics

```text
A starts
B arrives
A completes
C arrives
```

Determine which values are accepted.

---

## 26. Mastery Exercises

### Exercise 1 — Minimal Observable

Build:

```js
Observable
Subscription
Observer
```

with correct terminal semantics.

### Exercise 2 — Operator library

Implement:

```text
map
filter
tap
take
```

### Exercise 3 — Subscription composition

Ensure:

```text
downstream unsubscribe
→ all upstream teardowns
```

### Exercise 4 — Timer operators

Implement:

```text
debounce
throttle
sample
```

### Exercise 5 — Flattening operators

Implement:

```text
merge
concat
switch
exhaust
```

### Exercise 6 — Bounded flattening

Add:

```text
maxConcurrency
```

### Exercise 7 — Hot shared source

Implement:

```text
multicast
```

with:

```text
connect
disconnect
subscriber management
```

### Exercise 8 — Replay

Implement:

```text
replay(N)
```

and enforce:

```text
bounded memory
```

### Exercise 9 — Backpressure experiment

Create a producer faster than a consumer.

Measure:

```text
queue depth
memory
latency
drop count
```

Compare:

```text
unbounded
drop-oldest
drop-newest
sample-latest
```

### Exercise 10 — Production reactive system

Build:

```text
event source
→ validation
→ transformation
→ bounded concurrency
→ retry/backoff
→ timeout
→ metrics
→ graceful teardown
```

Document:

- cardinality;
- ownership;
- ordering;
- failure;
- cancellation;
- backpressure;
- resource limits.

---

## 27. Key Takeaways

1. Reactive programming models values and events over time.
2. Observables commonly represent zero, one, or many notifications across a subscription lifetime.
3. Observable abstractions are generally library/application level rather than one universal ECMAScript primitive.
4. Promises and Observables solve different temporal problems.
5. Async iterators are naturally pull-oriented; Observables are often push-oriented.
6. Subscriptions establish execution and ownership relationships.
7. Teardown is essential for resource safety.
8. Cold sources may create independent producer executions per subscriber.
9. Hot sources can exist independently of individual subscribers.
10. Multicasting can avoid duplicate producer work.
11. Replay trades memory for late-subscriber visibility.
12. `next`, `error`, and `complete` represent distinct notification channels.
13. Completion is not the same as cancellation.
14. Push systems can overwhelm consumers.
15. Buffers must have explicit memory and overflow policies.
16. Debounce/throttle/sample alter temporal semantics, not merely performance.
17. Flattening operators encode concurrency and ordering policy.
18. Merge permits overlap, concat preserves sequential ordering, switch favors the latest, and exhaust ignores new work while active.
19. Unbounded reactive concurrency recreates the problems from Chapter 39.
20. Reactive systems can have races and reentrancy hazards.
21. Long-lived subscriptions can retain memory and resources.
22. Reactive abstraction should be used when temporal composition genuinely improves the design.
23. The central principle is:

> Reactive programming is a model for composing values, events, time, cancellation, and resource lifetime—not merely a different spelling of asynchronous JavaScript.

---

## 28. Concept Connections

### Depends On

- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism

### Builds Toward

- Chapter 40 — Observables / Reactive Programming
- Chapter 45 — Memory / GC
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 58 — Node Architecture
- Chapter 60 — Node Streams
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 106 — Real-time WebSocket
- Chapter 107 — Job Queue
- Chapter 109 — Event-driven App
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

### Related Concepts

- Promises
- Async iterators
- EventEmitter
- Web Streams
- Push / Pull
- Hot / Cold
- Unicast / Multicast
- Replay
- Subscription
- Teardown
- Backpressure
- Buffering
- Debounce
- Throttle
- Sampling
- Windowing
- Flattening
- Merge
- Concat
- Switch
- Exhaust
- Scheduling
- Cancellation
- Reentrancy
- Retry
- Timeouts
- Resource lifetime

### Concepts Revisited

This chapter revisits:

- asynchronous execution;
- Promise semantics;
- cancellation;
- async iteration;
- streaming;
- concurrency;
- backpressure;
- error handling;
- resource cleanup.

### Why This Chapter Matters Later

Reactive programming connects asynchronous JavaScript to systems where values do not arrive once, but continuously:

```text
events
signals
messages
telemetry
UI interaction
network updates
state changes
```

It provides a vocabulary for:

```text
what happens over time
+
what happens when another value arrives
+
what work is cancelled
+
what work overlaps
+
what resources remain alive
```

Those questions are central to real-time systems, browser applications, event-driven backends, observability pipelines, and distributed architectures.

The key engineering discipline is:

> Model time explicitly, then make concurrency, cancellation, ordering, buffering, and ownership explicit.

---

## 29. Completion Criteria

Mark Chapter 40 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Define reactive programming.
- [ ] Define Observable.
- [ ] Compare Observable and Promise.
- [ ] Compare Observable and async iterator.
- [ ] Explain push vs pull.
- [ ] Explain subscription.
- [ ] Explain teardown.
- [ ] Explain cold and hot sources.
- [ ] Explain unicast and multicast.
- [ ] Explain replay.
- [ ] Explain backpressure.
- [ ] Explain buffering.
- [ ] Explain flattening.
- [ ] Explain merge/concat/switch/exhaust semantics.
- [ ] Explain reentrancy.
- [ ] Explain completion vs cancellation.

### Predictive Mastery

- [ ] Predict synchronous emissions.
- [ ] Predict teardown after unsubscribe.
- [ ] Predict cold-source behavior.
- [ ] Predict multicast behavior.
- [ ] Predict merge ordering.
- [ ] Predict concat ordering.
- [ ] Predict switch cancellation.
- [ ] Predict exhaust dropping.
- [ ] Predict queue/buffer growth.
- [ ] Predict retry amplification.

### Implementation

- [ ] Implement Observable.
- [ ] Implement Subscription.
- [ ] Implement terminal-state handling.
- [ ] Implement teardown composition.
- [ ] Implement map.
- [ ] Implement filter.
- [ ] Implement take.
- [ ] Implement debounce.
- [ ] Implement throttle.
- [ ] Implement flattening.
- [ ] Implement bounded concurrency.
- [ ] Implement multicast.
- [ ] Implement bounded replay.
- [ ] Implement cancellation-aware production cleanup.

### Debugging

- [ ] Diagnose subscription leaks.
- [ ] Diagnose duplicate subscriptions.
- [ ] Diagnose stale-response bugs.
- [ ] Diagnose event storms.
- [ ] Diagnose queue growth.
- [ ] Diagnose replay memory leaks.
- [ ] Diagnose reentrancy.
- [ ] Diagnose retry storms.
- [ ] Diagnose reconnect duplication.
- [ ] Diagnose missing teardown.

### Production Engineering

- [ ] Define subscription ownership.
- [ ] Define cancellation policy.
- [ ] Define completion/error policy.
- [ ] Define buffering policy.
- [ ] Define overflow policy.
- [ ] Define concurrency limits.
- [ ] Define retry/timeout behavior.
- [ ] Define replay limits.
- [ ] Define tenant isolation.
- [ ] Define lifecycle/shutdown.
- [ ] Define observability.

### Interview Readiness

- [ ] Explain Observable vs Promise.
- [ ] Explain push vs pull.
- [ ] Explain cold vs hot.
- [ ] Explain subscription and teardown.
- [ ] Explain flattening strategies.
- [ ] Implement a minimal Observable.
- [ ] Design backpressure policy.
- [ ] Diagnose reactive leaks/races.
- [ ] Choose reactive vs Promise/async iterator/EventEmitter/Web Streams.
- [ ] Defend reactive architecture at principal level.

### Track A — Core Theory

- [ ] Understand Observable semantics.
- [ ] Understand temporal composition.
- [ ] Understand push/pull.
- [ ] Understand subscription lifecycle.
- [ ] Understand concurrency and flattening.
- [ ] Understand backpressure.
- [ ] Understand hot/cold/multicast/replay.
- [ ] Understand reactive failure modes.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented reactive pipeline reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed reactive debugging.
- [ ] Completed code review.
- [ ] Completed reactive architecture comparison.
- [ ] Completed backpressure design.
- [ ] Completed subscription-lifecycle design.

### Mastery Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Do not mark `[*] Mastered` until the learner can independently:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# Chapter 40 — Revision / Retrieval Record

### Retrieval Prompts

1. What is reactive programming?
2. What is an Observable?
3. How is Observable different from Promise?
4. How is Observable different from async iterator?
5. What is push vs pull?
6. What is subscription?
7. What is teardown?
8. What is cold vs hot?
9. What is multicast?
10. What is replay?
11. What are next/error/complete?
12. What is cancellation vs completion?
13. What is backpressure?
14. Why can a push system overwhelm a consumer?
15. What is buffering?
16. What are debounce/throttle/sample?
17. What is flattening?
18. Compare merge, concat, switch, exhaust.
19. Why does switch-style behavior help search UIs?
20. When is switch-style behavior wrong?
21. Why can concat cause queue growth?
22. What is subscription leakage?
23. What is reentrancy?
24. Why is synchronous emission important?
25. What is a multicast source lifecycle?
26. What is replay-memory risk?
27. How should retries be bounded?
28. How does Chapter 40 connect to Chapter 39?
29. When should a system prefer async iterators?
30. When should a system avoid reactive abstraction?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit Observable vs Promise
- [ ] Revisit Observable vs async iterator
- [ ] Revisit push vs pull
- [ ] Revisit subscription lifecycle
- [ ] Revisit teardown
- [ ] Revisit cold/hot/multicast
- [ ] Revisit replay
- [ ] Revisit backpressure
- [ ] Revisit buffering
- [ ] Revisit debounce/throttle/sample
- [ ] Revisit flattening strategies
- [ ] Revisit bounded reactive concurrency
- [ ] Revisit reentrancy
- [ ] Revisit reactive memory leaks
- [ ] Revisit reactive production design
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

# Chapter 40 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — Promise, async function, iterator, async iterator, Job, and agent semantics.
2. WHATWG browser standards — events, streams, Web Workers, scheduling, and browser lifecycle behavior.
3. Node.js official documentation — event, stream, worker, timer, and runtime APIs.
4. Observable library specifications/documentation — use the exact semantics of the chosen implementation.
5. Application architecture documentation — lifecycle, backpressure, retry, ownership, and observability policies.

Always distinguish:

```text
ECMAScript language semantics
vs
host event APIs
vs
Observable library semantics
vs
application-specific reactive policy
```

Do not claim that all Observable libraries share identical behavior.

Do not claim that Observable is universally asynchronous.

Do not claim that Observable is a built-in ECMAScript primitive unless the specific environment/specification establishes that.

Do not equate unsubscribe with guaranteed physical cancellation of an underlying network or OS operation.

Do not assume backpressure exists automatically in a push-based Observable implementation.

---

# Chapter 40 — Completion Snapshot

```text
Chapter: 40
Title: Observables and Reactive Programming
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
