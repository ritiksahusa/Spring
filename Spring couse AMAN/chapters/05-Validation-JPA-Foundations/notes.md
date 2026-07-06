# Chapter 5+6 — Request Validation & Hibernate/JPA Foundations
**Timestamps:** `2:48:35 → 3:33:42`
**Topics:** Bean Validation (@Valid, @Email, @NotBlank, @Size), JPA architecture, Hibernate as ORM, Spring Data JPA abstraction, Entity class deep-dive, DDL auto modes, JpaRepository, @GeneratedValue strategies, testing with @SpringBootTest

---

## 1. Why Validation?

APIs accept user input — but that input can be wrong, empty, or malformed.

**Problem example:**
```
POST /students with body: { "name": "A", "email": "123.com" }
```
Without validation this saves garbage data to the DB. We need to **reject bad input early**, at the controller boundary, before it reaches the service or DB.

---

## 2. Adding Spring Boot Validation Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

This brings in `jakarta.validation` (Bean Validation API) backed by **Hibernate Validator** (the reference implementation).

---

## 3. Constraint Annotations (from `jakarta.validation.constraints`)

Apply these on DTO **fields**:

| Annotation | Purpose | Example |
|---|---|---|
| `@NotNull` | Field must not be null | ID fields |
| `@NotBlank` | String must not be null or empty/whitespace | Name, Email |
| `@NotEmpty` | Collection/String must not be empty | Lists |
| `@Email` | Must be a well-formed email address | Email field |
| `@Size(min, max)` | String/Collection length range | Name length |
| `@Min(value)` | Number ≥ value | Age ≥ 0 |
| `@Max(value)` | Number ≤ value | Age ≤ 120 |
| `@Positive` | Number > 0 | Price, Count |
| `@Past` | Date must be in the past | Birth date |
| `@Future` | Date must be in the future | Appointment |
| `@Pattern(regexp)` | Must match regex | Phone number |

### 3.1 Example — Annotated DTO

```java
import jakarta.validation.constraints.*;

public class AddStudentRequestDto {

    @NotBlank(message = "Name is required")
    @Size(min = 3, max = 30, message = "Name should be of length 3 to 30 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Must be a well-formed email address")
    private String email;

    // getters / setters
}
```

---

## 4. Activating Validation — @Valid on Controller

Adding constraints to the DTO does nothing unless you tell Spring to **validate** the input. Add `@Valid` before the `@RequestBody` parameter:

```java
@PostMapping
public ResponseEntity<StudentDto> createStudent(
        @Valid @RequestBody AddStudentRequestDto dto) {   // <-- @Valid here
    return ResponseEntity.status(HttpStatus.CREATED)
                         .body(studentService.createNewStudent(dto));
}
```

**What @Valid does:**
- Before the method body executes, Spring runs all `@Constraint` annotations on the DTO
- If any constraint fails → Spring immediately returns **400 Bad Request** with validation error details
- The method body is never reached

---

## 5. Validation Error Response (400 Bad Request)

When validation fails, Spring returns something like:
```json
{
  "status": 400,
  "errors": ["Name should be of length 3 to 30 characters"],
  "timestamp": "..."
}
```

You can customise the error format with `@RestControllerAdvice` and `@ExceptionHandler(MethodArgumentNotValidException.class)`.

---

## 6. The JPA Architecture Stack

Understanding the layers is critical for interviews:

```
Your Spring Boot Code
        |
        v
[ Spring Data JPA ]   ← Abstraction layer; provides Repository interfaces, findBy methods
        |
        v
[ JPA Specification ] ← jakarta.persistence; defines @Entity, @Id, JPQL, EntityManager APIs
        |
        v
[ Hibernate ORM ]     ← JPA implementation; translates JPQL → SQL, handles ORM mapping
        |
        v
[ JDBC ]              ← Java Database Connectivity; sends SQL to DB, gets results back
        |
        v
[ PostgreSQL Driver ] ← Translates JDBC calls into wire protocol for the specific DB
        |
        v
[ PostgreSQL Database ]
```

