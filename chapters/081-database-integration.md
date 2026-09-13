# Chapter 81 — Database Integration

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Integrating JavaScript applications with relational and non-relational databases while preserving correctness, performance, security, reliability, and architectural boundaries.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what database integration means at application architecture level.
2. Distinguish database technology from database access patterns.
3. Design a database boundary inside a production JavaScript application.
4. Choose between raw SQL, query builders, ORMs, and database-specific APIs.
5. Design repositories and data-access modules without creating useless abstractions.
6. Understand connection pools and their relationship to application concurrency.
7. Design transactions deliberately.
8. Explain ACID properties and their practical trade-offs.
9. Understand isolation levels and anomalies.
10. Prevent lost updates and race conditions.
11. Design optimistic and pessimistic concurrency controls.
12. Design efficient queries and indexes.
13. Diagnose N+1 query behavior.
14. Understand pagination and large dataset access.
15. Handle database errors and translate them into application errors.
16. Prevent SQL injection and unsafe dynamic query construction.
17. Design database constraints as part of correctness.
18. Distinguish application validation from database enforcement.
19. Design migrations and schema evolution safely.
20. Handle transaction boundaries in asynchronous JavaScript code.
21. Reason about connection lifetime, request lifetime, and transaction lifetime.
22. Design retries without duplicating non-idempotent database effects.
23. Understand deadlocks, lock contention, and retry policies.
24. Design caching around database access.
25. Reason about read replicas, eventual consistency, and stale reads.
26. Design multi-tenant database access safely.
27. Understand soft deletion and its trade-offs.
28. Design audit history and temporal state when required.
29. Integrate queues/outbox workflows with database transactions.
30. Monitor query performance and database saturation.
31. Build a production-style Node.js database integration layer.
32. Debug database-related correctness and performance failures.
33. Evaluate database architecture at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript objects, functions, modules, promises, async/await.
- Error handling and cleanup.
- Node.js architecture and lifecycle.
- Streams and concurrency.
- Production architecture.
- API design.
- Basic SQL.
- Primary keys and foreign keys.
- Basic transactions.
- Basic indexes.
- Basic HTTP/API concepts.

Recommended prior chapters:

- **15–21** — Objects / Prototypes / Metaprogramming
- **29–30** — Errors / Resource Management
- **31–40** — Async / Concurrency / Cancellation
- **55** — HTTP Networking
- **58–63** — Node.js
- **64–70** — Modules / Packages / Tooling
- **74–77** — Programming Paradigms / Patterns
- **78** — Production JavaScript Architecture
- **79** — API Design
- **80** — Library Authoring

---

# 3. What Is It?

Database integration is the boundary between application behavior and persistent data.

At the simplest level:

```text
JavaScript application
      ↓
database client
      ↓
database
```

In production:

```text
HTTP / job / CLI
      ↓
application use case
      ↓
transaction boundary
      ↓
repository / query module
      ↓
connection pool
      ↓
database
```

The database is not merely a place to “store objects.”

It is an independent system with:

```text
concurrency
constraints
transactions
locks
indexes
execution plans
resource limits
durability
failure modes
security boundaries
```

A production JavaScript engineer must reason about both sides:

```text
JavaScript runtime semantics
+
database semantics
```

---

# 4. Why Does It Exist?

Applications need durable state.

Examples:

```text
users
orders
payments
inventory
sessions
audit records
configuration
events
```

In-memory JavaScript state disappears when the process exits.

A database provides durable shared state across processes and machines.

But persistence creates new problems:

```text
network latency
concurrency
transactions
schema evolution
connection management
query performance
locking
failure recovery
```

Therefore:

> Database integration is not “calling a query.” It is coordinating two different execution and consistency systems.

---

# 5. Mental Model

Use this mental model:

```text
                    APPLICATION
                         │
                  use case / domain
                         │
                  data-access boundary
                         │
                  ┌──────▼──────┐
                  │ Connection  │
                  │    Pool     │
                  └──────┬──────┘
                         │
                    SQL / protocol
                         │
                  ┌──────▼──────┐
                  │  Database   │
                  │             │
                  │ constraints │
                  │ indexes      │
                  │ transactions│
                  │ locks       │
                  └─────────────┘
```

Then add time:

```text
request begins
  ↓
connection acquired
  ↓
transaction begins
  ↓
queries execute
  ↓
commit / rollback
  ↓
connection released
  ↓
response
```

A critical rule:

> A transaction is a database-state boundary, not a JavaScript function boundary.

---

# 6. Core Rules

## Rule 1 — The database is part of the correctness model

Do not put every invariant only in JavaScript.

Example:

```text
email must be unique
```

The application can check it.

But concurrent requests can both pass:

```text
SELECT ...
```

before either inserts.

A database `UNIQUE` constraint can enforce the invariant atomically.

---

## Rule 2 — Constraints are executable business protection

Useful constraints:

```text
PRIMARY KEY
UNIQUE
FOREIGN KEY
NOT NULL
CHECK
```

The application should still provide good user-facing validation.

Use both where appropriate.

---

## Rule 3 — Parameterize values

Unsafe:

```js
const sql = `
  SELECT *
  FROM users
  WHERE email = '${email}'
`;
```

Safe pattern:

```js
const result = await client.query(
  `
    SELECT *
    FROM users
    WHERE email = $1
  `,
  [email],
);
```

