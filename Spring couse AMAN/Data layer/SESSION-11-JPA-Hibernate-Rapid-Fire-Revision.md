# Session 11 — JPA/Hibernate Rapid-Fire Revision

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** broad interview revision across lifecycle, repositories, relationships, transactions, fetching, performance, internals, locking, caching, and debugging

## How to Use This Session

This is not a replacement for Sessions 1–10. Use it to find weak spots without restarting everything.

1. Answer each batch aloud before reading the model points.
2. Give yourself credit for correct reasoning even if wording is imperfect.
3. Mark only genuine conceptual gaps for revision.
4. For code traces, write the entity state, SQL possibility, and transaction outcome separately.
5. Do not confuse “I have seen this term” with “I can make a production decision.”

### Score guide

- **Strong:** correct mental model, trade-off, and scenario reasoning.
- **Working:** mostly correct with a terminology or edge-case gap.
- **Needs revision:** wrong layer, wrong lifecycle state, or unsafe production conclusion.

---

## Batch 1 — Persistence Context and Lifecycle

### Questions

1. What is the persistence context?
2. What is the difference between a managed and detached entity?
3. What does `persist()` do to a new entity?
4. What does `merge()` return, and what happens to the supplied object?
5. What does `detach(entity)` affect?
6. What does `clear()` affect?
7. What does `refresh(entity)` do?
8. What does `remove(entity)` mean before flush?
9. Why is flush not commit?
10. What does the first-level cache guarantee inside one context?

### Model points

1. A unit-of-work context that maintains managed entity identity, tracking, snapshots/dirty state, associations, and pending persistence actions.
2. Managed is tracked by the current context and eligible for dirty checking; detached is no longer tracked and mutations are not automatically synchronized.
3. It makes the original new object managed; an insert is synchronized at flush/transaction completion according to provider behavior.
4. It copies state into a managed instance and returns that managed instance; the supplied detached object remains detached.
5. One managed object is removed from tracking; it is not deleted or rolled back.
6. All managed objects in that context are detached and the first-level cache is cleared; it does not commit/rollback or undo flushed SQL.
7. It reads database state into a managed entity, overwriting unflushed local state while keeping the entity managed.
8. It marks a managed entity REMOVED; delete is normally synchronized at flush and becomes durable only on commit.
9. Flush sends SQL; commit determines transaction durability. Flushed work can still roll back.
10. Repeated lookup of the same entity key normally returns the same managed Java instance in that context.

---

## Batch 2 — Lifecycle Code Trace

### Code

```java
@Transactional
public void process(Long id) {
    Order order = entityManager.find(Order.class, id); // A
    order.setStatus(Status.PAID);                       // B
    entityManager.detach(order);                       // C
    order.setStatus(Status.SHIPPED);                    // D
    Order managed = entityManager.merge(order);         // E
    managed.setStatus(Status.DISPATCHED);               // F
    entityManager.refresh(managed);                     // G
    entityManager.remove(managed);                      // H
}
```

### Questions

1. State after A?
2. Is B eligible for dirty checking?
3. Does D automatically update the database?
4. Which object should be used after E?
5. What happens to F at G?
6. What state does H create?
7. Does H guarantee a durable delete immediately?

### Model points

1. `order` is MANAGED.
2. Yes, until detachment or transaction/context change.
3. No. The object is detached after C, so D is not automatically tracked.
4. `managed`, the returned managed instance. The original `order` remains detached.
5. `refresh()` reloads database state and discards the unflushed `DISPATCHED` value.
6. `managed` becomes REMOVED.
7. No. Flush can issue a delete; commit makes it durable and rollback can undo uncommitted work.

---

## Batch 3 — Spring Data JPA

### Questions

1. What is `JpaRepository`?
2. How does an interface method execute if there is no implementation class in application code?
3. What role does `SimpleJpaRepository` play?
4. What does `save()` usually do for a new entity?
5. What does `save()` usually do for an existing/detached entity?
6. Why use the return value of `save()`?
7. Why can an assigned identifier confuse new-state detection?
8. When is calling `save()` unnecessary?
9. What is a derived query?
10. What is the difference between JPQL and native SQL?
11. Why can a `Page` execute two queries?
12. When is `Slice` preferable?