### 6.1 JPA vs Hibernate vs Spring Data JPA

| Layer | What it is | Role |
|---|---|---|
| **JPA** | A specification (interface/contract) from Jakarta | Defines annotations (@Entity, @Id) and EntityManager API |
| **Hibernate** | An implementation of JPA | Implements the spec; generates SQL; optimises queries |
| **Spring Data JPA** | An abstraction layer on top of JPA | Provides `JpaRepository`, derived query methods, pagination |
| **JDBC** | Low-level Java API for DB connection | Opens connections, sends raw SQL, reads ResultSets |

> Analogy: JPA is the government specification for translators. Hibernate is a translator who implements those rules. Spring Data JPA is an agency that makes it even easier to hire translators.

### 6.2 Other JPA Implementations
- **EclipseLink** (reference implementation)
- **OpenJPA**
- Spring Boot uses **Hibernate by default** because it includes caching, optimisations, and a mature ecosystem.

---

## 7. What is JDBC?

JDBC = **Java Database Connectivity**

- Low-level API for connecting Java to any relational database
- Works with **JDBC Drivers** — each DB vendor provides a driver (PostgreSQL has its own, MySQL has its own)
- Hibernate uses JDBC internally; developers rarely write raw JDBC in Spring Boot apps

---

## 8. Entity Class — Deep Dive

An Entity is a Java class that maps to a database table (Object-Relational Mapping).

```java
package com.example.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDate;

@Entity                          // Required: makes this class a JPA entity
@Getter @Setter
@ToString                        // Lombok: generates readable toString()
public class Patient {

    @Id                          // Marks the primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private LocalDate birthDate;
    private String email;

    @Enumerated(EnumType.STRING) // Store enum as string in DB
    private Gender gender;       // Enum: MALE, FEMALE, OTHER
}
```

### 8.1 @GeneratedValue Strategies

| Strategy | Behaviour |
|---|---|
| `IDENTITY` | DB auto-increments: 1, 2, 3... (most common for PostgreSQL/MySQL) |
| `SEQUENCE` | Uses a named DB sequence; allows batch-inserts and optimisation |
| `TABLE` | Uses a separate table to track next ID value (overkill, rarely used) |
| `UUID` | Generates a 128-bit universally unique ID (e.g., `550e8400-e29b-...`) |
| `AUTO` | Hibernate picks the best strategy based on the dialect/DB |

> For most applications: use `IDENTITY` or `UUID`.

### 8.2 Field Type Mappings

| Java Type | DB Column Type |
|---|---|
| `Long` | BIGINT |
| `String` | VARCHAR(255) |
| `LocalDate` | DATE |
| `LocalDateTime` | TIMESTAMP |
| `Boolean` | BOOLEAN |
| `Double` / `BigDecimal` | DOUBLE / NUMERIC |
| Enum (with `@Enumerated(STRING)`) | VARCHAR |

### 8.3 Useful Table-Level Annotations

```java
@Entity
@Table(name = "patients",       // custom table name
       uniqueConstraints = {
           @UniqueConstraint(columnNames = "email") // unique constraint on email
       })
public class Patient { ... }
```

---

## 9. DDL Auto Options — Full Reference

`spring.jpa.hibernate.ddl-auto` controls what Hibernate does with the schema on startup:

| Value | On Startup | On Shutdown | When to use |
|---|---|---|---|
| `create` | Drops existing + creates fresh tables | Nothing | Local testing (wipes data every run) |
| `create-drop` | Drops + creates fresh tables | Drops all tables | In-memory DBs, test suites |
| `update` | Creates missing tables/columns; never drops | Nothing | Local development ✅ |
| `validate` | Checks entity ↔ schema match; no changes | Nothing | Production (fail fast if mismatch) |
| `none` | Absolutely nothing | Nothing | Production (schema managed externally) ✅ |