OWASP recommends parameterized queries / prepared statements as a primary defense against SQL injection and recommends allow-list validation or query redesign for query parts that cannot be represented as bind parameters, such as dynamic column names. citeturn325140search0turn325140search2

---

## Rule 4 — Connection pools are finite resources

A database pool is not:

```text
infinite database capacity
```

It is a bounded resource.

Too much application concurrency can become:

```text
request surge
→ pool saturation
→ queueing
→ latency increase
→ timeout
→ retry storm
```

---

## Rule 5 — Keep transactions short

Long transactions can hold:

```text
locks
connections
snapshots
resources
```

for too long.

Do not perform slow unrelated external calls inside a transaction unless the design explicitly requires it.

---

## Rule 6 — Query count matters

A query taking 5 ms executed once:

```text
5 ms
```

is not the same as:

```text
5 ms × 10,000
```

Database performance is often dominated by call count and data movement, not only individual query speed.

---

## Rule 7 — Make ordering explicit

If application behavior depends on order:

```sql
ORDER BY created_at, id
```

Do not rely on accidental database output order.

---

## Rule 8 — Retry only when semantics allow it

A transaction retry may be safe for a serialization failure.

A blind retry of:

```text
charge payment
```

may duplicate a business operation.

---

## Rule 9 — Schema is an API

Tables, columns, constraints, and indexes are dependencies.

Schema evolution is therefore compatibility engineering.

---

## Rule 10 — Measure before optimizing

Use:

```text
query plans
latency metrics
pool metrics
database wait metrics
application traces
```

rather than guessing.

---

# 7. Syntax

A generic Node.js database access pattern:

```js
const result = await db.query(
  "SELECT id, email FROM users WHERE id = $1",
  [userId],
);

const user = result.rows[0] ?? null;
```

A transaction pattern:

```js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  await client.query(
    "INSERT INTO orders (id, total) VALUES ($1, $2)",
    [orderId, total],
  );

  await client.query(
    "UPDATE inventory SET quantity = quantity - $1 WHERE sku = $2",
    [quantity, sku],
  );

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
```

The exact API varies by driver, but the lifecycle is conceptually:

```text
acquire
→ begin
→ execute
→ commit/rollback
→ release
```

---

# 8. Basic Examples

## Example 1 — Select

```js
const { rows } = await db.query(
  `
    SELECT id, status, total
    FROM orders
    WHERE customer_id = $1
    ORDER BY created_at DESC
    LIMIT $2
  `,
  [customerId, limit],
);
```

---

## Example 2 — Insert

```js
await db.query(
  `
    INSERT INTO orders (id, customer_id, total)
    VALUES ($1, $2, $3)
  `,
  [id, customerId, total],
);
```

---

## Example 3 — Database constraint

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT NOT NULL UNIQUE
);
```

The application can check for duplicate email.

The database still protects the invariant under concurrency.

---

# 9. Execution Walkthrough

Suppose:

```http
POST /orders
```

causes:

```js
await placeOrder(command);
```

The application path might be:

```text
HTTP request
  ↓
authentication
  ↓
authorization
  ↓
validation
  ↓
placeOrder use case
  ↓
BEGIN
  ↓
read customer
  ↓
read inventory
  ↓
insert order
  ↓
decrement inventory
  ↓
COMMIT
  ↓
publish/record integration event
```

At each step, ask:

```text
What can fail?
What is locked?
What is visible to other transactions?
What happens if process dies?
What happens if client retries?
```

---

# 10. Internal Mechanics

## 10.1 Connection pool

Conceptually:

```text
          application
       /      |      \
      /       |       \
   request request request
       \       |       /
        \      |      /
         connection pool
          /   |   \
       conn conn conn
         \    |    /
           database
```

The pool limits concurrent database work.

If:

```text
pool size = 20
active database operations = 20
new requests = 500
```

the remaining work must queue, reject, or time out according to the client/application policy.

---

## 10.2 Transaction lifecycle

```text
BEGIN
  ↓
statement 1
  ↓
statement 2
  ↓
statement 3
  ↓
COMMIT
```

If something fails:

```text
BEGIN
  ↓
statement 1
  ↓
statement 2 → error
  ↓
ROLLBACK
```

The application must reliably release the connection afterward.

---

## 10.3 Autocommit

Databases often execute statements in their own transaction when explicit transaction blocks are not used.

PostgreSQL documents this behavior as autocommit-like operation and supports explicit `BEGIN` / `START TRANSACTION` blocks with configurable transaction characteristics. citeturn325140search4

Do not assume that:

```js
await query("UPDATE ...");
await query("INSERT ...");
```

is one atomic operation.

Unless an explicit transaction surrounds them, they may commit independently.

---

# 11. ECMAScript / Specification Semantics

Database behavior is not ECMAScript behavior.

Keep these layers separate:

```text
ECMAScript
  ↓
Promise / async function behavior

Node.js
  ↓
socket / stream / process behavior

Database driver
  ↓
protocol + pooling + query API

Database engine
  ↓
transactions / locks / constraints / planner

Application
  ↓
