# Session 12 — Full SDE-2 Mock Interview

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** architecture -> entity design -> repository internals -> transactions -> fetching -> concurrency -> performance -> production debugging

## How to Run the Mock

Treat this as an interview, not a reading exercise.

- Answer aloud before opening the model points.
- Spend 3–5 minutes on a normal question and 8–12 minutes on a design/debugging scenario.
- Think in layers: API/service -> transaction -> persistence context -> Hibernate/JDBC -> database.
- State assumptions when the prompt is underspecified.
- Do not claim a SQL statement or provider behavior is universal without configuration/evidence.
- The interviewer is evaluating reasoning, trade-offs, and clarity more than annotation recall.

### Evaluation rubric

| Dimension | Strong answer |
| --- | --- |
| Mental model | names the correct abstraction/layer |
| Correctness | lifecycle, transaction, and concurrency claims are accurate |
| Design | maps domain and schema deliberately |
| Performance | predicts query/row/connection behavior |
| Operations | asks for evidence and proposes reversible fixes |
| Communication | states assumptions, trade-offs, and conclusion clearly |

Score each round from 0–3:

- `0`: fundamental misconception or no workable approach;
- `1`: partial recall, missing layer/trade-off;
- `2`: mostly correct and workable;
- `3`: precise, scenario-driven, production-ready.

---

## Round 1 — Core Mental Model

### Question 1

Explain the persistence context to a new engineer. Include identity, managed state, dirty checking, and flush.

**Listen for:**

- unit of work/identity map;
- one managed instance per entity key in the context;
- dirty checking only for managed state;
- write-behind and flush;
- commit still determines durability.

### Question 2

Compare `detach`, `clear`, `refresh`, and `remove` in one answer.

**Listen for:**

- one versus all;
- reload while remaining managed;
- REMOVED state and scheduled deletion;
- none of these means commit/rollback by itself.

### Question 3

Why can `merge` produce a bug when the return value is ignored?

**Listen for:**

- state transfer into a managed instance;
- supplied object can remain detached;
- later mutation of the original may not be dirty-checked.

### Model answer shape

> The persistence context is the unit of managed identity and change tracking. `detach` removes one object, `clear` detaches all, `refresh` reloads one while keeping it managed, and `remove` marks one managed entity for deletion. Dirty checking and write-behind synchronize managed changes at flush; commit, not flush alone, determines durability.

---

## Round 2 — Entity and Schema Design

### Prompt

Design the persistence model for:

- customers place orders;
- each order has many lines;
- a line references a product;
- line quantity and purchase price are required;
- products are shared and cannot be deleted when an order is deleted;
- order history must retain the price at purchase;
- an order-list API should not load lines.

Talk through entities, FKs, ownership, cascades, orphan removal, fetch, and API DTOs.

### Strong design points

```text
Customer 1 -- * Order 1 -- * OrderLine * -- 1 Product
```

- `Order.customer` owns a non-null `customer_id` FK and is usually lazy.
- `Customer.orders` is inverse with `mappedBy` and helper methods.
- `Order.lines` is private child state; `cascade = {PERSIST, MERGE}` and `orphanRemoval = true` may fit if line lifecycle is private.
- `OrderLine.order` owns its FK and is non-null.
- `OrderLine.product` is many-to-one without remove cascade.
- `OrderLine` stores purchase price/description snapshot.
- list endpoint uses DTO/projection and stable ordering; it does not serialize the graph.
- database constraints enforce non-null/positive/unique rules where appropriate.

### Follow-up

Why is a direct many-to-many between order and product weaker here?

**Answer:** It cannot naturally store quantity, purchase price, discounts, or order-specific lifecycle. The association is a business entity.

---

## Round 3 — Spring Data `save()`

### Question 1

Trace `repository.save(entity)` from service call to possible SQL.

**Strong answer:**

1. Call crosses the repository proxy.
2. Standard implementation such as `SimpleJpaRepository` receives entity metadata.
3. New-state detection chooses `persist` or `merge`.
4. New path makes original managed; existing path returns managed merge result.
5. SQL is normally synchronized at flush, not guaranteed at the source line.
6. Commit/rollback determines the final database outcome.

### Question 2

An entity has an assigned non-null ID but is new. What can go wrong?

**Answer:** ID presence can cause existing classification and a merge path. Configure reliable new-state detection, for example `Persistable` with a tested lifecycle strategy, and test new/reloaded/detached cases.

### Question 3

When is `save()` unnecessary?

**Answer:** When a managed entity is loaded and mutated within a transaction. Dirty checking can synchronize it. `save` may still express an API operation, but it does not replace the transaction.

### Question 4

Derived query or `@Query`?