### Model points

1. A Spring Data repository contract combining CRUD, paging/sorting, and JPA-specific operations.
2. Spring creates a proxy that dispatches standard methods, derived queries, declared queries, and custom fragments.
3. It is the standard repository implementation that uses entity metadata and `EntityManager` operations.
4. It usually calls `persist()` and returns the same managed instance.
5. It usually calls `merge()` and returns the managed instance produced by merge.
6. The existing path can return a different managed instance; mutating the original detached object may not be tracked.
7. A new object can have a non-null assigned ID, so presence of an ID does not prove a row exists.
8. For an entity loaded and mutated inside a transaction, dirty checking normally persists the change.
9. Spring parses a method name into a query using entity properties/operators.
10. JPQL uses entities/attributes and is more portable; native SQL uses tables/columns and permits database-specific features.
11. Content plus total count metadata commonly requires a count query.
12. When the client needs current content and “more available?” but not an exact total.

---

## Batch 4 — Relationships and Aggregates

### Questions

1. Which side commonly owns `Customer` -> `Order`?
2. What does `mappedBy` name?
3. Why maintain both sides of a bidirectional association in helper methods?
4. When is a join table used?
5. Why model an association entity instead of many-to-many?
6. What does `CascadeType.PERSIST` mean?
7. Why is `CascadeType.REMOVE` dangerous for many-to-many?
8. How does orphan removal differ from remove cascade?
9. Is cascade the same as fetch type?
10. Why should API responses usually use DTOs?

### Model points

1. `Order.customer`, because the orders table normally stores `customer_id`.
2. The Java property on the owning side; it is not a database column.
3. The owning side controls FK persistence, while the inverse collection controls navigation; inconsistent memory graphs create bugs.
4. For many-to-many or relationship representations where neither side can store one FK directly.
5. When the link has attributes such as quantity, role, status, or timestamps; it needs its own identity/invariants.
6. Persisting a parent propagates persist to configured related entities.
7. A shared target can be referenced by other parents; deleting one parent should not delete the shared target.
8. Remove cascade applies when the parent is deleted; orphan removal can delete a private child when disassociated.
9. Cascade controls operation propagation; fetch controls loading behavior.
10. DTOs prevent recursive graphs, lazy loading during serialization, overfetching, and entity/API coupling.

---

## Batch 5 — Transactions

### Questions

1. What does `@Transactional` add through Spring?
2. Why is the service often the transaction boundary?
3. What is `REQUIRED`?
4. What is `REQUIRES_NEW`?
5. How is isolation different from propagation?
6. Why can a checked exception commit work?
7. Why can catching an exception still result in rollback?
8. Why does self-invocation matter?
9. What does read-only mean?
10. Why can OSIV hide an architecture problem?

### Model points

1. Proxy-based transaction advice creates/joins a resource transaction and commits/rolls back according to configuration and outcome.
2. It surrounds one business use case whose database changes must succeed/fail together.
3. Join an existing transaction or create one if absent.
4. Suspend the outer transaction and create an independent one; it changes atomicity and consumes another connection.
5. Propagation chooses interaction with an existing transaction; isolation controls concurrent visibility/behavior.
6. Default rollback rules commonly target unchecked exceptions; checked exceptions need explicit configuration.
7. The transaction can be marked rollback-only before the exception is caught, so an outer commit can fail or roll back.
8. A direct `this.method()` call bypasses the proxy interceptor, so the method annotation may not apply.
9. A read-intent/performance hint whose enforcement varies; it is not a universal write firewall.
10. Lazy access during serialization can issue hidden SQL and hide that service-layer fetch planning is incomplete.

---

## Batch 6 — Fetching and N+1

### Questions

