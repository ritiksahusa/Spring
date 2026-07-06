# Chapter 4 — CRUD APIs with Database Setup
**Timestamps:** `1:29:29 → 2:48:28`
**Topics:** PostgreSQL setup, Spring Data JPA, Entity/Repository, 3-layer architecture, full CRUD REST APIs, ResponseEntity, ModelMapper, Postman

---

## 1. Goal of This Chapter

Move from returning hard-coded/dummy data to a **real persistent database** using:

- **PostgreSQL** as the relational database
- **Spring Data JPA** (+ Hibernate as the ORM provider) as the persistence layer
- Full **CRUD API** endpoints (GET, POST, PUT, PATCH, DELETE)
- Proper **3-layer architecture**: Controller → Service → Repository

---

## 2. PostgreSQL — Installation and Setup

### 2.1 Install PostgreSQL

| OS | Method |
|---|---|
| macOS | `brew install postgresql@17` |
| macOS (GUI) | `postgres.app` — easiest click-to-install |
| Windows | Download installer from `postgresql.org` → installs pgAdmin + Stack Builder |
| Any OS | Docker image (`docker run -e POSTGRES_PASSWORD=... -p 5432:5432 postgres`) |

**Start the service (macOS Homebrew):**
```bash
brew services start postgresql
brew services list          # verify status shows "started"
brew services restart postgresql
```

Default port: **5432**, default database: `postgres`, default user: system username (no password needed for local dev).

### 2.2 GUI Tool — DBeaver

- Free, cross-platform database GUI supporting ALL database types (not just Postgres)
- Alternative: **pgAdmin** (Windows installer bundles it)
- Connect in DBeaver:
  - `New Database Connection` → PostgreSQL
  - Host: `localhost`, Port: `5432`, Database: `postgres`
  - Username/Password: leave empty if using Homebrew install (uses OS user)
  - Click `Test Connection` → Finish

### 2.3 Create a dedicated database

Inside DBeaver, right-click `Databases` → Create Database → name it `students_db`.

---

## 3. Adding Dependencies to pom.xml

Two new dependencies are needed:

```xml
<!-- Spring Data JPA (includes Hibernate ORM) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- PostgreSQL JDBC Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

After adding, click **Sync Maven Changes** (Maven reload button) so IntelliJ downloads the JARs.

---

## 4. Configuring application.properties

```properties
# DataSource — tells Spring Boot where the DB lives
spring.datasource.url=jdbc:postgresql://localhost:5432/students_db
spring.datasource.username=          # leave empty → uses OS system user
spring.datasource.password=          # leave empty for local dev

# JPA / Hibernate settings
spring.jpa.hibernate.ddl-auto=update
# update: auto-creates/alters tables to match your Entities. 
# create-drop: recreates table on every restart (destroys data — avoid!)

spring.jpa.show-sql=true             # prints raw SQL queries in console
spring.jpa.properties.hibernate.format_sql=true   # pretty-prints those queries
```

### DDL Auto Options

| Value | Behaviour |
|---|---|
| `update` | Creates missing tables/columns; keeps existing data ✅ |
| `create` | Drops and recreates schema on startup (data lost) |
| `create-drop` | Like `create`, also drops on shutdown |
| `validate` | Only validates schema matches entity — no DDL changes |
| `none` | Completely disabled |

> Use `update` for local dev. Use `validate` or `none` in production (manage schema with Flyway/Liquibase).

---

## 5. Creating the Entity Class

An **Entity** is a Java class that maps to a database table (ORM — Object Relational Mapping).

```java
package com.example.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Entity                            // maps this class to a DB table named "student"
@Getter
@Setter
public class Student {

    @Id                            // marks this field as the Primary Key
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    // IDENTITY → DB auto-increments: 1, 2, 3...
    // UUID    → generates a UUID string
    // SEQUENCE → uses a DB sequence generator
    private Long id;

    private String name;
    private String email;
}
```

**What happens when the app starts:**
- Hibernate reads the `@Entity` class
- With `ddl-auto=update`, it runs:
  ```sql
  CREATE TABLE student (
      id BIGSERIAL NOT NULL,
      email VARCHAR(255),
      name VARCHAR(255),
      PRIMARY KEY (id)
  );
  ```
- The table appears in DBeaver under `students_db → public → Tables → student`

### Key JPA Annotations

| Annotation | Purpose |
|---|---|
| `@Entity` | Marks a class as a JPA entity (maps to a table) |
| `@Id` | Marks the primary key field |
| `@GeneratedValue` | Configures auto-generation of the PK |
| `@Column(name="...")` | Customise the column name (optional) |
| `@Table(name="...")` | Customise the table name (optional) |

---

## 6. Repository Layer — JpaRepository

The repository is the **Data Access Layer** (DAL). It handles all database CRUD operations.

```java
package com.example.repository;