business semantics
```

For example:

```js
await db.query(...);
```

means JavaScript waits for the Promise to settle.

It does not by itself mean:

```text
durable
atomic
isolated
retriable
idempotent
```

Those properties come from the database and application design.

---

# 12. Advanced Behavior

## 12.1 ACID

### Atomicity

All operations in a transaction succeed or none do.

### Consistency

A committed transaction preserves database-defined invariants.

### Isolation

Concurrent transactions observe changes according to the chosen isolation semantics.

### Durability

Committed state survives according to the database's durability guarantees.

These are properties of the database system, not of `async/await`.

---

## 12.2 Isolation levels

Common relational isolation terminology:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

The actual behavior is database-specific.

PostgreSQL currently documents transaction modes including `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`; its default and exact anomaly behavior should be checked against the PostgreSQL version you deploy. citeturn325140search4

---

## 12.3 Concurrency anomalies

### Dirty read

Transaction A observes uncommitted changes from B.

### Non-repeatable read

A row read twice can produce different values because another transaction committed between reads.

### Phantom behavior

A repeated predicate can observe a changed set of rows.

### Lost update

Two transactions read the same state and one overwrites another's work.

Do not learn anomalies as vocabulary only.

Learn them as schedules:

```text
T1: read x
T2: read x
T1: write x
T2: write x
```

---

## 12.4 Optimistic concurrency

Use a version:

```text
id = 42
version = 7
```

Update:

```sql
UPDATE orders
SET status = $1,
    version = version + 1
WHERE id = $2
  AND version = $3;
```

If affected rows:

```text
0 → someone else changed it
1 → update succeeded
```

This converts a race into an explicit conflict.

---

## 12.5 Pessimistic locking

A design may intentionally lock rows while performing a critical operation.

Conceptually:

```sql
SELECT ...
FOR UPDATE
```

This can protect concurrency-sensitive state but introduces:

```text
waiting
lock contention
deadlocks
reduced throughput
```

Use only when its correctness benefit is justified.

---

## 12.6 Deadlocks

Example:

```text
T1 locks A → waits for B
T2 locks B → waits for A
```

The database may abort one transaction.

Application behavior should classify that error and decide whether retry is appropriate.

Good retry design includes:

```text
bounded attempts
backoff
jitter where appropriate
idempotent or transaction-safe operation
observability
```

---

# 13. Edge Cases

## 13.1 Process crash after commit

Sequence:

```text
DB COMMIT
  ↓
process crashes
  ↓
HTTP response never reaches client
```

The operation may already exist.

Client retry can duplicate side effects unless the API/use case is idempotent.

---

## 13.2 Process crash before commit

```text
BEGIN
  ↓
queries
  ↓
process dies
```

The database should resolve the incomplete transaction according to its transaction/recovery semantics.

---

## 13.3 Connection returned incorrectly

Bad:

```js
const client = await pool.connect();

await client.query(...);
// forgotten release
```

This can exhaust the pool.

Correct lifecycle uses:

```js
try {
  ...
} finally {
  client.release();
}
```

---

## 13.4 Transaction uses multiple connections

Bad design:

```text
query A → connection 1
query B → connection 2
```

while believing both belong to one transaction.

A database transaction is generally associated with a specific session/connection.

Use a transaction-aware abstraction that keeps the sequence on the same connection.

---

## 13.5 External API inside database transaction

Potentially:

```text
BEGIN
  DB update
  wait 2 seconds for HTTP API
  DB update
COMMIT
```

Now the transaction holds resources during external latency.

Often better:

```text
commit durable intent
→ perform external work
→ record result/reconcile
```

or use an explicit outbox/workflow design.

---

# 14. Common Misconceptions

### “ORM means I don't need SQL.”

You still need to understand query shape, joins, indexes, transactions, locks, and execution plans.

### “Database constraints duplicate application validation.”

They serve different purposes.

Application validation:

```text
good UX
early feedback
business-specific messages
```

Database constraints:

```text
final integrity enforcement
concurrency-safe protection
```

### “A transaction makes everything safe.”

A transaction does not automatically solve:

```text
external side effects
poor isolation choices
deadlocks
long locks
bad queries
idempotency
```

### “Connection pools make databases infinitely concurrent.”

No. Pools bound concurrent connections and can also hide queueing.

### “NoSQL means no consistency.”

No. Different databases offer different consistency and transaction models.

### “Retries are free.”

Retries consume more:

```text
connections
CPU
database capacity
network capacity
```

and can worsen outages.

---

# 15. Common Mistakes

## Mistake 1 — String-concatenated SQL

```js
`SELECT * FROM users WHERE email = '${email}'`
```

Avoid.

Use parameterized queries. citeturn325140search0turn325140search2

---

## Mistake 2 — `SELECT *`

Returning every column can:

```text
increase I/O
increase memory
expose sensitive data
couple code to schema
```

Select what the use case needs.

---

## Mistake 3 — Query in loops

```js
for (const order of orders) {
  await loadCustomer(order.customerId);
}
```

Can produce N+1 database access.

---

## Mistake 4 — Long transactions

Do not hold transactions open during:

```text
user interactions
slow external calls
long CPU work
unbounded loops
```

---

## Mistake 5 — No explicit ordering

```sql
SELECT * FROM orders LIMIT 20;
```

does not define which 20 rows you mean.

---

## Mistake 6 — Treating soft delete as simple

```sql
UPDATE users SET deleted_at = now();
```

creates questions around:

```text
unique constraints
foreign keys
queries
indexes
restoration
cascades
analytics
```

---

## Mistake 7 — Schema changes without compatibility planning

Dropping a column before consumers stop reading it causes failures.

---

# 16. Comparison With Related Concepts

| Approach | Strength | Risk |
|---|---|---|
| Raw SQL | Maximum control | More SQL responsibility |
| Query builder | Structured dynamic queries | Adds abstraction |
| ORM | Productivity/domain mapping | Hidden query costs / impedance mismatch |
| Repository | Boundary around data access | Can become generic wrapper |
| Data mapper | Separates persistence model | More transformation work |
| Active Record | Convenient entity persistence | Couples domain to persistence |
| Relational DB | Strong constraints / transactions | Schema and coordination overhead |
| Document DB | Flexible document model | Cross-document consistency trade-offs |
| Key-value store | Simple high-speed access | Limited query semantics |
| Cache | Reduces read pressure | Invalidation / staleness |
| Read replica | Scales reads | Replication lag / stale reads |

---

# 17. Performance Considerations

## 17.1 Query latency model

A database request can cost:

```text
connection acquisition
+ network
+ parsing/planning
+ execution
+ lock waiting
+ result transfer
+ application mapping
```

Optimizing only SQL execution time can miss the actual bottleneck.

---

## 17.2 Indexes

An index can improve reads but costs:

```text
storage
write overhead
maintenance
cache space
```

Do not index every column.

Design indexes around actual access patterns.

Example:

```sql
CREATE INDEX orders_customer_created_idx
ON orders (customer_id, created_at DESC);
```

This may support:

```sql
WHERE customer_id = ?
ORDER BY created_at DESC
```

but verify using a query plan.

---

## 17.3 N+1

Bad:

```js
const users = await userRepository.findAll();

