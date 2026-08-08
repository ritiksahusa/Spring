# JPA SDE-2 Mastery — Proper Current Checklist

**Last updated:** 2026-08-08  
**Current phase:** Spring Data JPA Internals  
**Exact next topic:** `JpaRepository` -> repository proxy -> `SimpleJpaRepository` -> `save()` end-to-end

## Status Legend

- [ ] Not started
- [~] In progress
- [x] Mastered / interview ready
- [!] Needs revision

# STUDY / MASTERY PROTOCOL — IMPORTANT

The goal of this guide is to **complete the full checklist efficiently while keeping fundamentals strong**.

## How each topic should be taught

1. **Teach the concept first**
   - Explain the mental model clearly.
   - Use a small practical example when useful.
   - Focus on what matters for an SDE-2 interview.

2. **Question batches, not endless loops**
   - Ask interview questions in **small batches** (typically 3–5 questions).
   - The learner answers the batch together.
   - Evaluate the batch as a whole.

3. **Fix fundamentals, but do not over-drill**
   - If the learner is fundamentally wrong or has a real conceptual gap, stop and fix it.
   - If the learner understands the concept but fumbles, uses imperfect wording, or makes a minor mistake, correct it briefly and **move on**.
   - Do not keep repeating the same question until the answer becomes perfect.
   - Do not spend an entire long chat session trying to perfect one small topic.

4. **Use a reasonable mastery threshold**
   A topic can be marked mastered when the learner can:
   - explain the core mental model,
   - distinguish it from closely related concepts,
   - reason through a basic code/scenario question,
   - and give an interview-quality answer with minor wording mistakes allowed.

5. **Keep momentum across the roadmap**
   - Prefer breadth + solid fundamentals over perfectionism.
   - Revisit weak topics later through revision batches, rapid-fire questions, or mock interviews.
   - The checklist is the primary goal: **finish all important topics, then deepen through revision.**

6. **Time-boxing principle**
   - A normal topic should generally take **one focused teaching + question cycle**, not an unlimited mastery loop.
   - If a topic is unusually difficult, spend enough time to fix the fundamental misunderstanding, then move forward and flag it for revision.
   - Do not repeatedly restart previously mastered topics unless the learner's answer shows a genuine fundamental gap.

## Question-batch format

Preferred flow:

**Teach → 3–5 questions → evaluate → fix important gaps → mark status → move on**

If answers are mostly correct:
**brief corrections → mastery/working status → next topic**

If fundamentals are wrong:
**targeted explanation → 1–3 focused follow-ups → move on once the core is clear**

## Progress philosophy

> **Fundamentals must be correct; wording does not need to be perfect.**
>
> **Do not confuse fumbling with not understanding.**
>
> **Do not optimize for mastering one topic at the expense of completing the roadmap.**

---

# 1. JPA Fundamentals

## Core JPA

- [ ] What is JPA?
- [ ] JPA vs Hibernate
- [ ] JPA vs JDBC
- [ ] Entity and entity mapping
- [ ] `@Entity`
- [ ] `@Id`
- [ ] `@GeneratedValue`

## Persistence Context

- [x] Persistence Context — core mental model
- [x] EntityManager — basic mental model
- [x] Entity lifecycle concepts
- [x] NEW vs MANAGED
- [x] `persist()` — basic behavior
- [ ] `find()` vs `getReference()`
- [x] Dirty checking — fundamentals
- [x] Flush vs commit
- [x] First-level cache — fundamentals

**Section status:** Persistence-context core mastered; remaining foundation topics are tracked in [JPA-SDE-2-Foundation-Bridge.md](JPA-SDE-2-Foundation-Bridge.md).

---

# 2. Entity Lifecycle & EntityManager

## Entity States

- [x] NEW / TRANSIENT
- [x] MANAGED
- [x] DETACHED — core behavior demonstrated
- [x] REMOVED

## EntityManager Operations

- [x] `persist()` — NEW → MANAGED
- [x] `merge()` — managed-instance/state-transfer behavior
- [x] `remove()` — core behavior mastered
- [x] `detach()` — core behavior mastered
- [x] `clear()` — core behavior mastered
- [x] `refresh()` — core behavior mastered

## Persistence Context Behavior

- [x] Identity / same-instance behavior
- [x] Dirty checking fundamentals
- [ ] Flush modes
- [ ] Action Queue
- [x] Write-behind — conceptual
- [ ] Write-behind internals

## `merge()` mastery

The learner has demonstrated:

- detached object remains detached after `merge()`
- returned instance is managed
- managed instance is what dirty checking follows
- detached mutations after `merge()` are not tracked
- `merge()` with a new entity can result in INSERT
- existing entity state can result in UPDATE
- `persist()` and `merge()` have different semantics

