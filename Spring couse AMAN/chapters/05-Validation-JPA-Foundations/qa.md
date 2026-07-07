# Chapter 5+6 — Q&A: Request Validation & JPA Foundations

---

## Part A — Short Answer Questions

<a id="q1"></a>**1. What dependency do you add to enable Bean Validation in Spring Boot?** [↓ Answer](#a1)

<a id="a1"></a>`spring-boot-starter-validation` (brings in Hibernate Validator, the reference implementation of `jakarta.validation`). [↑ Question](#q1)

---

<a id="q2"></a>**2. Where do you place `@NotBlank`, `@Email`, `@Size` etc. — on the Entity or the DTO?** [↓ Answer](#a2)

<a id="a2"></a>On the **DTO** (Data Transfer Object) — specifically on the fields of the class annotated with `@RequestBody`. The DTO is the entry point for user input. [↑ Question](#q2)

---

<a id="q3"></a>**3. Adding `@Email` to a DTO field alone won't validate input. What else is required?** [↓ Answer](#a3)

<a id="a3"></a>You must add `@Valid` before the `@RequestBody` parameter in the controller method:
```java
public ResponseEntity<StudentDto> create(@Valid @RequestBody AddStudentRequestDto dto)
```

[↑ Question](#q3)

---

<a id="q4"></a>**4. What HTTP status code does Spring return when Bean Validation fails?** [↓ Answer](#a4)

<a id="a4"></a>**400 Bad Request** — the constraint violation is detected before the controller method body runs, and Spring returns a 400 automatically. [↑ Question](#q4)

---

<a id="q5"></a>**5. What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?** [↓ Answer](#a5)

<a id="a5"></a>

| Annotation | Accepts | Rejects |
|---|---|---|
| `@NotNull` | Any non-null value, including `""` | `null` |
| `@NotEmpty` | Non-empty string/collection | `null`, `""` |
| `@NotBlank` | Non-whitespace string | `null`, `""`, `"   "` |

For user-facing text fields, `@NotBlank` is almost always the right choice. [↑ Question](#q5)

---

<a id="q6"></a>**6. What is JPA? Is it a library or a specification?** [↓ Answer](#a6)

<a id="a6"></a>JPA (Jakarta Persistence API) is a **specification** — a set of rules, annotations, and interfaces that defines how Java objects should map to relational databases. It does not provide an implementation itself. [↑ Question](#q6)

---

<a id="q7"></a>**7. What is Hibernate? How does it relate to JPA?** [↓ Answer](#a7)

<a id="a7"></a>Hibernate is a **JPA implementation** (ORM framework). It implements the JPA specification by:
- Translating JPQL → SQL using **Dialects**
- Mapping Java objects ↔ DB rows (ORM)
- Wrapping JDBC calls

Spring Boot uses Hibernate as its default JPA provider. [↑ Question](#q7)

---

<a id="q8"></a>**8. What is Spring Data JPA and how does it differ from plain JPA/Hibernate?** [↓ Answer](#a8)

<a id="a8"></a>Spring Data JPA is an **abstraction layer over JPA** that:
- Provides `JpaRepository` interface with ready-made CRUD methods
- Auto-generates queries from method names (derived queries)
- Adds pagination and sorting support
- Eliminates the need to write `EntityManager` code manually

Plain JPA requires you to inject `EntityManager` and write JPQL manually. Spring Data JPA hides all of that. [↑ Question](#q8)

---

<a id="q9"></a>**9. What is JDBC? Why does Hibernate need it?** [↓ Answer](#a9)

<a id="a9"></a>JDBC = Java Database Connectivity. It is the low-level Java API that:
- Opens connections to any relational database
- Sends SQL strings
- Reads `ResultSet` objects

Hibernate uses JDBC internally to actually execute the generated SQL. Developers using Spring Data JPA rarely write JDBC directly. [↑ Question](#q9)

---

<a id="q10"></a>**10. What is the full architectural stack from your Spring code to the DB?** [↓ Answer](#a10)

<a id="a10"></a>

```
Your Code
  → Spring Data JPA (abstracts repository operations)
    → JPA Spec / EntityManager (JPQL processing)
      → Hibernate (translates JPQL → SQL, ORM)
        → JDBC (sends SQL, reads results)
          → PostgreSQL Driver
            → PostgreSQL Database
```

[↑ Question](#q10)

---

<a id="q11"></a>**11. What are the 5 `ddl-auto` options and when should each be used?** [↓ Answer](#a11)

<a id="a11"></a>

| Value | Use case |
|---|---|
| `create` | Local testing — wipes data every restart |
| `create-drop` | Test suites / in-memory DBs — drops on shutdown |
| `update` | Local dev — auto-adds missing tables/columns |
| `validate` | Production — fails startup if schema mismatch |
| `none` | Production — Hibernate never touches the schema |

[↑ Question](#q11)

---

<a id="q12"></a>**12. What does `@GeneratedValue(strategy = GenerationType.IDENTITY)` mean?** [↓ Answer](#a12)

<a id="a12"></a>The database engine auto-generates the primary key as an incrementing integer (1, 2, 3...). The `IDENTITY` strategy delegates key generation to the DB column (e.g., PostgreSQL's `BIGSERIAL` or MySQL's `AUTO_INCREMENT`). [↑ Question](#q12)

---

<a id="q13"></a>**13. When would you use `GenerationType.UUID` instead of `IDENTITY`?** [↓ Answer](#a13)

<a id="a13"></a>When you need globally unique IDs that:
- Can be generated without querying the DB (useful for distributed systems)
- Don't reveal the count/sequence of records
- Are safe to generate on the client side or across microservices [↑ Question](#q13)

---

<a id="q14"></a>**14. You annotate an entity class with `@Entity`. What happens when the app starts with `ddl-auto=update`?** [↓ Answer](#a14)

<a id="a14"></a>Hibernate checks if a table exists for that entity. If not, it generates a `CREATE TABLE` statement and runs it. If the table exists but is missing columns, it adds the missing columns. Existing data is never deleted. [↑ Question](#q14)

---

<a id="q15"></a>**15. Why is `@Repository` optional on a `JpaRepository` interface?** [↓ Answer](#a15)

<a id="a15"></a>Because Spring Data JPA auto-detects interfaces extending `JpaRepository` and creates a proxy bean for them at startup. `@Repository` is just a stereotype annotation for documentation/clarity — it doesn't change the behaviour for interfaces. [↑ Question](#q15)

---

## Part B — Multiple Choice Questions

<a id="q16"></a>**16. Which annotation must you place on a `@RequestBody` parameter to trigger Jakarta Bean Validation?** [↓ Answer](#a16)

a) `@Validated`
b) `@Valid`
c) `@Constraint`
d) `@BindingResult`

<a id="a16"></a>**Answer: b) `@Valid`** (from `jakarta.validation`) [↑ Question](#q16)

---

<a id="q17"></a>**17. Which Jakarta validation annotation checks that a String matches a valid email pattern?** [↓ Answer](#a17)

a) `@NotBlank`
b) `@Pattern`
c) `@Email`
d) `@Format`

<a id="a17"></a>**Answer: c) `@Email`** [↑ Question](#q17)

---

<a id="q18"></a>**18. Which `ddl-auto` setting is SAFEST for a production database?** [↓ Answer](#a18)

a) `create-drop`
b) `update`
c) `create`
d) `validate`

<a id="a18"></a>**Answer: d) `validate`** — it detects mismatches without making any changes. [↑ Question](#q18)

---

<a id="q19"></a>**19. What does Hibernate do with the following annotation: `@GeneratedValue(strategy = GenerationType.AUTO)`?** [↓ Answer](#a19)

a) Generates a random UUID
b) Chooses the best strategy based on the underlying database dialect
c) Creates a separate sequence table
d) Uses the `IDENTITY` strategy always

<a id="a19"></a>**Answer: b) Chooses the best strategy based on the underlying database dialect** [↑ Question](#q19)

---

<a id="q20"></a>**20. Spring Data JPA is described as an "abstraction layer over JPA". What does this mean?** [↓ Answer](#a20)

a) It replaces JPA completely with its own implementation
b) It provides higher-level convenience features (JpaRepository, derived queries) on top of JPA/Hibernate
c) It is just another name for Hibernate
d) It connects directly to JDBC, bypassing JPA

<a id="a20"></a>**Answer: b) It provides higher-level convenience features on top of JPA/Hibernate** [↑ Question](#q20)

---

<a id="q21"></a>**21. What does `@SpringBootTest` do on a test class?** [↓ Answer](#a21)

a) Runs the test without loading any Spring context
b) Loads the full Spring application context, including all beans and DB connection
c) Only loads the web layer (MockMvc)
d) Disables all Spring auto-configuration

<a id="a21"></a>**Answer: b) Loads the full Spring application context** [↑ Question](#q21)

---

<a id="q22"></a>**22. You added a new field `phoneNumber` to your `Patient` entity with `ddl-auto=update`. What happens to the existing DB?** [↓ Answer](#a22)

a) The DB is dropped and recreated
b) A new `phone_number` column is added to the existing `patient` table
c) An error is thrown
d) The field is silently ignored

<a id="a22"></a>**Answer: b) A new `phone_number` column is added to the existing `patient` table** [↑ Question](#q22)

---

<a id="q23"></a>**23. Which connection pool does Spring Boot use by default?** [↓ Answer](#a23)

a) Apache DBCP
b) C3P0
c) HikariCP
d) Tomcat JDBC Pool

<a id="a23"></a>**Answer: c) HikariCP** [↑ Question](#q23)

---

<a id="q24"></a>**24. You delete the `email` field from your `Patient` entity with `ddl-auto=update`. What happens to the `email` column in the DB?** [↓ Answer](#a24)

a) It is automatically dropped
b) It stays in the DB — `update` never drops columns
c) A validation error is thrown on startup
d) The column is set to NULL

<a id="a24"></a>**Answer: b) It stays in the DB — `update` never drops columns** [↑ Question](#q24)

---

<a id="q25"></a>**25. Which Lombok annotation generates a human-readable `toString()` for an entity?** [↓ Answer](#a25)

a) `@Data`
b) `@Builder`
c) `@ToString`
d) `@Value`

