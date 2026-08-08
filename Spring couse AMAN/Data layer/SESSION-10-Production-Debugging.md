# Session 10 — Production Debugging

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** production symptoms -> evidence -> JPA/Hibernate hypotheses -> safe fixes -> incident reasoning

## Session Outcome

You should be able to:

- build an evidence-first debugging path for JPA incidents;
- diagnose N+1, connection-pool exhaustion, long transactions, lazy-loading failures, unexpected updates, excessive selects, batch memory, deadlocks, and stale state;
- separate application, ORM, JDBC, pool, and database causes;
- choose mitigations that protect correctness while reducing impact;
- explain a production incident in an SDE-2 interview with symptoms, hypothesis, evidence, root cause, fix, and prevention;
- avoid common “fixes” that merely hide the problem.

---

## 1. The Production Debugging Loop

Use this loop for every incident:

```text
symptom
  -> scope and impact
  -> correlated evidence
  -> ranked hypotheses
  -> smallest safe experiment/mitigation
  -> verify metrics and correctness
  -> root cause and prevention
```

Capture:

- affected endpoint/job/tenant;
- start time and deploy/config/schema changes;
- p50/p95/p99 latency and error rate;
- request volume and page sizes;
- SQL count/shapes and slowest statements;
- transaction duration and rollback count;
- pool active/wait metrics;
- database CPU, locks, waits, rows scanned;
- heap/GC for batch jobs;
- exact exception and stack location.

Do not change five settings at once. It destroys causal evidence and makes rollback harder.

---

## 2. Symptom-to-Layer Map

| Symptom | First evidence | Likely layers |
| --- | --- | --- |
| latency grows with result count | SQL count/request | fetch plan, N+1, serialization |
| pool timeout | connection wait/active count | transaction duration, slow SQL, pool, remote calls |
| lazy exception | stack trace and boundary | transaction scope, fetch plan, OSIV |
| unexpected UPDATE | SQL and entity path | dirty checking, merge, flush, mapping |
| many SELECTs | repeated SQL/binds | lazy graph, mapper, repository loop |
| heap growth in job | heap/context/flush time | persistence context, retained references, batch size |
| deadlocks | DB deadlock graph | lock order, indexes, transaction duration |
| stale response | cache/context timeline | bulk DML, L2/cache invalidation, isolation |
| slow page | content/count plans | indexes, joins, count, offset, row multiplication |

The map chooses the first instrument, not the final diagnosis.

---

## 3. N+1 Incident

### Symptom

An order-list endpoint changes from 150 ms to 4 seconds as page size increases. Database query count is close to one plus the number of orders.

### Investigation

1. Add a request correlation ID to SQL metrics/logs.
2. Count root query and repeated relationship queries.
3. Identify the mapper/serializer access causing initialization.
4. Compare query count at 10, 50, and 100 roots.
5. Inspect response fields and required graph.

### Worked trace: before and after

Bad path:

```java
List<Order> orders = repository.findRecent(pageable); // 1 query
return orders.stream()
  .map(order -> new OrderRow(order.getId(),
           order.getCustomer().getName()))
  .toList();                                        // N lazy loads
```

Observed SQL shape for three orders:

```text
1. select ... from orders order by created_at desc limit 3
2. select ... from customer where id = 11
3. select ... from customer where id = 14
4. select ... from customer where id = 19
```

Possible fix for a summary endpoint:

```java
@Query("""
       select new com.example.api.OrderRow(o.id, c.name)
       from Order o join o.customer c
       order by o.createdAt desc, o.id desc
       """)
Slice<OrderRow> findRecentRows(Pageable pageable);
```

Now the intended shape is one query returning only the fields the endpoint needs. Verify the generated SQL, count behavior, indexes, and stable ordering; a fetch join or batch fetch may be better for a different response shape.

### Fix options

- DTO query for summary fields;
- fetch join/entity graph for a small to-one graph;
- batch fetching for repeated lazy loads;
- two-step load for a page plus collection;
- remove accidental serialization traversal.

### Prevention

- query-count integration tests for critical endpoints;
- endpoint-specific DTOs;
- SQL/latency dashboards;
- code review checklist for collection mapping.

Do not make every relation EAGER as the emergency fix.

---

## 4. Connection-Pool Exhaustion

### Symptoms

Requests fail with pool timeout while database CPU is moderate, or active connections stay near maximum.

### Ranked hypotheses

- long transactions hold connections;
- slow query or lock wait holds connections;
- remote call happens inside transaction;
- `REQUIRES_NEW` doubles connection demand;
- connection leak/resource configuration;
- pool is too small for legitimate concurrency;
- database/network failures reduce usable connections.

### Evidence

```text
connection acquisition wait
active/idle/max pool count
transaction duration
query/lock duration
thread stacks at pool wait
REQUIRES_NEW call paths
DB connection count and wait events
```

### Fix order

1. stop or reduce harmful traffic if needed;
2. identify held-connection duration;
3. remove remote calls from transaction where design permits;
4. fix slow SQL/lock contention;
5. review propagation and pool/database capacity;
6. resize only after the workload is understood.

