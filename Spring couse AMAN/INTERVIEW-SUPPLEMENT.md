# Spring Boot — Interview Supplement
**Covers critical topics commonly asked in product-company and service-company interviews
that go beyond the course transcripts.**

---

## GAP ANALYSIS — What the Course Covers vs What Interviewers Ask

| Topic | Course | This File |
|---|---|---|
| IoC, DI, beans | ✅ Ch02 | Extended below |
| Bean Scopes | ❌ | ✅ Section 1 |
| @Primary / @Qualifier | ❌ | ✅ Section 2 |
| Spring AOP | ❌ | ✅ Section 3 |
| Bean Lifecycle | ❌ | ✅ Section 4 |
| Spring Profiles | ❌ | ✅ Section 5 |
| Global Exception Handling | ❌ | ✅ Section 6 |
| @RequestParam vs @PathVariable | Partial | ✅ Section 7 |
| @Transactional deep-dive | Partial | ✅ Section 8 |
| Transaction Isolation Levels | ❌ | ✅ Section 9 |
| Optimistic vs Pessimistic Locking | ❌ | ✅ Section 10 |
| Hibernate 2nd Level Cache | ❌ | ✅ Section 11 |
| Custom Validators | ❌ | ✅ Section 12 |
| Testing (@MockBean, @WebMvcTest) | Partial | ✅ Section 13 |
| Filter vs Interceptor vs AOP | ❌ | ✅ Section 14 |
| Flyway / Liquibase | Mentioned | ✅ Section 15 |
| Spring Boot Actuator | ❌ | ✅ Section 16 |
| Spring vs Spring Boot differences | ❌ | ✅ Section 17 |
| Common Interview Q&A | ❌ | ✅ Section 18 |

---

## Section 1 — Bean Scopes

**Q: What are Spring bean scopes?**

Bean scope defines **how many instances of a bean Spring creates and how long they live**.

| Scope | Instances | Lifetime | When to use |
|---|---|---|---|
| `singleton` | 1 per Application Context | Entire app lifetime | Default. Stateless services, repositories |
| `prototype` | New instance per injection | Until garbage collected | Stateful beans; each client needs its own copy |
| `request` | 1 per HTTP request | Duration of HTTP request | Web only: per-request state |
| `session` | 1 per HTTP session | Duration of HTTP session | Web only: per-user login state |
| `application` | 1 per ServletContext | Entire web app lifetime | Web only: app-wide shared state |

### Singleton (Default)

```java
@Component
// @Scope("singleton") // redundant — singleton is the default
public class StudentService {
    // Spring creates exactly ONE instance
    // SAME bean injected everywhere it's needed
}
```

### Prototype

```java
@Component
@Scope("prototype")
public class ReportGenerator {
    // Spring creates a NEW instance every time it is injected or requested
}
```

```java
@Service
public class ReportService {
    @Autowired
    private ApplicationContext context;

    public void generate() {
        // Get a new ReportGenerator each time
        ReportGenerator gen = context.getBean(ReportGenerator.class);
    }
}
```

> **Interview trap:** If you inject a prototype bean into a singleton bean with `@Autowired`, the prototype bean is only created ONCE (at singleton creation time). To get a new instance each time, use `ApplicationContext.getBean()` or `@Lookup`.

### Request Scope (Web)

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String userId;
}
```

`proxyMode = TARGET_CLASS` is needed so the request-scoped bean can be injected into singletons.

---

## Section 2 — @Primary and @Qualifier

### Problem: Multiple beans of same type

```java
@Component public class RazorPayService implements PaymentService { ... }
@Component public class StripeService implements PaymentService { ... }

@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // Spring throws: NoUniqueBeanDefinitionException
}
```

### Solution 1: @Primary — Default bean

```java
@Component
@Primary  // Spring picks this one when no explicit qualifier is given
public class RazorPayService implements PaymentService { ... }
```

### Solution 2: @Qualifier — Explicit selection

```java
@Component
@Qualifier("stripe")
public class StripeService implements PaymentService { ... }

