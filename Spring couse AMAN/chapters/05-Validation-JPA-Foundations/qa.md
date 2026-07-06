# Chapter 5+6 — Q&A: Request Validation & JPA Foundations

---

## Part A — Short Answer Questions

**1. What dependency do you add to enable Bean Validation in Spring Boot?**

`spring-boot-starter-validation` (brings in Hibernate Validator, the reference implementation of `jakarta.validation`).

---

**2. Where do you place `@NotBlank`, `@Email`, `@Size` etc. — on the Entity or the DTO?**

On the **DTO** (Data Transfer Object) — specifically on the fields of the class annotated with `@RequestBody`. The DTO is the entry point for user input.

---

**3. Adding `@Email` to a DTO field alone won't validate input. What else is required?**

You must add `@Valid` before the `@RequestBody` parameter in the controller method:
```java
public ResponseEntity<StudentDto> create(@Valid @RequestBody AddStudentRequestDto dto)
```

---

**4. What HTTP status code does Spring return when Bean Validation fails?**

**400 Bad Request** — the constraint violation is detected before the controller method body runs, and Spring returns a 400 automatically.

---

**5. What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?**

| Annotation | Accepts | Rejects |
|---|---|---|
| `@NotNull` | Any non-null value, including `""` | `null` |
| `@NotEmpty` | Non-empty string/collection | `null`, `""` |
| `@NotBlank` | Non-whitespace string | `null`, `""`, `"   "` |

For user-facing text fields, `@NotBlank` is almost always the right choice.

---

**6. What is JPA? Is it a library or a specification?**

JPA (Jakarta Persistence API) is a **specification** — a set of rules, annotations, and interfaces that defines how Java objects should map to relational databases. It does not provide an implementation itself.

---

**7. What is Hibernate? How does it relate to JPA?**

Hibernate is a **JPA implementation** (ORM framework). It implements the JPA specification by:
- Translating JPQL → SQL using **Dialects**
- Mapping Java objects ↔ DB rows (ORM)
- Wrapping JDBC calls

Spring Boot uses Hibernate as its default JPA provider.

---

**8. What is Spring Data JPA and how does it differ from plain JPA/Hibernate?**

Spring Data JPA is an **abstraction layer over JPA** that:
- Provides `JpaRepository` interface with ready-made CRUD methods
- Auto-generates queries from method names (derived queries)
- Adds pagination and sorting support
- Eliminates the need to write `EntityManager` code manually

Plain JPA requires you to inject `EntityManager` and write JPQL manually. Spring Data JPA hides all of that.

---

**9. What is JDBC? Why does Hibernate need it?**

JDBC = Java Database Connectivity. It is the low-level Java API that:
- Opens connections to any relational database
- Sends SQL strings
- Reads `ResultSet` objects

Hibernate uses JDBC internally to actually execute the generated SQL. Developers using Spring Data JPA rarely write JDBC directly.

---

**10. What is the full architectural stack from your Spring code to the DB?**

```
Your Code
  → Spring Data JPA (abstracts repository operations)
    → JPA Spec / EntityManager (JPQL processing)
      → Hibernate (translates JPQL → SQL, ORM)
        → JDBC (sends SQL, reads results)
          → PostgreSQL Driver
            → PostgreSQL Database
```

---

**11. What are the 5 `ddl-auto` options and when should each be used?**

| Value | Use case |
|---|---|
| `create` | Local testing — wipes data every restart |
| `create-drop` | Test suites / in-memory DBs — drops on shutdown |
| `update` | Local dev — auto-adds missing tables/columns |
| `validate` | Production — fails startup if schema mismatch |
| `none` | Production — Hibernate never touches the schema |

---

**12. What does `@GeneratedValue(strategy = GenerationType.IDENTITY)` mean?**

The database engine auto-generates the primary key as an incrementing integer (1, 2, 3...). The `IDENTITY` strategy delegates key generation to the DB column (e.g., PostgreSQL's `BIGSERIAL` or MySQL's `AUTO_INCREMENT`).

---