Increasing the pool can make lock and database contention worse.

---

## 5. Long-Running Transactions

### Symptoms

Large transaction duration, stale locks, growing persistence contexts, delayed commits, or rollback of a huge unit of work.

### Causes

- remote API calls inside transaction;
- user interaction or file processing inside transaction;
- unbounded batch;
- slow query or lock wait;
- accidental lazy graph traversal;
- OSIV/serialization work associated with request lifecycle;
- a transaction spanning unrelated operations.

### Fix

Define the business atomicity first. Then:

- reduce the transaction to database work;
- use separate transactions for chunks if partial completion is acceptable;
- use an outbox/workflow for remote side effects;
- flush/clear bounded batches;
- instrument transaction duration and connection hold time.

Do not split a transaction merely to make a metric look smaller if the business operation must remain atomic.

---

## 6. `LazyInitializationException`

### Symptom

A controller, mapper, serializer, or asynchronous task accesses a lazy association after the transaction/context closes.

### Diagnosis questions

- Where was the entity loaded?
- When did the persistence context close?
- Which exact field triggered access?
- Does the response really need that relationship?
- Is OSIV hiding the issue in some environments?

### Corrective design

Load/map inside a service transaction with a fetch join, graph, DTO query, or batch plan. Return data, not a live entity graph that requires later lazy access.

For asynchronous work, pass identifiers/commands and load in the worker transaction. Do not pass detached entities assuming their lazy associations remain usable.

---

## 7. Unexpected `UPDATE`s

### Symptoms

An endpoint that should be read-only emits updates, or a save path updates many columns.

### Likely causes

- managed entity mutated during mapping or serialization;
- `merge()` copies a detached object with stale/full state;
- dirty checking sees a changed value or collection;
- audit timestamps update on flush;
- cascade merge traverses a graph;
- bulk/native path or trigger changes values;
- entity equality/collection mutation produces association changes.

### Investigation

- capture SQL and binds;
- identify entity state at mutation;
- inspect transaction read-only/flush mode;
- compare before/after fields and snapshots if available;
- trace mapper/setter/domain method;
- check merge return-value usage;
- inspect cascade mappings and listeners.

### Fix

Use read-only DTO projections for reads, load managed entities and mutate only intended fields, validate detached merge flows, and keep audit/cascade behavior explicit. Do not disable dirty checking globally to hide an accidental mutation.

---

## 8. Excessive `SELECT`s

### Symptom

A request has many selects but no obvious collection loop.

### Investigation paths

- lazy to-one access in mapper;
- `find()` called repeatedly instead of using identity map/bulk query;
- existence/count query per row;
- authorization check per entity;
- query-cache misses/entity-cache misses;
- pagination count query;
- proxy initialization from logging/serialization;
- repository call inside a stream/loop.

### Fix choices

- batch IDs and query once;
- fetch the required to-one graph;
- use projection/aggregation;
- batch lazy loads;
- move authorization logic to a query predicate where safe;
- cache stable reference data with an invalidation contract;
- remove accidental `toString()`/serialization traversal.

Measure result cardinality and semantics before replacing queries with broad joins.

---

## 9. Batch Processing and Persistence-Context Pressure

### Symptoms

A nightly job runs out of memory or slows after millions of rows.

### Evidence

- heap retained by managed entities/snapshots;
- flush time increasing per chunk;
- no `clear()` or too-large batch;
- application collection retaining processed objects;
- one transaction covering entire job;
- database transaction log/locks growing.

### Fix

Use bounded chunks, flush/clear safely, release references, select only required data, and choose transaction chunking based on recovery/atomicity. For set-based operations, consider bulk DML or database-native processing while documenting skipped entity lifecycle behavior.

Test restartability and duplicate handling. A fast batch that cannot safely restart is not production-ready.

---

## 10. Deadlocks and Lock Contention

### Symptoms

Deadlock exceptions, lock timeouts, high DB wait time, or requests blocked behind one transaction.

### Evidence

- database deadlock graph/lock report;
- transaction SQL sequence and row order;
- query plan/index coverage;
- transaction duration and remote calls;
- concurrent code paths touching the same tables.

### Fix

Use consistent lock order, shorter transactions, narrow predicates, useful indexes, bounded timeouts, and idempotent retry where appropriate. Separate remote workflows from held locks.

A deadlock is a graph problem. Changing one timeout may reduce visibility without removing the cycle.

---

## 11. Transaction-Boundary Bugs

### Symptoms

- writes do not persist;
- updates appear only sometimes;
- rollback does not happen after caught exception;
- self-invocation bypasses transaction;
- lazy access works in one environment but not another;
- an inner operation commits unexpectedly.

### Diagnostic checklist

- Did the call cross a Spring proxy?
- Which method owns the transaction?
- What propagation applies?
- Is the EntityManager joined to the transaction?
- When did flush happen?
- Was the transaction marked rollback-only?
- Did OSIV extend access beyond service work?
- Did `REQUIRES_NEW` suspend the outer transaction?

Trace an actual request rather than relying on annotation placement alone.

---

