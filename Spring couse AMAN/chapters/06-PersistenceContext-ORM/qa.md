# Chapter 7+8 — Q&A: PersistenceContext, EntityManager & ORM

---

## Part A — Short Answer Questions

**1. What is `SimpleJpaRepository` and why is it important?**

`SimpleJpaRepository<T, ID>` is the concrete class provided by Spring Data JPA that implements all `JpaRepository` interface methods. It uses an injected `EntityManager` to perform all DB operations. When you call `patientRepository.findAll()`, Spring is actually calling `SimpleJpaRepository.findAll()` which calls `entityManager.createQuery(...)`.

---

**2. What is `EntityManager` in JPA?**

`EntityManager` is the core JPA interface that manages entity operations:
- `persist()` — INSERT
- `find()` — SELECT by PK
- `merge()` — UPDATE (re-attach detached entity)
- `remove()` — DELETE
- `createQuery()` — JPQL query execution

All JpaRepository methods ultimately delegate to `EntityManager` methods.

---

**3. What are the four states in the Hibernate Entity Lifecycle?**

1. **Transient** — new Java object, not tracked by JPA, not in DB
2. **Persistent** — entity is managed by the PersistenceContext; DB changes are tracked
3. **Detached** — entity was previously persistent but the session ended; changes not auto-saved
4. **Removed** — entity marked for deletion; deleted from DB on transaction commit

---

**4. What is the PersistenceContext?**

PersistenceContext is an in-memory cache (first-level cache) associated with a single transaction. It stores all entities retrieved or saved within that transaction. If the same entity is fetched twice within one transaction, the second call returns the cached object without hitting the DB.

---

**5. What is dirty checking in Hibernate?**

Dirty checking is Hibernate's mechanism to automatically detect changes made to persistent entities within a transaction. At commit time, Hibernate compares the entity's current state against its snapshot when it was loaded. If changes are detected, it generates and executes an UPDATE SQL query automatically — without requiring an explicit `save()` call.

---

**6. What does `@Transactional` do?**

It marks a method (or class) as running within a single database transaction. All repository calls within the method share one `PersistenceContext`. If the method completes without error, the transaction is **committed**. If an unchecked exception is thrown, the transaction is **rolled back**.

---

**7. Why does `p1 == p2` return `true` when both use `findById(1L)` inside a `@Transactional` method?**

Because PersistenceContext caches entities within a transaction. The first `findById(1L)` hits the DB and stores the entity in the cache. The second `findById(1L)` finds the same entity already in the cache and returns the exact same object reference. They point to the same memory address → `p1 == p2` is `true`.

---

**8. What is the difference between `@UniqueConstraint` on `@Table` and `unique = true` on `@Column`?**

| Approach | Use when | Syntax |
|---|---|---|
| `@Column(unique = true)` | Single column uniqueness | On the field |
| `@UniqueConstraint` on `@Table` | Multi-column uniqueness OR when you need a named constraint | In `@Table(uniqueConstraints = {...})` |

For single-column uniqueness, `@Column(unique = true)` is simpler. For composite unique constraints (e.g., name + birthDate together must be unique), you must use `@UniqueConstraint`.

---

**9. What is a database index and how does `@Index` help?**

A DB index is a separate data structure (B-tree or hash) that maps column values to their row locations, enabling fast lookups. `@Index` in JPA tells Hibernate to generate a `CREATE INDEX` statement for that column. This makes `SELECT WHERE birth_date = ?` queries faster.

Trade-off: faster reads (O(log n) vs O(n)) but slower writes (index must be updated on INSERT/UPDATE).

---

**10. What is `@Enumerated(EnumType.STRING)` and why is it preferred over `EnumType.ORDINAL`?**

`@Enumerated(EnumType.STRING)` stores the enum constant's name as a string (e.g., "A_POSITIVE") in the DB.
`@Enumerated(EnumType.ORDINAL)` stores the position index (0, 1, 2...).

`STRING` is preferred because:
- It is human-readable in the DB
- It is stable — adding or reordering enum values doesn't corrupt existing data
- `ORDINAL` breaks if you insert a new enum value between existing ones

---

**11. What does `@Column(nullable = false)` do?**

It adds a `NOT NULL` constraint at the **database level**. If you try to insert a record without that column's value, the DB will reject the INSERT with a constraint violation. This is different from `@NotBlank` which validates at the application/controller level.

---

**12. What is `@CreationTimestamp` and where does it come from?**