for (const user of users) {
  user.orders = await orderRepository.findByUserId(user.id);
}
```

Better:

```js
const users = await userRepository.findAll();
const orders = await orderRepository.findByUserIds(
  users.map(user => user.id),
);
```

Or use a join where appropriate.

The correct solution depends on:

```text
result size
cardinality
indexing
payload
database planner
```

---

## 17.4 Pagination

Offset:

```sql
LIMIT 50 OFFSET 50000
```

can become expensive on large datasets.

Cursor/keyset:

```sql
WHERE created_at < $1
ORDER BY created_at DESC
LIMIT 50
```

can be more efficient when the ordering is appropriate and indexed.

---

## 17.5 Pool sizing

Larger pools are not automatically faster.

A large pool can increase:

```text
database contention
CPU
context switching
lock competition
memory
```

Pool size should be derived from workload, database capacity, query duration, and number of application instances.

---

# 18. Memory Considerations

Database integration can create memory pressure through:

```text
large query results
result buffering
ORM object graphs
caches
queued queries
connection pools
serialized payloads
```

Avoid loading millions of rows into memory.

Prefer:

```text
bounded queries
pagination
streaming where supported
incremental processing
batch operations
```

A database driver's streaming API can be useful for large results, but streaming changes error, cancellation, transaction, and connection-lifecycle considerations.

---

# 19. Security Considerations

## 19.1 SQL injection

Primary defense:

```text
parameterized queries
```

For dynamic identifiers or sort expressions that cannot be bound as values, use strict allow-lists or safe query construction. OWASP explicitly warns against relying on escaping as the primary defense. citeturn325140search0turn325140search1

---

## 19.2 Least privilege

Application database credentials should not automatically have:

```text
schema ownership
DROP DATABASE
CREATE EXTENSION
administrative privileges
```

Use the smallest required permission set.

---

## 19.3 Tenant isolation

For multi-tenant systems:

```text
tenant_id
```

should participate in the access-control model.

Do not rely solely on:

```js
WHERE id = $1
```

if the object can belong to another tenant.

Prefer:

```sql
SELECT *
FROM orders
WHERE id = $1
  AND tenant_id = $2;
```

The tenant boundary should be enforced close to the data access.

---

## 19.4 Sensitive data

Do not casually log:

```text
password hashes
access tokens
payment secrets
identity documents
private customer data
```

Observability must respect data classification.

---

# 20. Production Usage

## 20.1 Recommended architecture

```text
                    Application
                         │
                    Use Case
                         │
                 Repository Port
                         │
                Data Access Adapter
                         │
                 Database Client
                         │
                  Connection Pool
                         │
                      Database
```

Infrastructure should own:

```text
driver configuration
pool creation
SQL
transaction implementation
database-specific errors
```

The application should own:

```text
business decisions
workflow
domain invariants
transaction intent
```

---

## 20.2 Repository design

Good:

```js
orderRepository.findPendingForCustomer(customerId);
orderRepository.save(order);
```

Less useful:

```js
orderRepository.query(sql, params);
```

The latter can turn the repository into a thin driver proxy.

---

## 20.3 Error translation

Database:

```text
unique_violation
```

Application:

```js
new DuplicateOrderError();
```

API:

```http
409 Conflict
```

The mapping should happen at a boundary.

---

## 20.4 Migrations

Treat schema migration as a deployment concern.

### Expand-and-contract pattern

```text
old schema
  ↓
add new compatible structure
  ↓
deploy code that understands both
  ↓
backfill
  ↓
