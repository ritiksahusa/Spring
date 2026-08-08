# Session 6 — Performance and Database Interaction

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** JDBC batching -> bulk DML -> persistence-context memory -> indexes -> query plans -> pools -> observability

## Session Outcome

You should be able to:

- distinguish ORM batching, JDBC batching, and bulk DML;
- design a bounded batch insert/update loop;
- explain why identifier strategy can affect insert batching;
- manage persistence-context memory during large jobs;
- connect query predicates to database indexes and execution plans;
- diagnose connection-pool exhaustion without blaming JPA generically;
- reason about slow-query latency, database time, and application time;
- instrument SQL and transaction behavior for production investigation;
- propose a measured performance fix with correctness safeguards.

---

## 1. Follow the Whole Path

A JPA performance issue can originate at any layer:

```text
API/request volume
    -> service transaction duration
    -> persistence-context size and fetch graph
    -> Hibernate SQL generation
    -> JDBC batching / round trips
    -> connection pool wait
    -> database locks, indexes, plan, I/O
```

Do not optimize an ORM method name without measuring the generated SQL and database behavior. A repository call can be fast in isolation while the endpoint is slow because of N+1, serialization, pool waiting, or a count query.

### Core measurements

- request latency percentiles;
- database time versus application time;
- SQL count and repeated query shape per request;
- rows returned and rows affected;
- connection acquisition/wait time;
- active/idle pool metrics;
- transaction duration;
- flush duration;
- heap usage and persistence-context growth;
- database execution plan and lock waits.

---

## 2. JDBC Batching versus Bulk DML

### JDBC batching

Hibernate can group similar prepared statements into JDBC batches:

```text
insert into order_line (...) values (...)
insert into order_line (...) values (...)
insert into order_line (...) values (...)
        -> one/fewer JDBC round-trip batches
```

The database still processes individual row operations, but network and driver overhead can decrease.

### Bulk DML

A bulk `UPDATE` or `DELETE` changes many rows through one SQL statement:

```sql
update orders
set status = 'EXPIRED'
where status = 'PENDING'
  and expires_at < ?
```

Bulk DML is often efficient for set-based work, but it bypasses normal entity dirty checking and can leave the persistence context stale.

| Technique | Unit of work | Entity callbacks/dirty checking | Main risk |
| --- | --- | --- | --- |
| JDBC batching | many entity statements | usually participates | memory/order/configuration |
| Bulk DML | set-based SQL | bypassed | stale managed state |
| loop with individual flush | entity by entity | participates | round trips and slow jobs |

Choose based on whether entity lifecycle rules, callbacks, auditing, and invariants must run.

---

## 3. Batch Inserts and Updates

A bounded batch pattern:

```java
for (int index = 0; index < commands.size(); index++) {
    Order order = mapper.toEntity(commands.get(index));
    entityManager.persist(order);

    if ((index + 1) % batchSize == 0) {
        entityManager.flush();
        entityManager.clear();
    }
}
entityManager.flush();
entityManager.clear();
```

The final flush matters when the total count is not an exact multiple of the batch size.

### Why clear matters

Every managed entity can carry state, snapshots, relationship wrappers, and references. A single transaction with millions of managed objects can create heap pressure and make dirty checking expensive. `flush()` sends work; `clear()` releases tracking state.

### Transaction size

One giant transaction may provide atomicity but increase:

- lock duration;
- undo/redo or transaction-log pressure;
- recovery time;
- memory usage;
- failure blast radius.

Chunking into transactions can improve throughput and recovery, but it changes atomicity. Decide whether partial completion is acceptable and make retries idempotent.

### Identifier strategy

Some ID generation strategies, particularly identity columns, can require an insert before the identifier is known and may limit batching behavior. Sequence-based strategies with allocation can batch more effectively, depending on provider/database configuration. Verify with generated SQL rather than assuming all strategies behave the same.

### Ordering and batch settings

Hibernate can order inserts/updates by entity type to improve batching. Settings such as batch size and ordered operations are provider-specific. Larger is not always better: parameter limits, memory, lock behavior, and database workload matter.

---