### Correct mental model

    NEW
      │
      │ persist()
      ▼
    MANAGED — same object

    DETACHED
      │
      │ merge()
      ▼
    MANAGED INSTANCE — returned by merge()

### Interview-grade wording

> `merge()` copies the supplied entity's state into a managed instance and returns that managed instance.

Do not make the absolute claim that `merge()` must always create a brand-new Java object.

**Section status:** Entity lifecycle core mastered (`persist()` / `merge()` / `detach()` / `clear()` / `refresh()` / `remove()`).

---

# 3. Entity Relationships

- [ ] `@ManyToOne`
- [ ] `@OneToMany`
- [ ] `@OneToOne`
- [ ] `@ManyToMany`
- [ ] Owning side
- [ ] `mappedBy`
- [ ] Foreign keys
- [ ] Join tables
- [ ] Cascade types
- [ ] `orphanRemoval`
- [ ] LAZY vs EAGER
- [ ] Bidirectional relationships
- [ ] JSON serialization pitfalls

**Status:** Not started.

---

# 4. Transactions

- [x] Basic transaction boundary
- [x] Commit vs rollback — basic
- [x] Flush vs transaction commit distinction
- [ ] `@Transactional`
- [ ] Spring transaction interceptor
- [ ] Transaction manager
- [ ] Persistence Context + transaction boundaries — deep dive
- [ ] Propagation
- [ ] Isolation
- [ ] Read-only transactions
- [ ] Self-invocation / proxy pitfalls
- [ ] Checked vs unchecked rollback
- [ ] Transaction synchronization
- [ ] OSIV

**Status:** Foundational pieces only.

---

# 5. Spring Data JPA

- [ ] `JpaRepository`
- [ ] `SimpleJpaRepository`
- [ ] `save()` end-to-end
- [ ] `save()` → `persist()` vs `merge()`
- [ ] Derived queries
- [ ] `@Query`
- [ ] JPQL
- [ ] Native queries
- [ ] Specifications
- [ ] Criteria API
- [ ] Pagination
- [ ] Sorting
- [ ] Interface projections
- [ ] DTO projections
- [ ] Query optimization

**Status:** Not started in this mastery track.

---

# 6. Fetching & Performance

- [ ] N+1 query problem
- [ ] `JOIN FETCH`
- [ ] Entity Graphs
- [ ] Batch fetching / `@BatchSize`
- [ ] JDBC batching
- [ ] Pagination pitfalls
- [ ] `LazyInitializationException`
- [ ] EAGER-loading pitfalls
- [ ] Persistence Context memory growth
- [ ] Bulk updates/deletes
- [ ] DB indexes
- [ ] Query plans
- [ ] Connection pool interaction
- [ ] Slow-query diagnosis
- [ ] SQL logging / observability

**Status:** Not started.

---

# 7. Hibernate Internals

- [ ] JPA vs Hibernate architecture
- [ ] Hibernate `Session`
- [ ] Persistence Context implementation
- [ ] Entity state tracking internals
- [ ] Snapshots
- [ ] Dirty checking implementation
- [x] Write-behind — conceptual
- [ ] Action Queue
- [ ] SQL generation
- [ ] Flush process
- [ ] Proxy concepts
- [ ] Bytecode enhancement
- [ ] Hibernate-specific features
- [ ] Hibernate batching internals

**Status:** Only conceptual write-behind exposure.

---

# 8. Locking & Concurrency

- [ ] `@Version`
- [ ] Optimistic locking
- [ ] `OptimisticLockException`
- [ ] Pessimistic locking
- [ ] `PESSIMISTIC_READ`
- [ ] `PESSIMISTIC_WRITE`
- [ ] Lost-update scenarios
- [ ] Lock contention
- [ ] Deadlocks
- [ ] Choosing optimistic vs pessimistic locking

**Status:** Not started.

---

# 9. Caching

- [x] First-level cache fundamentals
- [ ] Second-level cache
- [ ] Query cache
- [ ] Cache invalidation
- [ ] Cache consistency
- [ ] When caching helps/hurts

**Status:** First-level only.

---

# 10. Production Debugging

- [ ] N+1 causing slow APIs
- [ ] Connection pool exhaustion
- [ ] Long-running transactions
- [ ] `LazyInitializationException`
- [ ] Unexpected UPDATEs
- [ ] Excessive SELECTs
- [ ] Large batch processing
- [ ] Deadlocks
- [ ] Lock contention
- [ ] Transaction boundary bugs
- [ ] Persistence Context memory pressure
- [ ] Slow-query diagnosis
- [ ] Production incident case studies