import com.example.entity.Student;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository   // optional (interface is auto-detected), but good practice for clarity
public interface StudentRepository extends JpaRepository<Student, Long> {
    // JpaRepository<EntityType, IDType>
    // Spring Boot auto-generates the implementation at runtime (SimpleJpaRepository)
}
```

### Methods you get for free from JpaRepository

| Method | SQL equivalent |
|---|---|
| `findAll()` | `SELECT * FROM student` |
| `findById(id)` | `SELECT * FROM student WHERE id = ?` |
| `save(entity)` | `INSERT` (new) or `UPDATE` (existing) |
| `deleteById(id)` | `DELETE FROM student WHERE id = ?` |
| `existsById(id)` | `SELECT COUNT(*) … WHERE id = ?` |
| `count()` | `SELECT COUNT(*)` |

> `JpaRepository` extends `CrudRepository` → `PagingAndSortingRepository`, so all those methods are also available.

---

## 7. Three-Layer Architecture (Enforced)

```
Client Request
      |
      v
[ Controller Layer ]   ← Presentation; only sees DTOs, never Entities
      |
      v
[ Service Layer ]      ← Business logic; converts Entity ↔ DTO
      |
      v
[ Repository Layer ]   ← Data access; interacts with the DB
      |
      v
[ PostgreSQL Database ]
```

### Why enforce this?
- **Separation of Concerns**: each layer has one job
- **Security**: Entity (with all DB fields) is never directly exposed to client
- **Reusability**: service logic can be called from multiple controllers
- **Testability**: each layer can be unit-tested independently

---

## 8. StudentDTO — Data Transfer Object

The **DTO** is what flows between the controller and the outside world. The **Entity** is what flows between the service and the database.

```java
package com.example.dto;

import lombok.*;

@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
public class StudentDto {
    private Long id;
    private String name;
    private String email;
}
```

**Separate Request DTO** (for POST/PUT — no id from client):
```java
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
public class AddStudentRequestDto {
    private String name;
    private String email;
    // No 'id' field — server generates it
}
```

---

## 9. ModelMapper — Automatic DTO ↔ Entity Conversion

Without ModelMapper you write boilerplate like:
```java
new StudentDto(student.getId(), student.getName(), student.getEmail()); // fragile
```

### 9.1 Add the dependency
```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.1.1</version>
</dependency>
```

### 9.2 Create a @Configuration @Bean
```java
package com.example.config;

import org.modelmapper.ModelMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MapperConfig {

    @Bean
    public ModelMapper modelMapper() {
        return new ModelMapper();
    }
}
```

> `@Configuration` is a Spring stereotype that marks a class as containing `@Bean` definitions. It is scanned by Spring just like `@Component`.

### 9.3 Using ModelMapper
```java
// Entity → DTO
StudentDto dto = modelMapper.map(student, StudentDto.class);

// DTO → Entity  (source, destination object)
Student student = modelMapper.map(addStudentRequestDto, Student.class);

// DTO fields INTO an existing Entity object (for partial update)
modelMapper.map(updateDto, existingStudent);
```

**How it works:** ModelMapper matches fields by **name**. If `Student.name` and `StudentDto.name` both exist, it maps them automatically. Ensure `@NoArgsConstructor` is present on the target class.

---

## 10. Service Layer — Interface + Implementation

### 10.1 Interface
```java
package com.example.service;

import com.example.dto.StudentDto;
import com.example.dto.AddStudentRequestDto;
import java.util.List;

public interface StudentService {
    List<StudentDto> getAllStudents();
    StudentDto getStudentById(Long id);
    StudentDto createNewStudent(AddStudentRequestDto dto);
    void deleteStudentById(Long id);
    StudentDto updateStudent(Long id, AddStudentRequestDto dto);
    StudentDto updatePartialStudent(Long id, Map<String, Object> updates);
}
```

### 10.2 Implementation
```java
package com.example.service.impl;

import com.example.dto.*;
import com.example.entity.Student;
import com.example.repository.StudentRepository;
import com.example.service.StudentService;
import lombok.RequiredArgsConstructor;
import org.modelmapper.ModelMapper;
import org.springframework.stereotype.Service;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor   // generates constructor for all 'final' fields → auto DI
public class StudentServiceImpl implements StudentService {

