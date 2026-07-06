# Chapter 9-11 — JPA Query Methods, JPQL, Native Queries, Projection, Pagination
**Timestamps:** `4:17:02 → 5:01:54`
**Topics:** Derived query methods, AND/OR/Between/Containing/OrderBy, @Query JPQL, named parameters, native queries, @Modifying, projection with NEW keyword, Pageable, PageRequest, Page<T>

---

## 1. JPA Derived Query Methods

Spring Data JPA can generate SQL queries automatically from method names in your repository interface. This is called **Derived Query Methods** (also known as JPA Query Methods).

### 1.1 Basic Naming Rules

```java
public interface PatientRepository extends JpaRepository<Patient, Long> {

    // Find by single field
    Optional<Patient> findByName(String name);

    // Find by LocalDate field
    List<Patient> findByBirthDate(LocalDate birthDate);

    // Find by Email
    Optional<Patient> findByEmail(String email);
}
```

**How it works:**
- Spring parses the method name at startup
- It validates that `name`, `birthDate`, `email` are actual fields in `Patient`
- It generates JPQL: `SELECT p FROM Patient p WHERE p.name = :name`
- If a field doesn't exist → **startup fails** (fail-fast, not runtime)

### 1.2 Combining Conditions

```java
// AND: both conditions must match
List<Patient> findByBirthDateAndEmail(LocalDate birthDate, String email);

// OR: either condition matches
List<Patient> findByBirthDateOrEmail(LocalDate birthDate, String email);
```

### 1.3 Comparison Operators in Method Names

| Keyword | SQL Equivalent | Example |
|---|---|---|
| `Between` | `BETWEEN ? AND ?` | `findByBirthDateBetween(start, end)` |
| `LessThan` | `< ?` | `findByAgeIsLessThan(age)` |
| `GreaterThan` | `> ?` | `findByIdGreaterThan(id)` |
| `IsNull` | `IS NULL` | `findByEmailIsNull()` |
| `IsNotNull` | `IS NOT NULL` | `findByEmailIsNotNull()` |
| `In` | `IN (?)` | `findByNameIn(List<String>)` |
| `NotIn` | `NOT IN (?)` | `findByNameNotIn(List<String>)` |
| `Like` | `LIKE ?` | `findByNameLike("%Anuj%")` |
| `StartingWith` | `LIKE ?%` | `findByNameStartingWith("A")` |
| `Containing` | `LIKE %?%` | `findByNameContaining("anuj")` |
| `IgnoreCase` | `LOWER(col) = LOWER(?)` | `findByNameContainingIgnoreCase(q)` |
| `True`/`False` | `= TRUE` / `= FALSE` | `findByActiveTrue()` |
| `Distinct` | `SELECT DISTINCT` | `findDistinctByName(name)` |

### 1.4 Sorting in Method Names

```java
// OrderBy + field + Asc/Desc
List<Patient> findByNameContainingOrderByIdDesc(String query);
// → SELECT ... WHERE name LIKE %?% ORDER BY id DESC
```

### 1.5 Examples with Code

```java
// Between: all patients born between two dates
List<Patient> findByBirthDateBetween(LocalDate startDate, LocalDate endDate);

// Containing + sort
List<Patient> findByNameContainingOrderByIdDesc(String query);

// Result type can be Optional<T>, T, or List<T>:
Optional<Patient> findByEmail(String email);     // 0 or 1 result
Patient findByName(String name);                 // expects exactly 1
List<Patient> findByBloodGroup(BloodGroupType bg); // 0+ results
```

---

## 2. @Query — Custom JPQL Queries

When derived query methods aren't flexible enough (e.g., GROUP BY, complex joins, subqueries), use `@Query` with JPQL.

### 2.1 JPQL vs SQL

| JPQL | SQL Equivalent |
|---|---|
| `FROM Patient` | `FROM patient` |
| `SELECT p FROM Patient p` | `SELECT * FROM patient` |
| `p.name` | `patient.name` (column name) |
| Operates on **entities/classes** | Operates on **tables/columns** |
| Database-independent | Database-specific |

