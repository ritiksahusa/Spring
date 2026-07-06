# Chapter 10 — QA: FetchType and N+1 Query Optimization

---

## Part A — Short Answer Questions (20)

**A1.** What is FetchType in JPA?

> **Answer:** FetchType defines when a related entity/collection should be loaded from DB:
> - `EAGER` = load immediately with parent
> - `LAZY` = load only when accessed

---

**A2.** What are default fetch types in JPA for major mappings?

> **Answer:**
> - `@OneToOne` → EAGER
> - `@ManyToOne` → EAGER
> - `@OneToMany` → LAZY
> - `@ManyToMany` → LAZY

---

**A3.** Define the N+1 query problem.

> **Answer:** N+1 happens when one query fetches parent rows (1 query), and then each parent row triggers one additional query for children (N queries). Total queries = `1 + N`. Example: fetch 100 patients, then fetch appointments for each patient separately → 101 queries.

---

**A4.** Why is N+1 dangerous in production systems?

> **Answer:** It causes many DB round trips, increased latency, poor throughput, high DB CPU usage, and performance collapse under load. It can look fine with small local data but fail badly with large datasets.

---

**A5.** What does `LEFT JOIN FETCH` do in JPQL?

> **Answer:** It joins related entities and fetches them in the same SQL query. `LEFT JOIN` ensures parents are still included even if they have no children; `FETCH` tells Hibernate to initialize the association immediately.

---

**A6.** Why is `SELECT DISTINCT` often needed with fetch join on one-to-many?

> **Answer:** SQL join returns one row per child, duplicating parent rows for parents with multiple children. `DISTINCT` in JPQL helps Hibernate deduplicate parent entities in memory.

---

**A7.** Why can eager loading trigger StackOverflowError in bidirectional relationships when printing?

> **Answer:** Because recursive object references are traversed repeatedly. Example: `Patient.toString()` prints appointments, each `Appointment.toString()` prints patient, which prints appointments again, causing infinite recursion until stack overflows.

---

**A8.** How do `@ToString.Exclude` and `@JsonIgnore` help in fetch-related issues?

> **Answer:**
> - `@ToString.Exclude` prevents Lombok toString from traversing a relationship, avoiding recursive logs and lazy-loading side effects.
> - `@JsonIgnore` prevents serialization frameworks from traversing that field, reducing accidental lazy initialization and N+1 in APIs.

---

**A9.** Why is "make everything EAGER" a bad optimization strategy?

> **Answer:** It removes control from query design and causes over-fetching, huge joins, duplicate rows, heavy memory usage, and hidden N+1 cascades through deeper associations.

---

**A10.** In the transcript, why did doctor queries still appear after patient+appointment fetch join?

> **Answer:** Because Appointment → Doctor (`@ManyToOne`) was still eager by default. So when appointments were loaded, Hibernate also loaded their doctors (extra queries). To avoid that, either fetch join doctor explicitly in same query or set `@ManyToOne(fetch = LAZY)`.

---

**A11.** Write JPQL to fetch all patients with appointments in one query.

> **Answer:**
> ```java
> @Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
> List<Patient> findAllPatientsWithAppointments();
> ```

---

**A12.** Write JPQL to fetch patients + appointments + each appointment's doctor in one query.

> **Answer:**
> ```java
> @Query("""
>        SELECT DISTINCT p
>        FROM Patient p
>        LEFT JOIN FETCH p.appointments a
>        LEFT JOIN FETCH a.doctor
>        """)
> List<Patient> findAllPatientsWithAppointmentsAndDoctor();
> ```

---

**A13.** Should we always use deep nested fetch joins to avoid N+1?

> **Answer:** No. Deep fetch joins can produce massive result sets with duplication and poor memory behavior. Use them only for specific use cases where the full graph is actually needed.

---

**A14.** Which side is best to optimize first for N+1: mapping defaults or query methods?

> **Answer:** Start with mapping defaults (`LAZY` by default) and then optimize use-case-specific repository queries using fetch joins or DTO projections.

---

**A15.** Why is DTO projection recommended in APIs for performance-sensitive endpoints?

> **Answer:** DTO projection fetches only needed fields, avoids entity graph traversal, prevents lazy/eager side effects, reduces payload size, and gives explicit control over query shape.

---

**A16.** What happens when you fetch 5 patients and each has EAGER appointments?

> **Answer:** Typically one query loads patients, then five more queries load appointments (N+1 behavior), unless optimized via join fetch or entity graph.

---

**A17.** What is a practical signal in logs that N+1 is occurring?

