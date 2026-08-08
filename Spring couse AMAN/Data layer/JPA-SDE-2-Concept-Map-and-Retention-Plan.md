# JPA SDE-2 Concept Map and Retention Plan

**Purpose:** Make the 12-session material easier to understand, retrieve, and resume.  
**Canonical roadmap:** [JPA_SDE-2_Mastery_Guide_Export.md](JPA_SDE-2_Mastery_Guide_Export.md)  
**Coverage bridge:** [JPA-SDE-2-Foundation-Bridge.md](JPA-SDE-2-Foundation-Bridge.md)

## 1. The Master Mental Model

Most JPA/Hibernate questions become clearer when you trace the same request through five layers:

```text
1. Domain intent
   What business operation is being performed?
        |
2. Entity/object state
   NEW, MANAGED, DETACHED, or REMOVED?
        |
3. Persistence context and transaction
   What is tracked? What transaction is active? What can flush?
        |
4. Hibernate/JDBC/database execution
   What SQL, indexes, locks, rows, and connections are involved?
        |
5. Result and consistency
   Commit/rollback? stale object/cache? response shape? retry needed?
```

### Example: confirm an order

```text
Domain: confirm order 42
  -> load managed Order
  -> transaction owns the unit of work
  -> change status; dirty checking detects it
  -> flush emits UPDATE; database checks version/index/locks
  -> commit makes it durable; DTO returns confirmed state
```

When an answer feels confusing, name the layer where the behavior occurs. Most incorrect answers mix layers, for example:

- “`clear()` rolls back the update” mixes persistence-context state with transaction outcome.
- “EAGER fixes N+1” mixes mapping defaults with SQL query shape.
- “The cache says stock is 1, so stock is 1” mixes cached display data with database authority.
- “`save()` inserted the row” confuses repository dispatch with flush/commit timing.

---

## 2. The Four Core Decision Trees

### A. Updating an entity

```text
Do you already have a managed entity in the current transaction?
    |
    +-- yes -> validate/authorize -> mutate it -> dirty checking
    |
    +-- no
          |
          +-- new object -> persist/save as new -> use original managed object
          |
          +-- detached state
                |
                +-- command-style update -> load current managed entity, apply command
                |
                +-- intentional detached state transfer -> merge/save, use returned object
```

**Default SDE-2 preference:** For a business update, load the current managed entity and apply a command. It makes authorization, invariants, and optimistic conflicts visible.

### B. Choosing a query tool

```text
Is the predicate short and stable?
    -> derived query

Does the entity query need explicit joins/fetch/grouping?
    -> JPQL with @Query

Are filters optional or assembled dynamically?
    -> Specification or Criteria API

Does the caller need a narrow read shape?
    -> DTO/record or interface projection

Does the query need database-specific/reporting SQL?
    -> native query/JDBC, with measured justification
```

### C. Choosing a fetch plan

```text
What does the response actually need?
    |
    +-- a few scalar fields -> DTO/projection
    |
    +-- root plus to-one data -> fetch join/entity graph/batch fetch
    |
    +-- root page plus collection -> page root IDs, then fetch graph
    |
    +-- many roots and deferred to-one data -> batch fetching may fit
    |
    +-- data needed after transaction -> load/map before returning
```

Do not choose EAGER as a substitute for this decision.

### D. Choosing concurrency control

```text
Is the state authoritative for a correctness invariant?
    |
    +-- no -> bounded-stale cache/display read may be acceptable
    |
    +-- yes
          |
          +-- conflicts uncommon and retry/409 is acceptable
          |       -> optimistic @Version/conditional update
          |
          +-- conflicts common and critical section is short
                  -> pessimistic lock or atomic database operation

For every retry: fresh transaction + fresh state + revalidation + bounded attempts.
```

---

## 3. Contrast Pairs to Memorize

