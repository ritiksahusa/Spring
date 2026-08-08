# Session 1 — Entity Lifecycle Completion

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** `detach()` → `clear()` → `refresh()` → `remove()` → lifecycle comparison → interview reasoning

## Session Outcome

By the end of this session, you should be able to:

- identify an entity's JPA state at every point in a code path;
- explain what the persistence context is tracking and why that matters;
- distinguish `detach()`, `clear()`, `refresh()`, and `remove()` precisely;
- predict whether dirty checking will produce an `UPDATE`;
- explain when an `INSERT`, `UPDATE`, or `DELETE` is likely to be sent to the database;
- reason about flush, commit, rollback, and the first-level cache in lifecycle scenarios;
- give an interview-quality answer without making absolute claims about SQL timing.

This session assumes that you already know the basic persistence-context model, `EntityManager`, `persist()`, `merge()`, dirty checking, and flush-versus-commit. It intentionally does not restart `merge()`.

---

## 1. The Mental Model: Object State Plus Tracking State

A JPA entity has two related but different realities:

1. **The Java object** — an object in application memory.
2. **Its relationship with the persistence context** — whether JPA is currently tracking it.

The persistence context is an identity map and a unit of tracking. Within one persistence context, a database identity normally maps to one managed Java instance. Hibernate takes snapshots of managed state and can compare the current state with those snapshots during flush.

The four lifecycle states are:

| State | Persistence-context tracking | Typical entry | What happens to changes? |
| --- | --- | --- | --- |
| NEW / TRANSIENT | Not tracked; no persistent identity yet | `new Patient()` | Not persisted merely because fields change |
| MANAGED | Tracked by the persistence context | `find()`, `persist()`, `merge()` return value | Dirty checking can synchronize changes |
| DETACHED | Previously persistent, no longer tracked | `detach()`, `clear()`, closed context | Changes are not automatically synchronized |
| REMOVED | Tracked and scheduled for deletion | `remove()` | Delete is synchronized during flush, subject to transaction outcome |

A useful state graph is:

```text
                         persist()
             NEW ----------------------------> MANAGED
              |                                  |   ^
              |                                  |   |
              |                                  |   | find()/query
              |                                  |   |
              |                                  |   | merge() returns managed copy
              |                                  |   |
              |                                  v   |
              |       detach()/clear()/close() DETACHED
              |                                  |
              |                                  | merge()
              |                                  v
              +----------------------------> MANAGED

MANAGED --remove()--> REMOVED --flush--> DELETE attempt
   ^                    |
   |                    | rollback / provider-specific recovery paths are not a
   |                    | substitute for treating REMOVED as a normal managed object
   |                    v
   +------------- transaction outcome determines database result
```

Important: **Java object identity and database row existence are not the same thing.** Calling `remove()` changes how the persistence context treats the entity; the database row is affected when the persistence operation is flushed and committed.

---

## 2. `detach(entity)`: Stop Tracking One Entity

### Definition

`entityManager.detach(entity)` removes one managed entity from the current persistence context. After detachment, changes made through that Java object are not observed by dirty checking.

```java
@Transactional
public void changeWithoutSaving(Long id) {
    Patient patient = entityManager.find(Patient.class, id); // MANAGED
    entityManager.detach(patient);                           // DETACHED
    patient.setStatus("DISCHARGED");                        // not tracked
}
```

The method may complete successfully, but no `UPDATE` should be generated for `status` merely because the detached object was mutated. If the object is later passed to `merge()`, its state can become persistent again through the managed instance returned by `merge()`.

```java
Patient detached = entityManager.find(Patient.class, id);
entityManager.detach(detached);
detached.setStatus("DISCHARGED");

Patient managed = entityManager.merge(detached);
// managed is MANAGED; detached is still DETACHED.
```

### Why use it?

- release a large object graph from a long-running persistence context;
- prevent accidental dirty checking for an object;
- deliberately work with detached data before an explicit merge;
- control memory and tracking cost in batch-style processing.

### Interview traps

- `detach()` does not delete the row.
- `detach()` does not undo changes already flushed to the database.
- `detach()` does not make a copy; the same Java object is no longer tracked.
- `detach()` only applies to a managed entity in that persistence context. Detaching an already detached or new object has no useful persistence effect.
- Detaching a parent does not magically detach every related object in all cases; graph behavior depends on what is managed and provider semantics. Use `clear()` when the requirement is to detach all entities in the context.

---

## 3. `clear()`: Stop Tracking Everything in the Context

### Definition

`entityManager.clear()` detaches all managed entities from the current persistence context.

```java
List<Patient> patients = entityManager
        .createQuery("select p from Patient p", Patient.class)
        .getResultList();

entityManager.clear(); // every managed entity in this context becomes DETACHED
```

After `clear()`:

- the first-level cache is emptied;
- managed references become detached;
- subsequent mutations to those objects are not dirty-checked;
- the database is not automatically rolled back;
- the database is not automatically committed;
- already-flushed SQL is not undone by `clear()`.

### `clear()` versus `flush()`

This distinction is a common SDE-2 interview boundary:

| Operation | Main effect | Sends pending SQL? | Reverses already-flushed SQL? |
| --- | --- | ---: | ---: |
| `flush()` | Synchronizes persistence-context changes with the database | Usually yes | No |
| `clear()` | Detaches all managed entities and empties first-level cache | No | No |
| `rollback()` | Ends transaction and discards uncommitted database work | No new SQL | Usually yes for uncommitted work |
| `commit()` | Completes transaction | May flush first | No |

A pending in-memory change can be discarded from tracking by `clear()` if it has not been flushed. But `clear()` itself is not a transaction operation. A transaction rollback is what gives database-level atomicity.

### Typical batch pattern

```java
for (int index = 0; index < commands.size(); index++) {
    Patient patient = new Patient(commands.get(index));
    entityManager.persist(patient);

    if (index % 100 == 0) {
        entityManager.flush(); // send the batch to the database
        entityManager.clear(); // release managed objects and snapshots
    }
}
```

The pair is useful because `flush()` synchronizes work and `clear()` prevents the persistence context from growing without bound. The pair does not replace a sensible transaction boundary.

---

## 4. `refresh(entity)`: Replace In-Memory State from the Database

### Definition

`entityManager.refresh(entity)` reloads the entity's state from the database and keeps the entity managed.

```java
Patient patient = entityManager.find(Patient.class, id); // MANAGED
patient.setStatus("LOCAL_VALUE");
entityManager.refresh(patient);                          // database state wins
```

After refresh:

- the entity remains MANAGED;
- locally changed, unflushed state is overwritten by the database state;
- a database read is required;
- later changes can still be detected by dirty checking;
- refresh is not a rollback operation for the whole transaction.

### Why use it?

- another process or database trigger changed the row;
- you need to discard local unflushed changes for one managed entity;
- you need database-generated values or current database state after an external change;
- you need to re-read an entity while keeping it managed.

### Refresh is not the same as clear

```java
Patient patient = entityManager.find(Patient.class, id);
entityManager.refresh(patient);
patient.setStatus("NEW_VALUE"); // still tracked; may produce UPDATE
```

With `clear()`, the reference becomes detached. With `refresh()`, the reference remains managed. This is the decisive behavioral difference.

### Constraints and caution

- Refresh generally requires a managed entity.
- Refresh can overwrite application changes that have not been flushed.
- A refresh does not guarantee that every lazily loaded relationship is initialized.
- The exact SQL and lock behavior can vary with provider, mappings, and lock mode; reason from the contract first and avoid promising a specific SQL shape unless configured.

---

## 5. `remove(entity)`: Mark a Managed Entity for Deletion

### Definition

`entityManager.remove(entity)` marks a managed entity as REMOVED. The provider will usually issue a `DELETE` during flush, but the final database result still depends on transaction completion.

```java
@Transactional
public void deletePatient(Long id) {
    Patient patient = entityManager.find(Patient.class, id); // MANAGED
    entityManager.remove(patient);                           // REMOVED
    // DELETE is commonly emitted at flush/commit, not necessarily here.
}
```

### State and timing

```text
find()       -> MANAGED
remove()     -> REMOVED, deletion scheduled
flush()      -> DELETE sent to database
commit()     -> deletion becomes durable
rollback()   -> uncommitted deletion is rolled back
```

Do not describe `remove()` as "immediately deleting the row." The accurate interview wording is:

> `remove()` marks a managed entity for removal. The provider synchronizes that deletion during flush, and the transaction commit determines whether it becomes durable.

### What if the entity is detached?

`remove()` expects a managed entity. For a detached object, the usual pattern is:

```java
Patient detached = loadFromRequest();
Patient managed = entityManager.merge(detached);
entityManager.remove(managed);
```

For deletion by identifier, loading a managed reference and removing that managed reference is often clearer. Repository APIs may provide a direct delete operation, but the transaction and relationship semantics still matter.

### Common pitfalls

- Removing a detached instance directly can fail with an exception.
- A later flush can make the deletion visible to the database before commit, but rollback can still undo it.
- Cascades and foreign-key constraints can change whether deletion succeeds.
- A removed entity is not a safe normal object to keep using as if it were managed; application code should stop treating it as active persistent state.

---

## 6. Comparing the Four Operations