> **Answer:** One initial SELECT for parent table followed by many repeated SELECT statements differing only by FK parameter (`where patient_id=?`), often exactly equal to number of parent rows.

---

**A18.** How can setting `@ManyToOne(fetch = LAZY)` reduce extra queries?

> **Answer:** It stops automatic loading of parent objects for each child row. Related entity is loaded only if accessed, which avoids unnecessary joins/selects in list endpoints.

---

**A19.** What is the trade-off of reducing queries to one big join?

> **Answer:** Fewer round trips, but potentially much larger result set and more duplicate rows. You trade network chatter for heavier single-query processing. Optimal choice depends on data shape and endpoint needs.

---

**A20.** What is the safest high-level fetch strategy in enterprise projects?

> **Answer:** Keep associations LAZY by default, and for each endpoint create dedicated query methods (`JOIN FETCH` / DTO projection) that fetch exactly what that use case needs.

---

## Part B — Multiple Choice Questions (12)

**B1.** What is the default fetch type for `@ManyToOne`?

- a) LAZY
- b) EAGER ✅
- c) NONE
- d) FETCH

---

**B2.** If `findAll()` returns 20 patients and each patient triggers one query for appointments, total query count is:

- a) 20
- b) 1
- c) 21 ✅
- d) 40

---

**B3.** Which JPQL keyword avoids loading children in separate queries for each parent?

- a) INCLUDE
- b) FETCH JOIN ✅
- c) AUTOLOAD
- d) CASCADE

---

**B4.** Which combination is best for "patient list page" that doesn't show appointments?

- a) appointments EAGER + serialize entities directly
- b) appointments LAZY + DTO projection ✅
- c) appointments EAGER + manual null assignment
- d) appointments LAZY + call getAppointments() for each row

---

**B5.** Why use `DISTINCT` with `LEFT JOIN FETCH p.appointments`?

- a) to sort the results
- b) to remove duplicate parent entities caused by one-to-many join ✅
- c) to avoid foreign keys
- d) required syntax for all JPQL joins

---

**B6.** Which annotation helps prevent recursive logging with Lombok in bidirectional entities?

- a) `@NoArgsConstructor`
- b) `@Builder`
- c) `@ToString.Exclude` ✅
- d) `@GeneratedValue`

---

**B7.** N+1 problem primarily harms:

- a) compile time
- b) database/network runtime performance ✅
- c) code readability only
- d) JVM startup speed

---

**B8.** In a deep graph fetch, what often grows quickly and hurts performance?

- a) number of Java classes
- b) number of DB indexes
- c) join result set size and duplicate rows ✅
- d) thread count

---

**B9.** Which is the best default for collections (`@OneToMany`, `@ManyToMany`)?

- a) EAGER
- b) LAZY ✅
- c) NONE
- d) CASCADE

---

**B10.** What is a common reason for extra doctor queries after fetching patient + appointments?

- a) Missing `@Entity`
- b) Doctor relation default EAGER on `@ManyToOne` ✅
- c) Missing `@Service`
- d) Null patient IDs

---

**B11.** Which approach gives the best control over response shape and query payload?

- a) Direct entity serialization everywhere
- b) EAGER all relations globally
- c) DTO projections with custom query methods ✅
- d) Disable Hibernate logging

---

**B12.** FetchType decides:

- a) how long transaction runs
- b) when association data is loaded ✅
- c) whether table is created
- d) primary key generation strategy

---

## Part C — Scenario and Code Questions (10)

**C1.** You have `Patient -> appointments` and you're seeing N+1 in logs. Write a repository method to fix this.

> **Answer:**
> ```java
> @Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
> List<Patient> findAllPatientsWithAppointments();
> ```

---

**C2.** Now you also need each appointment's doctor in same call. Write optimized JPQL.

> **Answer:**
> ```java
> @Query("""
>        SELECT DISTINCT p
>        FROM Patient p
>        LEFT JOIN FETCH p.appointments a
>        LEFT JOIN FETCH a.doctor
>        """)
> List<Patient> findAllPatientsWithAppointmentsAndDoctors();
> ```

---

**C3.** You don't need doctor details in current API, but doctor relation is `@ManyToOne` default EAGER. What change should you make?

> **Answer:**
> ```java
> @ManyToOne(fetch = FetchType.LAZY)
> @JoinColumn(name = "doctor_id", nullable = false)
> private Doctor doctor;
> ```
> This avoids unnecessary doctor queries.

---

**C4.** A developer sets `fetch = FetchType.EAGER` on `@OneToMany` to "optimize". Explain why this may worsen performance.