## 12. Slow Query and Count-Query Incident

### Symptom

The content query is fast, but paged endpoint latency is high.

### Possible cause

The `Page` count query scans a large table or retains complex joins. Collection fetch and generated count transformations can make the count plan worse than the content plan.

### Fix

Capture both SQL statements and actual plans. Consider `Slice`, a custom count query, a simpler count predicate, keyset pagination, or a separate search-count product requirement. Do not remove total metadata without confirming the client contract.

---

## 13. Production Incident Case Studies

### Case 1: “The API became slow after adding a field”

A response mapper starts reading `customer.address.city`. Query count rises from 2 to 202.

**Root cause:** lazy traversal introduced N+1.  
**Fix:** DTO query/fetch plan for the required fields.  
**Prevention:** query-count test and endpoint SQL dashboard.

### Case 2: “Pool timeouts after payment integration”

A transaction locks an order, calls a payment provider for 3 seconds, then commits. Traffic spikes.

**Root cause:** connections and locks are held during remote latency.  
**Fix:** redesign reservation/state transition and idempotent asynchronous workflow.  
**Prevention:** transaction duration and pool-wait alerts.

### Case 3: “Bulk cleanup returned old data”

A scheduled bulk update changes statuses. The same job returns entities loaded before the update.

**Root cause:** persistence context/cache stale after bulk DML.  
**Fix:** flush/clear/reload or return update counts.  
**Prevention:** bulk-DML invalidation contract and integration test.

### Case 4: “Only production has lazy exceptions”

Local OSIV is enabled; production disables it. Controller serialization accesses lazy lines.

**Root cause:** API relied on context scope outside service transaction.  
**Fix:** fetch/map DTO inside service transaction.  
**Prevention:** environment parity and boundary tests.

### Case 5: “Deadlock retries increased failure rate”

A retry loop immediately retries deadlock victims with no backoff.

**Root cause:** retries amplified contention and lock order was inconsistent.  
**Fix:** consistent order, short critical section, bounded jittered retry.  
**Prevention:** deadlock graph monitoring and concurrency tests.

---

## 14. Safe Incident Response Checklist

During impact:

- identify affected endpoint/job and severity;
- stop dangerous traffic or disable an optional path if possible;
- preserve logs/metrics/plans before changing levels;
- apply the smallest reversible mitigation;
- avoid destructive cache/database operations without scope;
- communicate correctness risk, not only latency.

After stabilization:

- reproduce with representative data;
- identify root cause and contributing factors;
- add a regression test/metric/alert;
- document rollback and operational runbook;
- review whether the fix changed transaction, cache, or consistency semantics.

---

## 15. SDE-2 Incident Answer Framework

Use this structure in an interview or incident review:

1. **Symptom:** what users/metrics observed.
2. **Scope:** affected path, volume, and time.
3. **Evidence:** SQL, traces, pool, DB plan, heap, locks.
4. **Hypotheses:** ranked by evidence.
5. **Experiment/mitigation:** smallest safe test.
6. **Root cause:** the mechanism, not the symptom.
7. **Fix:** code/config/schema/transaction change.
8. **Prevention:** test, metric, alert, review rule.
9. **Trade-off:** consistency, throughput, cost, or complexity.

This demonstrates production judgment rather than naming a fashionable annotation.

---

## 16. Practice Batch

1. An endpoint's query count grows linearly with result size. What do you inspect first?
2. A pool is exhausted but database CPU is low. Name three plausible causes.
3. A `LazyInitializationException` happens only after deployment. What boundary questions do you ask?
4. A batch job's flush time grows continuously. What does that suggest?
5. How do you explain a deadlock fix without claiming retries solve the root cause?

### Model answer key

1. Correlate request SQL and look for repeated parameterized lazy/loop queries, then inspect mapper/serialization access.
2. Long transactions, lock waits/slow queries, remote calls inside transactions, `REQUIRES_NEW` connection demand, leaks, or a pool/database configuration mismatch.
3. Compare OSIV, service transaction scope, serializer timing, async boundaries, and the exact lazy field access. Fix the fetch plan inside the transaction.
4. The persistence context may be growing, snapshots/dirty-checking are expanding, references are retained, or database contention is increasing. Measure heap, context clearing, and DB time.
5. State that retries are bounded resilience after consistent lock order/short transactions/index fixes; they do not remove the wait cycle and can amplify load without backoff.

---

## Session Handoff

**Prepared study material:** Session 10 — Production Debugging  
**Topics covered:** evidence-first runbook, N+1, pool exhaustion, long transactions, lazy failures, unexpected updates, excessive selects, batch memory, deadlocks, transaction bugs, count-query incidents, case studies, and prevention.  
**Next session:** Session 11 — JPA/Hibernate Rapid-Fire Revision  
**Next starting topic:** short questions across lifecycle, repositories, relationships, transactions, fetching, performance, internals, locking, and caching

## One-Line Revision Summary

Production debugging is a layered evidence problem: correlate symptoms with SQL, transaction, pool, heap, lock, and plan data, then choose a reversible fix that preserves correctness and prevents recurrence.
