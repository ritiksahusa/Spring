# Chapter 3 — Q&A, MCQ & Practice Questions

> **Topic:** Dispatcher Servlet and MVC Flow
> Review these after reading `notes.md`. Answers are at the bottom.

---

## Part A — Short Answer Questions

<a id="q1"></a>**Q1.** What does API stand for, and what is its purpose? [↓ Answer](#a1)

<a id="q2"></a>**Q2.** What are the three layers in Spring Boot's three-layer architecture? [↓ Answer](#a2)

<a id="q3"></a>**Q3.** What is the role of DispatcherServlet in a Spring Boot application? [↓ Answer](#a3)

<a id="q4"></a>**Q4.** What is the difference between `@Controller` and `@RestController`? [↓ Answer](#a4)

<a id="q5"></a>**Q5.** What is Jackson, and what is its role in Spring Boot REST APIs? [↓ Answer](#a5)

<a id="q6"></a>**Q6.** What is a DTO (Data Transfer Object)? Why is it used instead of directly exposing entities? [↓ Answer](#a6)

<a id="q7"></a>**Q7.** What is Lombok? Name three annotations it provides and what each generates. [↓ Answer](#a7)

<a id="q8"></a>**Q8.** What is Handler Mapping, and who is responsible for it in Spring MVC? [↓ Answer](#a8)

<a id="q9"></a>**Q9.** In the traditional MVC flow, what did the View Resolver do? Why is it not needed in modern REST APIs? [↓ Answer](#a9)

<a id="q10"></a>**Q10.** What is the default port Tomcat listens on, and where does the embedded Tomcat server come from in a Spring Boot project? [↓ Answer](#a10)

---

## Part B — Multiple Choice Questions (MCQ)

<a id="q11"></a>**Q11.** What is the correct order of components in a Spring Boot REST request flow? [↓ Answer](#a11)

- A) Client → DispatcherServlet → Tomcat → Controller → Jackson → Client
- B) Client → Tomcat → DispatcherServlet → Handler Mapping → Controller → Jackson → Tomcat → Client
- C) Client → Controller → DispatcherServlet → Tomcat → Client
- D) Client → Jackson → Tomcat → DispatcherServlet → Client

<a id="q12"></a>**Q12.** `@RestController` is equivalent to which combination of annotations? [↓ Answer](#a12)

- A) `@Component` + `@ResponseBody`
- B) `@Controller` + `@Autowired`
- C) `@Controller` + `@ResponseBody`
- D) `@Service` + `@ResponseBody`

<a id="q13"></a>**Q13.** Which HTTP method mapping annotation would you use to create a resource (e.g., a new student)? [↓ Answer](#a13)

- A) `@GetMapping`
- B) `@DeleteMapping`
- C) `@PostMapping`
- D) `@PatchMapping`

<a id="q14"></a>**Q14.** What does Lombok's `@Data` annotation generate? [↓ Answer](#a14)

- A) Only getters
- B) Only setters
- C) Getters, setters, equals(), hashCode(), and toString()
- D) Constructors only

<a id="q15"></a>**Q15.** Where does the embedded Tomcat server come from in a `spring-boot-starter-web` project? [↓ Answer](#a15)

- A) It must be installed separately
- B) It is bundled inside `spring-boot-starter-tomcat` which is a transitive dependency of `spring-boot-starter-web`
- C) It is part of the JDK
- D) You must explicitly add `tomcat-embed-core` to `pom.xml`

<a id="q16"></a>**Q16.** What component converts a Java object (like `StudentDto`) to JSON in a Spring Boot response? [↓ Answer](#a16)

- A) DispatcherServlet
- B) Handler Mapping
- C) View Resolver
- D) HTTP Message Converter (Jackson)

<a id="q17"></a>**Q17.** Which layer in the three-layer architecture should contain database queries and JPA/repository logic? [↓ Answer](#a17)

- A) Presentation Layer (Controller)
- B) Business Logic Layer (Service)
- C) Data Access Layer (Repository)
- D) All three layers equally

<a id="q18"></a>**Q18.** In Spring Boot's MVC flow, when does DispatcherServlet know which controller method to call? [↓ Answer](#a18)

- A) At compile time
- B) By checking Handler Mappings registered during application startup (component scanning)
- C) It always calls the first method in the class
- D) By reading `application.properties`

<a id="q19"></a>**Q19.** Why should you NOT return an Entity directly from a controller? [↓ Answer](#a19)

- A) Entities are too slow to serialize
- B) Entities expose the database schema/structure to clients, creating a security and coupling issue
- C) Spring Boot cannot serialize entities
- D) Entities don't have getters by default

