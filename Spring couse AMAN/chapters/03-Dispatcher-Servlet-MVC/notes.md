# Chapter 3: Dispatcher Servlet and MVC Flow

> **Video time:** `59:13` → `1:29:27`

---

## 1. What is an API?

**API** = Application Programming Interface — the **interface between two communicating systems** over the internet.

**Without internet (old days):** To get goods/services, you visited a physical location.

**With internet:** Clients (web apps, mobile apps) send requests to a **server** over the internet. The server processes the request and sends back a **response**. This request-response exchange is done via **APIs**.

```
Client (Browser/App)
    ↓  HTTP Request
Server (Spring Boot App)
    ↓  calls database
    ↓  business logic
    ↑  HTTP Response (JSON)
Client receives data
```

**REST API** = Representational State Transfer API — the most widely used type of API today.

---

## 2. MVC Architecture

**MVC** = **Model-View-Controller** — a design pattern that enforces **Separation of Concerns**.

| Layer | Responsibility |
|---|---|
| **Controller** | Entry point — receives the request, routes it, validates it |
| **Model** | Data/business object that carries the processed information |
| **View** | How the data is presented to the client (HTML, JSON, XML) |

**Traditional (JSP-based):**
```
Client → Controller → Service (Model) → View (HTML via JSP) → Client
```

**Modern (REST API):**
```
Client → Controller → Service (Model) → HTTP Message Converter (JSON) → Client
```

In REST APIs, **View Resolver** is not needed. The HTTP Message Converter directly serializes the Java object to JSON.

---

## 3. Three-Layer Architecture (Spring Boot)

The Spring Boot convention maps MVC layers to this practical structure:

```
Presentation Layer   →  Controller
Business Logic Layer →  Service
Data Access Layer    →  Repository (+ Entity)
```

### Data flow:

```
Client
  ↓ HTTP request
Controller  ←→  DTO (Data Transfer Object)
  ↓
Service     ←→  Business Logic
  ↓
Repository  ←→  Entity (database model)
  ↓
Database
```

### Why this separation?

- **Separation of Concerns** — each layer has a single responsibility
- **Encapsulation** — database schema (entities) are never exposed directly to clients
- **Scalability** — multiple developers can work on different layers independently
- **Security** — DTOs control exactly what data is exposed, preventing data leakage

### DTO vs Entity

| | DTO (Data Transfer Object) | Entity |
|--|---|---|
| Used in | Controller ↔ Service communication | Service ↔ Repository/DB communication |
| Maps to | What the client sends/receives | What the database stores |
| Exposed to client? | Yes | No |

---

## 4. How DispatcherServlet Works

The `DispatcherServlet` is the **front controller** of Spring MVC. Every incoming HTTP request goes through it.

### Full request flow:

```
1. Client sends HTTP Request
2. Embedded Tomcat Server receives the raw HTTP request
3. Tomcat passes it to DispatcherServlet
4. DispatcherServlet creates:
   - HttpServletRequest  (wraps request data: body, headers, params)
   - HttpServletResponse (will carry the response)
5. DispatcherServlet does Handler Mapping:
   - Scans registered @GetMapping/@PostMapping etc.
   - Finds the matching controller method
6. DispatcherServlet invokes the Controller method
7. Controller runs business logic (via Service → Repository → DB)
8. Controller returns a Java object (DTO/POJO)
9. HTTP Message Converter (Jackson) serializes Java object → JSON
10. DispatcherServlet writes JSON into HttpServletResponse
11. Tomcat sends HTTP response to client
12. Client receives JSON
```

### Where does DispatcherServlet come from?

It's auto-configured by `spring-boot-starter-web`. Startup log shows:
```
Initializing Spring DispatcherServlet 'dispatcherServlet'
```

---

## 5. Embedded Tomcat Server

Spring Boot ships with **Apache Tomcat embedded** — no separate server installation needed.

**Where does Tomcat come from?**

`spring-boot-starter-web` → includes → `spring-boot-starter-tomcat` → embedded Tomcat

Tomcat's job:
- Listen on port 8080 (default)
- Accept raw HTTP connections
- Convert HTTP bytes → Java HttpServletRequest
- Delegate to DispatcherServlet
- Send back the HttpServletResponse as HTTP bytes

---

## 6. HTTP Message Converter (Jackson)

Spring Boot uses **Jackson** (`jackson-databind`) to serialize/deserialize Java objects to/from JSON.

- Included automatically via `spring-boot-starter-json` (bundled with `spring-boot-starter-web`)
- Works transparently — you just return a Java object and it becomes JSON
- Works for complex nested objects too

```java
// This POJO:
public class StudentDto {
    private Long id;
    private String name;
    private String email;
}

// Gets automatically serialized to:
// {"id":1,"name":"Aman","email":"aman@gmail.com"}
```

---

## 7. REST Controller Annotations

### `@RestController` vs `@Controller`

```java
@RestController  // = @Controller + @ResponseBody
public class StudentController { ... }
```