| Pair | Difference in one sentence |
| --- | --- |
| JPA vs Hibernate | JPA is the standard contract; Hibernate is a provider and implementation with extra features. |
| JPA vs JDBC | JPA manages entity state/relationships; JDBC exposes SQL/connection primitives. |
| `find()` vs `getReference()` | `find()` reads now and can return null; a reference supplies an identity and may load later. |
| `persist()` vs `merge()` | `persist()` makes the supplied new object managed; `merge()` transfers state and returns the managed instance. |
| `detach()` vs `clear()` | Detach one object; clear the whole persistence context. |
| `refresh()` vs `clear()` | Refresh reloads one and keeps it managed; clear detaches all and does not reload. |
| `remove()` vs `DELETE` | Remove marks a managed entity; flush/commit determine SQL execution and durability. |
| flush vs commit | Flush synchronizes ORM changes with the database; commit makes the transaction durable. |
| cascade vs fetch | Cascade propagates operations; fetch controls loading. |
| remove cascade vs orphan removal | Parent deletion propagation versus private-child deletion on disassociation. |
| lazy vs EAGER | Deferred/explicit loading choice versus mapping-required loading; neither guarantees a good query. |
| entity vs DTO projection | Managed domain object versus narrow read result. |
| `Page` vs `Slice` | Total-count metadata versus content plus “more exists?” without requiring a total. |
| `REQUIRED` vs `REQUIRES_NEW` | Join/create the current transaction versus suspend/create an independent one. |
| propagation vs isolation | Transaction interaction versus concurrent visibility/behavior. |
| JDBC batch vs bulk DML | Grouped per-entity statements versus set-based SQL that bypasses entity tracking. |
| first-level vs second-level cache | One persistence context versus shared SessionFactory/EntityManagerFactory scope. |
| optimistic vs pessimistic locking | Detect conflict at write versus coordinate by database locking. |
| cache vs source of truth | A performance copy versus the authoritative consistency mechanism. |

Use the pair table for daily retrieval. If you cannot state the difference in one sentence, revisit the owning session.

---

## 4. Memory Hooks for the Roadmap

- **Persistence context:** identity map plus change tracker.
- **Managed:** Hibernate is watching this object.
- **Detached:** Java object remains; automatic watching stops.
- **`merge()`:** copy state in; use the returned managed object.
- **Dirty checking:** compare managed state with what was loaded.
- **Write-behind:** record now, generate SQL at flush.
- **Flush:** send work; do not call it commit.
- **Repository proxy:** interface method dispatch, not magic persistence.
- **`save()`:** new detection chooses persist or merge.
- **Owning side:** the association that writes the FK/join table.
- **`mappedBy`:** points to the owning Java property.
- **Cascade:** operation propagation, not relationship loading.
- **N+1:** one root query plus one relationship query per root.
- **Fetch plan:** data required by this use case, not a global entity default.
- **Transaction boundary:** the business unit that commits or rolls back together.
- **Propagation:** what to do when a transaction already exists.
- **Isolation:** what concurrent work can observe.
- **`@Version`:** stale write becomes an explicit conflict.
- **Pessimistic lock:** hold a short database-coordinated critical section.
- **Batch:** fewer round trips, not automatically fewer rows or lifecycle operations.
- **Persistence-context pressure:** flush sends; clear releases tracking.
- **Query plan:** evidence of how the database will execute the SQL.
- **Pool exhaustion:** connections held too long or demanded too aggressively.
- **First-level cache:** local identity/tracking, not shared freshness.
- **Second-level cache:** shared optimization with an invalidation contract.
- **Production debugging:** symptom -> evidence -> hypothesis -> experiment -> prevention.

---

## 5. The Better Study Loop

Use this loop for every session. It is deliberately active and short.

### Pass 1: orient, 3 minutes

Read only:

- session outcome;
- scope and prerequisites;
- the first mental model/table;
- the session one-line summary.

Write one sentence predicting what the session will explain.

### Pass 2: learn, 15–25 minutes

Read the concepts and worked examples. For every major section, write the five answers:

```text
Problem:
Owning layer:
Affected state/object:
SQL timing:
Production failure mode:
```

Do not copy paragraphs. Compress them into your own words.

### Pass 3: retrieve, 10 minutes

Close the note. Answer the practice batch and one code trace from memory. If you look at an answer, mark it as a retrieval miss, not as mastered.

### Pass 4: discriminate, 10 minutes

Choose one contrast pair and explain why the wrong alternative would fail in a real scenario. Examples:

- `find()` instead of `getReference()` for authorization;
- DTO instead of entity serialization;
- optimistic instead of pessimistic locking under low contention;
- `flush()` plus `clear()` instead of `clear()` alone in a batch.

### Pass 5: handoff, 3 minutes

Record:

- what was actually covered;
- one genuine gap;
- one topic understood but worth revisiting;
- the exact next starting point;
- the next review date/interval.

A session is not complete because the file was read. It is complete when retrieval and scenario reasoning produce evidence.

---

## 6. Spaced Review Schedule

Use relative intervals so the plan works regardless of the calendar:

| Review point | Time | Task | Passing evidence |
| --- | ---: | --- | --- |
| R0 | End of session | 3–5 questions + one code trace | Explain the core without reopening notes |
| R1 | Next day | Contrast-pair recall | Correctly distinguish three nearby concepts |
| R2 | Three days later | Production scenario | Identify layer, evidence, and fix |
| R3 | Seven days later | Blank-page model | Draw the five-layer flow and state transitions |
| R4 | Fourteen days later | Mixed rapid-fire | 80% correct without fundamental errors |
| R5 | After Session 12 | Mock interview | Explain trade-offs naturally under time pressure |

If a review fails, mark the exact mechanism `[!]` and repeat only that topic after 24–48 hours. Do not reread the entire roadmap.

### Review rotation

- Session 1: lifecycle state table and flush/commit.
- Session 2: save decision tree and query-selection ladder.
- Session 3: owning side/schema drawing.
- Session 4: transaction proxy/propagation trace.
- Session 5: SQL count and fetch-plan choice.
- Session 6: performance evidence and batch design.
- Session 7: snapshot/action-queue/flush trace.
- Session 8: two-transaction conflict timeline.
- Session 9: cache scope/invalidation map.
- Session 10: symptom-to-evidence incident response.

---

## 7. Session Review Card Template

Copy this into the checklist after a live session:

```text
Session:
Date studied:
Sections actually covered:

Core model in my own words:

Three concepts I can distinguish:
1.
2.
3.

Code/scenario I traced:

Retrieval misses:
- 

Fundamental gap to fix:

Topic understood but flagged for later:

Status: mastered / working / needs revision
Next starting topic:
Next review: R1 / R2 / R3 / R4 / R5
```

### Good versus weak evidence

**Weak:** “Read Session 4 and it made sense.”  
**Strong:** “Traced a self-invocation call, identified proxy bypass, and explained why the outer transaction still determines the outcome. Need revision on `NESTED` versus `REQUIRES_NEW`.”

---

## 8. Partial-Session Resume Protocol

If a study chat ends halfway through:

1. Record the last completed heading or question batch.
2. Record the last concept you can explain without looking.
3. Record unresolved questions, not a vague “continue later.”
4. Keep the session status `[~]` or `[!]`; do not mark it mastered.
5. Resume at the next heading, not at the beginning.
6. Start with one two-minute retrieval question from the previous section.

Example:

```text
Session 4 paused after Propagation batch.
Can explain REQUIRED and REQUIRES_NEW.
Need to revisit NESTED savepoint behavior and self-invocation.
Resume with Isolation section, then answer the remaining practice batch.
```

This is the recovery contract for future chats.

---

## 9. Mastery Gate with Observable Evidence

Mark a session `[x]` only when all are true:

- **Explain:** one-minute explanation without notes.
- **Contrast:** three nearby concepts distinguished correctly.
- **Trace:** one code path with entity state, SQL timing, and transaction outcome.
- **Decide:** one production trade-off justified with evidence.
- **Retrieve:** at least 80% of the review batch without opening the answer key.

Minor wording mistakes are acceptable. A wrong layer or lifecycle state is a fundamental gap and should be marked for revision.

---

## 10. The End-to-End Interview Answer Pattern

For any SDE-2 JPA question, answer in this order:

1. **Definition:** one precise sentence.
2. **Mental model:** what state/flow changes.
3. **Execution:** when SQL/transaction/cache behavior occurs.
4. **Scenario:** one concrete example.
5. **Trade-off:** when not to use it.
6. **Verification:** what logs, SQL, metrics, or test would prove it.

Example for N+1:

> N+1 is one root query followed by one relationship query per root. Lazy access in a mapper can cause it. I would verify repeated parameterized SQL and query count growth, then choose a DTO/fetch join/entity graph/batch plan based on cardinality and pagination. I would add a query-count regression test.

This structure keeps answers clear under interview pressure.

---

## Retention Handoff

**Use with:** all Sessions 1–12.  
**Current live checkpoint:** Session 2 — `JpaRepository` -> repository proxy -> `SimpleJpaRepository` -> `save()` end-to-end.  
**Required bridge:** Read [JPA-SDE-2-Foundation-Bridge.md](JPA-SDE-2-Foundation-Bridge.md) for the explicit coverage gaps.  
**Next process change:** Begin every session with retrieval, end with a review card, and update the checklist with actual evidence.

## One-Line Revision Summary

Understand JPA by tracing domain intent through entity state, persistence context, transaction, SQL/database, and consistency; retain it by retrieving contrast pairs and scenario decisions at spaced intervals.