## 4. Bulk DML Correctness

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Order o set o.status = :status where o.id in :ids")
int updateStatus(Collection<Long> ids, OrderStatus status);
```

Before bulk DML, pending managed changes may need to be flushed so the statement sees the intended database state. Afterward, managed objects may hold old values. Clear or reload before continuing with entity operations.

Bulk DML also bypasses or changes assumptions around:

- entity listeners;
- per-entity validation;
- auditing callbacks;
- optimistic version handling, depending on query;
- domain invariants implemented in entity methods.

Use bulk operations for a justified set-based use case and document which lifecycle behavior is intentionally skipped.

---

## 5. Persistence-Context Memory and Dirty-Checking Cost

A persistence context is not a free pointer collection. It can retain:

- managed entity references;
- original snapshots;
- collection wrappers and snapshots;
- identity-map entries;
- pending actions.

As the context grows, flush can compare more state and the heap can grow. Symptoms include long flushes, high GC, out-of-memory errors, and batch throughput declining over time.

### Bounded-context pattern

```text
read/process bounded chunk
    -> mutate managed entities
    -> flush
    -> clear
    -> release chunk references
```

Also avoid retaining processed entities in application collections. Clearing the persistence context does not help if the job itself stores every entity in a list.

### Read-only streaming

Streaming a result can reduce application memory but requires a transaction/resource plan. A cursor, fetch size, persistence context, and connection must be managed carefully. Do not return an open stream beyond the transaction or serialize it after resources close.

---

## 6. Indexes: Match the Access Pattern

An index can reduce search work, but it has write and storage costs. Design it from real predicates and ordering:

```sql
where customer_id = ?
  and status = ?
