# Session 8 — Locking and Concurrency

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** lost updates -> `@Version` -> optimistic locking -> pessimistic locks -> isolation -> deadlocks -> strategy choice

## Session Outcome

You should be able to:

- explain a lost update with two concurrent transactions;
- use `@Version` to detect stale writes;
- distinguish optimistic conflict detection from pessimistic blocking;
- choose lock modes for inventory, payments, and ordinary edits;
- explain how isolation and row locks interact;
- diagnose deadlocks and reduce their probability;
- design retry and conflict responses safely;
- identify concurrency holes in detached `merge()` and bulk-update flows.

---

## 1. The Lost-Update Problem

Two transactions read the same value:

```text
initial stock = 10

T1 reads 10              T2 reads 10
T1 writes 9              T2 writes 8
T1 commits               T2 commits

final stock = 8
```

If the business expected both decrements, one update was lost. Default database isolation does not necessarily detect this because each transaction can legally update a row last.

Concurrency design starts with the business invariant:

- must every update be preserved?
- can a user retry after a conflict?
- can two changes be merged?
- must only one worker claim a record?
- is temporary stale data acceptable?

---

## 2. Optimistic Locking with `@Version`

```java
@Entity
public class InventoryItem {
    @Id
    private Long id;

    private int available;

    @Version
    private long version;
}
```

Conceptually, the update becomes:

```sql
update inventory_item
set available = ?, version = version + 1
where id = ? and version = ?
```

If another transaction already changed the row, the `where version = ?` predicate matches zero rows. Hibernate detects that and raises an optimistic-lock conflict, commonly `OptimisticLockException` or a Spring-translated data-access exception.

### What `@Version` provides

- stale-write detection;
- a monotonically changing row version in the normal entity path;
- a clear conflict signal instead of silently overwriting.

### What it does not provide

- automatic retry with correct business semantics;
- protection for every native/bulk update unless version handling is designed;
- a guarantee that all external writers use the version predicate;
- prevention of reads of stale data;
- a replacement for authorization or validation.

Use a nullable wrapper version or provider-supported type according to mapping rules; choose the type consistently and test schema generation/migrations.

---

## 3. Optimistic Conflict Timeline

```text
T1: load item(id=7, version=4)
T2: load item(id=7, version=4)
T1: set quantity=9
T1: flush -> update where id=7 and version=4; version becomes 5
T2: set quantity=8
T2: flush -> update where id=7 and version=4; 0 rows
T2: conflict
```

A correct application response depends on the use case:

- return HTTP 409 and ask the user to reload;
- retry by re-reading and reapplying an idempotent command;
- merge non-overlapping fields;
- reject the command when the invariant cannot be reconstructed safely.

Do not blindly catch an optimistic conflict and call `save()` again with the stale object. That can repeat the same conflict or overwrite a newer decision if version handling is bypassed.

---

## 4. Detached Objects and `merge()`

A web request often carries an old detached representation:

```java
Order detached = mapper.toEntity(request);
Order managed = entityManager.merge(detached);
```

With `@Version`, merge can detect that the detached version is stale when synchronization occurs. But a safer update design often loads the current managed entity and applies a command:

```java
@Transactional
public void changeAddress(Long orderId, AddressCommand command) {
    Order order = repository.findById(orderId).orElseThrow();
    authorization.check(order);
    order.changeAddress(command);
}
```

This makes authorization, invariant checks, and conflict behavior visible. If the API intentionally uses full detached state, require and validate a version token.

### HTTP versioning

Expose a version or ETag to the client for edit flows. The client submits the version it read; the service rejects stale updates. This turns an invisible data race into an explicit conflict.

---

## 5. Pessimistic Locking