**Status:** Not started.

---

# 11. SDE-2 Interview Mastery

## Completed / demonstrated

- [x] Explain Persistence Context
- [x] Explain EntityManager fundamentals
- [x] Explain dirty checking
- [x] Explain flush vs commit
- [x] Explain `persist()`
- [x] Explain `merge()`
- [x] Explain `persist()` vs `merge()`
- [x] Trace entity states through code
- [x] Predict managed vs detached behavior
- [x] Reason about INSERT vs UPDATE
- [x] Handle modified `merge()` scenarios
- [x] Explain `detach()` vs `clear()`
- [x] Explain `refresh()`
- [x] Explain `remove()` and REMOVED state
- [x] Trace full entity lifecycle scenarios

## Remaining

- [ ] Explain `save()` end-to-end
- [ ] Explain N+1 + multiple solutions
- [ ] Debug transaction scenarios
- [ ] Debug concurrency scenarios
- [ ] Design entity relationships
- [ ] Performance optimization case studies
- [ ] Hibernate rapid-fire
- [ ] JPA rapid-fire
- [ ] Full SDE-2 mock interview

---

# CURRENT EXACT POSITION

    JPA Fundamentals
            │
            ▼
    Persistence Context              ✓
            │
            ▼
    EntityManager                    ✓
            │
            ▼
    Entity Lifecycle                 ✓ core
            │
            ▼
    persist()                        ✓
            │
            ▼
    Dirty Checking                   ✓
            │
            ▼
    Flush vs Commit                  ✓
            │
            ▼
    First-Level Cache                ✓
            │
            ▼
    DETACHED + merge()               ✓ MASTERED
            │
            ▼
    persist() vs merge()             ✓ MASTERED
            │
            ▼
    detach()                         ✓ MASTERED
            │
            ▼
    clear()                          ✓ MASTERED
            │
            ▼
    refresh()                        ✓ MASTERED
            │
            ▼
    remove()                         ✓ MASTERED
            │
            ▼
    Lifecycle interview batch        ✓ COMPLETED
            │
            ▼
    Spring Data JPA save()           ← NEXT

---

# EXACT NEXT LESSON

## Session 2 — Spring Data JPA Internals

Start with:

1. `JpaRepository`
2. Repository proxy concept
3. `SimpleJpaRepository`
4. `save()` end-to-end
5. `save()` → `persist()` vs `merge()`
6. Entity information / new-vs-existing decision
7. Derived queries
8. `@Query`
9. JPQL vs native query
10. Basic pagination and sorting

**Do not restart `merge()`.**

---

# CHAT SESSION ROADMAP — FIXED PATH

## Purpose

This is the **fixed session-by-session plan** for the JPA SDE-2 mastery track.

The objective is not to finish one tiny concept perfectly in one chat. The objective is to reach **SDE-2 interview-level mastery across the entire roadmap within a constrained number of sessions**.

### Session rules

- Each session covers a **meaningful group of related topics**.
- A small topic should normally be grouped with adjacent concepts.
- We go deep enough to understand the mental model and handle SDE-2 questions, but not into implementation trivia unless it is interview-relevant.
- Questions are asked in **batches of 3–5**.
- Minor fumbling is corrected briefly; genuine conceptual gaps are fixed.
- If the fundamentals are clear, **we move forward even if the answer is not perfectly worded**.
- Difficult topics can be flagged for later revision rather than consuming the whole session.
- At the end of every session, record the exact next session number/topic.
- **Do not create a new roadmap in a later chat. Continue this fixed path.**

---

## Fixed Session Plan

### Session 1 — Entity Lifecycle Completion

**Completed:** `detach()` → `clear()` → `refresh()` → `remove()` → lifecycle interview batch

Cover:

- `detach()`
- `clear()`
- `refresh()`
- `remove()`
- REMOVED state
- Lifecycle state transitions
- `detach()` vs `clear()` vs `refresh()` vs `remove()`
- Short lifecycle interview batch

**Depth:** Core mental model + code/scenario reasoning.

**Do NOT:** spend the whole session on `detach()` alone.

---

### Session 2 — Spring Data JPA Internals

Cover:

- `JpaRepository`
- Repository proxy concept
- `SimpleJpaRepository`
- `save()` end-to-end
- `save()` → `persist()` vs `merge()`
- Entity information / new-vs-existing decision
- Derived queries
- `@Query`
- JPQL vs native query
- Basic pagination and sorting

**Depth:** Strong SDE-2 internal flow. Avoid framework-source-code trivia.