1. What is lazy loading?
2. What is `LazyInitializationException` telling you?
3. Define N+1.
4. How do you prove N+1?
5. When is a fetch join useful?
6. Why are collection fetch joins risky with pagination?
7. What is an entity graph?
8. When is batch fetching useful?
9. Why is EAGER not a universal solution?
10. When is a DTO projection best?

### Model points

1. Association state is deferred until accessed while an active context/provider mechanism can load it.
2. Required lazy state was accessed after the context/session was unavailable.
3. One root query plus N relationship/loop queries for N roots.
4. Correlate request SQL and show repeated parameterized queries with count growing linearly with result size.
5. To load a known association in one entity-oriented query when row multiplication/cardinality is safe.
6. Collection rows multiply roots, making page boundaries/counts ambiguous and potentially expensive.
7. A declared attribute fetch plan around a repository query.
8. When many roots need a deferred association but a join would create excessive row multiplication.
9. It can load unnecessary graphs, issue extra selects, and create global performance problems.
10. When a read endpoint needs a narrow stable shape and should avoid entity graph/lazy traversal.

---

## Batch 7 — Performance and Hibernate Internals

### Questions

1. JDBC batching versus bulk DML?
2. Why flush plus clear in a batch?
3. What does write-behind mean?
4. What does dirty checking compare?
5. What is the action queue conceptually?
6. Why can identity IDs limit batching?
7. What should a slow-query investigation capture?
8. Why can a larger connection pool worsen an incident?
9. What is a Hibernate proxy?
10. JPA versus Hibernate `Session`?

### Model points

1. JDBC batching groups similar per-entity statements; bulk DML changes a set directly and bypasses per-entity tracking/lifecycle.
2. Flush sends the chunk; clear releases managed objects/snapshots and bounds context memory.
3. In-memory changes are scheduled and translated to SQL during flush rather than every setter call.
4. Current managed state versus loaded snapshot/enhanced dirty markers at flush.
5. Pending insert/update/delete/collection/orphan actions ordered for execution.
6. The database insert may need to run to obtain the ID, reducing deferred grouping.
7. Exact SQL/binds, actual plan, rows scanned/returned, locks, pool wait, transaction duration, and repeated query context.
8. More concurrent DB work can exceed DB CPU/locks/connections; it increases capacity demand without fixing query/transaction duration.
9. A lazy stand-in for an entity whose state can be loaded later within the context.
10. `EntityManager` is the JPA API; `Session` is Hibernate's native provider API with extra behavior/features.

---

## Batch 8 — Locking and Concurrency

### Questions

1. What is a lost update?
2. How does `@Version` prevent silent stale writes?
3. What does optimistic locking assume?
4. When choose pessimistic write?
5. Is higher isolation enough for every lost-update problem?
6. How should an optimistic conflict be retried?
7. What creates a deadlock?
8. How can you reduce deadlocks?
9. What is dangerous about remote work while holding a lock?
10. How can bulk DML bypass version protection?

### Model points

1. Concurrent transactions read the same state and the later write overwrites the earlier write.
2. The update includes the expected version and increments it; a zero-row update signals a conflict.
3. Conflicts are uncommon and can be handled after detection.
4. For short, highly contended critical sections such as resource claims.
5. Isolation controls visibility; explicit version predicates/locks may still be required.
6. Start a fresh transaction, reload, revalidate, reapply only if safe/idempotent, and bound attempts.
7. Transactions acquire resources in a cycle, each waiting for the other.
8. Consistent order, short transactions, narrow locks, proper indexes, bounded timeout, and measured retry.
9. It holds rows/connections during unpredictable latency and increases contention/deadlock risk.
10. It can skip entity dirty checking/version predicates unless the bulk statement includes them deliberately.

---

## Batch 9 — Caching and Production

### Questions

1. What is the scope of the first-level cache?
2. What is second-level cache scope?
3. What does a query cache usually store?
4. Why can native SQL create stale cache data?
5. When should inventory not be authoritative from cache?
6. Name three causes of pool exhaustion.
7. What does a growing flush time suggest?
8. How do you debug unexpected updates?
9. What is the production debugging loop?
10. What belongs in an incident answer?

