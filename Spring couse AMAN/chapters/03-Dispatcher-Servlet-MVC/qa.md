# Chapter 3 — Q&A, MCQ & Practice Questions

> **Topic:** Dispatcher Servlet and MVC Flow
> Review these after reading `notes.md`. Answers are at the bottom.

---

## Part A — Short Answer Questions

**Q1.** What does API stand for, and what is its purpose?

**Q2.** What are the three layers in Spring Boot's three-layer architecture?

**Q3.** What is the role of DispatcherServlet in a Spring Boot application?

**Q4.** What is the difference between `@Controller` and `@RestController`?

**Q5.** What is Jackson, and what is its role in Spring Boot REST APIs?

**Q6.** What is a DTO (Data Transfer Object)? Why is it used instead of directly exposing entities?

**Q7.** What is Lombok? Name three annotations it provides and what each generates.

**Q8.** What is Handler Mapping, and who is responsible for it in Spring MVC?

**Q9.** In the traditional MVC flow, what did the View Resolver do? Why is it not needed in modern REST APIs?

**Q10.** What is the default port Tomcat listens on, and where does the embedded Tomcat server come from in a Spring Boot project?

---

## Part B — Multiple Choice Questions (MCQ)

**Q11.** What is the correct order of components in a Spring Boot REST request flow?

- A) Client → DispatcherServlet → Tomcat → Controller → Jackson → Client
- B) Client → Tomcat → DispatcherServlet → Handler Mapping → Controller → Jackson → Tomcat → Client
- C) Client → Controller → DispatcherServlet → Tomcat → Client
- D) Client → Jackson → Tomcat → DispatcherServlet → Client

**Q12.** `@RestController` is equivalent to which combination of annotations?

- A) `@Component` + `@ResponseBody`
- B) `@Controller` + `@Autowired`
- C) `@Controller` + `@ResponseBody`
- D) `@Service` + `@ResponseBody`

**Q13.** Which HTTP method mapping annotation would you use to create a resource (e.g., a new student)?

- A) `@GetMapping`
- B) `@DeleteMapping`
- C) `@PostMapping`
- D) `@PatchMapping`

**Q14.** What does Lombok's `@Data` annotation generate?

- A) Only getters
- B) Only setters
- C) Getters, setters, equals(), hashCode(), and toString()
- D) Constructors only

**Q15.** Where does the embedded Tomcat server come from in a `spring-boot-starter-web` project?

- A) It must be installed separately
- B) It is bundled inside `spring-boot-starter-tomcat` which is a transitive dependency of `spring-boot-starter-web`
- C) It is part of the JDK
- D) You must explicitly add `tomcat-embed-core` to `pom.xml`

**Q16.** What component converts a Java object (like `StudentDto`) to JSON in a Spring Boot response?

- A) DispatcherServlet
- B) Handler Mapping
- C) View Resolver
- D) HTTP Message Converter (Jackson)

**Q17.** Which layer in the three-layer architecture should contain database queries and JPA/repository logic?

- A) Presentation Layer (Controller)
- B) Business Logic Layer (Service)
- C) Data Access Layer (Repository)
- D) All three layers equally

**Q18.** In Spring Boot's MVC flow, when does DispatcherServlet know which controller method to call?

- A) At compile time
- B) By checking Handler Mappings registered during application startup (component scanning)
- C) It always calls the first method in the class
- D) By reading `application.properties`

**Q19.** Why should you NOT return an Entity directly from a controller?

- A) Entities are too slow to serialize
- B) Entities expose the database schema/structure to clients, creating a security and coupling issue
- C) Spring Boot cannot serialize entities
- D) Entities don't have getters by default

**Q20.** What is the difference between `@GetMapping("/students")` and `@GetMapping("/students/{id}")`?

- A) They are identical
- B) The first returns all students; the second uses a path variable to return one student by ID
- C) The first requires authentication; the second doesn't
- D) The second one triggers a POST request

---