| Operation | Input expectation | Resulting state | Reads database? | Main effect |
| --- | --- | --- | ---: | --- |
| `detach(x)` | Usually MANAGED | DETACHED | No | Stop tracking one object |
| `clear()` | Persistence context | All become DETACHED | No | Stop tracking everything |
| `refresh(x)` | MANAGED | MANAGED | Yes | Replace state from database |
| `remove(x)` | MANAGED | REMOVED | Usually no required read if already managed | Schedule deletion |

### One-sentence distinctions

- **`detach()`**: “Forget this one object.”
- **`clear()`**: “Forget every managed object in this context.”
- **`refresh()`**: “Reload this object from the database but keep tracking it.”
- **`remove()`**: “Track this object as scheduled for deletion.”

### Decision table

| Requirement | Likely operation |
| --- | --- |
| Prevent further dirty checking for one entity | `detach()` |
| Release all managed entities in a batch | `clear()` |
| Discard local state and reload one managed entity | `refresh()` |
| Delete a managed row as part of a transaction | `remove()` |
| Send pending changes before clearing | `flush()` then `clear()` |
| Undo uncommitted database work | transaction rollback, not `clear()` |

---

## 7. Full Lifecycle Trace

Consider this entity:

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    private String status;

    // constructors, getters, setters omitted
}
```

Trace the following service method:

```java
@Transactional
public void process(Long id) {
    Order order = entityManager.find(Order.class, id); // A
    order.setStatus("PACKED");                         // B
    entityManager.flush();                              // C
    entityManager.detach(order);                        // D
    order.setStatus("SHIPPED");                        // E

    Order managedAgain = entityManager.merge(order);    // F
    entityManager.refresh(managedAgain);                // G
    entityManager.remove(managedAgain);                 // H
}                                                        // I
```

Expected reasoning:

- **A:** `order` is MANAGED.
- **B:** The change is eligible for dirty checking.
- **C:** Hibernate can issue an `UPDATE` for `PACKED`; the change is now flushed, but commit has not necessarily happened.
- **D:** `order` becomes DETACHED.
- **E:** `SHIPPED` is only a Java-side detached change.
- **F:** `merge()` copies the detached state into a managed instance; use `managedAgain` for further persistence operations.
- **G:** The managed instance is reloaded from the database. The database value, likely `PACKED` if the earlier flush is visible in the transaction, replaces the merged in-memory value. The exact observed result depends on transaction/database isolation and provider behavior.
- **H:** `managedAgain` becomes REMOVED.
- **I:** On successful completion, the transaction commonly flushes and commits the delete. If the transaction rolls back, the uncommitted database work is not durable.

The important skill is not memorizing a single SQL sequence. It is separating **state transitions**, **flush timing**, and **transaction outcome**.

---

## 8. SDE-2 Scenario Reasoning

### Scenario A: `clear()` after mutation

```java
Patient patient = entityManager.find(Patient.class, 7L);
patient.setStatus("DISCHARGED");
entityManager.clear();
```

**Answer:** `patient` is detached after `clear()`. If the change was not flushed before `clear()`, it will normally not be synchronized through dirty checking. `clear()` does not roll back anything that was already flushed and does not commit or rollback the transaction.

### Scenario B: `refresh()` after mutation

```java
Patient patient = entityManager.find(Patient.class, 7L);
patient.setStatus("DISCHARGED");
entityManager.refresh(patient);
patient.setStatus("ARCHIVED");
```

**Answer:** The first local value is overwritten by the database value. The entity remains managed. The second value, `ARCHIVED`, is again eligible for dirty checking and can be flushed.

### Scenario C: remove then rollback

```java
Patient patient = entityManager.find(Patient.class, 7L);
entityManager.remove(patient);
entityManager.flush();
// transaction rolls back later
```

**Answer:** A `DELETE` may have been sent to the database at flush, but rollback makes the uncommitted deletion non-durable. SQL emission and transaction durability are different questions.

### Scenario D: detached delete

```java
Patient patient = requestMapper.toPatient(request);
entityManager.remove(patient);
```

**Answer:** This is unsafe because the mapped object is normally NEW or DETACHED, not managed. Load/find a managed entity or merge first, then remove the managed instance. Also consider authorization and stale-data checks before deletion.

### Scenario E: `detach()` and relationships

A managed `Order` has a lazily loaded `Customer`. The order is detached and then code accesses `order.getCustomer().getName()`.

**Answer:** Detaching can make later lazy loading impossible if the required association was not initialized while the persistence context was open. This can result in `LazyInitializationException`. The solution is not to blindly switch everything to EAGER; fetch the required data deliberately inside the transaction or use a suitable projection/entity graph/fetch join.

---

## 9. Interview-Quality Answers

### What is the difference between `detach()` and `clear()`?

`detach(entity)` removes one managed entity from the persistence context. `clear()` removes all managed entities from that persistence context. Both produce detached objects and stop dirty checking for them, but neither commits, rolls back, or reverses SQL that has already been flushed.

### What does `refresh()` do?

`refresh()` reloads an entity's current state from the database, overwriting unflushed in-memory changes, while keeping the entity managed. It is useful when database-side changes or external writers must become visible to the current managed instance.

### Does `remove()` immediately execute `DELETE`?

Not necessarily. `remove()` changes the managed entity to the REMOVED state and schedules deletion. The provider usually sends the `DELETE` during flush; transaction commit makes the result durable, while rollback can undo uncommitted work.

### What happens to a managed entity after `clear()`?

It becomes detached, the persistence context's first-level cache is cleared, and later changes to that Java object are not tracked automatically. `clear()` does not itself affect transaction outcome.

### Can a detached entity be deleted?

The safe standard flow is to obtain a managed instance, then call `remove()` on that managed instance. For example, `managed = entityManager.find(...)` followed by `remove(managed)`. If starting with detached state, `merge()` returns a managed instance, but deletion semantics and stale state should be considered carefully.

---

## 10. Practice Batch: Answer Before Reading the Key

1. An entity is managed, changed, flushed, and then detached. Does detaching undo the database update?
2. What is the difference between `clear()` and transaction rollback?
3. After `refresh(entity)`, is the entity managed or detached? What happens to an unflushed local field change?
4. Why should `remove()` normally receive a managed entity?
5. A loop persists 100,000 entities in one persistence context. Why might `flush()` plus `clear()` be used periodically?

### Model answer key

1. No. `detach()` stops future tracking; it does not undo an update already sent during flush. Transaction rollback is the mechanism that can undo uncommitted database work.
2. `clear()` changes persistence-context tracking by detaching all managed entities. Rollback changes the transaction outcome by discarding uncommitted database work. They operate at different layers.
3. It remains managed. Its unflushed local state is replaced by the database state, and future mutations can again be dirty-checked.
4. `remove()` is defined around a managed persistent instance. A detached or new object is not currently associated with the persistence context, so load or merge the state and remove the managed instance.
5. `flush()` synchronizes pending work, while `clear()` releases managed entities and snapshots. Together they reduce persistence-context memory growth and accidental tracking during batch work.

---

## 11. Five-Minute Whiteboard Drill

Draw this sequence and label the state after every arrow:

```text
new Order()
    -> persist()
    -> setStatus("PAID")
    -> flush()
    -> detach()
    -> setStatus("SHIPPED")
    -> merge()
    -> refresh()
    -> remove()
    -> commit()