> **Answer:** EAGER collection loading may trigger N+1 under normal repository calls, increases payload size, and loads unused child rows. It may also combine badly with other eager relations, creating heavy and duplicate-prone joins.

---

**C5.** Given 1000 patients and each has 10 appointments, compare rough row volume in one fetch join query.

> **Answer:** A patient-appointment fetch join can return roughly 10,000 joined rows (1000 × 10), even though parent objects are only 1000. This can be heavy in memory and transfer size, so use it only when needed.

---

**C6.** Write a test-like snippet to count and print patients using optimized query.

> **Answer:**
> ```java
> @Test
> void testFetchPatientsWithAppointments() {
>     List<Patient> patients = patientRepository.findAllPatientsWithAppointments();
>     System.out.println("patients = " + patients.size());
>     patients.forEach(System.out::println);
> }
> ```

---

**C7.** Your `toString()` causes StackOverflowError due to bidirectional fields. Give one safe fix.

> **Answer:**
> ```java
> // Patient.java
> @ToString.Exclude
> @OneToMany(mappedBy = "patient")
> private List<Appointment> appointments = new ArrayList<>();
>
> // Appointment.java
> @ToString.Exclude
> @ManyToOne(fetch = FetchType.LAZY)
> private Patient patient;
> ```

---

**C8.** Why might query count drop but response time still remain high after adding nested fetch joins?

> **Answer:** Because one huge join may still be expensive: wider rows, larger data transfer, duplicate rows, and heavy hydration in JVM. Query count alone is not enough; measure total time and payload too.

---

**C9.** You only need patient name and number of appointments. What's better than returning full entities?

> **Answer:** Use a DTO projection query:
> ```java
> @Query("SELECT new com.app.dto.PatientSummaryDto(p.id, p.name, COUNT(a)) " +
>        "FROM Patient p LEFT JOIN p.appointments a GROUP BY p.id, p.name")
> List<PatientSummaryDto> findPatientSummaries();
> ```
> This avoids loading full appointment/doctor graphs.

---

**C10.** Explain practical sequence to fix N+1 in a failing API endpoint.

> **Answer:**
> 1. Enable SQL logs and confirm N+1 pattern.
> 2. Identify exactly what fields endpoint needs.
> 3. Set mappings to LAZY where appropriate.
> 4. Add dedicated query (`JOIN FETCH` or DTO projection).
> 5. Re-run and compare query count + response time.
> 6. Ensure no recursive serialization (`@JsonIgnore`/DTOs).

---

## Part D — Fill in the Blanks (8)

**D1.** N+1 means one query for ______ and N queries for ______.

> **Answer:** parents / children (associations)

---

**D2.** Default fetch type for `@OneToMany` is ______.

> **Answer:** LAZY

---

**D3.** `LEFT JOIN FETCH` loads parent and child in the ______ SQL query.

> **Answer:** same (single)

---

**D4.** `@ManyToOne` is by default ______ fetch, which may cause extra queries.

> **Answer:** EAGER

---

**D5.** To avoid duplicate parent entities in fetch join results, use JPQL keyword ______.

> **Answer:** DISTINCT

---

**D6.** Recursive toString in bidirectional entities can cause ______ error.

> **Answer:** StackOverflowError

---

**D7.** Best general strategy is to keep mappings ______ by default and optimize queries per use case.

> **Answer:** LAZY

---

**D8.** Overusing deep joins may reduce query count but increase result set ______ and memory cost.

> **Answer:** size (volume)

---

## Part E — True or False (8)

**E1.** `@OneToOne` is lazy by default in JPA.

> **Answer:** False — default is EAGER.

---

**E2.** N+1 can occur even when your code seems to execute only one repository method.

> **Answer:** True — hidden lazy/eager loading can trigger additional SQL statements.

---

**E3.** `JOIN FETCH` can be used in JPQL to load associations in one query.

> **Answer:** True.

---

**E4.** Making every relation EAGER always improves performance.

> **Answer:** False — it often worsens performance via over-fetching and large joins.

---

**E5.** `@ToString.Exclude` can prevent recursive logging issues in bidirectional entities.

> **Answer:** True.

---

**E6.** Query count is the only metric needed to validate fetch optimization.

> **Answer:** False — response time, payload size, and DB CPU also matter.

---

**E7.** `@ManyToOne(fetch = LAZY)` can help avoid unnecessary associated entity loads.

> **Answer:** True.

---

**E8.** Using DTO projections can reduce both data transfer and ORM hydration overhead.

> **Answer:** True.

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

**F1.** What's the difference between JPA's `FetchType` and Hibernate's `@Fetch(FetchMode...)` — why do engineers say "FetchType defines WHEN, FetchMode defines HOW"?

