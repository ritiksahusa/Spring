# Chapter 4 — Q&A: CRUD APIs with Database Setup

---

## Part A — Short Answer Questions

<a id="q1"></a>**1. What two Maven dependencies are needed to connect a Spring Boot app to a PostgreSQL database?** [↓ Answer](#a1)

<a id="a1"></a>`spring-boot-starter-data-jpa` (provides Spring Data JPA + Hibernate) and `postgresql` (the JDBC driver). [↑ Question](#q1)

---

<a id="q2"></a>**2. What does `spring.jpa.hibernate.ddl-auto=update` do?** [↓ Answer](#a2)

<a id="a2"></a>It tells Hibernate to automatically create tables that don't exist and add columns that are missing, without dropping existing data. It keeps the DB schema in sync with your Entity classes. [↑ Question](#q2)

---

<a id="q3"></a>**3. What is ORM? What ORM library does Spring Data JPA use by default?** [↓ Answer](#a3)

<a id="a3"></a>ORM = Object Relational Mapping. It maps Java objects (Entities) to relational database tables. Spring Data JPA uses **Hibernate** as the ORM provider behind the scenes. [↑ Question](#q3)

---

<a id="q4"></a>**4. What is the purpose of `@Entity` annotation?** [↓ Answer](#a4)

<a id="a4"></a>It tells JPA that this Java class corresponds to a database table. Hibernate will use this class to generate/validate the table schema. [↑ Question](#q4)

---

<a id="q5"></a>**5. What does `@GeneratedValue(strategy = GenerationType.IDENTITY)` do?** [↓ Answer](#a5)

<a id="a5"></a>It tells the database to auto-generate the primary key value. With `IDENTITY`, the DB increments it sequentially (1, 2, 3...). The developer never needs to set the `id` manually. [↑ Question](#q5)

---

<a id="q6"></a>**6. What is a Repository in Spring Boot? What interface do you extend?** [↓ Answer](#a6)

<a id="a6"></a>A Repository is the Data Access Layer — it handles all SQL operations. You extend `JpaRepository<EntityType, IdType>` to get free CRUD methods like `findAll()`, `save()`, `deleteById()`, etc. [↑ Question](#q6)

---

<a id="q7"></a>**7. What is the difference between an Entity and a DTO?** [↓ Answer](#a7)

<a id="a7"></a>

| | Entity | DTO |
|---|---|---|
| Purpose | Represents a DB row | Data sent to/from the client |
| Layer | Service ↔ Repository | Controller ↔ Client |
| Contains | All DB fields (including sensitive ones) | Only the fields the client needs |

Never return an Entity directly from a controller — always convert to DTO first. [↑ Question](#q7)

---

<a id="q8"></a>**8. Why create a separate `AddStudentRequestDto` instead of reusing `StudentDto` for POST requests?** [↓ Answer](#a8)

<a id="a8"></a>Because `StudentDto` contains an `id` field — the client should not provide an `id` when creating a student (the DB auto-generates it). A separate request DTO with only `name` and `email` prevents accidental ID injection. [↑ Question](#q8)

---

<a id="q9"></a>**9. What is `ResponseEntity<T>` and why is it preferred over returning the object directly?** [↓ Answer](#a9)

<a id="a9"></a>`ResponseEntity<T>` wraps the response body AND the HTTP status code. Using it lets you explicitly set status codes like `201 CREATED` for POST or `204 NO_CONTENT` for DELETE, which makes REST APIs semantically correct. [↑ Question](#q9)

---

<a id="q10"></a>**10. What does `@PathVariable` do? Give an example.** [↓ Answer](#a10)

<a id="a10"></a>It binds a URL path segment to a method parameter.

```java
@GetMapping("/students/{id}")
public StudentDto getById(@PathVariable Long id) { ... }
// URL: GET /students/42  →  id = 42
```

[↑ Question](#q10)

---

<a id="q11"></a>**11. What does `@RequestBody` do?** [↓ Answer](#a11)

<a id="a11"></a>It tells Spring to deserialise the HTTP request body (JSON) into a Java object using Jackson.

```java
@PostMapping
public ResponseEntity<StudentDto> create(@RequestBody AddStudentRequestDto dto) { ... }
```

[↑ Question](#q11)

---

<a id="q12"></a>**12. What is the difference between PUT and PATCH HTTP methods?** [↓ Answer](#a12)

<a id="a12"></a>

- **PUT**: Replaces the **entire** resource. All fields must be provided.
- **PATCH**: Partially updates a resource. Only the changed fields are sent.

[↑ Question](#q12)

---

<a id="q13"></a>**13. What does `@RequiredArgsConstructor` do and why is it used for dependency injection?** [↓ Answer](#a13)

<a id="a13"></a>It is a Lombok annotation that generates a constructor for all `final` fields. Spring Boot sees this constructor and uses it for constructor injection. It eliminates the need to manually write constructor code. Fields must be declared `final` for this to work. [↑ Question](#q13)

---

<a id="q14"></a>**14. What is ModelMapper and what problem does it solve?** [↓ Answer](#a14)

<a id="a14"></a>ModelMapper is a library that automatically maps fields between two objects (e.g., `Student` entity → `StudentDto`) by matching field names. It eliminates manual boilerplate like `new StudentDto(s.getId(), s.getName(), s.getEmail())`. [↑ Question](#q14)

---

<a id="q15"></a>**15. What is the context path and how do you configure it?** [↓ Answer](#a15)

<a id="a15"></a>The context path is a global URL prefix applied to all endpoints. Configured in `application.properties`:

```properties
server.servlet.context-path=/api
```

After this, all endpoints are accessed as `/api/students` instead of `/students`. [↑ Question](#q15)

---

## Part B — Multiple Choice Questions

<a id="q16"></a>**16. Which annotation is used to mark the primary key field in a JPA Entity?** [↓ Answer](#a16)

a) `@PrimaryKey`
b) `@Id`
c) `@Key`
d) `@GeneratedValue`

<a id="a16"></a>**Answer: b) `@Id`** [↑ Question](#q16)

---

<a id="q17"></a>**17. Which `ddl-auto` value would DESTROY all data every time the application restarts?** [↓ Answer](#a17)

a) `update`
b) `validate`
c) `create-drop`
d) `none`

<a id="a17"></a>**Answer: c) `create-drop`** [↑ Question](#q17)

---

<a id="q18"></a>**18. What HTTP status code should a POST endpoint return when a resource is successfully created?** [↓ Answer](#a18)

a) 200 OK
b) 204 No Content
c) 201 Created
d) 400 Bad Request

<a id="a18"></a>**Answer: c) 201 Created** [↑ Question](#q18)

---

<a id="q19"></a>**19. What is the correct way to get a free `save()` method for your Entity without writing any SQL?** [↓ Answer](#a19)

a) Annotate the class with `@Service`
b) Implement `CrudOperations` interface
c) Extend `JpaRepository<Entity, IdType>`
d) Use `@Autowired EntityManager`

<a id="a19"></a>**Answer: c) Extend `JpaRepository<Entity, IdType>`** [↑ Question](#q19)

---

<a id="q20"></a>**20. Which layer should NEVER directly access the Repository?** [↓ Answer](#a20)

a) Service Layer
b) Controller Layer
c) Persistence Layer
d) Configuration Layer

<a id="a20"></a>**Answer: b) Controller Layer** — the controller should only talk to the Service. [↑ Question](#q20)

---

<a id="q21"></a>**21. What does `modelMapper.map(source, DestClass.class)` return?** [↓ Answer](#a21)

a) The original source object
b) A new object of `DestClass` with fields copied from `source`
c) A JSON string
d) A proxy object

<a id="a21"></a>**Answer: b) A new object of `DestClass` with fields copied from `source`** [↑ Question](#q21)

---

<a id="q22"></a>**22. Which of these is NOT a method provided by `JpaRepository`?** [↓ Answer](#a22)

a) `findAll()`
b) `save()`
c) `executeQuery(String sql)`
d) `deleteById()`

<a id="a22"></a>**Answer: c) `executeQuery(String sql)` — this is not a JpaRepository method** [↑ Question](#q22)

---

<a id="q23"></a>**23. What does `ResponseEntity.noContent().build()` return?** [↓ Answer](#a23)

a) An empty JSON object `{}`
b) A response with HTTP status 204 and no body
c) A response with HTTP status 200 and null body
d) A response with HTTP status 400

<a id="a23"></a>**Answer: b) A response with HTTP status 204 and no body** [↑ Question](#q23)

---

<a id="q24"></a>**24. What happens when you call `studentRepository.findById(id)` and the ID doesn't exist?** [↓ Answer](#a24)

a) It throws `NullPointerException`
b) It returns `null`
c) It returns an empty `Optional<Student>`
d) It throws `EntityNotFoundException` automatically

<a id="a24"></a>**Answer: c) It returns an empty `Optional<Student>`** [↑ Question](#q24)

---

<a id="q25"></a>**25. In DBeaver, what is the default PostgreSQL port?** [↓ Answer](#a25)

a) 3306
b) 8080
c) 5432
d) 27017