```

Then answer three separate questions:

1. Which changes are eligible for dirty checking?
2. At which points might SQL be emitted?
3. Which event determines whether the final deletion is durable?

Expected outline:

- `persist()` makes the original object managed; changing it before flush is tracked.
- `flush()` can emit the insert and any tracked update.
- The change after `detach()` is not tracked until its state is merged into a managed instance.
- `merge()` returns the instance whose changes matter.
- `refresh()` replaces that managed state from the database.
- `remove()` schedules deletion; flush may emit it; commit determines durability.

---

## 12. Common Incorrect Statements to Remove From Your Vocabulary

- “`detach()` rolls back the entity.”
- “`clear()` commits all pending work.”
- “`refresh()` makes the entity detached.”
- “`remove()` always executes `DELETE` immediately.”
- “If SQL was logged, the transaction definitely succeeded.”
- “A detached object is deleted when its fields are changed.”
- “`merge()` makes the original detached object managed.”

Use precise replacements:

- “`detach()` stops tracking this entity.”
- “`clear()` detaches all entities in the persistence context.”
- “`refresh()` reloads database state and keeps the entity managed.”
- “`remove()` marks a managed entity for deletion.”
- “Flush sends SQL; commit determines transaction durability.”
- “Detached changes require an explicit reattachment/state-transfer path such as `merge()`.”
- “`merge()` returns the managed instance; the supplied object remains detached.”

---

## Session Handoff

**Completed session:** Session 1 — Entity Lifecycle Completion  
**Topics covered:** `detach()`, `clear()`, `refresh()`, `remove()`, REMOVED state, lifecycle transitions, operation comparison, flush/commit/rollback boundaries, and lifecycle interview scenarios.  
**Fundamental gaps to watch:** `clear()` versus `flush()`; SQL emission versus transaction durability; using the managed instance returned by `merge()`.  
**Current status:** Mastered / interview ready according to the guide checkpoint.  
**Next session:** Session 2 — Spring Data JPA Internals  
**Next starting topic:** `JpaRepository` → repository proxy → `SimpleJpaRepository` → `save()` end-to-end

## One-Line Revision Summary

`detach()` forgets one, `clear()` forgets all, `refresh()` reloads one while keeping it managed, and `remove()` schedules one managed entity for deletion; flush synchronizes SQL, while commit determines durability.