### Model points

1. One persistence context/EntityManager.
2. Shared across persistence contexts under one SessionFactory/EntityManagerFactory, subject to provider configuration.
3. Query key/result representation such as IDs or scalar results; entity data may still need another cache/database lookup.
4. It bypasses ORM/cache invalidation paths.
5. When stale availability could violate allocation correctness; use transactional version/lock/conditional update.
6. Long transactions/locks, slow queries, remote calls in transactions, `REQUIRES_NEW`, leaks, or undersized capacity.
7. Growing persistence context/snapshots, retained references, or database contention.
8. Capture SQL/binds, trace mutation/state, inspect merge/cascade/listeners, and compare intended fields.
9. Symptom -> impact -> evidence -> ranked hypotheses -> safe experiment -> verify -> root cause/prevention.
10. Symptom, scope, evidence, hypothesis, mitigation, root cause, fix, prevention, and trade-offs.

---

## Mini Code Trace 1 — Save and Merge

```java
Order input = mapper.toEntity(request); // existing id
Order result = repository.save(input);
input.setStatus(Status.CONFIRMED);
```

**Expected:** Existing state can route to `merge`; `result` is managed and `input` remains detached. The later mutation of `input` is not necessarily tracked. Use `result` or load a managed entity and apply a command.

## Mini Code Trace 2 — Clear and Flush

```java
entity.setStatus(Status.PAID);
entityManager.clear();
entityManager.flush();
```

**Expected:** If the change was not flushed before `clear`, the entity is detached and the later flush has no managed change to synchronize. `clear` is not rollback, but it stops tracking this pending in-memory mutation.

## Mini Code Trace 3 — Page with Collection

```java
@Query("select o from Order o join fetch o.lines where o.status = :status")
Page<Order> findPage(Status status, Pageable pageable);
```

**Expected:** Collection row multiplication makes pagination/count semantics risky. Prefer root-ID paging plus a second graph query or a summary DTO depending on the endpoint.

## Mini Code Trace 4 — Version Conflict

```java
Item a = repository.findById(id).orElseThrow();
Item b = repository.findById(id).orElseThrow();
a.decrement();
repository.save(a);
b.decrement();
repository.save(b);
```

**Expected:** If separate transactions/contexts and `@Version` exist, one update succeeds and the other conflicts. Retry only with fresh state and safe command semantics.

## Mini Code Trace 5 — Transaction Proxy

```java
public void importRows(List<Row> rows) {
    rows.forEach(this::saveOne);
}

@Transactional
public void saveOne(Row row) { repository.save(row); }
```

**Expected:** Direct self-invocation can bypass the Spring proxy. Put the transaction around the outer operation or move `saveOne` to another proxied service.

---

## Weak-Spot Matrix

After the batches, mark each area:

| Area | Strong / Working / Needs revision | Evidence or question missed |
|---|---|---|
| Lifecycle |  |  |
| `save()` internals |  |  |
| Relationships |  |  |
| Transactions |  |  |
| Fetching/N+1 |  |  |
| Performance |  |  |
| Hibernate internals |  |  |
| Locking |  |  |
| Caching |  |  |
| Production debugging |  |  |

Revise the smallest weak area first. Do not restart the entire roadmap because one question was fumbled.

---

## Session Handoff

**Prepared study material:** Session 11 — JPA/Hibernate Rapid-Fire Revision  
**Format:** nine question batches, five code traces, model points, and a weak-spot matrix.  
**Next session:** Session 12 — Full SDE-2 Mock Interview  
**Next starting topic:** architecture and entity-design prompt, followed by `save()`, transactions, N+1, concurrency, performance, and rapid-fire rounds

## One-Line Revision Summary

For rapid-fire answers, name the layer first, state the lifecycle/transaction behavior precisely, separate SQL timing from commit, and finish with the production trade-off or evidence you would inspect.