**Answer:** Derived methods suit short stable predicates. Use JPQL for explicit entity query shape, joins, grouping, or fetch plans; use native SQL for justified database-specific/complex queries. Measure SQL and plans.

---

## Round 4 — Transaction Tracing

### Prompt

```java
@Transactional
public void approve(Long id) {
    Loan loan = loans.findById(id).orElseThrow();
    loan.approve();
    auditService.record(loan);
    paymentClient.reserve(loan.paymentReference());
}
```

Discuss transaction boundary, flush, remote call, failure, and redesign.

### Strong answer points

- External call enters through Spring proxy only if the service call crosses it.
- The transaction usually joins/creates a persistence context and database transaction.
- `loan.approve()` is tracked; SQL may flush before query/commit.
- Holding a DB connection/locks during the payment call can increase latency/contention.
- If payment fails, local DB work may roll back, but remote side effect has different consistency.
- Use state transition/idempotency/outbox/workflow; keep database transaction focused.
- If audit must survive outer rollback, `REQUIRES_NEW` changes atomicity and pool demand; use deliberately.

### Follow-up

Why can a caught exception still roll back?

**Answer:** It may have already marked the transaction rollback-only, or an outer transaction may classify the outcome for rollback. Conversely, catching a checked exception may allow commit unless rollback rules are configured.

---

## Round 5 — N+1 Production Scenario

### Prompt

An endpoint loads 50 orders in 180 ms locally. In production, it takes 3 seconds and logs 51 nearly identical selects. The response includes customer name and total, not lines.

### Strong diagnosis

- one root query plus 50 lazy customer queries is N+1;
- correlate SQL to request and verify query count at different page sizes;
- use a DTO query selecting order id/status/total/customer name or a safe to-one fetch plan;
- add stable ordering and appropriate indexes;
- do not make all mappings EAGER or rely on OSIV;
- add query-count/performance regression coverage.

### Follow-up

What if the endpoint also needs lines?

**Answer:** A collection fetch join with pagination can multiply rows. Page root IDs then fetch the graph in a second query, or use a purpose-built projection/aggregation.

---

## Round 6 — Concurrency Scenario

### Prompt

Two checkout requests see stock 1 and both succeed. Design a fix.

### Candidate approaches

**Optimistic:**

- add `@Version`;
- one update succeeds, the other gets a conflict;
- retry only after fresh read and safe command revalidation;
- return conflict/unavailable response when stock is gone.

**Pessimistic:**

- load row with `PESSIMISTIC_WRITE` in a short transaction;
- recheck stock while lock is held;
- decrement and commit;
- use timeout/deadlock handling.

**Conditional update:**

```sql
update inventory
set available = available - 1,
    version = version + 1
where id = ?
  and available > 0
  and version = ?
```

Check affected-row count. This can be efficient but must be integrated with transaction, cache, and entity-state handling.

### Trade-off question

Which strategy would you choose?

**Expected reasoning:** contention rate, critical-section duration, user retry ability, inventory correctness, database capacity, and idempotency. There is no universal answer.

---

## Round 7 — Performance and Database Interaction

### Prompt

A nightly import processes 2 million rows. It starts at 20,000 rows/minute, falls to 2,000, and eventually runs out of memory.

### Strong diagnosis

- one persistence context likely retains entities/snapshots;
- dirty-checking and flush cost grow; application collections may also retain rows;
- use bounded `flush()` + `clear()` and release references;
- consider transaction chunking if partial completion/restart semantics are acceptable;
- use JDBC batching for entity lifecycle-compatible writes;
- use bulk/set-based SQL when callbacks/invariants do not need per-entity execution;
- inspect ID strategy, batch settings, DB indexes, transaction log, and lock time;
- test restart/idempotency and verify generated SQL.

### Follow-up

Why is `clear()` alone insufficient?

**Answer:** It detaches but does not send pending work. Without flush, changes may not reach the database. The usual bounded pattern is flush then clear at a safe chunk boundary.

---

## Round 8 — Hibernate Internals

### Questions

1. What does write-behind mean?
2. What does dirty checking use?
3. Why does Hibernate have an action queue?
4. What is a proxy?
5. How does a flush get triggered?
6. Why can batch settings not produce a batch?

### Strong points

- managed state changes are queued and translated to SQL at flush;
- snapshots or enhanced dirty markers identify changes;
- action queue orders inserts/updates/deletes/collection actions for constraints and batching;
- proxies/persistent collections defer loads;
- explicit flush, commit, or provider/query synchronization can trigger flush;
- identity IDs, differing SQL shapes, frequent flushes, constraints, and driver/database limitations reduce batching.

Do not claim internal private class names are stable API.

---

## Round 9 — Caching

### Prompt

A team wants to cache inventory entities because reads are frequent.

### Strong response