**Goal:** Be able to explain what happens when a service calls `repository.save(entity)`.

---

### Session 3 — Entity Relationships

Cover:

- `@ManyToOne`
- `@OneToMany`
- `@OneToOne`
- `@ManyToMany`
- Owning side
- `mappedBy`
- Foreign keys
- Join tables
- Cascade types
- `orphanRemoval`
- LAZY vs EAGER
- Bidirectional relationships
- JSON serialization pitfalls

**Depth:** Mapping decisions + database representation + common bugs.

**Goal:** Design a realistic Order/Customer/Product relationship and explain the choices.

---

### Session 4 — Transactions & Spring Transaction Management

Cover:

- `@Transactional`
- Transaction interceptor / proxy
- Transaction manager
- Persistence Context + transaction boundary
- Commit / rollback
- Propagation
- Isolation
- Read-only transactions
- Self-invocation / proxy pitfalls
- Checked vs unchecked rollback
- Transaction synchronization
- OSIV

**Depth:** High. This is a major SDE-2 topic.

**Goal:** Trace a service call from proxy → transaction → EntityManager → flush → commit/rollback.

---

### Session 5 — Fetching & N+1

Cover:

- Lazy loading
- `LazyInitializationException`
- N+1 query problem
- `JOIN FETCH`
- Entity Graphs
- Batch fetching / `@BatchSize`
- EAGER-loading pitfalls
- Pagination + fetching pitfalls

**Depth:** High and practical.

**Goal:** Diagnose an API that executes hundreds of SQL queries and explain multiple valid fixes.

---

### Session 6 — Performance & Database Interaction

Cover:

- JDBC batching
- Bulk updates/deletes
- Persistence Context memory growth
- DB indexes
- Query plans
- Connection pool interaction
- Slow-query diagnosis
- SQL logging / observability

**Depth:** SDE-2 production reasoning, not database-administration depth.

**Goal:** Connect JPA behavior → Hibernate → JDBC → connection pool → database.

---

### Session 7 — Hibernate Internals

Cover:

- JPA vs Hibernate architecture
- Hibernate `Session`
- Persistence Context implementation
- Entity state tracking
- Snapshots
- Dirty checking implementation
- Write-behind
- Action Queue
- SQL generation
- Flush process
- Proxy concepts
- Bytecode enhancement
- Hibernate-specific features
- Hibernate batching internals

**Depth:** Conceptual internals with enough detail for SDE-2 interviews.

**Do NOT:** spend excessive time memorizing Hibernate source-code classes.

---

### Session 8 — Locking & Concurrency

Cover:

- `@Version`
- Optimistic locking
- `OptimisticLockException`
- Pessimistic locking
- `PESSIMISTIC_READ`
- `PESSIMISTIC_WRITE`
- Lost updates
- Lock contention
- Deadlocks
- Choosing optimistic vs pessimistic locking

**Depth:** High for scenarios.

**Goal:** Explain how concurrent updates behave and choose the correct locking strategy.

---

### Session 9 — Caching

Cover:

- First-level cache recap
- Second-level cache
- Query cache
- Cache invalidation
- Cache consistency
- When caching helps/hurts

**Depth:** Moderate. Focus on when/why, not provider-specific trivia.

---

### Session 10 — Production Debugging

Cover:

- N+1 causing slow APIs
- Connection pool exhaustion
- Long-running transactions
- `LazyInitializationException`
- Unexpected UPDATEs
- Excessive SELECTs
- Large batch processing
- Deadlocks
- Lock contention
- Transaction boundary bugs
- Persistence Context memory pressure
- Slow-query diagnosis
- Production incident case studies

**Depth:** Very practical SDE-2 troubleshooting.

**Goal:** Given symptoms + logs/SQL, identify likely JPA/Hibernate causes and propose fixes.

---

### Session 11 — JPA/Hibernate Rapid-Fire Revision

Cover:

- JPA fundamentals
- Entity lifecycle
- EntityManager
- `persist()` / `merge()` / `remove()` / `detach()` / `clear()` / `refresh()`
- Dirty checking
- Flush vs commit
- First-level cache
- Relationships
- Transactions
- Spring Data JPA
- Fetching
- Performance
- Hibernate internals
- Locking
- Caching

**Format:** Mostly interview questions, short scenarios, code tracing.

**Goal:** Find remaining weak spots without restarting entire chapters.

---

### Session 12 — Full SDE-2 Mock Interview

Cover:

- Architecture questions
- Entity relationship design
- Transaction scenarios
- `save()` internals
- N+1 diagnosis
- Concurrency scenarios
- Performance debugging
- Hibernate internals
- Production incident scenarios
- Rapid-fire fundamentals