@Service
public class CheckoutService {
    @Autowired
    @Qualifier("stripe")  // explicitly pick Stripe
    private PaymentService paymentService;
}
```

### Solution 3: @ConditionalOnProperty (most flexible)

```properties
payment.provider=stripe
```

```java
@Component
@ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
public class StripeService implements PaymentService { ... }
```

> **Interview rule:** `@Primary` is the default fallback. `@Qualifier` is explicit selection. `@ConditionalOnProperty` is config-driven and most production-friendly.

---

## Section 3 — Spring AOP (Aspect-Oriented Programming)

### Why AOP?

Some concerns are **cross-cutting** — they apply to many methods across many classes, but you don't want to repeat code in all of them:
- Logging every method entry/exit
- Performance monitoring
- Security checks
- Transaction management

AOP lets you write this logic ONCE and apply it to many methods automatically.

### Key Terminology

| Term | Meaning |
|---|---|
| **Aspect** | Class that contains cross-cutting logic |
| **Advice** | The actual code that runs (before/after/around a method) |
| **JoinPoint** | A specific point during program execution (usually a method call) |
| **Pointcut** | Expression that selects which JoinPoints to intercept |
| **Weaving** | Applying aspects to target objects |

### Advice Types

| Type | When it runs |
|---|---|
| `@Before` | Before the method executes |
| `@After` | After the method (whether it succeeds or throws) |
| `@AfterReturning` | Only if method returns successfully |
| `@AfterThrowing` | Only if method throws an exception |
| `@Around` | Wraps the method; you control whether it runs |

### Example — Logging Aspect

```java
@Aspect
@Component
public class LoggingAspect {

    // Pointcut: match all methods in controller package
    @Before("execution(* com.example.controller.*.*(..))")
    public void logBeforeMethod(JoinPoint joinPoint) {
        System.out.println("Calling: " + joinPoint.getSignature().getName());
    }

    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))",
                    returning = "result")
    public void logAfterMethod(JoinPoint jp, Object result) {
        System.out.println("Returned: " + result);
    }

    // @Around gives full control
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed(); // actually call the method
        long elapsed = System.currentTimeMillis() - start;
        System.out.println(pjp.getSignature() + " took " + elapsed + "ms");
        return result;
    }
}
```

**Dependency needed:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### @Transactional is AOP Under the Hood

`@Transactional` is itself implemented as Spring AOP advice — Spring wraps your method call with begin/commit/rollback logic.

---

## Section 4 — Bean Lifecycle

Every Spring bean goes through a lifecycle:

```
1. Instantiation   → Spring creates the bean (calls constructor)
2. DI             → Dependencies are injected (@Autowired, constructor)
3. @PostConstruct → Custom initialisation code you write
4. Bean Ready     → Bean is in use (handles requests, etc.)
5. @PreDestroy    → Custom cleanup code when context is shutting down
6. Destroyed      → Bean is garbage collected
```

### @PostConstruct and @PreDestroy

```java
@Component
public class DatabasePool {

    @PostConstruct  // called after DI is complete; before bean is used
    public void init() {
        System.out.println("Opening DB connections...");
    }

    @PreDestroy  // called when Spring context is shutting down
    public void cleanup() {
        System.out.println("Closing DB connections...");
    }
}
```

> `@PostConstruct` is from `jakarta.annotation`. If DI fails, `@PostConstruct` is never called.

### Alternative: InitializingBean / DisposableBean interfaces

```java
@Component
public class MyBean implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        // called after DI — same as @PostConstruct
    }

    @Override
    public void destroy() throws Exception {
        // called on shutdown — same as @PreDestroy
    }
}
```

### init-method / destroy-method via @Bean

```java
@Bean(initMethod = "start", destroyMethod = "stop")
public MyService myService() {
    return new MyService();
}
```

---

## Section 5 — Spring Profiles

Profiles let you have different configurations for different environments (dev, test, prod).

### Define profile-specific properties

```
application.properties          ← always loaded (common config)
application-dev.properties      ← loaded when profile = dev
application-prod.properties     ← loaded when profile = prod
application-test.properties     ← loaded when profile = test
```

```properties
# application-dev.properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb_dev
spring.jpa.hibernate.ddl-auto=update

