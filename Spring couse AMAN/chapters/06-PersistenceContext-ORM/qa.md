# Chapter 7+8 — Q&A: PersistenceContext, EntityManager & ORM

---

## Part A — Short Answer Questions

<a id="q1"></a>**1. What is `SimpleJpaRepository` and why is it important?** [↓ Answer](#a1)

<a id="a1"></a>`SimpleJpaRepository<T, ID>` is the concrete class provided by Spring Data JPA that implements all `JpaRepository` interface methods. It uses an injected `EntityManager` to perform all DB operations. When you call `patientRepository.findAll()`, Spring is actually calling `SimpleJpaRepository.findAll()` which calls `entityManager.createQuery(...)`. [↑ Question](#q1)

---

<a id="q2"></a>**2. What is `EntityManager` in JPA?** [↓ Answer](#a2)

<a id="a2"></a>`EntityManager` is the core JPA interface that manages entity operations:
- `persist()` — INSERT
- `find()` — SELECT by PK
- `merge()` — UPDATE (re-attach detached entity)
- `remove()` — DELETE
- `createQuery()` — JPQL query execution

All JpaRepository methods ultimately delegate to `EntityManager` methods. [↑ Question](#q2)

---

<a id="q3"></a>**3. What are the four states in the Hibernate Entity Lifecycle?** [↓ Answer](#a3)

<a id="a3"></a>

1. **Transient** — new Java object, not tracked by JPA, not in DB
2. **Persistent** — entity is managed by the PersistenceContext; DB changes are tracked
3. **Detached** — entity was previously persistent but the session ended; changes not auto-saved
4. **Removed** — entity marked for deletion; deleted from DB on transaction commit

[↑ Question](#q3)

---

<a id="q4"></a>**4. What is the PersistenceContext?** [↓ Answer](#a4)

<a id="a4"></a>PersistenceContext is an in-memory cache (first-level cache) associated with a single transaction. It stores all entities retrieved or saved within that transaction. If the same entity is fetched twice within one transaction, the second call returns the cached object without hitting the DB. [↑ Question](#q4)

---

<a id="q5"></a>**5. What is dirty checking in Hibernate?** [↓ Answer](#a5)

<a id="a5"></a>Dirty checking is Hibernate's mechanism to automatically detect changes made to persistent entities within a transaction. At commit time, Hibernate compares the entity's current state against its snapshot when it was loaded. If changes are detected, it generates and executes an UPDATE SQL query automatically — without requiring an explicit `save()` call. [↑ Question](#q5)

---

<a id="q6"></a>**6. What does `@Transactional` do?** [↓ Answer](#a6)

<a id="a6"></a>It marks a method (or class) as running within a single database transaction. All repository calls within the method share one `PersistenceContext`. If the method completes without error, the transaction is **committed**. If an unchecked exception is thrown, the transaction is **rolled back**. [↑ Question](#q6)

---

<a id="q7"></a>**7. Why does `p1 == p2` return `true` when both use `findById(1L)` inside a `@Transactional` method?** [↓ Answer](#a7)

<a id="a7"></a>Because PersistenceContext caches entities within a transaction. The first `findById(1L)` hits the DB and stores the entity in the cache. The second `findById(1L)` finds the same entity already in the cache and returns the exact same object reference. They point to the same memory address → `p1 == p2` is `true`. [↑ Question](#q7)

---

<a id="q8"></a>**8. What is the difference between `@UniqueConstraint` on `@Table` and `unique = true` on `@Column`?** [↓ Answer](#a8)

<a id="a8"></a>

| Approach | Use when | Syntax |
|---|---|---|
| `@Column(unique = true)` | Single column uniqueness | On the field |
| `@UniqueConstraint` on `@Table` | Multi-column uniqueness OR when you need a named constraint | In `@Table(uniqueConstraints = {...})` |

For single-column uniqueness, `@Column(unique = true)` is simpler. For composite unique constraints (e.g., name + birthDate together must be unique), you must use `@UniqueConstraint`. [↑ Question](#q8)

---

<a id="q9"></a>**9. What is a database index and how does `@Index` help?** [↓ Answer](#a9)

<a id="a9"></a>A DB index is a separate data structure (B-tree or hash) that maps column values to their row locations, enabling fast lookups. `@Index` in JPA tells Hibernate to generate a `CREATE INDEX` statement for that column. This makes `SELECT WHERE birth_date = ?` queries faster.

Trade-off: faster reads (O(log n) vs O(n)) but slower writes (index must be updated on INSERT/UPDATE). [↑ Question](#q9)

---

<a id="q10"></a>**10. What is `@Enumerated(EnumType.STRING)` and why is it preferred over `EnumType.ORDINAL`?** [↓ Answer](#a10)

<a id="a10"></a>`@Enumerated(EnumType.STRING)` stores the enum constant's name as a string (e.g., "A_POSITIVE") in the DB.
`@Enumerated(EnumType.ORDINAL)` stores the position index (0, 1, 2...).

`STRING` is preferred because:
- It is human-readable in the DB
- It is stable — adding or reordering enum values doesn't corrupt existing data
- `ORDINAL` breaks if you insert a new enum value between existing ones [↑ Question](#q10)

---

<a id="q11"></a>**11. What does `@Column(nullable = false)` do?** [↓ Answer](#a11)

<a id="a11"></a>It adds a `NOT NULL` constraint at the **database level**. If you try to insert a record without that column's value, the DB will reject the INSERT with a constraint violation. This is different from `@NotBlank` which validates at the application/controller level. [↑ Question](#q11)

---

<a id="q12"></a>**12. What is `@CreationTimestamp` and where does it come from?** [↓ Answer](#a12)

<a id="a12"></a>`@CreationTimestamp` is a **Hibernate annotation** (`org.hibernate.annotations`) that automatically fills the annotated `LocalDateTime` field with the current timestamp when the entity is first persisted. It does NOT update on subsequent saves. Use `@UpdateTimestamp` for last-modified time. [↑ Question](#q12)

---

<a id="q13"></a>**13. How do you seed initial data into the database on application startup?** [↓ Answer](#a13)

<a id="a13"></a>

1. Create `src/main/resources/data.sql` with INSERT statements
2. Add to `application.properties`:
   ```properties
   spring.jpa.defer-datasource-initialization=true
   spring.sql.init.mode=always
   spring.sql.init.data-locations=classpath:data.sql
   ```

`defer-datasource-initialization=true` ensures Hibernate creates tables first, then data.sql runs. [↑ Question](#q13)

---

<a id="q14"></a>**14. If you rename a Java entity field from `name` to `patientName` with `ddl-auto=update`, what happens to the DB?** [↓ Answer](#a14)

<a id="a14"></a>A new column `patient_name` is added; the old `name` column remains. Old records have `patient_name = NULL` and new records have `name = NULL`. This corrupts the data. To safely rename columns in production, you must use a migration tool like **Flyway** to run `ALTER TABLE ... RENAME COLUMN`. [↑ Question](#q14)

---

<a id="q15"></a>**15. What is the difference between a `@Transactional` method and a non-transactional method when making two `findById()` calls?** [↓ Answer](#a15)

<a id="a15"></a>

| | Without @Transactional | With @Transactional |
|---|---|---|
| DB queries | 2 separate queries | 1 query (2nd uses cache) |
| PersistenceContext | Separate context per call | Shared context for all calls |
| `p1 == p2` | `false` (different objects) | `true` (same cached object) |
| Dirty checking | Disabled | Active on commit |

[↑ Question](#q15)

---

## Part B — Multiple Choice Questions

<a id="q16"></a>**16. Which class is the concrete implementation of `JpaRepository` generated by Spring Data JPA?** [↓ Answer](#a16)

a) `HibernateJpaRepository`
b) `EntityManagerRepository`
c) `SimpleJpaRepository`
d) `DefaultJpaRepository`

<a id="a16"></a>**Answer: c) `SimpleJpaRepository`** [↑ Question](#q16)

---

<a id="q17"></a>**17. An entity is in the TRANSIENT state. You call `patientRepository.save(patient)`. What state does it transition to?** [↓ Answer](#a17)

a) Removed
b) Detached
c) Persistent
d) None — save() has no state effect

<a id="a17"></a>**Answer: c) Persistent** [↑ Question](#q17)

---

<a id="q18"></a>**18. Which annotation marks a method so that all its DB operations run in a single transaction?** [↓ Answer](#a18)

a) `@Persistent`
b) `@Transaction`
c) `@Atomic`
d) `@Transactional`

<a id="a18"></a>**Answer: d) `@Transactional`** [↑ Question](#q18)

---

<a id="q19"></a>**19. You call `em.detach(patient)` on a persistent entity. What happens?** [↓ Answer](#a19)

a) The entity is deleted from the DB
b) The entity is removed from the PersistenceContext; future changes are not tracked
c) The entity is committed to the DB immediately
d) The entity reverts to Transient state

<a id="a19"></a>**Answer: b) The entity is removed from the PersistenceContext; future changes are not tracked** [↑ Question](#q19)

---

<a id="q20"></a>**20. What is the PersistenceContext also called in Hibernate terminology?** [↓ Answer](#a20)

a) Second-level cache
b) Query cache
c) First-level cache
d) Session pool

<a id="a20"></a>**Answer: c) First-level cache** [↑ Question](#q20)

---

<a id="q21"></a>**21. You have `@Column(nullable = false)` on a field. Which level enforces this constraint?** [↓ Answer](#a21)

a) Application level (Java)
b) Controller level (Spring MVC)
c) Database level (SQL NOT NULL)
d) Service level

<a id="a21"></a>**Answer: c) Database level (SQL NOT NULL)** [↑ Question](#q21)

---

<a id="q22"></a>**22. Which `@Enumerated` type is SAFER if you add new values to an existing enum?** [↓ Answer](#a22)

a) `EnumType.ORDINAL`
b) `EnumType.STRING`
c) `EnumType.NUMERIC`
d) Both are equally safe

<a id="a22"></a>**Answer: b) `EnumType.STRING`** — adding new enum values won't affect stored string values. [↑ Question](#q22)

---

<a id="q23"></a>**23. Which statement about `@Index` is TRUE?** [↓ Answer](#a23)

a) Indexes speed up both SELECT and INSERT equally
b) Indexes slow down SELECT but speed up INSERT
c) Indexes speed up SELECT but slow down INSERT
d) Indexes have no effect on performance

<a id="a23"></a>**Answer: c) Indexes speed up SELECT but slow down INSERT** [↑ Question](#q23)

---

<a id="q24"></a>**24. `@CreationTimestamp` is from which package?** [↓ Answer](#a24)

a) `jakarta.persistence`
b) `org.springframework.data.jpa`
c) `org.hibernate.annotations`
d) `java.time`

<a id="a24"></a>**Answer: c) `org.hibernate.annotations`** [↑ Question](#q24)

---

<a id="q25"></a>**25. What happens when a `@Transactional` method throws an unchecked exception?** [↓ Answer](#a25)

a) Transaction is committed as normal
b) Transaction is rolled back
c) Transaction is paused and retried
d) Transaction continues in a new context