**13. When would you use `GenerationType.UUID` instead of `IDENTITY`?**

When you need globally unique IDs that:
- Can be generated without querying the DB (useful for distributed systems)
- Don't reveal the count/sequence of records
- Are safe to generate on the client side or across microservices

---

**14. You annotate an entity class with `@Entity`. What happens when the app starts with `ddl-auto=update`?**

Hibernate checks if a table exists for that entity. If not, it generates a `CREATE TABLE` statement and runs it. If the table exists but is missing columns, it adds the missing columns. Existing data is never deleted.

---

**15. Why is `@Repository` optional on a `JpaRepository` interface?**

Because Spring Data JPA auto-detects interfaces extending `JpaRepository` and creates a proxy bean for them at startup. `@Repository` is just a stereotype annotation for documentation/clarity — it doesn't change the behaviour for interfaces.

---

## Part B — Multiple Choice Questions

**16. Which annotation must you place on a `@RequestBody` parameter to trigger Jakarta Bean Validation?**

a) `@Validated`
b) `@Valid`
c) `@Constraint`
d) `@BindingResult`

**Answer: b) `@Valid`** (from `jakarta.validation`)

---

**17. Which Jakarta validation annotation checks that a String matches a valid email pattern?**

a) `@NotBlank`
b) `@Pattern`
c) `@Email`
d) `@Format`

**Answer: c) `@Email`**

---

**18. Which `ddl-auto` setting is SAFEST for a production database?**

a) `create-drop`
b) `update`
c) `create`
d) `validate`

**Answer: d) `validate`** — it detects mismatches without making any changes.

---

**19. What does Hibernate do with the following annotation: `@GeneratedValue(strategy = GenerationType.AUTO)`?**

a) Generates a random UUID
b) Chooses the best strategy based on the underlying database dialect
c) Creates a separate sequence table
d) Uses the `IDENTITY` strategy always

**Answer: b) Chooses the best strategy based on the underlying database dialect**

---

**20. Spring Data JPA is described as an "abstraction layer over JPA". What does this mean?**

a) It replaces JPA completely with its own implementation
b) It provides higher-level convenience features (JpaRepository, derived queries) on top of JPA/Hibernate
c) It is just another name for Hibernate
d) It connects directly to JDBC, bypassing JPA

**Answer: b) It provides higher-level convenience features on top of JPA/Hibernate**

---

**21. What does `@SpringBootTest` do on a test class?**

a) Runs the test without loading any Spring context
b) Loads the full Spring application context, including all beans and DB connection
c) Only loads the web layer (MockMvc)
d) Disables all Spring auto-configuration

**Answer: b) Loads the full Spring application context**

---

**22. You added a new field `phoneNumber` to your `Patient` entity with `ddl-auto=update`. What happens to the existing DB?**

a) The DB is dropped and recreated
b) A new `phone_number` column is added to the existing `patient` table
c) An error is thrown
d) The field is silently ignored

**Answer: b) A new `phone_number` column is added to the existing `patient` table**

---

**23. Which connection pool does Spring Boot use by default?**

a) Apache DBCP
b) C3P0
c) HikariCP
d) Tomcat JDBC Pool

**Answer: c) HikariCP**

---

**24. You delete the `email` field from your `Patient` entity with `ddl-auto=update`. What happens to the `email` column in the DB?**

a) It is automatically dropped
b) It stays in the DB — `update` never drops columns
c) A validation error is thrown on startup
d) The column is set to NULL

**Answer: b) It stays in the DB — `update` never drops columns**

---

**25. Which Lombok annotation generates a human-readable `toString()` for an entity?**

a) `@Data`
b) `@Builder`
c) `@ToString`
d) `@Value`

**Answer: c) `@ToString`**

---

## Part C — Scenario / Code-Reading Questions

**26. Analyse this DTO and identify any issues:**
```java
public class CreateAppointmentDto {
    private String doctorName;

    @Email
    private String patientEmail;

    private LocalDate appointmentDate;
}
```

Issues:
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

---