> JPQL is object-oriented SQL — you write against your Java entity class names, not table names.

### 2.2 Basic JPQL Query

```java
@Query("SELECT p FROM Patient p WHERE p.bloodGroup = :bloodGroup")
List<Patient> findByBloodGroupCustom(@Param("bloodGroup") BloodGroupType bloodGroup);
```

**Named parameters (`:`)**:
- Use `:paramName` in the JPQL string
- Use `@Param("paramName")` to bind the method argument
- Prevents SQL injection (values are escaped, not concatenated)

**Positional parameters (`?1`, `?2`...)**:
```java
@Query("SELECT p FROM Patient p WHERE p.bloodGroup = ?1")
List<Patient> findByBloodGroupPos(BloodGroupType bloodGroup);
```
Named parameters are preferred — they are clearer as the query grows.

### 2.3 Complex JPQL — GROUP BY

```java
@Query("SELECT p.bloodGroup, COUNT(p) FROM Patient p GROUP BY p.bloodGroup")
List<Object[]> countEachBloodGroupType();
```

Returns `List<Object[]>` — each `Object[]` has two elements:
- `objects[0]` = BloodGroupType enum value
- `objects[1]` = Long count

### 2.4 JPQL Update/Delete Queries

For modifying queries, you must add `@Modifying` and `@Transactional`:

```java
@Modifying
@Transactional
@Query("UPDATE Patient p SET p.name = :name WHERE p.id = :id")
int updateNameWithId(@Param("name") String name, @Param("id") Long id);
```

- Returns `int` = number of rows affected
- `@Modifying` tells JPA this query changes data
- `@Transactional` is required for UPDATE/DELETE queries — Spring throws `InvalidDataAccessApiUsageException` without it

---

## 3. Native Queries

Native queries bypass JPQL entirely and use the raw SQL of your database engine.

```java
@Query(value = "SELECT * FROM patient", nativeQuery = true)
List<Patient> findAllPatients();
```

| Feature | JPQL | Native |
|---|---|---|
| Language | Object-oriented SQL | Raw DB SQL (PostgreSQL/MySQL specific) |
| DB independent | Yes | No |
| Projection support | Yes | Limited |
| SELECT * | No (use entity alias) | Yes |
| Table name | Entity class name (e.g., `Patient`) | Actual table name (e.g., `patient`) |

---

## 4. Projection — Selecting Specific Fields

Sometimes you don't want to load all fields of an entity — just a few columns (e.g., bloodGroup + count). This is **projection**.

### 4.1 The Problem with Object[]

```java
@Query("SELECT p.bloodGroup, COUNT(p) FROM Patient p GROUP BY p.bloodGroup")
List<Object[]> countEachBloodGroupType();  // hard to use, not type-safe
```

### 4.2 JPQL NEW — Projection DTO

Create a dedicated DTO:
```java
@AllArgsConstructor
@NoArgsConstructor
@Getter @Setter @ToString
public class BloodGroupCountDto {
    private BloodGroupType bloodGroupType;
    private Long count;
}
```

Use `NEW` in JPQL with the **fully qualified class name**:
```java
@Query("SELECT NEW com.example.dto.BloodGroupCountDto(p.bloodGroup, COUNT(p)) " +
       "FROM Patient p GROUP BY p.bloodGroup")
List<BloodGroupCountDto> countEachBloodGroupType();
```

**How it works:**
1. JPQL extracts `p.bloodGroup` and `COUNT(p)` from DB
2. For each row, Hibernate calls `new BloodGroupCountDto(bloodGroup, count)`
3. You get a strongly-typed list of DTOs

> **Important:** `@AllArgsConstructor` is mandatory because Hibernate uses the constructor to create objects. Argument order must match the JPQL `NEW` call.

### 4.3 Projection DOES NOT work with nativeQuery=true

Projection using `NEW` is a JPQL/Hibernate feature. When `nativeQuery = true`, Hibernate doesn't process the result — you must use `List<Object[]>` or define a `@SqlResultSetMapping`.

---