**Format:** Interview simulation.

**Goal:** Answer naturally, reason aloud, and identify final revision areas.

---

## Session Allocation Summary

| Session | Chapter | Priority | Depth |
| --- | --- | --- | --- |
| 1 | Entity Lifecycle Completion | High | Core |
| 2 | Spring Data JPA | High | Deep |
| 3 | Relationships | High | Deep |
| 4 | Transactions | Very High | Deep |
| 5 | Fetching + N+1 | Very High | Deep |
| 6 | Performance + DB interaction | Very High | Deep |
| 7 | Hibernate Internals | High | Moderate/Deep |
| 8 | Locking + Concurrency | Very High | Deep |
| 9 | Caching | Medium | Moderate |
| 10 | Production Debugging | Very High | Deep |
| 11 | Rapid-Fire Revision | High | Broad |
| 12 | Full SDE-2 Mock | Very High | Interview |

### Expected total: **12 focused chat sessions**

This is intentionally a **12-session plan**, rather than one session per checklist subsection.

---

# SESSION HANDOFF PROTOCOL

At the end of every chat session, the guide must record:

**Completed session:** Session N  
**Topics covered:** actual topics discussed  
**Fundamental gaps fixed:** only genuine gaps  
**Topics flagged for revision:** optional  
**Current status:** mastered / working / needs revision  
**Next session:** Session N+1  
**Next starting topic:** exact first topic from the fixed roadmap

The next chat must start from that recorded position.

### Critical rule

> **Never redesign the roadmap because a previous session spent longer than expected on one topic.**
>
> If a topic takes longer, fix the fundamentals, mark the remaining depth for later revision, and continue with the next planned session.

---

# CURRENT SESSION POSITION

**Completed:** Session 1 — Entity Lifecycle Completion

Start with:

`detach()` → `clear()` → `refresh()` → `remove()` → lifecycle comparison → interview batch

**Next:** Session 2 — Spring Data JPA Internals

Start with:

`JpaRepository` → repository proxy → `SimpleJpaRepository` → `save()` end-to-end

Do not restart `merge()`.

---

# SESSION HANDOFF — SESSION 1

**Completed session:** Session 1  
**Topics covered:** `detach()`, `clear()`, `refresh()`, `remove()`, REMOVED state, lifecycle transitions, comparison, and lifecycle interview scenarios.  
**Fundamental gaps fixed:**  

- `remove()` marks an entity for removal; it does not necessarily execute SQL `DELETE` immediately.
- `detach()` does not roll back an entity; it makes the entity DETACHED and stops subsequent dirty checking.
- `refresh()` reloads state from the database and the entity remains MANAGED.
- `clear()` detaches all managed entities and does not itself mean flush or rollback.
- Whether a change made before `clear()` reaches the database depends on whether it was flushed.
**Topics flagged for revision:** `clear()` vs `flush()` distinction; lifecycle state tracing.  
**Current status:** Mastered / interview ready  
**Next session:** Session 2  
**Next starting topic:** `JpaRepository`

---

# LONG-TERM LEARNING PATH

Persistence Context
→ Entity lifecycle
→ `persist()`
→ dirty checking
→ flush vs commit
→ first-level cache
→ `detach()`
→ `clear()`
→ `refresh()`
→ `remove()`
→ lifecycle comparison
→ Spring Data JPA `save()`
→ relationships
→ transactions
→ fetching
→ N+1
→ batching / performance
→ locking / concurrency
→ caching
→ Hibernate internals
→ production debugging
→ JPA/Hibernate rapid-fire
→ SDE-2 mock interviews

**Important:** This path is a roadmap, not a requirement to perfect each topic before moving forward.

---

# EXPORT / HANDOFF RULE

When the learner says **"export"**:

1. Update progress first.
2. Record what was actually mastered.
3. Record corrected misconceptions.
4. Record topics that are understood but should be revisited later.
5. Record the exact stopping point.
6. Record the exact next lesson.
7. Preserve the long-term learning path.
8. Preserve the **Study / Mastery Protocol** above.
9. Export a self-contained continuation document.

The export must allow another ChatGPT instance to resume without restarting completed material or over-drilling a previously understood topic.

---

# ONE-LINE RESUME CHECKPOINT

> **Session 1 — Entity Lifecycle Completion is mastered; `persist()`, `merge()`, `detach()`, `clear()`, `refresh()`, and `remove()` are interview ready. Resume with Session 2: `JpaRepository` → repository proxy → `SimpleJpaRepository` → `save()`. Follow the fixed 12-session roadmap.**