<a id="a25"></a>**Answer: b) Transaction is rolled back** [↑ Question](#q25)

---

## Part C — Scenario / Code-Reading Questions

<a id="q26"></a>**26. You have this code:**

```java
@Service
public class PatientService {

    @Transactional
    public void updatePatientName(Long id, String newName) {
        Patient patient = patientRepository.findById(id).orElseThrow();
        patient.setName(newName);
        // Note: No patientRepository.save(patient) here
    }
}
```

**Will the patient's name actually be updated in the DB? Why?** [↓ Answer](#a26)

<a id="a26"></a>Yes. Because the method is `@Transactional`, the `patient` object is in the **Persistent state**. Hibernate tracks all changes via dirty checking. When the transaction commits at the end of the method, Hibernate detects that `name` changed and automatically generates:

```sql
UPDATE patient SET name = ? WHERE id = ?
```

No explicit `save()` is needed. [↑ Question](#q26)

---

<a id="q27"></a>**27. Analyse this code:**

```java
@SpringBootTest
class PatientTests {
    @Autowired
    private PatientRepository patientRepository;

    @Test
    void testTwoFindCalls() {
        Patient p1 = patientRepository.findById(1L).orElseThrow();
        Patient p2 = patientRepository.findById(1L).orElseThrow();
        System.out.println(p1 == p2);
    }
}
```

**What will be printed? Why?** [↓ Answer](#a27)

<a id="a27"></a>`false` will be printed. This test method is NOT annotated with `@Transactional`. Without a shared transaction, each `findById()` call opens its own transaction and creates its own PersistenceContext. Two separate SELECT queries are issued and two different object instances are created — they have the same values but are different Java objects. To get `p1 == p2 = true`, add `@Transactional` on the test method. [↑ Question](#q27)

---

<a id="q28"></a>**28. You want to make the combination of (doctorId, patientId) unique in an Appointment table. Write the appropriate annotation.** [↓ Answer](#a28)

<a id="a28"></a>

```java
@Entity
@Table(
    uniqueConstraints = {
        @UniqueConstraint(
            name = "unique_doctor_patient",
            columnNames = {"doctor_id", "patient_id"}
        )
    }
)
public class Appointment { ... }
```

This ensures that the same doctor cannot have two appointments with the same patient (if that's the business rule). [↑ Question](#q28)

---

<a id="q29"></a>**29. What is wrong with this entity and how would you fix it?**

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.ORDINAL)
    private OrderStatus status;   // Enum: PENDING=0, PROCESSING=1, SHIPPED=2, DELIVERED=3
}
```

[↓ Answer](#a29)

<a id="a29"></a>**Problem:** If you later insert `CANCELLED` between `PROCESSING` and `SHIPPED`, all existing `SHIPPED` records (stored as `2`) will now be mapped to `CANCELLED`, and `DELIVERED` (stored as `3`) will map to `SHIPPED`. **This silently corrupts all existing data.**

**Fix:** Use `EnumType.STRING`:

```java
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

[↑ Question](#q29)

---

<a id="q30"></a>**30. You run the app with `ddl-auto=create` and notice your `data.sql` runs but the tables are empty. What is the fix?** [↓ Answer](#a30)

<a id="a30"></a>The problem is that `data.sql` might run BEFORE Hibernate finishes creating the tables (race condition on startup). The fix:

```properties
spring.jpa.defer-datasource-initialization=true
```

This makes Spring wait until Hibernate has fully initialised (tables created) before running `data.sql`. [↑ Question](#q30)

---

## Part D — Fill in the Blanks

<a id="q31"></a>**31.** The concrete class that Spring Data JPA generates to implement all `JpaRepository` methods is called `________________`. [↓ Answer](#a31)

<a id="a31"></a>**Answer:** `SimpleJpaRepository` [↑ Question](#q31)

---

<a id="q32"></a>**32.** When an entity is in the `________________` state, it is NOT tracked by JPA and any changes will NOT be saved to the DB. [↓ Answer](#a32)

<a id="a32"></a>**Answer:** Transient [↑ Question](#q32)

---

<a id="q33"></a>**33.** Hibernate's `________________` mechanism automatically detects changes to persistent entities and generates UPDATE SQL at commit time. [↓ Answer](#a33)

<a id="a33"></a>**Answer:** dirty checking [↑ Question](#q33)

---

<a id="q34"></a>**34.** `@Transactional` ensures that all operations in a method share a single `________________`, enabling the first-level cache. [↓ Answer](#a34)

<a id="a34"></a>**Answer:** PersistenceContext (or transaction) [↑ Question](#q34)

---

<a id="q35"></a>**35.** To store an enum as its name string in the database, use `@Enumerated(EnumType.________________)`. [↓ Answer](#a35)

<a id="a35"></a>**Answer:** `STRING` [↑ Question](#q35)

---

<a id="q36"></a>**36.** `@CreationTimestamp` is a `________________` annotation (not a JPA annotation). [↓ Answer](#a36)

<a id="a36"></a>**Answer:** Hibernate [↑ Question](#q36)

---

<a id="q37"></a>**37.** With `ddl-auto=update`, renaming a field in an Entity causes Hibernate to create a `________________` column rather than renaming the existing one. [↓ Answer](#a37)

<a id="a37"></a>**Answer:** new [↑ Question](#q37)

---

<a id="q38"></a>**38.** To create a DB index on the `birth_date` column named `idx_patient_bd`, you write: `@Index(name = "idx_patient_bd", columnList = "________________")`. [↓ Answer](#a38)

<a id="a38"></a>**Answer:** `birth_date` [↑ Question](#q38)

---

## Part E — True / False

<a id="q39"></a>**39.** `SimpleJpaRepository` directly communicates with the database by writing raw SQL strings. [↓ Answer](#a39) <a id="a39"></a>**False** — it uses `EntityManager` which uses Hibernate, which generates SQL. [↑ Question](#q39)

<a id="q40"></a>**40.** A Persistent entity is automatically updated in the DB when you change its fields, even without calling `save()`, as long as you are inside a `@Transactional` context. [↓ Answer](#a40) <a id="a40"></a>**True** [↑ Question](#q40)

<a id="q41"></a>**41.** PersistenceContext is a database-level cache. [↓ Answer](#a41) <a id="a41"></a>**False** — it is a Java in-memory cache (first-level cache) per transaction. [↑ Question](#q41)

<a id="q42"></a>**42.** `@Enumerated(EnumType.ORDINAL)` is the safest choice for production because it saves storage. [↓ Answer](#a42) <a id="a42"></a>**False** — `ORDINAL` is fragile and `STRING` is preferred for safety. [↑ Question](#q42)

<a id="q43"></a>**43.** When a `@Transactional` method throws an unchecked exception, all DB changes made in that method are committed. [↓ Answer](#a43) <a id="a43"></a>**False** — they are rolled back. [↑ Question](#q43)

<a id="q44"></a>**44.** `@UniqueConstraint` with multiple column names creates a composite unique constraint (not separate unique constraints on each column). [↓ Answer](#a44) <a id="a44"></a>**True** [↑ Question](#q44)

<a id="q45"></a>**45.** Indexes improve the performance of both SELECT and INSERT operations equally. [↓ Answer](#a45) <a id="a45"></a>**False** — indexes speed up SELECT but slow down INSERT/UPDATE. [↑ Question](#q45)

<a id="q46"></a>**46.** `@Column(updatable = false)` prevents any Java code from changing the field's value. [↓ Answer](#a46) <a id="a46"></a>**False** — it only prevents the DB column from being included in UPDATE SQL. Java code can still change the field value, but Hibernate won't persist that change. [↑ Question](#q46)

<a id="q47"></a>**47.** `defer-datasource-initialization=true` is needed when using `data.sql` with JPA entities so that tables are created before the SQL runs. [↓ Answer](#a47) <a id="a47"></a>**True** [↑ Question](#q47)

<a id="q48"></a>**48.** A Detached entity can be re-attached to a PersistenceContext using `save()` or `merge()`. [↓ Answer](#a48) <a id="a48"></a>**True** [↑ Question](#q48)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** What's the actual behavioral difference between `entityManager.persist(entity)` and `entityManager.merge(entity)` on a DETACHED entity (has a non-null ID)? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** `persist()` on a detached entity throws `PersistentObjectException` ("detached entity passed to persist") — `persist` is strictly for TRANSIENT (new) entities. `merge()` is designed for exactly this case: it copies the detached object's field values onto the corresponding MANAGED entity (loading it if necessary) and RETURNS that managed instance — a classic gotcha is that `em.merge(detached) != detached`; you must use the returned reference, not the one you passed in. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** Why is putting Lombok's `@Data` directly on a bidirectional `@Entity` dangerous? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** `@Data` generates `equals()`/`hashCode()`/`toString()` using ALL fields, including relationships. On bidirectional mappings this causes infinite recursion (`Patient.toString()` → `Insurance.toString()` → `Patient.toString()`... → `StackOverflowError`) and subtle `equals`/`hashCode` bugs with Hibernate proxies (a lazy proxy's hashCode can differ from the fully-loaded entity's, breaking `HashSet`/`HashMap` semantics mid-collection). Best practice: use individual `@Getter`/`@Setter`, exclude relationship fields from `@ToString`/`@EqualsAndHashCode`, and never base identity on a mutable, nullable-before-persist `id`. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** What is the safest way to implement `equals()`/`hashCode()` on a JPA entity, and why is using the auto-generated `id` alone risky? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** A transient entity has `id == null`, so two DIFFERENT transient entities would incorrectly appear "equal" if compared naively. Worse, adding a transient entity to a `HashSet` and THEN persisting it changes its hashCode (null → generated value), corrupting the set's bucket placement. A common safe pattern: assign a `UUID` in the entity's constructor (stable, non-null even before persist) and base `equals`/`hashCode` on that instead of the DB-generated `id`. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** What's the difference between `flush()` and `commit()`, and when does a `SELECT` query trigger an automatic flush you never explicitly requested? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** `flush()` synchronizes in-memory changes to the DB (executes pending SQL) WITHOUT ending the transaction — visible within the same transaction, not yet permanent/external. `commit()` ends the transaction, making changes permanent. With `FlushMode.AUTO` (default), Hibernate auto-flushes BEFORE a JPQL/Criteria query if it might be affected by pending changes — e.g., save a new Patient, then immediately `findByName(...)` in the same transaction; Hibernate flushes first so the query can see the pending row. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** Why does calling `findById(id)` twice in one `@Transactional` method issue only ONE SQL query, but calling `findByName(name)` twice issues TWO — even for the same row? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** `findById()` maps to `EntityManager.find()`, which checks the L1 cache's identity map by primary key BEFORE hitting the DB — a true cache hit skips SQL entirely. `findByName()` always executes a `SELECT ... WHERE name = ?` against the database (Hibernate can't know in advance the row is already cached) — though once fetched, if the primary key is already in the L1 cache, the SAME managed instance is returned, the SQL round-trip still happens both times. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** What does `spring.jpa.open-in-view=true` (Spring Boot's default) actually do, and why do senior engineers often explicitly disable it? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** It keeps the Hibernate session/connection open for the ENTIRE HTTP request, letting lazy collections be accessed anywhere (even the view layer) without `LazyInitializationException`. The downside: it silently masks N+1 problems (lazy loads "just work" everywhere, hiding bad query design), holds DB connections open longer than necessary, and blurs transaction boundaries. Most production teams set `spring.jpa.open-in-view=false` and enforce explicit `@Transactional` service boundaries plus DTOs/fetch-joins. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** You mix a raw native SQL reporting query with JPA-managed writes in the SAME transaction. What ordering bug should you watch for? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** Native queries via `createNativeQuery(...)` bypass Hibernate's dirty-checking and don't reliably auto-flush pending JPA changes first. Forgetting an explicit `em.flush()` before the native query means it can read STALE data (changes not yet flushed to the DB) — a subtle bug that only appears when native queries and entity saves are mixed within one transaction. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** How would you stream millions of rows for a reporting job WITHOUT loading everything into memory or blowing up the PersistenceContext? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** Use a streaming repository method (`Stream<Patient>` return type with `@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "50"))`, wrapped in `@Transactional` to keep the DB cursor open) combined with periodic `entityManager.clear()` to detach processed entities — preventing the L1 cache from growing unbounded, a classic `OutOfMemoryError` cause when iterating huge result sets while every entity stays managed. [↑ Question](#qf8)

````