- inventory correctness must come from version/lock/conditional-update transactions;
- a stale cache may be acceptable for display but not allocation;
- identify first-level versus L2 versus application cache;
- define scope, invalidation, rollback behavior, multi-node coordination, hit ratio, and stale tolerance;
- bulk/native writes need explicit cache invalidation;
- fix N+1/index/query problems before adding cache complexity.

### Follow-up

What does query cache store?

**Answer:** A query key/result representation, commonly identifiers or scalar results, not necessarily full entity state. It can still require entity-cache/database work and can suffer high invalidation/cardinality costs.

---

## Round 10 — Production Incident

### Prompt

After a deployment, API p99 doubles, pool wait rises, and deadlock errors appear. No single SQL query is dramatically slower.

### Investigation plan

1. Compare deployment/config/mapping/query changes.
2. Correlate request, transaction, SQL, and pool metrics.
3. Inspect active connection/transaction duration and lock graphs.
4. Look for new `REQUIRES_NEW`, remote calls inside transactions, fetch/query count changes, and lock-order changes.
5. Capture deadlock reports and SQL sequence.
6. Apply reversible traffic/feature mitigation.
7. Fix root cause: shorten transactions, align lock order, remove N+1/slow count, or correct propagation.
8. Add alerts/regression tests/concurrency tests.

### Strong communication

State uncertainty and the next discriminating measurement. Do not say “increase the pool” until connection hold time and database capacity are understood.

---

## Final Rapid-Fire Set

Answer in one or two precise sentences:

1. `detach` versus `clear`?
2. `refresh` versus reload after clear?
3. `remove` versus delete SQL?
4. Flush versus commit?
5. Why use merge result?
6. `mappedBy`?
7. Cascade versus orphan removal?
8. Lazy versus EAGER?
9. N+1?
10. Page versus Slice?
11. `REQUIRED` versus `REQUIRES_NEW`?
12. Isolation versus propagation?
13. Self-invocation?
14. JDBC batch versus bulk DML?
15. `@Version`?
16. Pessimistic write?
17. Deadlock?
18. First-level versus second-level cache?
19. Query cache?
20. Production debugging first step?

### Answer key

1. One managed entity versus all entities in the current context; neither commits/rolls back.
2. Refresh keeps the entity managed and reloads it; clear detaches, requiring a new managed lookup/merge path.
3. Remove schedules a managed entity for deletion; SQL timing/durability are flush/transaction concerns.
4. Flush sends ORM SQL; commit makes the transaction durable.
5. Existing save/merge can return a different managed object; its mutations are tracked.
6. Inverse mapping points to the owning Java property, not a column.
7. Cascade propagates operations; orphan removal deletes a private child on disassociation.
8. Fetch plan choices; EAGER is not an efficient-query guarantee.
9. One root query plus N relationship/loop queries.
10. Page usually has total-count work; Slice indicates more without total.
11. Join/create versus suspend/create independent transaction.
12. Concurrent visibility versus transaction interaction.
13. Direct call bypasses transactional proxy advice.
14. Grouped per-entity statements versus set-based DML that bypasses entity tracking.
15. Version predicate detects stale writes.
16. Database-coordinated row lock for a short critical section.
17. Cyclic resource waits; fix ordering/critical section, then bounded retry if safe.
18. One context versus shared SessionFactory-level cache.
19. Query key/result representation with invalidation/cardinality trade-offs.
20. Scope symptom and collect correlated evidence before changing settings.

---

## Final Self-Assessment

| Round | Score 0–3 | Weak topic to revisit |
|---|---:|---|
| Core lifecycle |  |  |
| Entity design |  |  |
| `save()` |  |  |
| Transactions |  |  |
| Fetching/N+1 |  |  |
| Concurrency |  |  |
| Performance |  |  |
| Hibernate internals |  |  |
| Caching |  |  |
| Production incident |  |  |

Interpretation:

- **24–30:** interview-ready foundation; polish communication and trade-offs.
- **17–23:** workable foundation; revise the lowest-scoring two rounds.
- **0–16:** revisit fundamentals by layer, starting with lifecycle/transactions, then repeat the mock.

A score is evidence for planning, not a permanent label. Write down the exact missed mechanism and the smallest revision exercise.

---

## Final Track Handoff

**Prepared study material:** Session 12 — Full SDE-2 Mock Interview  
**Roadmap status:** All 12 study packs are prepared; personal mastery status must be updated after live answers and evaluation.  
**Next action:** Run the mock in batches, record weak spots, and revise only the lowest-scoring topics.  
**Do not restart:** mastered lifecycle fundamentals unless the mock exposes a genuine state/transaction misunderstanding.

## One-Line Revision Summary

An SDE-2 JPA answer should connect domain design to entity state, transaction boundary, fetch/query shape, database concurrency, operational evidence, and explicit trade-offs.
