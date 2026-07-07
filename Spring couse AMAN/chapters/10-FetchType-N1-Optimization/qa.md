# Chapter 10 — QA: FetchType and N+1 Query Optimization

---

## Part A — Short Answer Questions (20)

<a id="qA1"></a>**A1.** What is FetchType in JPA? [↓ Answer](#aA1)

<a id="aA1"></a>> **Answer:** FetchType defines when a related entity/collection should be loaded from DB:
>
> - `EAGER` = load immediately with parent
> - `LAZY` = load only when accessed
>
> [↑ Question](#qA1)

---

<a id="qA2"></a>**A2.** What are default fetch types in JPA for major mappings? [↓ Answer](#aA2)

<a id="aA2"></a>> **Answer:**
>
> - `@OneToOne` → EAGER
> - `@ManyToOne` → EAGER
> - `@OneToMany` → LAZY
> - `@ManyToMany` → LAZY
>
> [↑ Question](#qA2)

---

<a id="qA3"></a>**A3.** Define the N+1 query problem. [↓ Answer](#aA3)

<a id="aA3"></a>> **Answer:** N+1 happens when one query fetches parent rows (1 query), and then each parent row triggers one additional query for children (N queries). Total queries = `1 + N`. Example: fetch 100 patients, then fetch appointments for each patient separately → 101 queries. [↑ Question](#qA3)

---

<a id="qA4"></a>**A4.** Why is N+1 dangerous in production systems? [↓ Answer](#aA4)

<a id="aA4"></a>> **Answer:** It causes many DB round trips, increased latency, poor throughput, high DB CPU usage, and performance collapse under load. It can look fine with small local data but fail badly with large datasets. [↑ Question](#qA4)

---

<a id="qA5"></a>**A5.** What does `LEFT JOIN FETCH` do in JPQL? [↓ Answer](#aA5)

<a id="aA5"></a>> **Answer:** It joins related entities and fetches them in the same SQL query. `LEFT JOIN` ensures parents are still included even if they have no children; `FETCH` tells Hibernate to initialize the association immediately. [↑ Question](#qA5)

---

<a id="qA6"></a>**A6.** Why is `SELECT DISTINCT` often needed with fetch join on one-to-many? [↓ Answer](#aA6)

<a id="aA6"></a>> **Answer:** SQL join returns one row per child, duplicating parent rows for parents with multiple children. `DISTINCT` in JPQL helps Hibernate deduplicate parent entities in memory. [↑ Question](#qA6)

---

<a id="qA7"></a>**A7.** Why can eager loading trigger StackOverflowError in bidirectional relationships when printing? [↓ Answer](#aA7)

<a id="aA7"></a>> **Answer:** Because recursive object references are traversed repeatedly. Example: `Patient.toString()` prints appointments, each `Appointment.toString()` prints patient, which prints appointments again, causing infinite recursion until stack overflows. [↑ Question](#qA7)

---

<a id="qA8"></a>**A8.** How do `@ToString.Exclude` and `@JsonIgnore` help in fetch-related issues? [↓ Answer](#aA8)

<a id="aA8"></a>> **Answer:**
>
> - `@ToString.Exclude` prevents Lombok toString from traversing a relationship, avoiding recursive logs and lazy-loading side effects.
> - `@JsonIgnore` prevents serialization frameworks from traversing that field, reducing accidental lazy initialization and N+1 in APIs.
>
> [↑ Question](#qA8)

---

<a id="qA9"></a>**A9.** Why is "make everything EAGER" a bad optimization strategy? [↓ Answer](#aA9)

<a id="aA9"></a>> **Answer:** It removes control from query design and causes over-fetching, huge joins, duplicate rows, heavy memory usage, and hidden N+1 cascades through deeper associations. [↑ Question](#qA9)

---

<a id="qA10"></a>**A10.** In the transcript, why did doctor queries still appear after patient+appointment fetch join? [↓ Answer](#aA10)

<a id="aA10"></a>> **Answer:** Because Appointment → Doctor (`@ManyToOne`) was still eager by default. So when appointments were loaded, Hibernate also loaded their doctors (extra queries). To avoid that, either fetch join doctor explicitly in same query or set `@ManyToOne(fetch = LAZY)`. [↑ Question](#qA10)

---

<a id="qA11"></a>**A11.** Write JPQL to fetch all patients with appointments in one query. [↓ Answer](#aA11)

<a id="aA11"></a>> **Answer:**
> ```java
> @Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
> List<Patient> findAllPatientsWithAppointments();
> ```
>
> [↑ Question](#qA11)

---

<a id="qA12"></a>**A12.** Write JPQL to fetch patients + appointments + each appointment's doctor in one query. [↓ Answer](#aA12)

<a id="aA12"></a>> **Answer:**
> ```java
> @Query("""
>        SELECT DISTINCT p
>        FROM Patient p
>        LEFT JOIN FETCH p.appointments a
>        LEFT JOIN FETCH a.doctor
>        """)
> List<Patient> findAllPatientsWithAppointmentsAndDoctor();
> ```
>
> [↑ Question](#qA12)

---

<a id="qA13"></a>**A13.** Should we always use deep nested fetch joins to avoid N+1? [↓ Answer](#aA13)

<a id="aA13"></a>> **Answer:** No. Deep fetch joins can produce massive result sets with duplication and poor memory behavior. Use them only for specific use cases where the full graph is actually needed. [↑ Question](#qA13)

---

<a id="qA14"></a>**A14.** Which side is best to optimize first for N+1: mapping defaults or query methods? [↓ Answer](#aA14)

<a id="aA14"></a>> **Answer:** Start with mapping defaults (`LAZY` by default) and then optimize use-case-specific repository queries using fetch joins or DTO projections. [↑ Question](#qA14)

---

<a id="qA15"></a>**A15.** Why is DTO projection recommended in APIs for performance-sensitive endpoints? [↓ Answer](#aA15)

<a id="aA15"></a>> **Answer:** DTO projection fetches only needed fields, avoids entity graph traversal, prevents lazy/eager side effects, reduces payload size, and gives explicit control over query shape. [↑ Question](#qA15)

---

<a id="qA16"></a>**A16.** What happens when you fetch 5 patients and each has EAGER appointments? [↓ Answer](#aA16)

<a id="aA16"></a>> **Answer:** Typically one query loads patients, then five more queries load appointments (N+1 behavior), unless optimized via join fetch or entity graph. [↑ Question](#qA16)

---

<a id="qA17"></a>**A17.** What is a practical signal in logs that N+1 is occurring? [↓ Answer](#aA17)

<a id="aA17"></a>> **Answer:** One initial SELECT for parent table followed by many repeated SELECT statements differing only by FK parameter (`where patient_id=?`), often exactly equal to number of parent rows. [↑ Question](#qA17)

---

<a id="qA18"></a>**A18.** How can setting `@ManyToOne(fetch = LAZY)` reduce extra queries? [↓ Answer](#aA18)

<a id="aA18"></a>> **Answer:** It stops automatic loading of parent objects for each child row. Related entity is loaded only if accessed, which avoids unnecessary joins/selects in list endpoints. [↑ Question](#qA18)

---

<a id="qA19"></a>**A19.** What is the trade-off of reducing queries to one big join? [↓ Answer](#aA19)

<a id="aA19"></a>> **Answer:** Fewer round trips, but potentially much larger result set and more duplicate rows. You trade network chatter for heavier single-query processing. Optimal choice depends on data shape and endpoint needs. [↑ Question](#qA19)

---

<a id="qA20"></a>**A20.** What is the safest high-level fetch strategy in enterprise projects? [↓ Answer](#aA20)

<a id="aA20"></a>> **Answer:** Keep associations LAZY by default, and for each endpoint create dedicated query methods (`JOIN FETCH` / DTO projection) that fetch exactly what that use case needs. [↑ Question](#qA20)

---

## Part B — Multiple Choice Questions (12)

<a id="qB1"></a>**B1.** What is the default fetch type for `@ManyToOne`? [↓ Answer](#aB1)

- a) LAZY
- b) <a id="aB1"></a>EAGER ✅ [↑ Question](#qB1)
- c) NONE
- d) FETCH

---

<a id="qB2"></a>**B2.** If `findAll()` returns 20 patients and each patient triggers one query for appointments, total query count is: [↓ Answer](#aB2)

- a) 20
- b) 1
- c) <a id="aB2"></a>21 ✅ [↑ Question](#qB2)
- d) 40

---

<a id="qB3"></a>**B3.** Which JPQL keyword avoids loading children in separate queries for each parent? [↓ Answer](#aB3)

- a) INCLUDE
- b) <a id="aB3"></a>FETCH JOIN ✅ [↑ Question](#qB3)
- c) AUTOLOAD
- d) CASCADE

---

<a id="qB4"></a>**B4.** Which combination is best for "patient list page" that doesn't show appointments? [↓ Answer](#aB4)

- a) appointments EAGER + serialize entities directly
- b) <a id="aB4"></a>appointments LAZY + DTO projection ✅ [↑ Question](#qB4)
- c) appointments EAGER + manual null assignment
- d) appointments LAZY + call getAppointments() for each row

---

<a id="qB5"></a>**B5.** Why use `DISTINCT` with `LEFT JOIN FETCH p.appointments`? [↓ Answer](#aB5)

- a) to sort the results
- b) <a id="aB5"></a>to remove duplicate parent entities caused by one-to-many join ✅ [↑ Question](#qB5)
- c) to avoid foreign keys
- d) required syntax for all JPQL joins

---

<a id="qB6"></a>**B6.** Which annotation helps prevent recursive logging with Lombok in bidirectional entities? [↓ Answer](#aB6)

- a) `@NoArgsConstructor`
- b) `@Builder`
- c) <a id="aB6"></a>`@ToString.Exclude` ✅ [↑ Question](#qB6)
- d) `@GeneratedValue`

---

<a id="qB7"></a>**B7.** N+1 problem primarily harms: [↓ Answer](#aB7)

- a) compile time
- b) <a id="aB7"></a>database/network runtime performance ✅ [↑ Question](#qB7)
- c) code readability only
- d) JVM startup speed

---

<a id="qB8"></a>**B8.** In a deep graph fetch, what often grows quickly and hurts performance? [↓ Answer](#aB8)

- a) number of Java classes
- b) number of DB indexes
- c) <a id="aB8"></a>join result set size and duplicate rows ✅ [↑ Question](#qB8)
- d) thread count

---

<a id="qB9"></a>**B9.** Which is the best default for collections (`@OneToMany`, `@ManyToMany`)? [↓ Answer](#aB9)

- a) EAGER
- b) <a id="aB9"></a>LAZY ✅ [↑ Question](#qB9)
- c) NONE
- d) CASCADE

---

<a id="qB10"></a>**B10.** What is a common reason for extra doctor queries after fetching patient + appointments? [↓ Answer](#aB10)

- a) Missing `@Entity`
- b) <a id="aB10"></a>Doctor relation default EAGER on `@ManyToOne` ✅ [↑ Question](#qB10)
- c) Missing `@Service`
- d) Null patient IDs

---

<a id="qB11"></a>**B11.** Which approach gives the best control over response shape and query payload? [↓ Answer](#aB11)

- a) Direct entity serialization everywhere
- b) EAGER all relations globally
- c) <a id="aB11"></a>DTO projections with custom query methods ✅ [↑ Question](#qB11)
- d) Disable Hibernate logging

---

<a id="qB12"></a>**B12.** FetchType decides: [↓ Answer](#aB12)

- a) how long transaction runs
- b) <a id="aB12"></a>when association data is loaded ✅ [↑ Question](#qB12)
- c) whether table is created
- d) primary key generation strategy

---

## Part C — Scenario and Code Questions (10)

<a id="qC1"></a>**C1.** You have `Patient -> appointments` and you're seeing N+1 in logs. Write a repository method to fix this. [↓ Answer](#aC1)

<a id="aC1"></a>> **Answer:**
> ```java
> @Query("SELECT DISTINCT p FROM Patient p LEFT JOIN FETCH p.appointments")
> List<Patient> findAllPatientsWithAppointments();
> ```
>
> [↑ Question](#qC1)

---

<a id="qC2"></a>**C2.** Now you also need each appointment's doctor in same call. Write optimized JPQL. [↓ Answer](#aC2)

<a id="aC2"></a>> **Answer:**
> ```java
> @Query("""
>        SELECT DISTINCT p
>        FROM Patient p
>        LEFT JOIN FETCH p.appointments a
>        LEFT JOIN FETCH a.doctor
>        """)
> List<Patient> findAllPatientsWithAppointmentsAndDoctors();
> ```
>
> [↑ Question](#qC2)

---

<a id="qC3"></a>**C3.** You don't need doctor details in current API, but doctor relation is `@ManyToOne` default EAGER. What change should you make? [↓ Answer](#aC3)

<a id="aC3"></a>> **Answer:**
> ```java
> @ManyToOne(fetch = FetchType.LAZY)
> @JoinColumn(name = "doctor_id", nullable = false)
> private Doctor doctor;
> ```
> This avoids unnecessary doctor queries.
>
> [↑ Question](#qC3)

---

<a id="qC4"></a>**C4.** A developer sets `fetch = FetchType.EAGER` on `@OneToMany` to "optimize". Explain why this may worsen performance. [↓ Answer](#aC4)

<a id="aC4"></a>> **Answer:** EAGER collection loading may trigger N+1 under normal repository calls, increases payload size, and loads unused child rows. It may also combine badly with other eager relations, creating heavy and duplicate-prone joins. [↑ Question](#qC4)

---

<a id="qC5"></a>**C5.** Given 1000 patients and each has 10 appointments, compare rough row volume in one fetch join query. [↓ Answer](#aC5)

<a id="aC5"></a>> **Answer:** A patient-appointment fetch join can return roughly 10,000 joined rows (1000 × 10), even though parent objects are only 1000. This can be heavy in memory and transfer size, so use it only when needed. [↑ Question](#qC5)

---

<a id="qC6"></a>**C6.** Write a test-like snippet to count and print patients using optimized query. [↓ Answer](#aC6)

<a id="aC6"></a>> **Answer:**
> ```java
> @Test
> void testFetchPatientsWithAppointments() {
>     List<Patient> patients = patientRepository.findAllPatientsWithAppointments();
>     System.out.println("patients = " + patients.size());
>     patients.forEach(System.out::println);
> }
> ```
>
> [↑ Question](#qC6)

---

<a id="qC7"></a>**C7.** Your `toString()` causes StackOverflowError due to bidirectional fields. Give one safe fix. [↓ Answer](#aC7)

<a id="aC7"></a>> **Answer:**
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
>
> [↑ Question](#qC7)

---

<a id="qC8"></a>**C8.** Why might query count drop but response time still remain high after adding nested fetch joins? [↓ Answer](#aC8)

<a id="aC8"></a>> **Answer:** Because one huge join may still be expensive: wider rows, larger data transfer, duplicate rows, and heavy hydration in JVM. Query count alone is not enough; measure total time and payload too. [↑ Question](#qC8)

---

<a id="qC9"></a>**C9.** You only need patient name and number of appointments. What's better than returning full entities? [↓ Answer](#aC9)

<a id="aC9"></a>> **Answer:** Use a DTO projection query:
> ```java
> @Query("SELECT new com.app.dto.PatientSummaryDto(p.id, p.name, COUNT(a)) " +
>        "FROM Patient p LEFT JOIN p.appointments a GROUP BY p.id, p.name")
> List<PatientSummaryDto> findPatientSummaries();
> ```
> This avoids loading full appointment/doctor graphs.
>
> [↑ Question](#qC9)

---

<a id="qC10"></a>**C10.** Explain practical sequence to fix N+1 in a failing API endpoint. [↓ Answer](#aC10)

<a id="aC10"></a>> **Answer:**
>
> 1. Enable SQL logs and confirm N+1 pattern.
> 2. Identify exactly what fields endpoint needs.
> 3. Set mappings to LAZY where appropriate.
> 4. Add dedicated query (`JOIN FETCH` or DTO projection).
> 5. Re-run and compare query count + response time.
> 6. Ensure no recursive serialization (`@JsonIgnore`/DTOs).
>
> [↑ Question](#qC10)

---

## Part D — Fill in the Blanks (8)

<a id="qD1"></a>**D1.** N+1 means one query for ______ and N queries for ______. [↓ Answer](#aD1)

<a id="aD1"></a>> **Answer:** parents / children (associations)
>
> [↑ Question](#qD1)

---

<a id="qD2"></a>**D2.** Default fetch type for `@OneToMany` is ______. [↓ Answer](#aD2)

<a id="aD2"></a>> **Answer:** LAZY
>
> [↑ Question](#qD2)

---

<a id="qD3"></a>**D3.** `LEFT JOIN FETCH` loads parent and child in the ______ SQL query. [↓ Answer](#aD3)

<a id="aD3"></a>> **Answer:** same (single)
>
> [↑ Question](#qD3)

---

<a id="qD4"></a>**D4.** `@ManyToOne` is by default ______ fetch, which may cause extra queries. [↓ Answer](#aD4)

<a id="aD4"></a>> **Answer:** EAGER
>
> [↑ Question](#qD4)

---

<a id="qD5"></a>**D5.** To avoid duplicate parent entities in fetch join results, use JPQL keyword ______. [↓ Answer](#aD5)

<a id="aD5"></a>> **Answer:** DISTINCT
>
> [↑ Question](#qD5)

---

<a id="qD6"></a>**D6.** Recursive toString in bidirectional entities can cause ______ error. [↓ Answer](#aD6)

<a id="aD6"></a>> **Answer:** StackOverflowError
>
> [↑ Question](#qD6)

---

<a id="qD7"></a>**D7.** Best general strategy is to keep mappings ______ by default and optimize queries per use case. [↓ Answer](#aD7)

<a id="aD7"></a>> **Answer:** LAZY
>
> [↑ Question](#qD7)

---

<a id="qD8"></a>**D8.** Overusing deep joins may reduce query count but increase result set ______ and memory cost. [↓ Answer](#aD8)

<a id="aD8"></a>> **Answer:** size (volume)
>
> [↑ Question](#qD8)

---

## Part E — True or False (8)

<a id="qE1"></a>**E1.** `@OneToOne` is lazy by default in JPA. [↓ Answer](#aE1)

<a id="aE1"></a>> **Answer:** False — default is EAGER.
>
> [↑ Question](#qE1)

---

<a id="qE2"></a>**E2.** N+1 can occur even when your code seems to execute only one repository method. [↓ Answer](#aE2)

<a id="aE2"></a>> **Answer:** True — hidden lazy/eager loading can trigger additional SQL statements.
>
> [↑ Question](#qE2)

---

<a id="qE3"></a>**E3.** `JOIN FETCH` can be used in JPQL to load associations in one query. [↓ Answer](#aE3)

<a id="aE3"></a>> **Answer:** True.
>
> [↑ Question](#qE3)

---

<a id="qE4"></a>**E4.** Making every relation EAGER always improves performance. [↓ Answer](#aE4)

<a id="aE4"></a>> **Answer:** False — it often worsens performance via over-fetching and large joins.
>
> [↑ Question](#qE4)

---

<a id="qE5"></a>**E5.** `@ToString.Exclude` can prevent recursive logging issues in bidirectional entities. [↓ Answer](#aE5)

<a id="aE5"></a>> **Answer:** True.
>
> [↑ Question](#qE5)

---

<a id="qE6"></a>**E6.** Query count is the only metric needed to validate fetch optimization. [↓ Answer](#aE6)

<a id="aE6"></a>> **Answer:** False — response time, payload size, and DB CPU also matter.
>
> [↑ Question](#qE6)

---

<a id="qE7"></a>**E7.** `@ManyToOne(fetch = LAZY)` can help avoid unnecessary associated entity loads. [↓ Answer](#aE7)

<a id="aE7"></a>> **Answer:** True.
>
> [↑ Question](#qE7)

---

<a id="qE8"></a>**E8.** Using DTO projections can reduce both data transfer and ORM hydration overhead. [↓ Answer](#aE8)

<a id="aE8"></a>> **Answer:** True.
>
> [↑ Question](#qE8)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** What's the difference between JPA's `FetchType` and Hibernate's `@Fetch(FetchMode...)` — why do engineers say "FetchType defines WHEN, FetchMode defines HOW"? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** `FetchType` (JPA spec) is a hint about WHEN data loads (immediately vs on-access) but doesn't mandate HOW the SQL is generated. Hibernate's `@Fetch(FetchMode.JOIN | SELECT | SUBSELECT)` controls the actual SQL STRATEGY: `JOIN` = one query with SQL JOIN, `SELECT` = one query per parent (classic N+1, default for lazy), `SUBSELECT` = one extra query fetching ALL children for ALL loaded parents via a subselect — solving N+1 with just 2 total queries. You can combine `FetchType.LAZY` (WHEN) with `@Fetch(FetchMode.SUBSELECT)` (HOW).
>
> [↑ Question](#qf1)

<a id="qf2"></a>**F2.** What is `@BatchSize`, and how does it solve N+1 WITHOUT using `JOIN FETCH`? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** `@BatchSize(size = 10)` batches lazy-association loading: instead of one query per parent, it loads associations for up to 10 parents at once via `WHERE parent_id IN (?,?,...)`. For 100 patients, instead of 100 queries you get ~10 batched queries — without risking `MultipleBagFetchException` from simultaneous `JOIN FETCH`es. Can also be set globally via `hibernate.default_batch_fetch_size`.
>
> [↑ Question](#qf2)

<a id="qf3"></a>**F3.** Why can't you paginate correctly with `JOIN FETCH` on a `@OneToMany`, and what's the "two-query" pattern that solves BOTH pagination and N+1? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** Joining a one-to-many multiplies result rows, so SQL `LIMIT`/`OFFSET` can't cut cleanly at the parent level — Hibernate falls back to in-memory pagination (loading everything first), defeating the purpose. Fix: Query 1 fetches just the page of parent IDs (no fetch join, accurate pagination); Query 2 fetches full data for exactly those IDs with `JOIN FETCH ... WHERE id IN :ids`. Correct pagination AND no N+1, at the cost of 2 queries.
>
> [↑ Question](#qf3)

<a id="qf4"></a>**F4.** What tool would you add to CI to catch N+1 regressions automatically, before they reach production? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** `datasource-proxy` or Hypersistence Utils' query-count assertions wired into `@DataJpaTest`s, failing the build if a code change silently adds extra queries. In production, `hibernate.generate_statistics=true` plus APM tooling (Datadog, New Relic) can alert on abnormal query-per-request counts.
>
> [↑ Question](#qf4)

<a id="qf5"></a>**F5.** How does `spring.jpa.open-in-view=true` (Spring Boot's default) explain why N+1 problems often "hide" in dev but appear suddenly in production? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** With Open-Session-In-View, lazy associations can be accessed ANYWHERE during the request (even accidentally, in JSON serialization walking the object graph) and "just work" by lazily issuing more queries — masking N+1 entirely in small-scale dev/functional testing. At real production scale (hundreds of parent rows vs 2-3 test rows), those invisible extra queries multiply into visible latency/DB load problems.
>
> [↑ Question](#qf5)

<a id="qf6"></a>**F6.** How would you shape both the query AND the response for a `Patient` with nested `Appointments` (each with a `Doctor` summary), avoiding both the fetch-join pagination trap and over-exposing data? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** Use a DTO projection with a separate mapping step, not `JOIN FETCH` against full entities: run the two-query (ID-then-enrich) or Set-based fetch-join pattern to load raw entities efficiently, then map into a `PatientResponseDto` with a nested `List<AppointmentSummaryDto>` (itself embedding just `doctorName`/`appointmentTime`) — avoiding exposing full entity graphs and keeping payload minimal and purpose-built.
>
> [↑ Question](#qf6)

<a id="qf7"></a>**F7.** A nested `JOIN FETCH` query (`p.appointments` then `a.doctor`) still shows duplicate rows even after JPQL `DISTINCT`. Why? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** JPQL `DISTINCT` deduplicates at the entity level (collapsing duplicate `Patient` objects with the same ID after hydration). But with a SECOND level of fan-out (appointments→doctor), genuinely duplicate rows can persist at deeper nesting in older Hibernate versions (pre-Hibernate 6 sometimes needed `hibernate.query.passDistinctThrough=false`). Hibernate 6+ handles this better, but it's important to verify actual returned collection sizes in a test whenever nesting more than one level of fetch joins.
>
> [↑ Question](#qf7)

<a id="qf8"></a>**F8.** Why is "always use `FetchType.LAZY` everywhere" NOT a complete N+1 solution by itself? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** `LAZY` only guarantees data isn't loaded until accessed — it does nothing to prevent N+1 once you DO access it in a loop (`for (Patient p : findAll()) { p.getAppointments().size(); }` still triggers 1+N queries). Laziness just moves WHEN the N+1 happens (load time → access time); it doesn't eliminate the pattern. True fixes require query-level strategies: `JOIN FETCH`, `@EntityGraph`, `@BatchSize`/`SUBSELECT`, or DTO projections.
>
> [↑ Question](#qf8)

```