## Part C — Scenario / Code-Reading Questions

**Q21.** A developer writes this:

```java
@RestController
public class OrderController {

    @GetMapping("/orders")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
}
```

Where `Order` is a JPA Entity. What is wrong with this code, and how would you fix it?

---

**Q22.** Trace through this request step by step:
```
GET http://localhost:8080/student/42
```

Describe what happens at each stage (Tomcat, DispatcherServlet, Controller, Jackson).

---

**Q23.** A developer creates a `ProductDto` class but forgets to add `@NoArgsConstructor`. What runtime error might Jackson throw when it tries to deserialize an incoming request body, and why?

---

**Q24.** Look at this code:

```java
@Controller
public class ReportController {

    @GetMapping("/report")
    public ReportDto generateReport() {
        return new ReportDto("Q1 2025", 1500000.0);
    }
}
```

Will this work as expected for a REST API returning JSON? If not, what is missing?

---

**Q25.** A teammate says: "I put all my code — database calls, business logic, and controller routing — in one class inside the controller." What are the problems with this approach, and how would you restructure it?

---

## Part D — Fill in the Blanks

**Q26.** The component that receives all HTTP requests and routes them to the correct controller is called the ________.

**Q27.** Spring Boot uses ________ as the default JSON serialization/deserialization library.

**Q28.** The data object used to transfer data between the presentation layer and the client is called a ________.

**Q29.** `@RestController` = `@Controller` + ________.

**Q30.** Lombok's ________ annotation generates an all-arguments constructor and ________ generates a no-argument constructor.

---

## Part E — Draw / Design Questions

**Q31.** Draw (in text) the complete request-response flow for:

```
POST http://localhost:8080/students
Body: {"name":"Priya","email":"priya@example.com"}
```

Show every component from client to database and back, including what object each step holds.

---

**Q32.** Given this Entity:

```java
@Entity
public class Product {
    @Id
    private Long id;
    private String name;
    private double price;
    private String internalCostCode; // should NOT be visible to clients
}
```

Design a `ProductDto` class (with Lombok) that exposes only `id`, `name`, and `price` to the API client.

---

## Answers

### Part A

1. API = Application Programming Interface. Its purpose is to enable communication between two systems (client and server) over a defined contract (URLs, request/response formats).

2. Presentation Layer (Controller), Business Logic Layer (Service), Data Access Layer (Repository).

3. DispatcherServlet is Spring MVC's front controller. It receives ALL HTTP requests from Tomcat, performs Handler Mapping to find the right controller method, invokes the controller, and manages the response flow.

4. `@Controller` marks a class as a web controller that can return view names (HTML). `@RestController` = `@Controller` + `@ResponseBody`, meaning it returns data (JSON) directly to the response body, not a view.

5. Jackson (`jackson-databind`) is a Java JSON library. In Spring Boot, it's the HTTP Message Converter that automatically serializes Java objects to JSON (and deserializes JSON to Java objects) in REST APIs.

6. A DTO is a simple Java class used to transfer only the required data between layers or to clients. It prevents exposing the database schema (entities) directly to clients, improving security and decoupling.

7. Lombok reduces Java boilerplate. Three annotations: `@Data` — generates getters, setters, equals, hashCode, toString; `@AllArgsConstructor` — generates all-args constructor; `@NoArgsConstructor` — generates no-arg constructor.

8. Handler Mapping is the process of finding which controller method should handle a given request (based on URL path + HTTP method). DispatcherServlet performs Handler Mapping.

9. View Resolver resolved the view name returned by a controller to an actual view template (like a JSP/HTML file). In REST APIs, there's no HTML view — Jackson serializes the Java object to JSON directly, so View Resolver is skipped.

10. Port 8080 (default). Embedded Tomcat comes from `spring-boot-starter-tomcat`, which is included by `spring-boot-starter-web`.

### Part B