# application-prod.properties
spring.datasource.url=jdbc:postgresql://prod-server:5432/mydb
spring.jpa.hibernate.ddl-auto=validate
logging.level.root=WARN
```

### Activate a profile

```properties
# application.properties
spring.profiles.active=dev
```

Or pass as JVM argument:
```bash
java -Dspring.profiles.active=prod -jar myapp.jar
```

Or environment variable:
```bash
SPRING_PROFILES_ACTIVE=prod java -jar myapp.jar
```

### @Profile — conditional bean registration

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // H2 in-memory database for local dev
        return new EmbeddedDatabaseBuilder().setType(H2).build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        // Real Postgres connection pool
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://...");
        return ds;
    }
}
```

---

## Section 6 — Global Exception Handling

### The Problem

Without global handling, unhandled exceptions show Spring's default white-label error page or raw stack traces.

### @ControllerAdvice + @ExceptionHandler

`@ControllerAdvice` is a special component that handles exceptions thrown from any `@Controller` or `@RestController`.

```java
@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // Handle a specific custom exception
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiError handleNotFound(ResourceNotFoundException ex) {
        return new ApiError(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
    }

    // Handle validation failures (@Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidationError(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.toList());
        return new ApiError(400, errors.toString(), LocalDateTime.now());
    }

    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiError handleGeneric(Exception ex) {
        return new ApiError(500, "Internal server error", LocalDateTime.now());
    }
}
```

### Error Response DTO

```java
@Getter
@AllArgsConstructor
public class ApiError {
    private int status;
    private String message;
    private LocalDateTime timestamp;
}
```

### Custom Exception

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// Usage in service:
Patient patient = repo.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Patient not found: " + id));
```

> **@ControllerAdvice** applies globally. Without it, exceptions propagate to Spring's `BasicErrorController` which returns the Whitelabel Error Page.

---

## Section 7 — @RequestParam vs @PathVariable vs @RequestBody

### @PathVariable — URL path segment

```java
// URL: GET /students/42
@GetMapping("/students/{id}")
public StudentDto getById(@PathVariable Long id) { ... }

// Multiple:
@GetMapping("/departments/{deptId}/employees/{empId}")
public Employee get(@PathVariable Long deptId, @PathVariable Long empId) { ... }
```

### @RequestParam — Query string parameter

```java
// URL: GET /students?page=0&size=10&name=Anuj
@GetMapping("/students")
public Page<StudentDto> search(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(required = false) String name) { ... }
```

### @RequestBody — JSON body (for POST/PUT/PATCH)

```java
// Request body: {"name":"Anuj","email":"anuj@gmail.com"}
@PostMapping("/students")
public ResponseEntity<StudentDto> create(@Valid @RequestBody CreateStudentDto dto) { ... }
```

### @RequestHeader — HTTP header

```java
@GetMapping("/profile")
public UserDto profile(@RequestHeader("Authorization") String token) { ... }
```

### Comparison Table

| Annotation | Source | Use Case |
|---|---|---|
| `@PathVariable` | URL path `/items/{id}` | Identify a specific resource |
| `@RequestParam` | Query string `?key=value` | Filtering, sorting, pagination |
| `@RequestBody` | Request body (JSON/XML) | POST/PUT — create or update resource |
| `@RequestHeader` | HTTP headers | Auth tokens, content type |

---

## Section 8 — @Transactional Deep-Dive

### Propagation Levels

Propagation controls what happens when a `@Transactional` method is called FROM another `@Transactional` method.

| Propagation | Behaviour |
|---|---|
| `REQUIRED` (default) | Joins existing transaction; creates new one if none exists |
| `REQUIRES_NEW` | Always creates a new transaction; suspends existing one |
| `NESTED` | Creates a savepoint in existing transaction; can partially rollback |
| `SUPPORTS` | Runs in existing transaction if available; non-transactionally if not |
| `NOT_SUPPORTED` | Suspends any existing transaction; runs non-transactionally |
| `MANDATORY` | Must be called with an existing transaction; throws if none |
| `NEVER` | Must NOT be called with a transaction; throws if one exists |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String message) {
    // Always runs in its own transaction
    // Even if caller's transaction rolls back, this audit log is committed
}
```

### rollbackFor

By default `@Transactional` only rolls back for **unchecked exceptions** (RuntimeException and Error).

```java
@Transactional(rollbackFor = Exception.class)
// Rolls back for ALL exceptions including checked ones

@Transactional(noRollbackFor = ValidationException.class)
// Commits even if ValidationException is thrown
```

### readOnly

