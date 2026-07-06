# Chapter 14 — FetchType and N+1 Query Optimization
**Timestamps:** `6:36:38 → 6:46:36`
**Topics:** Lazy vs Eager loading, N+1 query problem, join fetch optimization, nested fetch joins, performance tradeoffs

---

## 1. What is FetchType in JPA?

FetchType decides **when** related entities are loaded from the database.

| FetchType | Meaning | Behavior |
|---|---|---|
| `EAGER` | Load immediately | Related entity is fetched along with parent |
| `LAZY` | Load on demand | Related entity/collection fetched only when accessed |

### 1.1 JPA default fetch types

| Relationship | Default |
|---|---|
| `@OneToOne` | `EAGER` |
| `@ManyToOne` | `EAGER` |
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |

---

## 2. StackOverflow Risk in Bidirectional Printing

When entities have bidirectional references and you print them (`toString` / JSON serialization), recursion can happen:

```text
Patient -> appointments -> appointment.patient -> patient.appointments -> ... (infinite)
```

### 2.1 Fix in Lombok

```java
@ToString.Exclude
@OneToMany(mappedBy = "patient")
private List<Appointment> appointments = new ArrayList<>();
```

Also exclude reverse references in child entities if needed.

> For APIs, prefer DTOs and `@JsonIgnore` / `@JsonManagedReference` / `@JsonBackReference` over exposing entities directly.

---

## 3. N+1 Query Problem

### 3.1 Scenario

Suppose you fetch all patients:

```java
List<Patient> patients = patientRepository.findAll();
```

And `Patient` has:

```java
@OneToMany(mappedBy = "patient", fetch = FetchType.EAGER)
private List<Appointment> appointments;
```

Now Hibernate performs:
1. **1 query** to fetch all patients
2. **N queries** (one per patient) to fetch appointments

If there are 5 patients → total 6 queries.

This is called **N+1**:
- `1` = initial parent query
- `N` = one query for each parent row

### 3.2 Why it's bad

- Too many DB round trips
- Slower response times
- High DB load under traffic
- Hidden performance bug (works in dev, fails at scale)

---

## 4. Basic Prevention Strategy

The first and safest optimization:

> **Don't fetch what you don't need.**

If endpoint only needs patient basic details, keep appointments lazy and ignore them in response.

```java
@OneToMany(mappedBy = "patient", fetch = FetchType.LAZY)
@JsonIgnore // or map to DTO
private List<Appointment> appointments;
```

This avoids automatic loading and prevents N+1 in generic list APIs.

---

## 5. Fetch Join Optimization (JPQL)

When you DO need related data, use a custom query with `JOIN FETCH`.

### 5.1 Fetch patients with appointments in one query

```java
@Query("SELECT p FROM Patient p LEFT JOIN FETCH p.appointments")
List<Patient> findAllPatientsWithAppointments();
```

This turns many small queries into a single joined query.

### 5.2 Why LEFT JOIN FETCH?

- `LEFT JOIN` keeps patients even if they have zero appointments
- `FETCH` tells Hibernate to initialize the relationship immediately from the same result set

---

## 6. Nested Fetch Join (appointments + doctor)

If appointment includes doctor and doctor is needed too:

```java
@Query("""
       SELECT p
       FROM Patient p
       LEFT JOIN FETCH p.appointments a
       LEFT JOIN FETCH a.doctor
       """)
List<Patient> findAllPatientsWithAppointmentsAndDoctor();
```

Now one query brings:
- Patient
- Their appointments
- Doctor for each appointment

### 6.1 Caution: More joins ≠ always faster

As joins increase:
- Result set width increases
- Duplicate rows increase (Cartesian-like expansion)
- Memory processing increases

So optimize per use-case, not globally.

---

## 7. Distinct with Fetch Join

Fetch join on one-to-many can return duplicate parent rows in SQL result. Use `DISTINCT` in JPQL:

```java
@Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
List<Patient> findDistinctPatientsWithAppointments();
```

Hibernate deduplicates entities in memory by ID.

---

## 8. ManyToOne EAGER Surprise

`@ManyToOne` is EAGER by default.

Example from transcript: while fetching appointments, Hibernate also fetched doctors automatically (extra queries), because `Appointment -> Doctor` was eager.

### 8.1 Fix

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "doctor_id", nullable = false)
private Doctor doctor;
```

Use explicit LAZY for `@ManyToOne` unless eager is truly required.

---

## 9. Practical Query Flow Comparison

### Case A: Naive `findAll()` + eager collection

```text
Query 1: SELECT * FROM patient
Query 2: SELECT * FROM appointment WHERE patient_id = 1
Query 3: SELECT * FROM appointment WHERE patient_id = 2
Query 4: SELECT * FROM appointment WHERE patient_id = 3
...
```

Total = `1 + N`

### Case B: Fetch join query

```text
Single query:
SELECT ...
FROM patient p
LEFT JOIN appointment a ON a.patient_id = p.id
```

Total = `1`

### Case C: Nested fetch join

```text
Single query:
SELECT ...
FROM patient p
LEFT JOIN appointment a ON a.patient_id = p.id
LEFT JOIN doctor d ON d.id = a.doctor_id
```

Total = `1` but wider and potentially heavier row set.

---

## 10. Recommended Fetch Guidelines

| Situation | Recommendation |
|---|---|
| Default mapping | Keep associations LAZY by default |
| API list endpoints | Return DTO projections, avoid entity graph expansion |
| Need child list immediately | Use query-specific `JOIN FETCH` |
| Need nested child graph | Add limited nested fetch joins intentionally |
| Unsure about performance | Inspect SQL logs and count queries |

---

## 11. Avoiding N+1 in Real Projects

### 11.1 Good baseline

- All collections (`@OneToMany`, `@ManyToMany`) should remain LAZY
- Most `@ManyToOne` should be explicitly LAZY
- Use dedicated repository queries per endpoint
- Use DTO mapping for response shape

### 11.2 Use-case-specific repository methods

```java
public interface PatientRepository extends JpaRepository<Patient, Long> {

    // Basic list (no appointments)
    @Query("SELECT p FROM Patient p")
    List<Patient> findAllBasic();

    // Detailed list with appointments
    @Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
    List<Patient> findAllWithAppointments();

    // Deep detail with appointments + doctor
    @Query("""
           SELECT DISTINCT p
           FROM Patient p
           LEFT JOIN FETCH p.appointments a
           LEFT JOIN FETCH a.doctor
           """)
    List<Patient> findAllWithAppointmentsAndDoctor();
}
```

---

## 12. Key Concepts Summary

| Concept | Description |
|---|---|
| FetchType.LAZY | Load related data only when accessed |
| FetchType.EAGER | Load related data immediately with parent |
| N+1 Problem | `1` parent query + `N` child queries |
| JOIN FETCH | Fetch association in same query to avoid N+1 |
| LEFT JOIN FETCH | Include parents even when child is missing |
| Nested Fetch Join | Load multi-level associations in one query |
| DISTINCT in JPQL | Prevent duplicate parent entities after fetch join |
| Over-joining risk | Single query can become heavy and memory-expensive |
| Best practice | Keep mappings LAZY; optimize per use case with custom queries |