<a id="a25"></a>**Answer: c) 5432** [↑ Question](#q25)

---

## Part C — Scenario / Code-Reading Questions

<a id="q26"></a>**26. You have this code:**

```java
Student student = studentRepository.findById(id)
    .orElseThrow(() -> new IllegalArgumentException("Not found: " + id));
```

**What happens if `id = 999` and there is no student with that ID?** [↓ Answer](#a26)

<a id="a26"></a>`findById(999)` returns an empty `Optional`. The `.orElseThrow()` call triggers, creating and throwing an `IllegalArgumentException` with the message `"Not found: 999"`. Spring will return a 500 status unless you add a global exception handler. [↑ Question](#q26)

---

<a id="q27"></a>**27. Analyse this service method:**

```java
public StudentDto createNewStudent(AddStudentRequestDto dto) {
    Student newStudent = modelMapper.map(dto, Student.class);
    Student saved = studentRepository.save(newStudent);
    return modelMapper.map(saved, StudentDto.class);
}
```

**Trace what happens step by step.** [↓ Answer](#a27)

<a id="a27"></a>

1. `modelMapper.map(dto, Student.class)` → creates a `Student` with name and email from the dto (no `id` yet)
2. `studentRepository.save(newStudent)` → Hibernate runs `INSERT INTO student (name, email) VALUES (?, ?)`, DB auto-assigns `id`
3. The returned `saved` Student now has the DB-assigned `id`
4. `modelMapper.map(saved, StudentDto.class)` → creates a `StudentDto` with all 3 fields including the new `id`
5. Returns the `StudentDto` which is serialised to JSON by Jackson

[↑ Question](#q27)

---

<a id="q28"></a>**28. What is wrong with this controller code?** [↓ Answer](#a28)

```java
@RestController
@RequestMapping("/students")
public class StudentController {
    @Autowired
    private StudentRepository studentRepository;
    
    @GetMapping
    public List<Student> getAll() {
        return studentRepository.findAll();
    }
}
```

<a id="a28"></a>Three violations of best practices:

1. **Controller accesses Repository directly** — should go through a Service layer
2. **Returns `Student` entity directly** — should return a DTO to avoid exposing internal DB structure
3. **Field injection with `@Autowired`** — discouraged; use constructor injection via `@RequiredArgsConstructor` instead

[↑ Question](#q28)

---

<a id="q29"></a>**29. You want to add a PATCH endpoint that only updates the `phoneNumber` field of a Student. The current `updatePartialStudent` implementation uses a `switch` statement. What changes do you need?** [↓ Answer](#a29)

<a id="a29"></a>

1. Add `private String phoneNumber` to the `Student` entity (+ getter/setter)
2. Add `private String phoneNumber` to `StudentDto`
3. In `updatePartialStudent` switch:
   ```java
   case "phoneNumber" -> student.setPhoneNumber((String) value);
   ```
4. Optionally add `phoneNumber` to `AddStudentRequestDto` if needed for full creation.

[↑ Question](#q29)

---

<a id="q30"></a>**30. Explain why ModelMapper needs `@NoArgsConstructor` on the destination class.** [↓ Answer](#a30)

<a id="a30"></a>ModelMapper creates a new instance of the destination class using reflection. To instantiate an object via reflection, Java needs a no-argument constructor. If `@NoArgsConstructor` (or a default constructor) is missing, ModelMapper throws `"Failed to instantiate..."` error. `@AllArgsConstructor` alone is not enough. [↑ Question](#q30)

---

## Part D — Fill in the Blanks

<a id="q31"></a>**31.** The `@Entity` annotation comes from the package `________________`. [↓ Answer](#a31)

<a id="a31"></a>**Answer:** `jakarta.persistence` [↑ Question](#q31)

---

<a id="q32"></a>**32.** `JpaRepository` takes two type parameters: `________________` and `________________`. [↓ Answer](#a32)

<a id="a32"></a>**Answer:** The Entity class type; The type of the Entity's `@Id` field (e.g., `Long`) [↑ Question](#q32)

---

<a id="q33"></a>**33.** To return a `201 Created` status from a controller using `ResponseEntity`, you write: `ResponseEntity.status(HttpStatus.________________).body(data)`. [↓ Answer](#a33)

<a id="a33"></a>**Answer:** `CREATED` [↑ Question](#q33)

---

<a id="q34"></a>**34.** To make Hibernate print all generated SQL queries in the console, set `spring.jpa.________________=true` in `application.properties`. [↓ Answer](#a34)

<a id="a34"></a>**Answer:** `show-sql` [↑ Question](#q34)

---

<a id="q35"></a>**35.** `@RequiredArgsConstructor` only generates a constructor argument for fields annotated with `________________` keyword. [↓ Answer](#a35)

<a id="a35"></a>**Answer:** `final` [↑ Question](#q35)

---

<a id="q36"></a>**36.** A `PUT` endpoint replaces the ________________ resource, while `PATCH` only updates ________________ fields. [↓ Answer](#a36)

<a id="a36"></a>**Answer:** entire; specific/partial [↑ Question](#q36)

---

<a id="q37"></a>**37.** In ModelMapper, the method to map a source object to an existing destination object is: `modelMapper.map(source, ________________)`. [↓ Answer](#a37)

<a id="a37"></a>**Answer:** `destinationObject` (passing the object instance, not the class) [↑ Question](#q37)

---

<a id="q38"></a>**38.** The `spring.jpa.hibernate.ddl-auto` value that drops and recreates the schema on every startup is `________________`. [↓ Answer](#a38)

<a id="a38"></a>**Answer:** `create-drop` [↑ Question](#q38)

---

## Part E — True / False

<a id="q39"></a>**39.** You must manually write the `findAll()` method in `StudentRepository`. [↓ Answer](#a39) <a id="a39"></a>**False** — it is inherited from `JpaRepository`. [↑ Question](#q39)

<a id="q40"></a>**40.** The `@Repository` annotation is mandatory on a `JpaRepository` interface. [↓ Answer](#a40) <a id="a40"></a>**False** — it is optional; Spring auto-detects it. [↑ Question](#q40)

<a id="q41"></a>**41.** `ResponseEntity.ok(data)` returns HTTP status 200 with the given data as the body. [↓ Answer](#a41) <a id="a41"></a>**True** [↑ Question](#q41)

<a id="q42"></a>**42.** A `Student` entity should be directly returned from the controller to keep code simple. [↓ Answer](#a42) <a id="a42"></a>**False** — always use DTOs in the controller layer. [↑ Question](#q42)

<a id="q43"></a>**43.** With `ddl-auto=update`, if you delete a field from your Entity class, the corresponding column is also dropped from the DB. [↓ Answer](#a43) <a id="a43"></a>**False** — `update` only adds missing columns/tables; it never drops existing ones. [↑ Question](#q43)

<a id="q44"></a>**44.** ModelMapper matches fields by their data type. [↓ Answer](#a44) <a id="a44"></a>**False** — it matches by **field name**. [↑ Question](#q44)

<a id="q45"></a>**45.** When `@RequiredArgsConstructor` is used, Spring Boot performs constructor injection automatically for `final` fields. [↓ Answer](#a45) <a id="a45"></a>**True** [↑ Question](#q45)

<a id="q46"></a>**46.** The PostgreSQL default port is 3306. [↓ Answer](#a46) <a id="a46"></a>**False** — it is 5432 (3306 is MySQL). [↑ Question](#q46)

<a id="q47"></a>**47.** `PATCH` is always preferable to `PUT` because it sends less data. [↓ Answer](#a47) <a id="a47"></a>**False** — both are valid for different use cases; `PUT` is used for full replacement, `PATCH` for partial updates. [↑ Question](#q47)

<a id="q48"></a>**48.** `@Configuration` is a specialised stereotype annotation that tells Spring the class contains `@Bean` method definitions. [↓ Answer](#a48) <a id="a48"></a>**True** [↑ Question](#q48)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** Why is `PUT` considered idempotent but `POST` is not, and how does this affect retry logic for flaky mobile clients? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** Idempotent means calling the same request N times produces the same end-state as calling it once. `PUT /students/5` with the same body always leaves student 5 with those exact values. `POST /students` creates a NEW resource every time, so retrying after a network timeout (client unsure if the first attempt succeeded) can create duplicates. This is why safe retries typically use PUT, or POST is paired with a client-generated **idempotency key** stored server-side to detect and reject duplicate submissions. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** Your auto-incrementing `Long id` is exposed directly in the public API (`/api/students/42`). What's the security concern, and how do you mitigate it? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** This is an Insecure Direct Object Reference (IDOR) / enumeration risk — attackers can iterate IDs to scrape all records or infer business metrics like signup rate. Mitigate by using `GenerationType.UUID` for externally-exposed identifiers, always enforcing authorization checks (does THIS user own record 42?) rather than relying on ID obscurity, and optionally keeping an internal sequential PK for DB performance plus a separate public-facing UUID/slug for API exposure. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** HikariCP's default pool size is 10. How do you determine the "correct" pool size for production, and why is "just make it bigger" wrong? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** The well-known formula ties pool size to CPU cores and disk parallelism (`connections ≈ (core_count × 2) + effective_spindle_count`), NOT to concurrent HTTP request count. Oversized pools cause MORE context switching and lock contention on the DB server, actually reducing throughput, while consuming more DB-side memory per idle connection. The right approach: start small, load test, and watch DB-side saturation and latency percentiles instead of guessing a large number. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** How would you detect and prevent a "lost update" when two users PATCH the same Student concurrently, without using DB row locks? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** Use optimistic concurrency: include a `version` (or `updatedAt`/`ETag`) in the GET response; require the client to send it back on update; if the current DB version doesn't match, return `409 Conflict` (or `412 Precondition Failed` for `If-Match`) instead of silently overwriting. This maps directly to JPA's `@Version` field under the hood. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** Why is `204 No Content` preferred over `200 OK` for a successful `DELETE`, and when might `200 OK` still be justified? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** `204 No Content` explicitly signals "operation succeeded, no body to expect," which is the REST-conventional default for pure deletes. `200 OK` with a body is justified when clients need the deleted object's data back (e.g., for optimistic UI updates showing "you deleted X") — but this must be a deliberate API design choice, not a default. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** Why does `ddl-auto=update` behave unpredictably when THREE replicas of the same Spring Boot app start simultaneously against the same database? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** Each instance independently runs schema-diffing/DDL logic on startup. Concurrent instances can race on `ALTER TABLE`/`CREATE TABLE` (duplicate index creation attempts, one instance querying a half-migrated table while another alters it), causing intermittent startup failures or lock contention. This is why production systems run a SINGLE controlled migration step (Flyway/Liquibase, e.g., as a deploy-time init step) and set `ddl-auto=validate`/`none` on the actual running application instances. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** A client sends an unexpected extra JSON field (e.g., `"isAdmin": true`) that isn't part of your DTO. What does Jackson do by default, and why is disciplined DTO design more important than strict unknown-property rejection? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** Spring Boot's Jackson auto-config silently ignores unknown properties by default (lenient client compatibility). The real mass-assignment risk isn't about strictness — it's DTO scope: if a "full" DTO happens to contain privileged fields (like `role`) and is reused across both admin and public creation endpoints, a client could set fields they shouldn't. The fix is disciplined, narrowly-scoped request DTOs per use case, never reusing an overly broad DTO for both trusted and untrusted callers. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** You're asked to add "soft delete" to Student. What JPA-level implications does this have for existing `findAll()`/unique constraints, and how do you implement it cleanly? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** Every query must now implicitly filter `deleted = false`, or soft-deleted rows reappear and can violate uniqueness (e.g., re-registering an email "held" by a deleted row). Hibernate's `@SQLRestriction` (formerly `@Where`) auto-appends a `deleted = false` filter to ALL queries against that entity, and `@SQLDelete` converts `deleteById()` into an `UPDATE ... SET deleted = true` instead of a physical delete. Unique constraints need to become partial/conditional indexes (e.g., Postgres `WHERE deleted = false`), and any raw/native query bypassing entity-level filtering needs manual handling. [↑ Question](#qf8)

````