```java
@Transactional(readOnly = true)
public List<Patient> getAllPatients() {
    // Hibernate optimisations: skip dirty checking at commit
    // DB: may use read replicas
    return patientRepository.findAll();
}
```

### timeout

```java
@Transactional(timeout = 5) // throws TransactionTimedOutException after 5 seconds
public void longOperation() { ... }
```

---

## Section 9 — Transaction Isolation Levels

**Problem:** Multiple concurrent transactions reading/writing the same data can cause anomalies.

### Read Phenomena (Problems)

| Problem | Description |
|---|---|
| **Dirty Read** | Transaction reads uncommitted data from another transaction |
| **Non-Repeatable Read** | Same row gives different values when read twice in same tx |
| **Phantom Read** | Same query returns different rows when run twice in same tx |

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible | Fastest |
| `READ_COMMITTED` | Not possible | Possible | Possible | High |
| `REPEATABLE_READ` | Not possible | Not possible | Possible | Medium |
| `SERIALIZABLE` | Not possible | Not possible | Not possible | Slowest |

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public Patient getPatient(Long id) { ... }
```

> **PostgreSQL default:** `READ_COMMITTED`
> **MySQL default:** `REPEATABLE_READ`

Most web apps work fine with `READ_COMMITTED` (the default in Spring is `DEFAULT` which delegates to DB default).

---

## Section 10 — Optimistic vs Pessimistic Locking

When two transactions update the same row simultaneously, you get **lost updates**.

### Optimistic Locking (@Version)

Assume conflicts are rare. Don't lock the DB row. Instead, detect conflicts at commit time.

```java
@Entity
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;

    @Version  // Hibernate adds a 'version' column to the table
    private int version;
}
```

**How it works:**
1. Transaction A reads Product (version=1)
2. Transaction B reads same Product (version=1)
3. Transaction A updates → version becomes 2 → committed
4. Transaction B tries to update (expects version=1) → Hibernate generates:
   `UPDATE product SET ... WHERE id=? AND version=1` → 0 rows updated → throws `OptimisticLockException`

> Use when conflicts are rare (read-heavy systems). No DB-level lock held.

### Pessimistic Locking

Assume conflicts will happen. Lock the DB row as soon as it's read.

```java
@Query("SELECT p FROM Patient p WHERE p.id = :id")
@Lock(LockModeType.PESSIMISTIC_WRITE)  // Hibernate: SELECT ... FOR UPDATE
Patient findByIdForUpdate(@Param("id") Long id);
```

**Lock modes:**
- `PESSIMISTIC_READ` → `SELECT ... FOR SHARE` (other readers allowed, no writers)
- `PESSIMISTIC_WRITE` → `SELECT ... FOR UPDATE` (exclusive lock — no reads or writes by others)

> Use when conflicts are frequent (write-heavy systems). Blocks other transactions. Risk of deadlock.

---

## Section 11 — Hibernate First and Second Level Cache

### First Level Cache (L1)

- **Scope:** One transaction/session (PersistenceContext)
- **Automatic:** Always ON, cannot be disabled
- Covered in Ch06 of course notes

### Second Level Cache (L2)

- **Scope:** Application-wide (shared across all sessions)
- **Optional:** Must be explicitly configured
- Stores entities that are frequently read and rarely changed

```xml
<!-- pom.xml: add cache provider -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```properties
# application.properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=jcache
spring.cache.type=caffeine
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // mark entity for L2 caching
public class Country {
    @Id private Long id;
    private String name;
}
```

### Query Cache

Caches the results of specific JPQL/named queries.