`@CreationTimestamp` is a **Hibernate annotation** (`org.hibernate.annotations`) that automatically fills the annotated `LocalDateTime` field with the current timestamp when the entity is first persisted. It does NOT update on subsequent saves. Use `@UpdateTimestamp` for last-modified time.

---

**13. How do you seed initial data into the database on application startup?**

1. Create `src/main/resources/data.sql` with INSERT statements
2. Add to `application.properties`:
   ```properties
   spring.jpa.defer-datasource-initialization=true
   spring.sql.init.mode=always
   spring.sql.init.data-locations=classpath:data.sql
   ```
`defer-datasource-initialization=true` ensures Hibernate creates tables first, then data.sql runs.

---

**14. If you rename a Java entity field from `name` to `patientName` with `ddl-auto=update`, what happens to the DB?**

A new column `patient_name` is added; the old `name` column remains. Old records have `patient_name = NULL` and new records have `name = NULL`. This corrupts the data. To safely rename columns in production, you must use a migration tool like **Flyway** to run `ALTER TABLE ... RENAME COLUMN`.

---

**15. What is the difference between a `@Transactional` method and a non-transactional method when making two `findById()` calls?**

| | Without @Transactional | With @Transactional |
|---|---|---|
| DB queries | 2 separate queries | 1 query (2nd uses cache) |
| PersistenceContext | Separate context per call | Shared context for all calls |
| `p1 == p2` | `false` (different objects) | `true` (same cached object) |
| Dirty checking | Disabled | Active on commit |

---

## Part B — Multiple Choice Questions

**16. Which class is the concrete implementation of `JpaRepository` generated by Spring Data JPA?**

a) `HibernateJpaRepository`
b) `EntityManagerRepository`
c) `SimpleJpaRepository`
d) `DefaultJpaRepository`

**Answer: c) `SimpleJpaRepository`**

---

**17. An entity is in the TRANSIENT state. You call `patientRepository.save(patient)`. What state does it transition to?**

a) Removed
b) Detached
c) Persistent
d) None — save() has no state effect

**Answer: c) Persistent**

---

**18. Which annotation marks a method so that all its DB operations run in a single transaction?**

a) `@Persistent`
b) `@Transaction`
c) `@Atomic`
d) `@Transactional`

**Answer: d) `@Transactional`**

---

**19. You call `em.detach(patient)` on a persistent entity. What happens?**

a) The entity is deleted from the DB
b) The entity is removed from the PersistenceContext; future changes are not tracked
c) The entity is committed to the DB immediately
d) The entity reverts to Transient state

**Answer: b) The entity is removed from the PersistenceContext; future changes are not tracked**

---

**20. What is the PersistenceContext also called in Hibernate terminology?**

a) Second-level cache
b) Query cache
c) First-level cache
d) Session pool

**Answer: c) First-level cache**

---

**21. You have `@Column(nullable = false)` on a field. Which level enforces this constraint?**

a) Application level (Java)
b) Controller level (Spring MVC)
c) Database level (SQL NOT NULL)
d) Service level

**Answer: c) Database level (SQL NOT NULL)**

---

**22. Which `@Enumerated` type is SAFER if you add new values to an existing enum?**

a) `EnumType.ORDINAL`
b) `EnumType.STRING`
c) `EnumType.NUMERIC`
d) Both are equally safe

**Answer: b) `EnumType.STRING`** — adding new enum values won't affect stored string values.

---

**23. Which statement about `@Index` is TRUE?**

a) Indexes speed up both SELECT and INSERT equally
b) Indexes slow down SELECT but speed up INSERT
c) Indexes speed up SELECT but slow down INSERT
d) Indexes have no effect on performance

**Answer: c) Indexes speed up SELECT but slow down INSERT**

---

**24. `@CreationTimestamp` is from which package?**

a) `jakarta.persistence`
b) `org.springframework.data.jpa`
c) `org.hibernate.annotations`
d) `java.time`

**Answer: c) `org.hibernate.annotations`**

---

**25. What happens when a `@Transactional` method throws an unchecked exception?**

a) Transaction is committed as normal
b) Transaction is rolled back
c) Transaction is paused and retried
d) Transaction continues in a new context

**Answer: b) Transaction is rolled back**

---

## Part C — Scenario / Code-Reading Questions

**26. You have this code:**
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
**Will the patient's name actually be updated in the DB? Why?**