11. **B** — Client → Tomcat → DispatcherServlet → Handler Mapping → Controller → Jackson → Tomcat → Client
12. **C** — `@Controller` + `@ResponseBody`
13. **C** — `@PostMapping`
14. **C** — Getters, setters, equals(), hashCode(), and toString()
15. **B** — Bundled inside `spring-boot-starter-tomcat`
16. **D** — HTTP Message Converter (Jackson)
17. **C** — Data Access Layer (Repository)
18. **B** — By checking Handler Mappings registered during application startup
19. **B** — Entities expose the database schema/structure
20. **B** — First returns all; second uses `{id}` path variable for one

### Part C

21. The entity `Order` is returned directly. Problems: exposes database schema, includes internal fields, creates tight coupling. Fix: create `OrderDto` with only client-needed fields, map `Order` → `OrderDto` in the service layer, return `OrderDto` from the controller.

22. Step-by-step:
- Client sends `GET http://localhost:8080/student/42`
- Embedded Tomcat receives the TCP connection and parses HTTP → creates `HttpServletRequest`
- Tomcat delegates to `DispatcherServlet`
- DispatcherServlet checks Handler Mappings — finds `StudentController.getStudentById(@PathVariable Long id)` mapped to `GET /student/{id}`
- DispatcherServlet invokes the method with `id=42`
- Controller (via Service → Repository) returns a `StudentDto` Java object
- HTTP Message Converter (Jackson) serializes `StudentDto` → `{"id":42,"name":"...","email":"..."}`
- JSON is written into `HttpServletResponse`
- Tomcat sends HTTP 200 response with JSON body back to client

23. Jackson requires a no-argument constructor to deserialize JSON into a Java object (it creates an empty instance then sets field values). Without `@NoArgsConstructor`, Jackson throws `InvalidDefinitionException: No suitable constructor found for...`.

24. No, it won't work as a REST API. The class is annotated with `@Controller`, not `@RestController`. Without `@ResponseBody`, Spring tries to resolve the return value as a view name, not JSON. Fix: change `@Controller` to `@RestController` (or add `@ResponseBody` to the method).

25. Problems: No separation of concerns, code is hard to maintain and test, a single change can break unrelated functionality, team members can't work in parallel. Restructure: Move controller routing logic to `@RestController`, move business logic to `@Service`, move database queries to `@Repository`.

### Part D

26. DispatcherServlet
27. Jackson (jackson-databind)
28. DTO (Data Transfer Object)
29. `@ResponseBody`
30. `@AllArgsConstructor` / `@NoArgsConstructor`

### Part E

31. POST request flow:
```
Client → sends POST /students with JSON body
  ↓ (Jackson reads JSON → StudentDto)
Tomcat Server → receives HTTP, creates HttpServletRequest
  ↓
DispatcherServlet → Handler Mapping → finds StudentController.createStudent()
  ↓
Controller → receives StudentDto (Jackson deserialized from request body)
  ↓
Service → validates and processes the DTO, maps to Student entity
  ↓
Repository → saves Student entity to database
  ↓
Service → maps saved entity back to StudentDto
  ↓
Controller → returns StudentDto
  ↓
Jackson → serializes StudentDto → JSON response body
  ↓
Tomcat → sends HTTP 201 Created response with JSON body
  ↓
Client receives {"id":10,"name":"Priya","email":"priya@example.com"}
```

32.
```java
32.
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ProductDto {
    private Long id;
    private String name;
    private double price;
    // internalCostCode is intentionally excluded
}
```

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

**F1.** Two controller methods are mapped `@GetMapping("/students/{id}")` and `@GetMapping("/students/active")`. What actually happens for a request to `/students/active`, and how do you make this unambiguous?

> **Answer:** Spring MVC ranks route matches by specificity — a literal path segment (`active`) is more specific than a path-variable pattern (`{id}`), so `/students/active` correctly routes to the literal mapping. The real danger is declaring the specific route in a DIFFERENT controller class processed later, or relying on implicit ordering. The safe, explicit fix is to constrain the variable's format: `@GetMapping("/students/{id:[0-9]+}")`, and always declare literal/specific routes before generic parameterized ones in the same class.