switch reads/writes
  ↓
verify
  ↓
remove old structure
```

Avoid:

```text
drop column
→ deploy new code
```

when old application instances may still run.

---

## 20.5 Outbox pattern

Dual write problem:

```text
DB transaction
+
message publish
```

Example failure:

```text
DB COMMIT succeeds
message publish fails
```

Or:

```text
message publishes
DB rollback
```

Outbox approach:

```text
BEGIN
  update business state
  insert outbox event
COMMIT

separate publisher
  ↓
publish event
  ↓
mark outbox delivered
```

This couples the durable intent to the database transaction.

It does not guarantee exactly-once external effects.

It gives a more reliable path toward eventual publication.

---

## 20.6 Read replicas

Architecture:

```text
write
  ↓
primary

read
  ↓
replica
```

Potential problem:

```text
write primary
immediate read replica
→ stale value
```

The application must understand replica lag.

For read-after-write requirements, route the read appropriately or use a consistency strategy that matches the use case.

---

## 20.7 Caching

Database:

```text
source of durable truth
```

Cache:

```text
performance optimization
```

Design:

```text
cache key
TTL
invalidations
stale policy
capacity
eviction
failure behavior
```

Never make a cache the accidental source of truth.

---

## 20.8 Health and readiness

A health endpoint might answer:

```text
process is alive
```

A readiness endpoint may need to answer:

```text
can this process safely serve traffic?
```

Do not make health checks expensive database queries at high frequency.

---

# 21. Implementation From Scratch

Build a production-style database layer for an orders service.

## Stage 1 — Guided

Create:

```text
src/
  database/
    pool.js
    transaction.js

  orders/
    repository.js
    service.js
```

---

## Stage 2 — Repository

```js
export function createOrderRepository(db) {
  return {
    async findById(id) {
      const { rows } = await db.query(
        `
          SELECT id, customer_id, status, total
          FROM orders
          WHERE id = $1
        `,
        [id],
      );

      return rows[0] ?? null;
    },

    async save(order) {
      await db.query(
        `
          INSERT INTO orders
            (id, customer_id, status, total)
          VALUES
            ($1, $2, $3, $4)
        `,
        [
          order.id,
          order.customerId,
          order.status,
          order.total,
        ],
      );
    },
  };
}
```

---

## Stage 3 — Transaction abstraction

Create:

```js
export async function withTransaction(pool, callback) {
  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    const result = await callback(client);

    await client.query("COMMIT");

    return result;
  } catch (error) {
    try {
      await client.query("ROLLBACK");
    } catch {
      // Preserve the original failure.
    }

    throw error;
  } finally {
    client.release();
  }
}
```

The abstraction must preserve:

```text
same connection
correct rollback
release on every path
original error
```

---

## Stage 4 — Concurrency protection

Implement:

```text
inventory decrement
```

Requirements:

```text
cannot go below zero
concurrent requests cannot oversell
failure rolls back
```

Possible SQL approach:

```sql
UPDATE inventory
SET quantity = quantity - $1
WHERE sku = $2
  AND quantity >= $1;
```

Then inspect affected rows:

```text
1 → decrement succeeded
0 → insufficient inventory / missing row
```

This can be safer than:

```text
SELECT quantity
→ JavaScript checks
→ UPDATE
```

because the condition and update can be expressed atomically.

---

## Stage 5 — Edge-case hardened

Add:

```text
deadlock retry
serialization retry
timeouts
statement timeout
pool exhaustion handling
query metrics
slow query logs
request cancellation
tenant isolation
```

---

## Stage 6 — Production-grade

Add:

```text
migrations
expand-contract deployment
outbox
idempotency record
read/write routing
cache
audit trail
backup verification
restore test
load test
query plan review
database observability
```

---

# 22. Debugging Exercises

## Exercise 1 — Connection leak

```js
const client = await pool.connect();

await client.query("SELECT 1");

// error path forgotten
```

Find why the pool eventually stops serving requests.

---

## Exercise 2 — N+1

A page returns:

```text
100 orders
```

and executes:

```text
1 query for orders
+
100 queries for customer data
```

Calculate the conceptual request count and design a better access strategy.

---

## Exercise 3 — Lost update

Two requests both:

```text
read balance = 100
subtract 80
write balance = 20
```

Both succeed.

Explain the race and propose:

```text
transaction
locking
conditional update
version check
```

alternatives.

---

## Exercise 4 — Deadlock

Transaction A:

```text
lock order
lock payment
```

Transaction B:

```text
lock payment
lock order
```

Find the cycle.

Design a consistent lock-order rule.

---

## Exercise 5 — Retry storm

Database becomes slow.

Requests timeout.

Application retries.

Traffic doubles.

Database becomes slower.

Explain the feedback loop.

---

## Exercise 6 — Replica lag

```text
POST /orders
→ write primary

GET /orders/123
→ read replica