order by created_at desc, id desc
```

A candidate composite index might begin with equality columns and include ordering columns, but the right order depends on database, cardinality, query variants, and write volume. An index on every column is not a strategy.

### Verify

- Does the query use the intended index?
- How many rows does it scan versus return?
- Is the predicate selective?
- Does a function/cast prevent index use?
- Does the composite index support the actual prefix/order?
- Is the count query using a different plan?
- Is the index maintained cost acceptable for writes?

JPA annotations do not replace migrations and plan analysis. Keep schema/index definitions versioned and review them with query changes.

---

## 7. Query Plans and Slow SQL

A slow query investigation should capture:

1. exact SQL with representative bind values;
2. execution plan, ideally with actual runtime/row counts;
3. rows examined and returned;
4. lock/wait information;
5. concurrent load and pool wait time;
6. whether the query is repeated N+1 or count work.

A query can be slow because of:

- missing/wrong index;
- stale statistics;
- poor cardinality estimate;
- join explosion;
- large offset;
- sorting without support;
- implicit casts/functions;
- lock waits rather than CPU;
- network/result-transfer volume.

Do not conclude “the database is slow” from one application timer. Separate connection acquisition, execution, result mapping, and serialization.

---

## 8. Connection Pool Interaction

The pool is a finite concurrency boundary. A request may wait for a connection because:

- transactions are too long;
- slow queries hold connections;
- remote calls occur inside transactions;
- pool size is too small for the workload;
- connections leak or are not returned;
- `REQUIRES_NEW` needs additional connections;
- database max connections or network failures reduce capacity.

Increasing pool size can make the database less stable by increasing concurrent work. First measure:

```text
pool wait time
active connections
idle connections
transaction duration
query duration
DB CPU/locks/connections
```

A healthy pool is not “as large as possible.” It is sized with database capacity, request concurrency, and transaction duration in mind.

### Little's Law intuition

If arrival rate rises or average transaction time rises, the number of concurrently occupied connections rises. Reducing transaction duration and query count often helps more than blindly increasing the pool.

---

## 9. Observability

Useful signals:

- SQL count per request or use case;
- slow-query log with correlation ID;
- transaction duration and rollback count;
- flush duration;
- pool acquisition wait and active count;
- rows returned/updated;
- endpoint cardinality and page size;
- database lock waits/deadlocks;
- heap/GC around batch jobs.

Logging bind values can expose sensitive data. Use safe sampling, redaction, and environment-specific levels. Production diagnosis should not require turning on unlimited SQL logging for every request.

### A practical investigation record

```text
Symptom:
Endpoint/job:
Time window:
Request volume:
SQL count and repeated shapes:
Slowest SQL and actual plan:
Pool wait:
Transaction duration:
Rows returned/affected:
Recent mapping/query/schema changes:
Hypothesis:
Smallest experiment:
Expected signal:
Rollback plan:
```

This forces evidence-based debugging.

---

## 10. Scenario Reasoning

### Scenario A: batch job slows over time

A job processes 500,000 rows. The first batches are fast, later batches slow, and heap grows.

**Likely causes:** one persistence context retains all entities, dirty checking grows, application lists retain references, or transaction/log pressure increases.

**Fix direction:** bounded flush/clear, release references, chunk transactions if atomicity allows, measure flush time and heap.

### Scenario B: pool exhaustion after adding an audit call

An outer transaction calls an audit service marked `REQUIRES_NEW` in a high-concurrency endpoint.

**Reasoning:** outer connections are suspended while inner calls acquire additional connections. Pool demand can approach two connections per request. Check pool wait and transaction duration before changing size.

### Scenario C: index added but query remains slow

The query filters by customer/status and sorts by timestamp, but the plan scans many rows.

**Reasoning:** index column order, selectivity, stale statistics, data distribution, or a different count query may be the issue. Inspect actual plan with representative values.

### Scenario D: bulk update followed by wrong response

A job bulk-updates statuses, then returns previously loaded entities.

**Reasoning:** the persistence context is stale. Flush/clear/reload or return a bulk-operation result rather than old managed objects.

---

## 11. Performance Fix Checklist

Before changing code:

- reproduce with realistic data volume;
- capture SQL count and exact SQL;
- measure pool wait and transaction duration;
- inspect the query plan;
- identify whether correctness depends on entity callbacks/dirty checking;
- define expected improvement.

After changing code:

- compare p50/p95/p99 latency;
- compare DB CPU and rows scanned;
- check query count and flush time;
- verify transaction/rollback behavior;
- run concurrency and large-data tests;
- check memory and pool metrics;
- keep a rollback path.

Never declare a performance fix from a local timing alone.

---

## 12. SDE-2 Interview Answers

### JDBC batching versus bulk update?

JDBC batching groups many similar entity SQL statements into fewer driver round trips while preserving per-row ORM operations. Bulk DML changes a set of rows directly in one SQL statement and bypasses per-entity dirty checking, so it can be faster but can leave the persistence context stale and skip lifecycle logic.

### Why use flush plus clear in a batch?

Flush synchronizes the current chunk with the database; clear detaches entities and releases persistence-context tracking/snapshots. Together they bound memory and dirty-checking cost, but transaction chunking and atomicity still need separate decisions.

### Why can a large pool make things worse?

It can increase simultaneous database work beyond database CPU, I/O, lock, or connection capacity. Pool size controls waiting, not query efficiency or database throughput.

### How do you investigate a slow JPA endpoint?

Measure request/database/pool time, count SQL and identify repeated shapes, capture representative SQL and actual execution plans, inspect row counts/locks/indexes, then test the smallest fetch/query/schema change with before/after metrics.

---

## 13. Practice Batch

1. Why does `flush()` without `clear()` fail to bound persistence-context memory?
2. Why can identity ID generation affect batching?
3. What evidence distinguishes pool exhaustion from a slow database query?
4. Why can a bulk DML fix introduce stale response data?
5. What should an SDE-2 engineer include in a slow-query diagnosis?

### Model answer key

1. Flush sends SQL but leaves managed objects, snapshots, and collection state in the context. The context can continue growing.
2. The insert may need to execute to obtain the generated ID, limiting how many inserts can be deferred/grouped. Provider and database behavior must be verified.
3. Pool metrics show connection-acquisition wait and active saturation; query timing/plan shows database execution or lock time. Both can coexist, so separate them.
4. Bulk DML bypasses entity state tracking. Previously loaded managed objects can still expose old values until the context is cleared or entities are reloaded.
5. Exact SQL/binds, query count, actual plan, rows scanned/returned, lock waits, pool wait, transaction duration, load context, recent changes, hypothesis, and measured experiment.

---

## Session Handoff

**Prepared study material:** Session 6 — Performance and Database Interaction  
**Topics covered:** JDBC batching, bulk DML, batch memory, flush/clear, transaction chunking, identifier strategies, indexes, query plans, connection pools, observability, and production diagnosis.  
**Next session:** Session 7 — Hibernate Internals  
**Next starting topic:** JPA versus Hibernate architecture -> `Session`/`EntityManager` -> persistence-context internals and dirty checking

## One-Line Revision Summary

Optimize the full path with evidence: bound the persistence context, distinguish batching from bulk DML, align indexes with real queries, separate pool wait from DB execution, and measure every fix against correctness and production signals.