    private final StudentRepository studentRepository;
    private final ModelMapper modelMapper;

    @Override
    public List<StudentDto> getAllStudents() {
        List<Student> students = studentRepository.findAll();
        return students.stream()
                .map(s -> modelMapper.map(s, StudentDto.class))
                .collect(Collectors.toList());
    }

    @Override
    public StudentDto getStudentById(Long id) {
        Student student = studentRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Student not found with id: " + id));
        return modelMapper.map(student, StudentDto.class);
    }

    @Override
    public StudentDto createNewStudent(AddStudentRequestDto dto) {
        Student newStudent = modelMapper.map(dto, Student.class);
        Student saved = studentRepository.save(newStudent);
        return modelMapper.map(saved, StudentDto.class);
    }

    @Override
    public void deleteStudentById(Long id) {
        if (!studentRepository.existsById(id)) {
            throw new IllegalArgumentException("Student does not exist with id: " + id);
        }
        studentRepository.deleteById(id);
    }

    @Override
    public StudentDto updateStudent(Long id, AddStudentRequestDto dto) {
        Student student = studentRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Student not found with id: " + id));
        modelMapper.map(dto, student);           // map dto fields INTO existing entity
        Student saved = studentRepository.save(student);
        return modelMapper.map(saved, StudentDto.class);
    }

    @Override
    public StudentDto updatePartialStudent(Long id, Map<String, Object> updates) {
        Student student = studentRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Student not found with id: " + id));
        updates.forEach((field, value) -> {
            switch (field) {
                case "name"  -> student.setName((String) value);
                case "email" -> student.setEmail((String) value);
                default      -> throw new IllegalArgumentException("Field not supported: " + field);
            }
        });
        Student savedStudent = studentRepository.save(student);
        return modelMapper.map(savedStudent, StudentDto.class);
    }
}
```

### @RequiredArgsConstructor
Lombok generates a constructor with all `final` fields as arguments. Spring uses this constructor for dependency injection.

```java
// Without Lombok — manual:
public StudentServiceImpl(StudentRepository repo, ModelMapper mapper) {
    this.studentRepository = repo;
    this.modelMapper = mapper;
}

// With Lombok @RequiredArgsConstructor — auto-generated ↑
```

---

## 11. Controller Layer — Full CRUD

### 11.1 @RequestMapping at class level
```java
@RestController
@RequestMapping("/api/students")   // base path for all endpoints in this controller
@RequiredArgsConstructor
public class StudentController {

    private final StudentService studentService;
    ...
}
```

### 11.2 Path Variables
```java
// URL: GET /api/students/42
@GetMapping("/{id}")
public ResponseEntity<StudentDto> getStudentById(@PathVariable Long id) {
    return ResponseEntity.ok(studentService.getStudentById(id));
}

// Multiple path variables
@GetMapping("/{id}/courses/{courseId}")
public String getStudentCourse(@PathVariable Long id, @PathVariable Long courseId) { ... }

// If variable name differs from parameter name:
@GetMapping("/{studentId}")
public ResponseEntity<StudentDto> get(@PathVariable("studentId") Long id) { ... }
```

### 11.3 ResponseEntity

`ResponseEntity<T>` wraps the response body AND the HTTP status code.

```java
// Full form:
return ResponseEntity.status(HttpStatus.OK).body(data);

// Shorthand for 200 OK:
return ResponseEntity.ok(data);

// 201 Created:
return ResponseEntity.status(HttpStatus.CREATED).body(data);

// 204 No Content (nothing to return):
return ResponseEntity.noContent().build();
```

### HTTP Status Codes Reference

| Code | Constant | When to use |
|---|---|---|
| 200 | `HttpStatus.OK` | GET success |
| 201 | `HttpStatus.CREATED` | POST — resource created |
| 204 | `HttpStatus.NO_CONTENT` | DELETE — nothing to return |
| 400 | `HttpStatus.BAD_REQUEST` | Invalid input |
| 404 | `HttpStatus.NOT_FOUND` | Resource not found |
| 500 | `HttpStatus.INTERNAL_SERVER_ERROR` | Unhandled server error |

### 11.4 Complete Controller Code
```java
@RestController
@RequestMapping("/api/students")
@RequiredArgsConstructor
public class StudentController {

    private final StudentService studentService;