response: 404
```

Explain why the application can see this apparently impossible state.

---

## Exercise 7 — Schema migration failure

Deploy sequence:

```text
drop old column
deploy new application
```

An old application instance is still running.

Explain the failure.

---

## Exercise 8 — Outbox duplication

Publisher crashes after:

```text
publish(event)
```

but before:

```text
markDelivered(event)
```

The same event publishes again.

Design consumer idempotency.

---

# 23. Code Review Exercise

Review:

```js
export async function transfer(from, to, amount) {
  const source = await db.query(
    `SELECT balance FROM accounts WHERE id = '${from}'`,
  );

  if (source.rows[0].balance < amount) {
    throw new Error("Insufficient funds");
  }

  await db.query(
    `UPDATE accounts
     SET balance = balance - ${amount}
     WHERE id = '${from}'`,
  );

  await db.query(
    `UPDATE accounts
     SET balance = balance + ${amount}
     WHERE id = '${to}'`,
  );
}
```

Identify at least 20 problems.

Expected areas:

```text
SQL injection
no transaction
race condition
lost update
partial transfer
missing account handling
negative amount
same-account case
currency precision
authorization
tenant isolation
deadlock ordering
connection lifecycle
error taxonomy
audit
idempotency
timeouts
observability
database constraints
retry semantics
input validation
```

Design a corrected architecture rather than only patching the SQL strings.

---

# 24. Interview Questions

## Fundamental

1. What is a connection pool?
2. Why use a transaction?
3. What is ACID?
4. Why use database constraints?
5. What is an index?
6. What is N+1?
7. Why parameterize queries?
8. What is a repository?
9. What is autocommit?
10. Why explicitly order query results?

## Intermediate

11. Explain lost updates.
12. Explain optimistic concurrency.
13. Explain pessimistic locking.
14. What is a deadlock?
15. How would you retry a deadlock?
16. Why can a transaction become a performance problem?
17. How does pool size affect concurrency?
18. Offset versus keyset pagination?
19. What is a read replica?
20. What is replica lag?

## Advanced

21. Explain expand-and-contract migrations.
22. Why is database schema a compatibility boundary?
23. Explain the outbox pattern.
24. How do you make outbox consumers idempotent?
25. How do you avoid distributed transaction requirements?
26. How do you design database access for multi-tenancy?
27. How do you diagnose slow queries?
28. How do you choose ORM versus raw SQL?
29. How do you prevent connection leaks?
30. How do you design transactions around async workflows?

## Principal

31. How do you determine transaction boundaries?
32. When should a business invariant be enforced in the database?
33. When would you reject an ORM?
34. How do you scale a database-bound Node.js application?
35. What would make you introduce read replicas?
36. How do you reason about consistency after introducing caching?
37. How do you migrate a billion-row table safely?
38. How do you handle a database outage without causing retry amplification?
39. How do database boundaries affect service boundaries?
40. When should the database become a domain boundary rather than an implementation detail?

---

# 25. Predict-the-Output Exercises

## Exercise A — Promise is not transaction

Predict:

```js
async function workflow() {
  await db.query("UPDATE accounts SET balance = balance - 10");
  await db.query("UPDATE accounts SET balance = balance + 10");
}
```

Question:

> Does `await` guarantee that both updates are atomic?

### Answer

No.

`await` sequences Promise settlement in JavaScript.

It does not automatically create a database transaction.

---

## Exercise B — Mutable JavaScript reference

Predict:

```js
const query = {
  text: "SELECT * FROM users",
  values: [],
};

function addValue(item) {
  item.values.push("x");
}

addValue(query);

console.log(query.values.length);
```

### Actual Result

```text
1
```

### Rule

Database access code is still JavaScript and follows ordinary object-reference semantics.

---

## Exercise C — Conditional update

Suppose the database operation is conceptually:

```sql
UPDATE inventory
SET quantity = quantity - 3
WHERE sku = 'A'
  AND quantity >= 3;