**27. You have `ddl-auto=create` and run tests. You notice your test data disappears on every run. Why?**

`create` drops and recreates all tables every time the Spring application context starts. So each `@SpringBootTest` run starts with empty tables. Switch to `ddl-auto=update` or `ddl-auto=none` with a `data.sql` / `@BeforeEach` setup if you need persistent test data.

---

**28. A colleague says: "I put `@Valid` on my method parameter but validation isn't triggering." What might be wrong?**

Possible causes:
1. The `spring-boot-starter-validation` dependency is missing from `pom.xml`
2. The constraint annotations are on the Entity class instead of the DTO class
3. The method is not called through the Spring MVC/HTTP pipeline (e.g., called directly in a unit test without MockMvc)
4. The class has `@Validated` only at the class level but `@Valid` is needed on the parameter

---

**29. Explain what this log line means:**
```
HikariPool-1 - Added connection conn0: url=jdbc:postgresql://localhost:5432/hospital_db
```

HikariCP (the connection pool) successfully opened a connection (`conn0`) to the PostgreSQL database `hospital_db` at `localhost:5432`. This confirms the Spring Boot application has connected to the database successfully on startup.

---

**30. You have this entity:**
```java
@Entity
public class Doctor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String specialisation;
}
```
**You run the app with `ddl-auto=update`. What SQL does Hibernate run?**

```sql
CREATE TABLE IF NOT EXISTS doctor (
    id BIGSERIAL NOT NULL,
    specialisation VARCHAR(255),
    PRIMARY KEY (id)
);
```

Hibernate generates the DDL automatically from the entity. The `@GeneratedValue(IDENTITY)` maps to `BIGSERIAL` in PostgreSQL (auto-incrementing integer column).

---

## Part D — Fill in the Blanks

**31.** The package where `@Email`, `@NotBlank`, `@Size` etc. come from is `________________`.

**Answer:** `jakarta.validation.constraints`

---

**32.** `@Valid` triggers validation on the method parameter before the ________________ body executes.

**Answer:** method / controller method

---

**33.** JPA stands for `________________ Persistence API`.

**Answer:** Jakarta (formerly Java)

---

**34.** Hibernate is a JPA ________________ that generates SQL queries from JPQL.

**Answer:** implementation / provider

---

**35.** `GenerationType.________________` generates a UUID as the primary key.

**Answer:** `UUID`

---

**36.** With `ddl-auto=________________`, existing tables are dropped and recreated every time the application starts.

**Answer:** `create`

---

**37.** Spring Boot uses ________________ as its default connection pool.

**Answer:** HikariCP

---

**38.** To exclude a field from Lombok's `@ToString` output, annotate the field with `@ToString.________________`.

**Answer:** `Exclude`

---

## Part E — True / False

**39.** Adding `@Email` to a DTO field is enough to validate email input — `@Valid` is not needed. **False** — `@Valid` must be added to the controller parameter.

**40.** JPA and Hibernate are the same thing. **False** — JPA is a specification; Hibernate is an implementation.

**41.** `ddl-auto=update` will drop a column if you remove the corresponding field from the entity. **False** — `update` only adds, never drops.

**42.** `@SpringBootTest` loads the full Spring context including DB connections. **True**

**43.** With `GenerationType.IDENTITY`, the developer must manually assign the `id` field before saving. **False** — the DB generates it automatically.

**44.** `@NotEmpty` and `@NotBlank` behave identically for String fields. **False** — `@NotBlank` also rejects strings that contain only whitespace.

**45.** Spring Data JPA replaces Hibernate — once you use Spring Data JPA, Hibernate is no longer involved. **False** — Spring Data JPA uses Hibernate internally; it is an abstraction on top of it, not a replacement.

**46.** It is safe to use `ddl-auto=update` in a production environment. **False** — production should use `validate` or `none` to prevent unintended schema changes.

**47.** `@Repository` annotation is mandatory on a `JpaRepository` interface. **False** — it is optional; Spring Data JPA auto-detects the interface.

**48.** JDBC is used by Hibernate internally to communicate with the database. **True**