> **Answer:** `FetchType` (JPA spec) is a hint about WHEN data loads (immediately vs on-access) but doesn't mandate HOW the SQL is generated. Hibernate's `@Fetch(FetchMode.JOIN | SELECT | SUBSELECT)` controls the actual SQL STRATEGY: `JOIN` = one query with SQL JOIN, `SELECT` = one query per parent (classic N+1, default for lazy), `SUBSELECT` = one extra query fetching ALL children for ALL loaded parents via a subselect — solving N+1 with just 2 total queries. You can combine `FetchType.LAZY` (WHEN) with `@Fetch(FetchMode.SUBSELECT)` (HOW).

**F2.** What is `@BatchSize`, and how does it solve N+1 WITHOUT using `JOIN FETCH`?

> **Answer:** `@BatchSize(size = 10)` batches lazy-association loading: instead of one query per parent, it loads associations for up to 10 parents at once via `WHERE parent_id IN (?,?,...)`. For 100 patients, instead of 100 queries you get ~10 batched queries — without risking `MultipleBagFetchException` from simultaneous `JOIN FETCH`es. Can also be set globally via `hibernate.default_batch_fetch_size`.

**F3.** Why can't you paginate correctly with `JOIN FETCH` on a `@OneToMany`, and what's the "two-query" pattern that solves BOTH pagination and N+1?

> **Answer:** Joining a one-to-many multiplies result rows, so SQL `LIMIT`/`OFFSET` can't cut cleanly at the parent level — Hibernate falls back to in-memory pagination (loading everything first), defeating the purpose. Fix: Query 1 fetches just the page of parent IDs (no fetch join, accurate pagination); Query 2 fetches full data for exactly those IDs with `JOIN FETCH ... WHERE id IN :ids`. Correct pagination AND no N+1, at the cost of 2 queries.

**F4.** What tool would you add to CI to catch N+1 regressions automatically, before they reach production?

> **Answer:** `datasource-proxy` or Hypersistence Utils' query-count assertions wired into `@DataJpaTest`s, failing the build if a code change silently adds extra queries. In production, `hibernate.generate_statistics=true` plus APM tooling (Datadog, New Relic) can alert on abnormal query-per-request counts.

**F5.** How does `spring.jpa.open-in-view=true` (Spring Boot's default) explain why N+1 problems often "hide" in dev but appear suddenly in production?

> **Answer:** With Open-Session-In-View, lazy associations can be accessed ANYWHERE during the request (even accidentally, in JSON serialization walking the object graph) and "just work" by lazily issuing more queries — masking N+1 entirely in small-scale dev/functional testing. At real production scale (hundreds of parent rows vs 2-3 test rows), those invisible extra queries multiply into visible latency/DB load problems.

**F6.** How would you shape both the query AND the response for a `Patient` with nested `Appointments` (each with a `Doctor` summary), avoiding both the fetch-join pagination trap and over-exposing data?

> **Answer:** Use a DTO projection with a separate mapping step, not `JOIN FETCH` against full entities: run the two-query (ID-then-enrich) or Set-based fetch-join pattern to load raw entities efficiently, then map into a `PatientResponseDto` with a nested `List<AppointmentSummaryDto>` (itself embedding just `doctorName`/`appointmentTime`) — avoiding exposing full entity graphs and keeping payload minimal and purpose-built.

**F7.** A nested `JOIN FETCH` query (`p.appointments` then `a.doctor`) still shows duplicate rows even after JPQL `DISTINCT`. Why?

> **Answer:** JPQL `DISTINCT` deduplicates at the entity level (collapsing duplicate `Patient` objects with the same ID after hydration). But with a SECOND level of fan-out (appointments→doctor), genuinely duplicate rows can persist at deeper nesting in older Hibernate versions (pre-Hibernate 6 sometimes needed `hibernate.query.passDistinctThrough=false`). Hibernate 6+ handles this better, but it's important to verify actual returned collection sizes in a test whenever nesting more than one level of fetch joins.

**F8.** Why is "always use `FetchType.LAZY` everywhere" NOT a complete N+1 solution by itself?

> **Answer:** `LAZY` only guarantees data isn't loaded until accessed — it does nothing to prevent N+1 once you DO access it in a loop (`for (Patient p : findAll()) { p.getAppointments().size(); }` still triggers 1+N queries). Laziness just moves WHEN the N+1 happens (load time → access time); it doesn't eliminate the pattern. True fixes require query-level strategies: `JOIN FETCH`, `@EntityGraph`, `@BatchSize`/`SUBSELECT`, or DTO projections.

```