> **Production rule of thumb:** Use `none` + migrate with **Flyway** or **Liquibase**, or use `validate` to detect mismatches.

> **Important:** `update` never drops columns, even if you delete a field from the entity. Old columns stay in DB.

---

## 10. application.properties — Complete Configuration

```properties
# Database connection
spring.datasource.url=jdbc:postgresql://localhost:5432/hospital_db
spring.datasource.username=
spring.datasource.password=

# Security for local: disable SSL
# spring.datasource.url=...&useSSL=false  (for MySQL)

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=false

# Dialect (usually auto-detected — can specify if needed)
# spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

---

## 11. JpaRepository — How it Works Internally

```java
public interface PatientRepository extends JpaRepository<Patient, Long> {
    // Empty interface — all CRUD methods come for free
}
```

**How Spring generates the implementation:**
1. At startup, Spring scans for interfaces extending `JpaRepository`
2. Spring Data JPA generates a proxy class (`SimpleJpaRepository`) at runtime
3. This proxy implements all CRUD methods using the `EntityManager`
4. The proxy is registered as a Spring Bean
5. It gets injected wherever `PatientRepository` is declared

> That's why `@Repository` is optional — Spring Data handles the bean lifecycle automatically.

---

## 12. Testing Repository Methods with @SpringBootTest

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import java.util.List;

@SpringBootTest  // Starts the full Spring application context for the test
class PatientTests {

    @Autowired
    private PatientRepository patientRepository;

    @Test
    void testPatientRepository() {
        List<Patient> patients = patientRepository.findAll();
        System.out.println(patients);
    }
}
```

**Why use test classes instead of running the app?**
- Test specific repository/service logic without starting the full HTTP server
- Faster to run; easier to isolate bugs
- No need to call Postman for every check

---

## 13. Lombok @ToString in Entities

By default, `System.out.println(patient)` prints the Java hashcode — not useful.

**Solution:** Use Lombok's `@ToString`:
```java
@Entity
@ToString
public class Patient { ... }

// If you want to exclude a lazy-loaded or large field:
@Entity
@ToString
public class Patient {
    @ToString.Exclude
    private List<Appointment> appointments;
}
```

Without Lombok, you'd write:
```java
@Override
public String toString() {
    return "Patient{id=" + id + ", name='" + name + "', email='" + email + "'}";
}
```

---

## 14. Connection Pooling — HikariCP

When Spring Boot connects to a database, it doesn't open a new connection for every request — that would be slow. Instead, it uses a **connection pool** (HikariCP by default).

- HikariCP maintains a pool of pre-opened DB connections
- Each thread borrows a connection, uses it, and returns it
- Visible in logs: `Added connection for database...` / `connection id: ...`
- You can configure pool size: `spring.datasource.hikari.maximum-pool-size=10`

---

## 15. Summary — Key Concepts

| Concept | Description |
|---|---|
| `@Valid` | Activates bean validation on a `@RequestBody` parameter |
| `@NotBlank` | Field must not be null, empty, or whitespace-only |
| `@Email` | Field must match email format |
| `@Size(min, max)` | String length must be within range |
| JPA | Jakarta Persistence API — the specification |
| Hibernate | JPA implementation; translates JPQL → SQL |
| Spring Data JPA | Abstraction over JPA; gives you `JpaRepository` |
| JDBC | Low-level Java DB API; Hibernate uses it internally |
| `ddl-auto=update` | Auto-creates missing tables/columns; safe for dev |
| `ddl-auto=validate` | Validates schema matches entity; used in production |
| `@GeneratedValue(IDENTITY)` | DB auto-increments the primary key |
| `@GeneratedValue(UUID)` | Generates a UUID as the primary key |
| `@SpringBootTest` | Loads full Spring context for integration tests |
| HikariCP | Default connection pool in Spring Boot |
| `@ToString.Exclude` | Excludes a field from Lombok's generated toString() |