<a id="a25"></a>**Answer: c) `@ToString`** [↑ Question](#q25)

---

## Part C — Scenario / Code-Reading Questions

<a id="q26"></a>**26. Analyse this DTO and identify any issues:** [↓ Answer](#a26)
```java
public class CreateAppointmentDto {
    private String doctorName;

    @Email
    private String patientEmail;

    private LocalDate appointmentDate;
}
```

<a id="a26"></a>Issues:
1. `doctorName` has no validation — a null or blank name can be saved
2. `patientEmail` has `@Email` but no `@NotBlank` — a `null` email will pass `@Email` (null is not checked by `@Email`)
3. `appointmentDate` has no `@Future` or `@NotNull` constraint — a past date or null could be accepted

**Fix:**
```java
@NotBlank(message = "Doctor name is required")
private String doctorName;

@NotBlank(message = "Patient email is required")
@Email(message = "Must be a valid email")
private String patientEmail;

@NotNull(message = "Appointment date is required")
@Future(message = "Appointment must be in the future")
private LocalDate appointmentDate;
```

[↑ Question](#q26)

---

<a id="q27"></a>**27. You have `ddl-auto=create` and run tests. You notice your test data disappears on every run. Why?** [↓ Answer](#a27)

<a id="a27"></a>`create` drops and recreates all tables every time the Spring application context starts. So each `@SpringBootTest` run starts with empty tables. Switch to `ddl-auto=update` or `ddl-auto=none` with a `data.sql` / `@BeforeEach` setup if you need persistent test data. [↑ Question](#q27)

---

<a id="q28"></a>**28. A colleague says: "I put `@Valid` on my method parameter but validation isn't triggering." What might be wrong?** [↓ Answer](#a28)

<a id="a28"></a>Possible causes:
1. The `spring-boot-starter-validation` dependency is missing from `pom.xml`
2. The constraint annotations are on the Entity class instead of the DTO class
3. The method is not called through the Spring MVC/HTTP pipeline (e.g., called directly in a unit test without MockMvc)
4. The class has `@Validated` only at the class level but `@Valid` is needed on the parameter [↑ Question](#q28)

---

<a id="q29"></a>**29. Explain what this log line means:** [↓ Answer](#a29)
```
HikariPool-1 - Added connection conn0: url=jdbc:postgresql://localhost:5432/hospital_db
```

<a id="a29"></a>HikariCP (the connection pool) successfully opened a connection (`conn0`) to the PostgreSQL database `hospital_db` at `localhost:5432`. This confirms the Spring Boot application has connected to the database successfully on startup. [↑ Question](#q29)

---

<a id="q30"></a>**30. You have this entity:**

```java
@Entity
public class Doctor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String specialisation;
}
```
**You run the app with `ddl-auto=update`. What SQL does Hibernate run?** [↓ Answer](#a30)

<a id="a30"></a>

```sql
CREATE TABLE IF NOT EXISTS doctor (
    id BIGSERIAL NOT NULL,
    specialisation VARCHAR(255),
    PRIMARY KEY (id)
);
```

Hibernate generates the DDL automatically from the entity. The `@GeneratedValue(IDENTITY)` maps to `BIGSERIAL` in PostgreSQL (auto-incrementing integer column). [↑ Question](#q30)

---

## Part D — Fill in the Blanks

<a id="q31"></a>**31.** The package where `@Email`, `@NotBlank`, `@Size` etc. come from is `________________`. [↓ Answer](#a31)

<a id="a31"></a>**Answer:** `jakarta.validation.constraints` [↑ Question](#q31)

---

<a id="q32"></a>**32.** `@Valid` triggers validation on the method parameter before the ________________ body executes. [↓ Answer](#a32)

<a id="a32"></a>**Answer:** method / controller method [↑ Question](#q32)

---

<a id="q33"></a>**33.** JPA stands for `________________ Persistence API`. [↓ Answer](#a33)

<a id="a33"></a>**Answer:** Jakarta (formerly Java) [↑ Question](#q33)

---

<a id="q34"></a>**34.** Hibernate is a JPA ________________ that generates SQL queries from JPQL. [↓ Answer](#a34)

<a id="a34"></a>**Answer:** implementation / provider [↑ Question](#q34)

---

<a id="q35"></a>**35.** `GenerationType.________________` generates a UUID as the primary key. [↓ Answer](#a35)

<a id="a35"></a>**Answer:** `UUID` [↑ Question](#q35)

---

<a id="q36"></a>**36.** With `ddl-auto=________________`, existing tables are dropped and recreated every time the application starts. [↓ Answer](#a36)

<a id="a36"></a>**Answer:** `create` [↑ Question](#q36)

---

<a id="q37"></a>**37.** Spring Boot uses ________________ as its default connection pool. [↓ Answer](#a37)

<a id="a37"></a>**Answer:** HikariCP [↑ Question](#q37)

---

<a id="q38"></a>**38.** To exclude a field from Lombok's `@ToString` output, annotate the field with `@ToString.________________`. [↓ Answer](#a38)

<a id="a38"></a>**Answer:** `Exclude` [↑ Question](#q38)

---

## Part E — True / False

<a id="q39"></a>**39.** Adding `@Email` to a DTO field is enough to validate email input — `@Valid` is not needed. [↓ Answer](#a39) <a id="a39"></a>**False** — `@Valid` must be added to the controller parameter. [↑ Question](#q39)

<a id="q40"></a>**40.** JPA and Hibernate are the same thing. [↓ Answer](#a40) <a id="a40"></a>**False** — JPA is a specification; Hibernate is an implementation. [↑ Question](#q40)

<a id="q41"></a>**41.** `ddl-auto=update` will drop a column if you remove the corresponding field from the entity. [↓ Answer](#a41) <a id="a41"></a>**False** — `update` only adds, never drops. [↑ Question](#q41)

<a id="q42"></a>**42.** `@SpringBootTest` loads the full Spring context including DB connections. [↓ Answer](#a42) <a id="a42"></a>**True** [↑ Question](#q42)

<a id="q43"></a>**43.** With `GenerationType.IDENTITY`, the developer must manually assign the `id` field before saving. [↓ Answer](#a43) <a id="a43"></a>**False** — the DB generates it automatically. [↑ Question](#q43)

<a id="q44"></a>**44.** `@NotEmpty` and `@NotBlank` behave identically for String fields. [↓ Answer](#a44) <a id="a44"></a>**False** — `@NotBlank` also rejects strings that contain only whitespace. [↑ Question](#q44)

<a id="q45"></a>**45.** Spring Data JPA replaces Hibernate — once you use Spring Data JPA, Hibernate is no longer involved. [↓ Answer](#a45) <a id="a45"></a>**False** — Spring Data JPA uses Hibernate internally; it is an abstraction on top of it, not a replacement. [↑ Question](#q45)

<a id="q46"></a>**46.** It is safe to use `ddl-auto=update` in a production environment. [↓ Answer](#a46) <a id="a46"></a>**False** — production should use `validate` or `none` to prevent unintended schema changes. [↑ Question](#q46)

<a id="q47"></a>**47.** `@Repository` annotation is mandatory on a `JpaRepository` interface. [↓ Answer](#a47) <a id="a47"></a>**False** — it is optional; Spring Data JPA auto-detects the interface. [↑ Question](#q47)

<a id="q48"></a>**48.** JDBC is used by Hibernate internally to communicate with the database. [↓ Answer](#a48) <a id="a48"></a>**True** [↑ Question](#q48)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** How would you validate that `password` and `confirmPassword` fields match, given `@NotBlank`/`@Size` only see one field at a time? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** Use a class-level custom constraint (cross-field validation): create an annotation like `@FieldsMatch(first="password", second="confirmPassword")` targeting `ElementType.TYPE`, with a `ConstraintValidator<FieldsMatch, YourDto>` whose `isValid()` receives the WHOLE DTO and compares both fields. Field-level annotations can't see sibling fields — that's exactly why class-level constraints exist. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** What are Bean Validation **groups**, and when would you actually need them? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** Groups let the SAME DTO apply DIFFERENT constraints depending on context — e.g., `id` might be `@Null` on create (client must not send one) but `@NotNull` on update. Define marker interfaces (`OnCreate`, `OnUpdate`), annotate constraints with `groups = OnCreate.class`, and trigger group-specific validation with Spring's `@Validated(OnCreate.class)` (plain `@Valid` has no group support). [↑ Question](#qf2)

<a id="qf3"></a>**F3.** What's the real difference between `@Valid` and `@Validated` beyond "one is Spring's"? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** `@Valid` is the standard `jakarta.validation` annotation with NO group support. `@Validated` (Spring's own) supports validation groups AND, when placed at the CLASS level on a `@Service`/`@Component`, enables method-level validation of plain parameters (e.g., `@NotNull String name` on a service method, not just `@RequestBody` DTOs) via `MethodValidationPostProcessor`. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** When would you deliberately drop from Spring Data JPA down to `JdbcTemplate`/raw SQL, even in a JPA-based project? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** For (1) bulk operations where entity-by-entity dirty-checking is too slow, (2) complex reporting queries needing vendor-specific SQL (window functions, CTEs) that JPQL can't express, (3) extremely hot performance paths wanting to bypass persistence-context bookkeeping, or (4) DDL/admin tasks. `JdbcTemplate` trades manual `RowMapper` mapping and lost cascading/dirty-checking for direct SQL control and lower ORM overhead. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** What is `hibernate.jdbc.batch_size`, and why does setting it NOT guarantee batched inserts for `GenerationType.IDENTITY` entities? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** It groups multiple INSERT/UPDATE statements into one JDBC round-trip. But `GenerationType.IDENTITY` forces Hibernate to execute EACH insert individually to retrieve the DB-generated key immediately (needed for cascades) — this DISABLES batching entirely for identity-strategy entities, a common hidden performance gotcha. To actually batch, you typically need `GenerationType.SEQUENCE` with `hibernate.order_inserts=true` instead. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** Production reports "connection pool exhausted" under load. What three root causes would you investigate first? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** (1) Connection leaks — code paths (manual native queries, `@Transactional` methods calling slow external services while holding a connection) that don't release connections; (2) long-running transactions holding connections longer than necessary (e.g., wrapping an external HTTP call inside `@Transactional`); (3) genuinely undersized pool for real concurrent load, requiring scaling or query optimization. HikariCP's `spring.datasource.hikari.leak-detection-threshold` is the first tool to enable to catch #1. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** Why is `@Email` alone (without `@NotBlank`) a common validation bug? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** Per the Bean Validation spec convention, most constraints (including `@Email`) treat `null` as VALID by design, deferring null-checks to `@NotNull`/`@NotBlank`. A field with ONLY `@Email` happily accepts a missing/null email — you must combine it with `@NotBlank` to actually require a present, well-formed value. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** How would you unit-test a custom `ConstraintValidator` WITHOUT starting any Spring context? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** Instantiate a standalone validator via `Validation.buildDefaultValidatorFactory().getValidator()` (pure Jakarta Bean Validation API), call `validator.validate(dtoInstance)` in a plain JUnit test, and assert on the returned `Set<ConstraintViolation<T>>` — faster and more isolated than spinning up a Spring context just to test validation logic. [↑ Question](#qf8)

````