    // GET all students
    @GetMapping
    public ResponseEntity<List<StudentDto>> getAllStudents() {
        return ResponseEntity.ok(studentService.getAllStudents());
    }

    // GET by ID
    @GetMapping("/{id}")
    public ResponseEntity<StudentDto> getStudentById(@PathVariable Long id) {
        return ResponseEntity.ok(studentService.getStudentById(id));
    }

    // POST — create
    @PostMapping
    public ResponseEntity<StudentDto> createStudent(@RequestBody AddStudentRequestDto dto) {
        return ResponseEntity.status(HttpStatus.CREATED)
                             .body(studentService.createNewStudent(dto));
    }

    // DELETE
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteStudent(@PathVariable Long id) {
        studentService.deleteStudentById(id);
        return ResponseEntity.noContent().build();
    }

    // PUT — full update
    @PutMapping("/{id}")
    public ResponseEntity<StudentDto> updateStudent(
            @PathVariable Long id,
            @RequestBody AddStudentRequestDto dto) {
        return ResponseEntity.ok(studentService.updateStudent(id, dto));
    }

    // PATCH — partial update
    @PatchMapping("/{id}")
    public ResponseEntity<StudentDto> patchStudent(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(studentService.updatePartialStudent(id, updates));
    }
}
```

---

## 12. PUT vs PATCH

| Feature | PUT | PATCH |
|---|---|---|
| Intent | Replace the **entire** resource | Update **part** of the resource |
| Body | Full object (all fields) | Only the fields that change |
| Example | Change name AND email together | Change only email |
| Implementation | `modelMapper.map(dto, existingEntity)` | `switch-case` on field names from a `Map<String,Object>` |

---

## 13. Context Path Configuration

```properties
# All API endpoints will be prefixed with /api
server.servlet.context-path=/api
```

After this, URLs become: `http://localhost:8080/api/students` instead of `http://localhost:8080/students`.

Useful in microservices to identify the service from the URL.

---

## 14. Postman — API Testing Tool

- Industry-standard API client (Indian company, globally used)
- Download free from `postman.com`
- Works as desktop app or browser
- Supports: GET, POST, PUT, PATCH, DELETE, and more
- Set request body as **Raw → JSON**

**Example POST request in Postman:**
```
Method: POST
URL: http://localhost:8080/api/students
Body (raw JSON):
{
  "name": "Anuj Kumar Sharma",
  "email": "anuj@gmail.com"
}
```

---

## 15. Complete Data Flow (End-to-End)

```
1. Client sends POST /api/students  {"name":"Anuj","email":"anuj@gmail.com"}
2. DispatcherServlet routes to StudentController.createStudent()
3. Jackson deserialises JSON body → AddStudentRequestDto
4. Controller calls StudentService.createNewStudent(dto)
5. Service: ModelMapper maps dto → Student entity (no id yet)
6. Service: studentRepository.save(student) 
7. Hibernate generates: INSERT INTO student (email, name) VALUES (?, ?)
8. PostgreSQL auto-increments id → returns saved row
9. Service: ModelMapper maps saved entity → StudentDto (now has id)
10. Controller wraps in ResponseEntity(201 CREATED, body=dto)
11. Jackson serialises StudentDto → JSON
12. Client receives: {"id":1,"name":"Anuj","email":"anuj@gmail.com"} [201]
```

---

## 16. Key Concepts Summary

| Concept | Description |
|---|---|
| `@Entity` | Maps a Java class to a DB table |
| `@Id` + `@GeneratedValue` | Auto-generated primary key |
| `JpaRepository<T, ID>` | Provides CRUD methods without writing SQL |
| ORM | Object Relational Mapping — Hibernate does this |
| `ddl-auto=update` | Hibernate auto-creates/updates tables |
| `@Repository` | Marks persistence layer (optional for interfaces) |
| `@Service` | Marks service layer bean |
| `@Configuration` + `@Bean` | Manual bean registration (e.g., ModelMapper) |
| `@RequiredArgsConstructor` | Lombok: generates constructor for `final` fields |
| `ModelMapper.map(src, DestClass)` | Converts between objects by field name matching |
| `ResponseEntity<T>` | Wraps response body + HTTP status code |
| `@PathVariable` | Extracts variable from URL path `/{id}` |
| `@RequestBody` | Deserialises request JSON body into a Java object |
| PUT vs PATCH | PUT=full replace; PATCH=partial update |
| Context path | Global URL prefix for all endpoints |
| Postman | API testing tool (GUI HTTP client) |