**F2.** What is content negotiation, and how does Spring MVC decide whether to return JSON or XML from the SAME endpoint?

> **Answer:** Content negotiation selects the response representation based on the client's `Accept` header (or a path suffix / query param, depending on configured strategy). Spring's `ContentNegotiationManager` checks configured strategies in order and picks the first registered `HttpMessageConverter` able to produce that media type. With both `jackson-databind` (JSON) and `jackson-dataformat-xml` (XML) on the classpath, the same method can serve `Accept: application/json` or `Accept: application/xml` clients differently, with zero extra code.

**F3.** A client receives `HTTP 406 Not Acceptable`. What causes this, and how do you fix it?

> **Answer:** 406 occurs when the client's `Accept` header requests a media type that NONE of the registered `HttpMessageConverter`s can produce (e.g., requesting XML when only Jackson JSON is on the classpath). Fix by adding the missing converter dependency, ensuring clients send a supported `Accept` header, or configuring `ContentNegotiationConfigurer.defaultContentType(MediaType.APPLICATION_JSON)` as a fallback.

**F4.** How does Jackson serialize a `LocalDateTime` field by default in a Spring Boot app, and what makes it output clean ISO-8601 instead of a numeric array?

> **Answer:** Vanilla Jackson doesn't understand `java.time` types out of the box and would fail or emit a numeric array. Spring Boot auto-configures the `jackson-datatype-jsr310` module (bundled transitively), registering proper serializers. Combined with `spring.jackson.serialization.write-dates-as-timestamps=false` (Boot's default, unlike raw Jackson which defaults to epoch-millis timestamps), `LocalDateTime` serializes as `"2025-03-15T10:00:00"`.

**F5.** What is `HandlerInterceptor.afterCompletion()` for, and why can't a simple `try/finally` in the controller replace it?

> **Answer:** `afterCompletion()` runs after the ENTIRE request-response cycle completes, regardless of whether an exception occurred — it still fires (for interceptors whose `preHandle` already succeeded) even if a LATER interceptor blocks the request or the controller is never reached at all. A `try/finally` inside a controller method can't cover cases where the controller method never executes in the first place.

**F6.** Why does a plain `@Controller` method returning `ResponseEntity<T>` still return raw JSON WITHOUT needing `@ResponseBody`, even though the class isn't `@RestController`?

> **Answer:** Spring MVC's `HttpEntityMethodProcessor` special-cases `ResponseEntity`/`HttpEntity` return types — returning a full `ResponseEntity` unambiguously means "this IS the raw HTTP response," so it's ALWAYS written directly to the body regardless of `@ResponseBody` presence, distinguishing it from a String return value (which would normally be treated as a view name on a plain `@Controller`).

**F7.** A teammate implements business validation logic inside a custom `HandlerInterceptor`. Why is this the wrong layer, and where should it live instead?

> **Answer:** `HandlerInterceptor` runs before the controller method executes but is meant for cross-cutting REQUEST-level concerns (auth checks, locale, simple header validation) — not domain/business rules, which depend on parsed request bodies. Business validation belongs in the Service layer (with `@Valid` + Bean Validation handling structural/format checks at the Controller boundary). Business rules hidden in interceptors are invisible to OpenAPI/Swagger docs and harder to unit test in isolation.

**F8.** What mechanism converts `@PathVariable Long id` from the URL string `"42"` to a `Long`, and what happens if a client sends `"abc"` instead?

> **Answer:** Spring's `ConversionService` (backed by registered `Converter`/`PropertyEditor` implementations) performs type conversion during argument resolution, BEFORE the controller method body runs. An invalid value like `"abc"` throws `MethodArgumentTypeMismatchException`, which by default results in `400 Bad Request` — entirely within `DispatcherServlet`'s argument-resolution phase, never reaching your method.

````
```