```

If current quantity is:

```text
2
```

and the application checks affected rows, predict:

```text
0
```

The update condition is false.

### Principal Lesson

Push concurrency-sensitive conditions into the database when atomic database semantics protect the invariant.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Explain:

```text
ACID
isolation
locks
deadlocks
optimistic concurrency
pessimistic concurrency
```

using one bank-transfer example.

### A2

Draw the lifecycle:

```text
HTTP request
→ pool
→ connection
→ transaction
→ query
→ commit
→ release
```

and identify every failure point.

---

## Track B — Implementation

### B1 — Production order system

Build:

```text
customers
orders
inventory
payments
```

Requirements:

```text
foreign keys
unique constraints
transactions
indexes
pagination
tenant isolation
```

### B2 — Concurrency

Implement a stock reservation operation that remains correct under concurrent requests.

Test:

```text
1 stock / 10 requests
```

Only one request should succeed.

### B3 — Reliability

Implement:

```text
outbox
idempotent consumer
retry-safe transaction handling
deadlock retry
graceful shutdown
```

---

## Track C — Interview / Reasoning

### C1

A Node service has:

```text
2,000 requests/sec
pool = 200
average query = 80 ms
database CPU = 95%
```

Decide whether increasing the pool is likely to help.

Defend the answer.

### C2

Design migration from:

```text
orders.total_cents INTEGER
```

to:

```text
orders.amount DECIMAL
orders.currency TEXT
```

without downtime.

### C3

A payment workflow requires:

```text
DB update
payment provider
email
event
```

Design a failure-safe architecture without pretending there is one atomic transaction across every system.

---

# 27. Key Takeaways

1. Databases are correctness systems, not storage buckets.
2. JavaScript async semantics and database transaction semantics are different layers.
3. Connection pools are finite resources.
4. Transactions must be explicit when atomicity is required.
5. Constraints protect invariants under concurrency.
6. Parameterized queries are a primary SQL-injection defense. citeturn325140search0turn325140search2
7. Dynamic identifiers require allow-listing or query redesign.
8. Query count matters.
9. N+1 is often an architecture/data-access problem.
10. Indexes trade write/storage cost for read efficiency.
11. Isolation levels change observable concurrency behavior.
12. Optimistic and pessimistic concurrency solve different problems.
13. Deadlocks are normal enough to design for.
14. Retry policies must respect business semantics.
15. A process crash can happen after database commit but before an HTTP response.
16. Outbox patterns address reliable publication intent, not magic exactly-once delivery.
17. Read replicas can introduce stale reads.
18. Caches introduce consistency and invalidation trade-offs.
19. Schema migrations are compatibility engineering.
20. Principal database design is about correctness first, then measured performance and operational resilience.

---

# 28. Concept Connections

## Depends On

- **Chapter 29** — Errors
- **Chapter 30** — Resource Management
- **Chapter 31–40** — Async / Concurrency / Cancellation / Streaming
- **Chapter 45–48** — Memory and engine behavior
- **Chapter 55** — HTTP networking
- **Chapter 58–63** — Node.js runtime
- **Chapter 67** — Dependency / supply-chain awareness
- **Chapter 70** — Production debugging
- **Chapter 78** — Production Architecture
- **Chapter 79** — API Design
- **Chapter 80** — Library Authoring

## Builds Toward

- **Chapter 82** — API Architecture
- **Chapter 83** — Observability
- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 87** — Deterministic Async Testing
- **Chapter 89** — Code Review / Refactoring
- **Chapter 101** — Real-world Production Scenarios
- **Chapter 105** — Node REST API
- **Chapter 107** — Job Queue
- **Chapter 108** — Cache System
- **Chapter 109** — Event-driven App
- **Chapter 110** — Production JavaScript Backend
- **Chapter 111** — Large-scale JavaScript Platform
- **Chapter 121** — Principal System Design

## Related Concepts

```text
Database Integration
  ├─ SQL
  ├─ transactions
  ├─ constraints
  ├─ indexes
  ├─ concurrency
  ├─ connection pools
  ├─ migrations
  ├─ replication
  ├─ caching
  ├─ outbox
  └─ observability
```

## Concepts Revisited

```text
Promises
async/await
errors
resource lifetime
dependency injection
API boundaries
idempotency
backpressure
security
performance
```

## Why This Chapter Matters Later

Many production failures attributed to “the database” are actually boundary failures:

```text
bad transaction boundary
bad query shape
bad pool sizing
bad retry behavior
bad migration
bad cache consistency
bad authorization
```

The database is only one component of the system.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
queries
connections
transactions
constraints
indexes
basic performance
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
async/await with transactions
application validation with constraints
pool size with database capacity
retry with idempotency
replica with strong consistency
cache with source of truth
```

Mark:

```text
[+] Completed
```

when you can:

- design a database access layer;
- use parameterized queries;
- define transaction boundaries;
- use constraints;
- prevent N+1;
- design indexes from access patterns;
- handle connection lifecycle;
- design safe migrations;
- explain concurrency behavior.

Mark:

```text
[*] Mastered
```

only when you can:

- diagnose database incidents from symptoms;
- reason about lock schedules;
- choose isolation deliberately;
- design migration strategies for live systems;
- balance ORM and SQL trade-offs;
- design retry-safe database workflows;
- integrate replicas and caches safely;
- defend database architecture at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 81 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain database integration boundary | ____ | `[ ]` |
| ____ | Explain connection pooling | ____ | `[ ]` |
| ____ | Design transaction boundary | ____ | `[ ]` |
| ____ | Explain isolation anomalies | ____ | `[ ]` |
| ____ | Design concurrency control | ____ | `[ ]` |
| ____ | Diagnose N+1 | ____ | `[ ]` |
| ____ | Design schema migration | ____ | `[ ]` |
| ____ | Design outbox reliability | ____ | `[ ]` |
| ____ | Threat-model database access | ____ | `[ ]` |
| ____ | Defend database architecture | ____ | `[ ]` |

## Spaced Retrieval

```text
Review 1 — same day
Review 2 — +1 day
Review 3 — +3 days
Review 4 — +7 days
Review 5 — +14 days
Review 6 — +30 days
Review 7 — +60 days
```

## Retrieval Prompts

Without reading:

1. Why is a database more than storage?
2. What does a connection pool control?
3. Why does `await` not make two queries atomic?
4. Explain lost update.
5. Explain optimistic concurrency.
6. Explain deadlock.
7. Why are constraints important?
8. Explain N+1.
9. Explain expand-and-contract migration.
10. Explain the outbox pattern.
11. Why can replicas return stale data?
12. Why can retries amplify a database outage?

---

# Chapter 81 — Canonical References and Source Discipline

## 1. PostgreSQL Documentation

Use the PostgreSQL documentation for:

```text
transactions
isolation
locking
constraints
indexes
queries
replication
maintenance
```

The current PostgreSQL documentation identifies version 18 as the current major release and documents explicit transaction blocks and transaction modes including `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`. Verify the exact behavior against the PostgreSQL version your production system runs. citeturn325140search4turn325140search5