Yes. Because the method is `@Transactional`, the `patient` object is in the **Persistent state**. Hibernate tracks all changes via dirty checking. When the transaction commits at the end of the method, Hibernate detects that `name` changed and automatically generates:
```sql
UPDATE patient SET name = ? WHERE id = ?
```
No explicit `save()` is needed.

---

**27. Analyse this code:**
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
**What will be printed? Why?**

`false` will be printed. This test method is NOT annotated with `@Transactional`. Without a shared transaction, each `findById()` call opens its own transaction and creates its own PersistenceContext. Two separate SELECT queries are issued and two different object instances are created — they have the same values but are different Java objects. To get `p1 == p2 = true`, add `@Transactional` on the test method.

---

**28. You want to make the combination of (doctorId, patientId) unique in an Appointment table. Write the appropriate annotation.**

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

This ensures that the same doctor cannot have two appointments with the same patient (if that's the business rule).

---

**29. What is wrong with this entity and how would you fix it?**
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
**Problem:** If you later insert `CANCELLED` between `PROCESSING` and `SHIPPED`, all existing `SHIPPED` records (stored as `2`) will now be mapped to `CANCELLED`, and `DELIVERED` (stored as `3`) will map to `SHIPPED`. **This silently corrupts all existing data.**

**Fix:** Use `EnumType.STRING`:
```java
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

---

**30. You run the app with `ddl-auto=create` and notice your `data.sql` runs but the tables are empty. What is the fix?**

The problem is that `data.sql` might run BEFORE Hibernate finishes creating the tables (race condition on startup). The fix:

```properties
spring.jpa.defer-datasource-initialization=true
```

This makes Spring wait until Hibernate has fully initialised (tables created) before running `data.sql`.

---

## Part D — Fill in the Blanks

**31.** The concrete class that Spring Data JPA generates to implement all `JpaRepository` methods is called `________________`.

**Answer:** `SimpleJpaRepository`

---

**32.** When an entity is in the `________________` state, it is NOT tracked by JPA and any changes will NOT be saved to the DB.

**Answer:** Transient

---

**33.** Hibernate's `________________` mechanism automatically detects changes to persistent entities and generates UPDATE SQL at commit time.

**Answer:** dirty checking

---

**34.** `@Transactional` ensures that all operations in a method share a single `________________`, enabling the first-level cache.

**Answer:** PersistenceContext (or transaction)

---

**35.** To store an enum as its name string in the database, use `@Enumerated(EnumType.________________)`.

**Answer:** `STRING`

---

**36.** `@CreationTimestamp` is a `________________` annotation (not a JPA annotation).

**Answer:** Hibernate

---

**37.** With `ddl-auto=update`, renaming a field in an Entity causes Hibernate to create a `________________` column rather than renaming the existing one.

**Answer:** new

---

**38.** To create a DB index on the `birth_date` column named `idx_patient_bd`, you write: `@Index(name = "idx_patient_bd", columnList = "________________")`.

**Answer:** `birth_date`

---

## Part E — True / False

**39.** `SimpleJpaRepository` directly communicates with the database by writing raw SQL strings. **False** — it uses `EntityManager` which uses Hibernate, which generates SQL.

**40.** A Persistent entity is automatically updated in the DB when you change its fields, even without calling `save()`, as long as you are inside a `@Transactional` context. **True**

**41.** PersistenceContext is a database-level cache. **False** — it is a Java in-memory cache (first-level cache) per transaction.

**42.** `@Enumerated(EnumType.ORDINAL)` is the safest choice for production because it saves storage. **False** — `ORDINAL` is fragile and `STRING` is preferred for safety.

**43.** When a `@Transactional` method throws an unchecked exception, all DB changes made in that method are committed. **False** — they are rolled back.

**44.** `@UniqueConstraint` with multiple column names creates a composite unique constraint (not separate unique constraints on each column). **True**

**45.** Indexes improve the performance of both SELECT and INSERT operations equally. **False** — indexes speed up SELECT but slow down INSERT/UPDATE.

**46.** `@Column(updatable = false)` prevents any Java code from changing the field's value. **False** — it only prevents the DB column from being included in UPDATE SQL. Java code can still change the field value, but Hibernate won't persist that change.

**47.** `defer-datasource-initialization=true` is needed when using `data.sql` with JPA entities so that tables are created before the SQL runs. **True**

**48.** A Detached entity can be re-attached to a PersistenceContext using `save()` or `merge()`. **True**