```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

```java
@Query("SELECT c FROM Country c")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Country> findAllCountries();
```

### Cache Strategies

| Strategy | Read | Write | Use Case |
|---|---|---|---|
| `READ_ONLY` | Yes | No | Immutable data (country codes, currencies) |
| `READ_WRITE` | Yes | Yes (serialised) | Normal mutable data |
| `NONSTRICT_READ_WRITE` | Yes | Yes (no locking) | Rarely updated data |
| `TRANSACTIONAL` | Yes | Yes (JTA) | XA transaction environments |

---

## Section 12 — Custom Validators

When built-in annotations (`@NotBlank`, `@Email`) aren't enough.

### Example: Validate phone number format

**Step 1: Create annotation**

```java
@Documented
@Constraint(validatedBy = PhoneNumberValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhone {
    String message() default "Invalid phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**Step 2: Create validator**

```java
public class PhoneNumberValidator implements ConstraintValidator<ValidPhone, String> {

    @Override
    public boolean isValid(String phone, ConstraintValidatorContext context) {
        if (phone == null) return true; // let @NotNull handle nulls
        return phone.matches("^[+]?[0-9]{10,15}$");
    }
}
```

**Step 3: Use it on DTO**

```java
public class AddPatientDto {
    @NotBlank
    private String name;

    @ValidPhone
    private String phone;
}
```

---

## Section 13 — Testing in Spring Boot

### Testing Strategy

| Layer | Test Type | Annotation | What gets loaded |
|---|---|---|---|
| Unit test (no Spring) | Pure JUnit | None (just JUnit) | Nothing — pure Java |
| Repository layer | Slice test | `@DataJpaTest` | JPA context + in-memory H2 |
| Controller layer | Slice test | `@WebMvcTest` | Spring MVC layer only |
| Full app | Integration | `@SpringBootTest` | Entire Spring context |

### Unit Test (Service) with Mockito

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)   // enables Mockito
class StudentServiceTest {

    @Mock
    private StudentRepository studentRepository;   // creates a mock

    @InjectMocks
    private StudentServiceImpl studentService;     // injects mocks

    @Test
    void getAllStudents_shouldReturnList() {
        // Arrange
        when(studentRepository.findAll())
            .thenReturn(List.of(new Student(1L, "Anuj", "anuj@gmail.com")));

        // Act
        List<StudentDto> result = studentService.getAllStudents();

        // Assert
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("Anuj");
        verify(studentRepository, times(1)).findAll();
    }
}
```

### Controller Test with @WebMvcTest

```java
@WebMvcTest(StudentController.class)  // loads ONLY the web layer
class StudentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // creates a Spring-managed mock
    private StudentService studentService;

    @Test
    void getStudent_shouldReturn200() throws Exception {
        when(studentService.getStudentById(1L))
            .thenReturn(new StudentDto(1L, "Anuj", "anuj@gmail.com"));

        mockMvc.perform(get("/api/students/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Anuj"));
    }
}
```

### Repository Test with @DataJpaTest

```java
@DataJpaTest  // uses in-memory H2 DB; rolls back after each test
class PatientRepositoryTest {

    @Autowired
    private PatientRepository patientRepository;

    @Test
    void findByEmail_shouldReturnPatient() {
        Patient p = new Patient();
        p.setName("Anuj");
        p.setEmail("anuj@gmail.com");
        patientRepository.save(p);

        Optional<Patient> found = patientRepository.findByEmail("anuj@gmail.com");
        assertThat(found).isPresent();
    }
}
```

### Key Annotations Summary

| Annotation | What it does |
|---|---|
| `@SpringBootTest` | Loads entire application context |
| `@WebMvcTest(Ctrl.class)` | Loads only web layer (controllers, filters) |
| `@DataJpaTest` | Loads only JPA layer with in-memory DB |
| `@MockBean` | Creates Spring-managed mock bean (replaces real bean) |
| `@Mock` | Mockito mock (no Spring context) |
| `@InjectMocks` | Injects @Mock fields into target class |
| `@Spy` | Partial mock — real methods unless stubbed |

---

## Section 14 — Filter vs Interceptor vs AOP

These three serve similar purposes (pre/post processing) but intercept at different points.

### Comparison

| Aspect | Filter (Servlet) | Interceptor (Spring MVC) | AOP Aspect |
|---|---|---|---|
| Level | Servlet container level | Spring MVC level | Method level |
| Spring context access | No | Yes | Yes |
| Applies to | All requests (including static resources) | Only requests handled by DispatcherServlet | Only Spring beans |
| Config | `@Component` + `WebFilter` | `@Component` + `HandlerInterceptor` | `@Aspect` + `@Component` |
| Use case | Logging raw request/response, CORS, auth token extraction | Role-based access, locale, logging per-request | Method timing, transactions, logging business logic |

### Filter Example

```java
@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        System.out.println("Incoming: " + request.getMethod() + " " + request.getRequestURI());
        chain.doFilter(req, res);  // pass to next filter/servlet
        System.out.println("Outgoing response: " + ((HttpServletResponse) res).getStatus());
    }
}
```

### Interceptor Example

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null) {
            response.sendError(401, "Unauthorized");
            return false; // stop processing
        }
        return true; // continue
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/**");
    }
}
```