## 5. Pagination with Pageable

When a table has millions of rows, loading all at once is slow and expensive. **Pagination** splits results into pages.

### 5.1 Adding Pageable to Repository

```java
// Any method can accept Pageable as an extra parameter
Page<Patient> findAll(Pageable pageable);

// Or with filtering:
Page<Patient> findByNameContaining(String query, Pageable pageable);
```

### 5.2 Creating a Pageable Instance

```java
// PageRequest.of(pageNumber, pageSize)
Pageable pageable = PageRequest.of(0, 20);  // page 0, 20 items per page

// With sorting:
Pageable pageable = PageRequest.of(0, 20, Sort.by("name"));           // ASC
Pageable pageable = PageRequest.of(0, 20, Sort.by("name").descending()); // DESC
Pageable pageable = PageRequest.of(0, 20, Sort.by("name", "birthDate")); // multiple
```

**Page numbers are 0-indexed** — page 0 is the first page.

### 5.3 Page<T> — The Return Type

```java
Page<Patient> page = patientRepository.findAll(pageable);

// Access page data:
List<Patient> patients = page.getContent();    // the actual records
int totalPages = page.getTotalPages();         // total number of pages
long totalElements = page.getTotalElements();  // total count of all records
boolean hasNext = page.hasNext();              // is there a next page?
int pageNumber = page.getNumber();             // current page number
int pageSize = page.getSize();                 // items per page
```

**Why Page<T> is better than List<T>:**
When returning `Page<T>` as JSON from a REST API, the client gets all metadata (total, current page, hasNext, etc.) which enables front-end pagination controls.

### 5.4 SQL Generated by Pagination

Two SQL queries are always run:
```sql
-- Query 1: Get the data for the requested page
SELECT * FROM patient LIMIT 20 OFFSET 0

-- Query 2: Count total records (for Page metadata)
SELECT COUNT(1) FROM patient
```

---

## 6. Using Pagination in a REST API

```java
@GetMapping("/patients")
public ResponseEntity<Page<PatientDto>> getAllPatients(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "name") String sortBy) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
    Page<Patient> patients = patientRepository.findAll(pageable);
    // convert to DTOs...
    return ResponseEntity.ok(dtoPage);
}
```

Client can call:
```
GET /patients?page=0&size=10&sortBy=name
GET /patients?page=1&size=10
```

---

## 7. Summary — Query Types Comparison

| Method | Use Case | Example |
|---|---|---|
| Derived query methods | Simple field-based queries | `findByName(String name)` |
| `@Query` JPQL | Complex queries, GROUP BY, joins | `@Query("SELECT ... ")` |
| Native query | Raw SQL, DB-specific features | `@Query(value="...", nativeQuery=true)` |
| `@Modifying` + `@Query` | UPDATE/DELETE queries | `@Modifying @Query("UPDATE ...")` |
| Projection (`NEW`) | Select specific fields, aggregate results | `@Query("SELECT NEW Dto(p.x, COUNT(p))...")` |
| `Pageable` | Page through large result sets | `findAll(Pageable pageable)` |

---

## 8. Key Concepts Summary

| Concept | Description |
|---|---|
| Derived Query Methods | Auto-generated SQL from method names (findByName, findByBirthDateBetween) |
| `@Query` | Custom JPQL or native SQL on repository methods |
| JPQL | Jakarta Persistence Query Language — uses entity names, not table names |
| Named Parameter | `:paramName` in JPQL + `@Param("paramName")` on method arg |
| Positional Parameter | `?1`, `?2`... in JPQL — less readable than named params |
| `@Modifying` | Required for UPDATE/DELETE @Query methods |
| `nativeQuery = true` | Runs raw DB SQL; bypasses Hibernate translation |
| Projection (JPQL) | `SELECT NEW FullyQualifiedClass(field1, field2) FROM ...` |
| `Pageable` | Carries page number, page size, and sort info |
| `PageRequest.of(n, size)` | Creates a Pageable; 0-indexed pages |
| `Page<T>` | Contains data + metadata (totalPages, totalElements, hasNext) |