Pessimistic locking asks the database to lock rows while the transaction works:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select i from InventoryItem i where i.id = :id")
Optional<InventoryItem> findForUpdate(Long id);
```

Or with JPA:

```java
InventoryItem item = entityManager.find(
        InventoryItem.class,
        id,
        LockModeType.PESSIMISTIC_WRITE
);
```

Conceptually:

```text
T1 obtains row lock
T2 attempts conflicting lock/update -> waits or fails
T1 updates and commits
T2 proceeds or receives timeout/deadlock
```

### Common lock modes

- `PESSIMISTIC_READ`: request a read lock compatible with provider/database semantics;
- `PESSIMISTIC_WRITE`: request a lock intended to prevent conflicting writes;
- `PESSIMISTIC_FORCE_INCREMENT`: lock and increment version in supported scenarios.

Exact behavior depends on database, dialect, isolation, and lock timeout configuration. “Pessimistic write always blocks all reads” is too broad.

### When it fits

- short critical sections with high conflict probability;
- inventory allocation or unique resource claiming;
- worker queues where one transaction must claim a row;
- correctness requires serialization and retry is expensive.

It is a poor fit for long transactions or code that performs remote calls while holding locks.

---

## 6. Optimistic versus Pessimistic

| Concern | Optimistic | Pessimistic |
| --- | --- | --- |
| Conflict assumption | conflicts are uncommon | conflicts are likely/expensive |
| Normal behavior | no blocking lock | acquire database lock |
| Failure signal | version conflict at update | wait, timeout, or deadlock |
| Throughput | good under low contention | can degrade under contention |
| User experience | retry/conflict response needed | may experience blocking |
| Critical section | can be longer if safe | keep very short |
| Good examples | ordinary edits, APIs | stock/resource claim |

Choose based on measured contention and business semantics, not fashion. Some flows combine an optimistic version with a short pessimistic lock for a specific critical operation.

---

## 7. Isolation Is Not a Locking Strategy

Isolation controls what concurrent transactions can observe. Locking controls explicit coordination around rows. A stronger isolation level may reduce some anomalies but can increase blocking and does not automatically encode the business invariant.

Example: `READ_COMMITTED` can prevent dirty reads while still allowing two transactions to read the same quantity and overwrite each other. `@Version` detects the stale update. A pessimistic lock can serialize the read-modify-write sequence.

Ask separately:

- what can this transaction see?
- which rows must be protected?
- when is the lock released?
- what is the conflict/timeout response?
- can the operation retry safely?

---

## 8. Deadlocks

A deadlock occurs when transactions wait in a cycle:

```text
T1 locks A -> waits for B
T2 locks B -> waits for A
```

Common causes:

- inconsistent row acquisition order;
- long transactions;
- broad range/table locks;
- multiple code paths updating the same entities in different order;
- missing indexes causing wider locking/scanning;
- foreign-key/index interactions;
- retries that increase concurrent pressure.

### Reducing deadlocks

- acquire resources in a consistent deterministic order;
- keep transactions short;
- lock only required rows;
- use appropriate indexes and predicates;
- avoid remote calls while holding locks;
- set bounded lock/query timeouts;
- retry deadlock victims only when the operation is idempotent and bounded;
- monitor and inspect database deadlock reports.

A retry is not a fix for an unbounded deadlock pattern. It can amplify load.

---

## 9. Lock Timeouts and Retry Design

A lock timeout or optimistic conflict is a business-visible outcome. A safe retry design should specify:

- maximum attempts;
- backoff/jitter;
- which exception categories are retryable;
- whether the transaction is recreated on each attempt;
- whether the command is idempotent;
- whether external side effects have already occurred;
- what response is returned after exhaustion.

Each retry should generally start a fresh transaction and re-read current state. Retrying inside a transaction that is already marked rollback-only is not a valid recovery path.

---

## 10. Bulk Updates and Versioning Holes

A bulk query can bypass normal version checks:

```java
@Modifying
@Query("update InventoryItem i set i.available = i.available - :amount where i.id = :id")
int decrement(...);
```

The update count can be used as a conditional guard, but the query must encode the invariant and any version predicate deliberately:

```sql
update inventory_item
set available = available - ?, version = version + 1
where id = ? and available >= ? and version = ?
```

After bulk DML, managed objects may be stale. Clear/reload and define whether entity listeners/auditing are intentionally skipped.

---

## 11. Scenario Reasoning

### Scenario A: two checkout requests

Both read inventory 1 and attempt to decrement it.

**Answer:** With optimistic locking, one succeeds and one receives a version conflict. With pessimistic write, one waits while the other completes; the second then rechecks availability. Do not let both update without a condition/version.

### Scenario B: retrying `OptimisticLockException`

The service catches the exception and retries the same managed object.

**Answer:** The transaction is typically rollback-only/ended. Retry in a new transaction, re-read state, and reapply an idempotent command. Blindly retrying stale state is unsafe.

### Scenario C: deadlock after adding a second update

Flow A updates customer then order; flow B updates order then customer.

**Answer:** Inconsistent lock order can create a cycle. Define a consistent order or redesign the transaction; use bounded retry only as resilience after reducing the cause.

### Scenario D: long remote call under lock

A service obtains `PESSIMISTIC_WRITE`, calls a payment provider, then commits.

**Answer:** The row is locked during network latency, increasing contention and deadlock/timeout risk. Separate the reservation/claim transaction from the remote workflow using state transitions and idempotency.

---

## 12. SDE-2 Interview Answers

### What problem does `@Version` solve?

It detects stale updates by adding the expected version to the update predicate and incrementing it on success. If another transaction changed the row, the update affects zero rows and the provider raises a conflict instead of silently overwriting.

### Optimistic versus pessimistic locking?

Optimistic locking assumes conflicts are uncommon and detects them with a version at write time. Pessimistic locking asks the database to coordinate access by blocking or rejecting conflicting transactions. Optimistic is often better for ordinary edits; pessimistic can fit short, highly contended critical sections.

### Does higher isolation prevent lost updates?

Not necessarily. Isolation controls visibility/anomalies, while lost-update prevention often needs version predicates, conditional updates, or explicit locks.

### How should an optimistic conflict be retried?

Start a new transaction, reload current state, revalidate the command/invariants, and reapply only if it is safe and idempotent. Bound attempts and return a conflict when it cannot be safely merged.

### How do you reduce deadlocks?

Use consistent lock/resource ordering, short transactions, appropriate indexes, narrow predicates, bounded timeouts, and avoid external calls while holding locks. Investigate database deadlock reports; retries alone are not the design.

---

## 13. Practice Batch

1. Trace the lost update with two transactions and explain how `@Version` changes the result.
2. Why is retrying the same entity inside a failed transaction unsafe?
3. When would a pessimistic lock be more appropriate than optimistic locking?
4. What creates a deadlock even when each transaction is individually correct?
5. How can a bulk update bypass version protection?

### Model answer key

1. Both read version 4; the first updates with `where version=4` and increments to 5; the second's same predicate affects zero rows and receives a conflict.
2. The transaction may be rollback-only and the object contains stale state. A retry needs a fresh transaction and current read, then must revalidate/reapply safely.
3. When contention is high, the critical section is short, and serializing access is cheaper/safer than repeated conflicts, such as a resource claim.
4. Different lock acquisition order creates a wait cycle. Consistent order and short transactions reduce it.
5. Bulk DML does not automatically use entity dirty checking/version predicates. Add explicit conditions/version increments if required and clear stale contexts.

---

## Session Handoff

**Prepared study material:** Session 8 — Locking and Concurrency  
**Topics covered:** lost updates, `@Version`, optimistic conflicts, detached updates, pessimistic lock modes, isolation, deadlocks, timeouts, retries, and bulk-DML concurrency holes.  
**Next session:** Session 9 — Caching  
**Next starting topic:** first-level cache recap -> second-level cache scope -> query cache and invalidation

## One-Line Revision Summary

Concurrency requires a business invariant plus a coordination strategy: versions detect stale writes, locks serialize contention, isolation controls visibility, and retries must recreate transactions and revalidate state.