---

## Section 15 — Database Migration with Flyway

### Why not ddl-auto=update in production?

- `update` cannot safely rename columns
- `update` never drops columns even when removed from entity
- No history of schema changes
- No rollback mechanism

### Flyway — Migration Tool

Flyway runs SQL migration scripts in version order and tracks which ones have been executed.

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

```properties
# application.properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.jpa.hibernate.ddl-auto=validate  # Flyway manages schema; Hibernate only validates
```

### Naming Convention for Scripts

```
src/main/resources/db/migration/
├── V1__create_patient_table.sql
├── V2__add_blood_group_column.sql
├── V3__create_appointment_table.sql
```

`V{version}__{description}.sql` — double underscore between version and description.

### Example Migration

```sql
-- V1__create_patient_table.sql
CREATE TABLE patient (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    birth_date DATE,
    blood_group VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

-- V2__add_gender_column.sql
ALTER TABLE patient ADD COLUMN gender VARCHAR(10);
```

Flyway tracks executed migrations in a `flyway_schema_history` table. It will NEVER run the same migration twice.

---

## Section 16 — Spring Boot Actuator

Actuator exposes production-ready monitoring endpoints without writing any code.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,info,metrics,env
management.endpoint.health.show-details=always
```

### Key Endpoints

| Endpoint | URL | What it shows |
|---|---|---|
| `/actuator/health` | Health of app, DB, disk | `UP` or `DOWN` with details |
| `/actuator/info` | App info (from application.properties) | Name, version |
| `/actuator/metrics` | JVM, CPU, memory, HTTP metrics | Performance data |
| `/actuator/env` | All environment properties | Config values |
| `/actuator/beans` | All Spring beans | What's loaded |
| `/actuator/mappings` | All URL → handler mappings | Routes |

> In production: restrict access with Spring Security or expose only `/health`.

---

## Section 17 — Spring vs Spring Boot

### Common Interview Question: "Difference between Spring and Spring Boot?"

| Feature | Spring Framework | Spring Boot |
|---|---|---|
| Configuration | Manual XML or Java `@Configuration` | Auto-configuration |
| Server | External (install Tomcat/JBoss separately) | Embedded (Tomcat/Jetty included in JAR) |
| Dependencies | Choose and configure each individually | Starter dependencies bundle related dependencies |
| Learning curve | Steep (lots of XML / boilerplate) | Faster to start |
| Deployment | WAR to servlet container | JAR with `java -jar` |
| Opinions | None — you configure everything | Opinionated defaults with override capability |

**Short answer for interview:**
> Spring Boot is built on top of Spring Framework. It adds auto-configuration, embedded servers, and starter dependencies that reduce boilerplate and let you build production-ready apps faster. Spring Boot is NOT a replacement — it's a productivity layer on top of Spring.

---

## Section 18 — Top Interview Questions (Q&A Format)

**Q1. What is the difference between @Component, @Service, @Repository, @Controller?**

> All are specialisations of `@Component` — they all register the class as a Spring bean. The difference is semantic/functional:
> - `@Repository` adds exception translation (converts DB exceptions to Spring's `DataAccessException`).
> - `@Controller` enables Spring MVC request routing.
> - `@RestController` adds `@ResponseBody` to return JSON.
> - `@Service` has no extra behaviour — it's a convention marker for business logic.

---

**Q2. What is @Autowired and what are its modes?**

> `@Autowired` tells Spring to inject a dependency automatically. Three modes:
> - **Constructor injection** (preferred) — Spring calls constructor with required bean
> - **Field injection** — Spring sets the field directly via reflection
> - **Setter injection** — Spring calls a setter method

---

**Q3. What happens if two beans of same type exist and you @Autowire without qualifier?**

> Spring throws `NoUniqueBeanDefinitionException`. Resolve with:
> 1. `@Primary` on the preferred bean
> 2. `@Qualifier("beanName")` on the injection point
> 3. `@ConditionalOnProperty` to only load one conditionally

---

**Q4. What is @Transactional? When should you NOT use it?**

> `@Transactional` wraps a method in a database transaction. Don't use it on:
> - Read-only bulk operations that don't need atomicity (use `@Transactional(readOnly=true)`)
> - Methods that call other `@Transactional` methods (propagation handles this — understand the propagation type first)
> - Non-public methods — Spring's proxy-based AOP cannot intercept private methods

---

**Q5. Can @Transactional on a private method work?**

> No. Spring uses proxy-based AOP. When you call a `@Transactional` method on a bean, Spring calls through a proxy. Proxy can only intercept public methods. Private/package-private methods called from within the same class bypass the proxy.

---

**Q6. What is the difference between save() and saveAndFlush() in JPA?**

> - `save()` — marks entity for persistence; actual SQL runs at transaction commit (Hibernate batches)
> - `saveAndFlush()` — runs SQL immediately (flushes the persistence context); useful when you need the DB state right away in the same transaction

---

**Q7. What is lazy loading and when does it cause issues?**

> Lazy loading means related entities are fetched from DB only when accessed. Issues:
> - `LazyInitializationException` — accessing lazy field after session closes
> - N+1 problem — accessing lazy collection in a loop triggers one query per parent
> Fix: `@Transactional` on service methods, or `JOIN FETCH` in JPQL

---

**Q8. What is the difference between @Entity and @Table?**

> - `@Entity` — marks a Java class as a JPA entity (required for Hibernate to manage it)
> - `@Table` — optional; customises the table name, schema, unique constraints, and indexes

---

**Q9. How would you handle concurrent updates to the same record?**

> Use optimistic locking (`@Version`). Hibernate adds a version column. On update, it checks if the version matches. If another transaction already updated it, it throws `OptimisticLockException`. The client can retry.

---

**Q10. What is the N+1 problem? How do you fix it?**

> N+1 = 1 query to fetch parents + N individual queries for children (one per parent). Fix:
> 1. Use `JOIN FETCH` in JPQL to load in one query
> 2. Use DTO projections to select only needed fields
> 3. Keep collections LAZY; only load when explicitly needed

---

**Q11. What is the difference between CascadeType.REMOVE and orphanRemoval=true?**

> - `CascadeType.REMOVE` — children are deleted when the parent entity is explicitly deleted
> - `orphanRemoval=true` — children are deleted when they are removed from the parent's collection OR the reference is set to null (even if parent is not deleted)

---

**Q12. What are the types of advice in Spring AOP?**

> `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`. `@Around` is most powerful — it wraps the method and you decide whether to call `pjp.proceed()`.

---

**Q13. How does Spring Boot auto-configuration work?**

> Spring Boot ships with `spring-boot-autoconfigure.jar`. It has a file listing 150+ `@Configuration` classes. Each is annotated with `@Conditional` — it only activates if certain classes are on the classpath. E.g., `HibernateJpaAutoConfiguration` activates when JPA classes are present.

---

**Q14. What is the difference between @SpringBootTest, @WebMvcTest, @DataJpaTest?**

> - `@SpringBootTest` — loads the entire application context (integration test)
> - `@WebMvcTest` — loads only the web/MVC layer (controller tests, no DB)
> - `@DataJpaTest` — loads only the JPA/repository layer with H2 in-memory DB

---

**Q15. What is Spring Bean Scope and which is default?**

> Singleton (default) — one instance per ApplicationContext. Others: Prototype, Request, Session, Application.

---

**Q16. What is the role of DispatcherServlet?**

> Front controller of Spring MVC. All HTTP requests go through it. It:
> 1. Uses HandlerMapping to find the right controller method
> 2. Calls the method
> 3. Uses HttpMessageConverter to serialize response to JSON
> 4. Writes response back to client

---

**Q17. What is the difference between PUT and PATCH?**

> PUT replaces the entire resource (client sends full object). PATCH partially updates (client sends only changed fields). Use PATCH with `Map<String, Object>` input or a separate patch DTO.

---

**Q18. What is @Valid vs @Validated?**

> - `@Valid` — standard Jakarta annotation; triggers bean validation on the annotated parameter
> - `@Validated` — Spring's variant; additionally supports validation groups (you can activate different constraint sets)

---

**Q19. How does Spring Security work at a high level?**

> Spring Security uses a **filter chain** (series of servlet filters). Key filters:
> - `UsernamePasswordAuthenticationFilter` — intercepts login requests
> - `JwtAuthenticationFilter` (custom) — validates JWT on each request
> - `SecurityContextHolder` stores the authenticated user info for the duration of the request

---

**Q20. What is ACID in database transactions?**

> - **A**tomicity — all operations in a transaction succeed or all fail (rollback)
> - **C**onsistency — DB moves from one valid state to another
> - **I**solation — concurrent transactions don't see each other's intermediate state
> - **D**urability — committed transactions survive crashes (written to disk)

---

## Quick Reference — Critical Annotations

| Annotation | Package | Purpose |
|---|---|---|
| `@SpringBootApplication` | `org.springframework.boot` | Main class, enables scanning + auto-config |
| `@RestController` | `org.springframework.web.bind.annotation` | REST API controller |
| `@RequestMapping` | `org.springframework.web.bind.annotation` | Base URL for controller |
| `@GetMapping` / `@PostMapping` | `org.springframework.web.bind.annotation` | HTTP method mapping |
| `@PathVariable` | `org.springframework.web.bind.annotation` | URL path variable |
| `@RequestParam` | `org.springframework.web.bind.annotation` | Query string param |
| `@RequestBody` | `org.springframework.web.bind.annotation` | JSON body to object |
| `@Valid` | `jakarta.validation` | Activate bean validation |
| `@Component` | `org.springframework.stereotype` | Generic bean |
| `@Service` | `org.springframework.stereotype` | Service layer bean |
| `@Repository` | `org.springframework.stereotype` | DAO layer bean |
| `@Configuration` | `org.springframework.context.annotation` | Config class |
| `@Bean` | `org.springframework.context.annotation` | Manual bean method |
| `@Autowired` | `org.springframework.beans.factory.annotation` | DI injection |
| `@Qualifier` | `org.springframework.beans.factory.annotation` | Explicit bean selection |
| `@Primary` | `org.springframework.context.annotation` | Default bean when ambiguous |
| `@Transactional` | `org.springframework.transaction.annotation` | DB transaction |
| `@Entity` | `jakarta.persistence` | JPA entity |
| `@Id` | `jakarta.persistence` | Primary key |
| `@GeneratedValue` | `jakarta.persistence` | PK generation strategy |
| `@Column` | `jakarta.persistence` | Column config |
| `@Table` | `jakarta.persistence` | Table config |
| `@OneToMany` | `jakarta.persistence` | 1:N relationship |
| `@ManyToOne` | `jakarta.persistence` | N:1 relationship |
| `@ManyToMany` | `jakarta.persistence` | N:N relationship |
| `@JoinColumn` | `jakarta.persistence` | FK column config |
| `@JoinTable` | `jakarta.persistence` | Join table config (N:N) |
| `@Query` | `org.springframework.data.jpa.repository` | Custom JPQL |
| `@Modifying` | `org.springframework.data.jpa.repository` | UPDATE/DELETE query |
| `@Param` | `org.springframework.data.repository.query` | Named param binding |
| `@ControllerAdvice` | `org.springframework.web.bind.annotation` | Global exception handler |
| `@ExceptionHandler` | `org.springframework.web.bind.annotation` | Handle specific exception |
| `@Aspect` | `org.aspectj.lang.annotation` | AOP aspect class |
| `@Before` / `@Around` | `org.aspectj.lang.annotation` | AOP advice |
| `@Profile` | `org.springframework.context.annotation` | Profile-specific bean |
| `@PostConstruct` | `jakarta.annotation` | Post-DI init method |
| `@PreDestroy` | `jakarta.annotation` | Pre-shutdown cleanup |
| `@Scope` | `org.springframework.context.annotation` | Bean scope |
| `@Version` | `jakarta.persistence` | Optimistic locking |
| `@SpringBootTest` | `org.springframework.boot.test.context` | Integration test |
| `@WebMvcTest` | `org.springframework.boot.test.autoconfigure.web.servlet` | Controller test |
| `@DataJpaTest` | `org.springframework.boot.test.autoconfigure.orm.jpa` | Repository test |
| `@MockBean` | `org.springframework.boot.test.mock.mockito` | Spring mock bean |