Primary:

- https://www.postgresql.org/docs/current/

---

## 2. OWASP SQL Injection Prevention

Use OWASP for:

```text
parameterized queries
prepared statements
allow-listing
safe dynamic SQL
```

OWASP recommends parameterized queries as a primary defense and specifically recommends allow-listing or query redesign when user input influences SQL identifiers such as table/column names. citeturn325140search0turn325140search1

Primary:

- https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

---

## 3. ECMAScript

Use the ECMAScript specification for:

```text
Promise semantics
async functions
objects
modules
language-level exceptions
```

Primary:

- https://tc39.es/ecma262/

---

## 4. Node.js

Use Node.js documentation for:

```text
sockets
streams
timers
process lifecycle
runtime behavior
```

Primary:

- https://nodejs.org/docs/latest/api/

Database drivers remain external packages/runtime integrations; verify driver-specific pooling, cancellation, transaction, and error behavior in the actual package/version used.

---

## Source Discipline

When documenting a database decision, classify it as:

```text
ECMAScript guarantee
Node.js runtime behavior
Database driver behavior
Database-engine behavior
SQL standard behavior
Database-specific behavior
Application policy
Operational convention
```

Never say:

> “JavaScript guarantees transaction rollback.”

Instead:

> “The database transaction and its driver/client lifecycle provide rollback semantics.”

Never say:

> “SQL always behaves this way.”

Instead identify:

```text
SQL standard
PostgreSQL
MySQL
SQLite
MongoDB
specific driver
```

where relevant.

---

# Chapter 81 — Completion Snapshot

## Core Theory

- [ ] Database as correctness boundary
- [ ] Connection pools
- [ ] Transactions
- [ ] ACID
- [ ] Isolation levels
- [ ] Concurrency anomalies
- [ ] Constraints
- [ ] Indexes
- [ ] Query planning
- [ ] N+1
- [ ] Pagination
- [ ] Optimistic concurrency
- [ ] Pessimistic locking
- [ ] Deadlocks
- [ ] Retry semantics
- [ ] Schema evolution
- [ ] Migrations
- [ ] Read replicas
- [ ] Replica lag
- [ ] Caching
- [ ] Outbox
- [ ] Multi-tenancy
- [ ] Security
- [ ] Observability

## Implementation

- [ ] Create database pool
- [ ] Implement repository
- [ ] Implement parameterized queries
- [ ] Implement transaction helper
- [ ] Add constraints
- [ ] Add indexes
- [ ] Implement pagination
- [ ] Implement optimistic concurrency
- [ ] Implement retry classification
- [ ] Handle deadlocks
- [ ] Implement migrations
- [ ] Implement outbox
- [ ] Add tenant isolation
- [ ] Add query metrics
- [ ] Add slow-query detection
- [ ] Test pool exhaustion
- [ ] Test rollback
- [ ] Test concurrent writes
- [ ] Test migration compatibility
- [ ] Test recovery

## Interview / Reasoning

- [ ] Explain transaction boundary
- [ ] Explain ACID
- [ ] Explain isolation
- [ ] Explain N+1
- [ ] Explain pooling
- [ ] Explain deadlocks
- [ ] Design concurrency control
- [ ] Design zero-downtime migration
- [ ] Design replica strategy
- [ ] Design cache consistency
- [ ] Design outbox reliability
- [ ] Defend ORM vs SQL

## Mastery Gate

```text
Understand      [ ]
Explain         [ ]
Predict         [ ]
Implement       [ ]
Debug           [ ]
Apply           [ ]
Compare         [ ]
Defend          [ ]
```

## Final Principal Test

Given an unfamiliar Node.js system using a database, can you determine:

```text
What database is used?
Why was it chosen?
What owns the data?
Where are transactions?
Which invariants are database-enforced?
Which are application-enforced?
How large is the connection pool?
How many application instances exist?
Can the pool saturate the database?
Where are the slow queries?
Where are N+1 patterns?
How are indexes chosen?
What isolation level is used?
How are concurrent writes protected?
What happens on deadlock?
What gets retried?
What is idempotent?
Can a process crash after commit?
How are events published?
Are replicas used?
Can reads be stale?
What is cached?
How is cache invalidation handled?
How are migrations deployed?
How are old and new application versions kept compatible?
How is tenant isolation enforced?
What database permissions does the application have?
What sensitive data can reach logs?
What happens if the database is unavailable?
What is the recovery strategy?
```

A principal engineer does not stop at:

> “The query works.”

They ask:

> **What happens under concurrency, failure, load, schema change, retry, tenant isolation, and partial system failure?**

---

## Principal Database Decision Framework

For every significant database choice, record:

```text
Business Invariant:
Access Pattern:
Data Ownership:
Transaction Boundary:
Consistency Requirement:
Expected Read Load:
Expected Write Load:
Query Shape:
Index Strategy:
Concurrency Model:
Failure Modes:
Retry Model:
Security Boundary:
Tenant Model:
Migration Strategy:
Observability:
Recovery Strategy:
Cost:
Operational Complexity:
Future Change:
Decision:
Revisit Trigger:
```

The strongest database integration is not the abstraction with the fewest SQL statements.

It is the design that makes **state ownership, correctness, concurrency, performance, failure behavior, and schema evolution explicit enough to operate safely at production scale.**