<a id="q20"></a>**Q20.** What is the difference between `@GetMapping("/students")` and `@GetMapping("/students/{id}")`? [↓ Answer](#a20)

- A) They are identical
- B) The first returns all students; the second uses a path variable to return one student by ID
- C) The first requires authentication; the second doesn't
- D) The second one triggers a POST request

---

## Part C — Scenario / Code-Reading Questions

<a id="q21"></a>**Q21.** A developer writes this:

```java
@RestController
public class OrderController {

    @GetMapping("/orders")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
}
```

Where `Order` is a JPA Entity. What is wrong with this code, and how would you fix it? [↓ Answer](#a21)

---

<a id="q22"></a>**Q22.** Trace through this request step by step:
```
GET http://localhost:8080/student/42
```

Describe what happens at each stage (Tomcat, DispatcherServlet, Controller, Jackson). [↓ Answer](#a22)

---

<a id="q23"></a>**Q23.** A developer creates a `ProductDto` class but forgets to add `@NoArgsConstructor`. What runtime error might Jackson throw when it tries to deserialize an incoming request body, and why? [↓ Answer](#a23)

---

<a id="q24"></a>**Q24.** Look at this code:

```java
@Controller
public class ReportController {

    @GetMapping("/report")
    public ReportDto generateReport() {
        return new ReportDto("Q1 2025", 1500000.0);
    }
}
```

Will this work as expected for a REST API returning JSON? If not, what is missing? [↓ Answer](#a24)

---

<a id="q25"></a>**Q25.** A teammate says: "I put all my code — database calls, business logic, and controller routing — in one class inside the controller." What are the problems with this approach, and how would you restructure it? [↓ Answer](#a25)

---

## Part D — Fill in the Blanks

<a id="q26"></a>**Q26.** The component that receives all HTTP requests and routes them to the correct controller is called the ________. [↓ Answer](#a26)

<a id="q27"></a>**Q27.** Spring Boot uses ________ as the default JSON serialization/deserialization library. [↓ Answer](#a27)

<a id="q28"></a>**Q28.** The data object used to transfer data between the presentation layer and the client is called a ________. [↓ Answer](#a28)

<a id="q29"></a>**Q29.** `@RestController` = `@Controller` + ________. [↓ Answer](#a29)

<a id="q30"></a>**Q30.** Lombok's ________ annotation generates an all-arguments constructor and ________ generates a no-argument constructor. [↓ Answer](#a30)

---

## Part E — Draw / Design Questions

<a id="q31"></a>**Q31.** Draw (in text) the complete request-response flow for: [↓ Answer](#a31)

```
POST http://localhost:8080/students
Body: {"name":"Priya","email":"priya@example.com"}
```

Show every component from client to database and back, including what object each step holds.

---

<a id="q32"></a>**Q32.** Given this Entity: [↓ Answer](#a32)

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

<a id="a1"></a>**1.** API = Application Programming Interface. Its purpose is to enable communication between two systems (client and server) over a defined contract (URLs, request/response formats). [↑ Question](#q1)

<a id="a2"></a>**2.** Presentation Layer (Controller), Business Logic Layer (Service), Data Access Layer (Repository). [↑ Question](#q2)

<a id="a3"></a>**3.** DispatcherServlet is Spring MVC's front controller. It receives ALL HTTP requests from Tomcat, performs Handler Mapping to find the right controller method, invokes the controller, and manages the response flow. [↑ Question](#q3)

<a id="a4"></a>**4.** `@Controller` marks a class as a web controller that can return view names (HTML). `@RestController` = `@Controller` + `@ResponseBody`, meaning it returns data (JSON) directly to the response body, not a view. [↑ Question](#q4)

<a id="a5"></a>**5.** Jackson (`jackson-databind`) is a Java JSON library. In Spring Boot, it's the HTTP Message Converter that automatically serializes Java objects to JSON (and deserializes JSON to Java objects) in REST APIs. [↑ Question](#q5)

<a id="a6"></a>**6.** A DTO is a simple Java class used to transfer only the required data between layers or to clients. It prevents exposing the database schema (entities) directly to clients, improving security and decoupling. [↑ Question](#q6)

<a id="a7"></a>**7.** Lombok reduces Java boilerplate. Three annotations: `@Data` — generates getters, setters, equals, hashCode, toString; `@AllArgsConstructor` — generates all-args constructor; `@NoArgsConstructor` — generates no-arg constructor. [↑ Question](#q7)

<a id="a8"></a>**8.** Handler Mapping is the process of finding which controller method should handle a given request (based on URL path + HTTP method). DispatcherServlet performs Handler Mapping. [↑ Question](#q8)

<a id="a9"></a>**9.** View Resolver resolved the view name returned by a controller to an actual view template (like a JSP/HTML file). In REST APIs, there's no HTML view — Jackson serializes the Java object to JSON directly, so View Resolver is skipped. [↑ Question](#q9)

<a id="a10"></a>**10.** Port 8080 (default). Embedded Tomcat comes from `spring-boot-starter-tomcat`, which is included by `spring-boot-starter-web`. [↑ Question](#q10)

### Part B

<a id="a11"></a>**11.** **B** — Client → Tomcat → DispatcherServlet → Handler Mapping → Controller → Jackson → Tomcat → Client [↑ Question](#q11)

<a id="a12"></a>**12.** **C** — `@Controller` + `@ResponseBody` [↑ Question](#q12)

<a id="a13"></a>**13.** **C** — `@PostMapping` [↑ Question](#q13)

<a id="a14"></a>**14.** **C** — Getters, setters, equals(), hashCode(), and toString() [↑ Question](#q14)

<a id="a15"></a>**15.** **B** — Bundled inside `spring-boot-starter-tomcat` [↑ Question](#q15)

<a id="a16"></a>**16.** **D** — HTTP Message Converter (Jackson) [↑ Question](#q16)

<a id="a17"></a>**17.** **C** — Data Access Layer (Repository) [↑ Question](#q17)

<a id="a18"></a>**18.** **B** — By checking Handler Mappings registered during application startup [↑ Question](#q18)

<a id="a19"></a>**19.** **B** — Entities expose the database schema/structure [↑ Question](#q19)

<a id="a20"></a>**20.** **B** — First returns all; second uses `{id}` path variable for one [↑ Question](#q20)

### Part C

<a id="a21"></a>**21.** The entity `Order` is returned directly. Problems: exposes database schema, includes internal fields, creates tight coupling. Fix: create `OrderDto` with only client-needed fields, map `Order` → `OrderDto` in the service layer, return `OrderDto` from the controller. [↑ Question](#q21)

<a id="a22"></a>**22.** Step-by-step: [↑ Question](#q22)
- Client sends `GET http://localhost:8080/student/42`
- Embedded Tomcat receives the TCP connection and parses HTTP → creates `HttpServletRequest`
- Tomcat delegates to `DispatcherServlet`
- DispatcherServlet checks Handler Mappings — finds `StudentController.getStudentById(@PathVariable Long id)` mapped to `GET /student/{id}`
- DispatcherServlet invokes the method with `id=42`
- Controller (via Service → Repository) returns a `StudentDto` Java object
- HTTP Message Converter (Jackson) serializes `StudentDto` → `{"id":42,"name":"...","email":"..."}`
- JSON is written into `HttpServletResponse`
- Tomcat sends HTTP 200 response with JSON body back to client

<a id="a23"></a>**23.** Jackson requires a no-argument constructor to deserialize JSON into a Java object (it creates an empty instance then sets field values). Without `@NoArgsConstructor`, Jackson throws `InvalidDefinitionException: No suitable constructor found for...`. [↑ Question](#q23)

<a id="a24"></a>**24.** No, it won't work as a REST API. The class is annotated with `@Controller`, not `@RestController`. Without `@ResponseBody`, Spring tries to resolve the return value as a view name, not JSON. Fix: change `@Controller` to `@RestController` (or add `@ResponseBody` to the method). [↑ Question](#q24)

<a id="a25"></a>**25.** Problems: No separation of concerns, code is hard to maintain and test, a single change can break unrelated functionality, team members can't work in parallel. Restructure: Move controller routing logic to `@RestController`, move business logic to `@Service`, move database queries to `@Repository`. [↑ Question](#q25)

### Part D

<a id="a26"></a>**26.** DispatcherServlet [↑ Question](#q26)

<a id="a27"></a>**27.** Jackson (jackson-databind) [↑ Question](#q27)

<a id="a28"></a>**28.** DTO (Data Transfer Object) [↑ Question](#q28)

<a id="a29"></a>**29.** `@ResponseBody` [↑ Question](#q29)

<a id="a30"></a>**30.** `@AllArgsConstructor` / `@NoArgsConstructor` [↑ Question](#q30)

### Part E

<a id="a31"></a>**31.** POST request flow: [↑ Question](#q31)
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

<a id="a32"></a>**32.** [↑ Question](#q32)
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

<a id="qf1"></a>**F1.** Two controller methods are mapped `@GetMapping("/students/{id}")` and `@GetMapping("/students/active")`. What actually happens for a request to `/students/active`, and how do you make this unambiguous? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** Spring MVC ranks route matches by specificity — a literal path segment (`active`) is more specific than a path-variable pattern (`{id}`), so `/students/active` correctly routes to the literal mapping. The real danger is declaring the specific route in a DIFFERENT controller class processed later, or relying on implicit ordering. The safe, explicit fix is to constrain the variable's format: `@GetMapping("/students/{id:[0-9]+}")`, and always declare literal/specific routes before generic parameterized ones in the same class. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** What is content negotiation, and how does Spring MVC decide whether to return JSON or XML from the SAME endpoint? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** Content negotiation selects the response representation based on the client's `Accept` header (or a path suffix / query param, depending on configured strategy). Spring's `ContentNegotiationManager` checks configured strategies in order and picks the first registered `HttpMessageConverter` able to produce that media type. With both `jackson-databind` (JSON) and `jackson-dataformat-xml` (XML) on the classpath, the same method can serve `Accept: application/json` or `Accept: application/xml` clients differently, with zero extra code. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** A client receives `HTTP 406 Not Acceptable`. What causes this, and how do you fix it? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** 406 occurs when the client's `Accept` header requests a media type that NONE of the registered `HttpMessageConverter`s can produce (e.g., requesting XML when only Jackson JSON is on the classpath). Fix by adding the missing converter dependency, ensuring clients send a supported `Accept` header, or configuring `ContentNegotiationConfigurer.defaultContentType(MediaType.APPLICATION_JSON)` as a fallback. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** How does Jackson serialize a `LocalDateTime` field by default in a Spring Boot app, and what makes it output clean ISO-8601 instead of a numeric array? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** Vanilla Jackson doesn't understand `java.time` types out of the box and would fail or emit a numeric array. Spring Boot auto-configures the `jackson-datatype-jsr310` module (bundled transitively), registering proper serializers. Combined with `spring.jackson.serialization.write-dates-as-timestamps=false` (Boot's default, unlike raw Jackson which defaults to epoch-millis timestamps), `LocalDateTime` serializes as `"2025-03-15T10:00:00"`. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** What is `HandlerInterceptor.afterCompletion()` for, and why can't a simple `try/finally` in the controller replace it? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** `afterCompletion()` runs after the ENTIRE request-response cycle completes, regardless of whether an exception occurred — it still fires (for interceptors whose `preHandle` already succeeded) even if a LATER interceptor blocks the request or the controller is never reached at all. A `try/finally` inside a controller method can't cover cases where the controller method never executes in the first place. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** Why does a plain `@Controller` method returning `ResponseEntity<T>` still return raw JSON WITHOUT needing `@ResponseBody`, even though the class isn't `@RestController`? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** Spring MVC's `HttpEntityMethodProcessor` special-cases `ResponseEntity`/`HttpEntity` return types — returning a full `ResponseEntity` unambiguously means "this IS the raw HTTP response," so it's ALWAYS written directly to the body regardless of `@ResponseBody` presence, distinguishing it from a String return value (which would normally be treated as a view name on a plain `@Controller`). [↑ Question](#qf6)

<a id="qf7"></a>**F7.** A teammate implements business validation logic inside a custom `HandlerInterceptor`. Why is this the wrong layer, and where should it live instead? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** `HandlerInterceptor` runs before the controller method executes but is meant for cross-cutting REQUEST-level concerns (auth checks, locale, simple header validation) — not domain/business rules, which depend on parsed request bodies. Business validation belongs in the Service layer (with `@Valid` + Bean Validation handling structural/format checks at the Controller boundary). Business rules hidden in interceptors are invisible to OpenAPI/Swagger docs and harder to unit test in isolation. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** What mechanism converts `@PathVariable Long id` from the URL string `"42"` to a `Long`, and what happens if a client sends `"abc"` instead? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** Spring's `ConversionService` (backed by registered `Converter`/`PropertyEditor` implementations) performs type conversion during argument resolution, BEFORE the controller method body runs. An invalid value like `"abc"` throws `MethodArgumentTypeMismatchException`, which by default results in `400 Bad Request` — entirely within `DispatcherServlet`'s argument-resolution phase, never reaching your method. [↑ Question](#qf8)

````
```