- `@Controller` — marks as a web controller (traditionally returns HTML view names)
- `@ResponseBody` — tells Spring to write the return value directly to HTTP response body (as JSON)
- `@RestController` — combines both: returns JSON, not HTML

### HTTP Method Mappings

| Annotation | HTTP Method | Typical Use |
|---|---|---|
| `@GetMapping` | GET | Read/retrieve data |
| `@PostMapping` | POST | Create new resource |
| `@PutMapping` | PUT | Full update of resource |
| `@PatchMapping` | PATCH | Partial update of resource |
| `@DeleteMapping` | DELETE | Delete resource |

---

## 8. Building a REST API — Step by Step

### Step 1: Project structure

```
src/main/java/com/codingshuttle/youtube/
├── controller/
│   └── StudentController.java
├── service/
│   └── StudentService.java
├── repository/
│   └── StudentRepository.java (interfaces DB)
├── entity/
│   └── Student.java (maps to DB table)
├── dto/
│   └── StudentDto.java (what we expose to clients)
└── LearningRestApiApplication.java
```

### Step 2: Create DTO

```java
@Data           // Lombok: generates getters, setters, hashCode, equals, toString
@AllArgsConstructor  // Lombok: generates constructor with all fields
@NoArgsConstructor   // Lombok: generates empty constructor
public class StudentDto {
    private Long id;
    private String name;
    private String email;
}
```

### Step 3: Create Controller

```java
@RestController
public class StudentController {

    @GetMapping("/student")
    public StudentDto getStudent() {
        return new StudentDto(1L, "Aman", "aman@gmail.com");
    }

    @GetMapping("/students/{id}")
    public StudentDto getStudentById(@PathVariable Long id) {
        // In real code: call service → repository → DB
        return new StudentDto(id, "Aman", "aman@gmail.com");
    }
}
```

### Step 4: Run and test

```
GET http://localhost:8080/student
Response: {"id":1,"name":"Aman","email":"aman@gmail.com"}
```

---

## 9. Lombok — Reducing Boilerplate Code

Lombok is a Java annotation processing library that generates boilerplate code at compile time.

### Without Lombok:

```java
public class StudentDto {
    private Long id;
    private String name;
    private String email;

    // Have to write manually:
    public StudentDto() {}
    public StudentDto(Long id, String name, String email) {
        this.id = id; this.name = name; this.email = email;
    }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    // ... getters/setters for all fields
    // ... equals, hashCode, toString
}
```

### With Lombok:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class StudentDto {
    private Long id;
    private String name;
    private String email;
    // ALL of the above is auto-generated at compile time!
}
```

### Key Lombok annotations:

| Annotation | Generates |
|---|---|
| `@Data` | getters, setters, `equals()`, `hashCode()`, `toString()` |
| `@AllArgsConstructor` | constructor with all fields |
| `@NoArgsConstructor` | empty (no-arg) constructor |
| `@RequiredArgsConstructor` | constructor for `final` fields only |
| `@Builder` | Builder design pattern |
| `@Slf4j` | Logger field |

### Dependency:

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

---

## 10. Summary — Complete Request Flow

```
Browser/Postman/App (Client)
    |
    | HTTP Request: GET /student
    ↓
Embedded Apache Tomcat Server
    |  converts HTTP bytes → HttpServletRequest
    ↓
DispatcherServlet
    |  HandlerMapping: "GET /student" → StudentController.getStudent()
    ↓
StudentController.getStudent()
    |  returns StudentDto { id=1, name="Aman", email="aman@gmail.com" }
    ↓
HTTP Message Converter (Jackson)
    |  StudentDto → {"id":1,"name":"Aman","email":"aman@gmail.com"}
    ↓
HttpServletResponse (populated with JSON body)
    ↓
Embedded Apache Tomcat Server
    |  HTTP response with JSON body
    ↓
Client receives JSON response
```

---

## 11. Key Terms Summary

| Term | Meaning |
|---|---|
| **API** | Application Programming Interface — contract between client and server |
| **REST** | Representational State Transfer — most common API style |
| **MVC** | Model-View-Controller design pattern |
| **Three-Layer Architecture** | Controller → Service → Repository |
| **DispatcherServlet** | Spring's front controller — routes all requests |
| **Handler Mapping** | Maps URL + HTTP method to the right controller method |
| **Tomcat** | Embedded web server inside Spring Boot |
| **HTTP Message Converter** | Serializes Java objects ↔ JSON |
| **Jackson** | The JSON library used by Spring Boot |
| **DTO** | Data Transfer Object — what we expose to clients |
| **Entity** | Java class mapped to a database table |
| **@RestController** | `@Controller` + `@ResponseBody` — returns JSON |
| **@GetMapping** | Maps GET requests to a method |
| **Lombok** | Library that generates boilerplate code via annotations |
| **@Data** | Lombok: generates getters, setters, equals, hashCode, toString |
| **POJO** | Plain Old Java Object — a simple class with no framework dependencies |
