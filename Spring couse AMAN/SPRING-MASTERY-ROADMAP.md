````markdown
# Spring Mastery Roadmap — Beyond the Course & Interview Supplement
**Purpose:** The course (Ch01-10) teaches core Spring Boot + JPA. `INTERVIEW-SUPPLEMENT.md` fills
common interview gaps (AOP, Bean Scopes, Locking, Testing, etc). **This file covers everything
still missing to call yourself a genuine Spring "master"** — the topics that separate a mid-level
Spring Boot developer from someone who can design, secure, scale, and operate real production systems.

---

## GAP MAP — What's covered where

| Domain | Course | Supplement | This File |
|---|---|---|---|
| Core Spring/DI/JPA/MVC | ✅ Ch01-10 | — | — |
| Bean scopes, AOP, Locking, Testing, Actuator basics | — | ✅ | — |
| **Spring Security (Auth/AuthZ, JWT, OAuth2)** | ❌ | ❌ | ✅ Section 1 |
| **Microservices & Spring Cloud** | ❌ | ❌ | ✅ Section 2 |
| **Reactive Programming (WebFlux)** | ❌ | ❌ | ✅ Section 3 |
| **Messaging (Kafka/RabbitMQ)** | ❌ | ❌ | ✅ Section 4 |
| **Scheduling & Async** | ❌ | ❌ | ✅ Section 5 |
| **JPA Auditing** | ❌ | ❌ | ✅ Section 6 |
| **API Documentation (OpenAPI/Swagger)** | ❌ | ❌ | ✅ Section 7 |
| **Design Patterns Spring uses/enables** | ❌ | ❌ | ✅ Section 8 |
| **Practical Caching (@Cacheable + Redis)** | ❌ | Partial (theory) | ✅ Section 9 |
| **CORS Deep Dive** | ❌ | ❌ | ✅ Section 10 |
| **Rate Limiting** | ❌ | ❌ | ✅ Section 11 |
| **HATEOAS** | ❌ | ❌ | ✅ Section 12 |
| **Logging & Observability (tracing, metrics)** | ❌ | ❌ | ✅ Section 13 |
| **Containerization (Docker)** | ❌ | ❌ | ✅ Section 14 |
| **CI/CD Pipelines** | ❌ | ❌ | ✅ Section 15 |
| **System Design with Spring Boot** | ❌ | ❌ | ✅ Section 16 |
| **Concurrency & Virtual Threads** | ❌ | ❌ | ✅ Section 17 |
| **Testcontainers (real-DB integration tests)** | ❌ | Partial (H2 only) | ✅ Section 18 |
| **Java 21 / Spring Boot 3 modern features** | ❌ | ❌ | ✅ Section 19 |
| **GraphQL with Spring** | ❌ | ❌ | ✅ Section 20 |
| Final battle-readiness checklist + study plan | ❌ | ❌ | ✅ Section 21 |

---

## Section 1 — Spring Security

Spring Security is asked in almost every mid/senior Spring interview. Skipping it is the single
biggest gap in the current notes.

### 1.1 Core Architecture — The Filter Chain

Spring Security is fundamentally a chain of **Servlet Filters** that run BEFORE `DispatcherServlet`.

```
Request → SecurityFilterChain (many filters) → DispatcherServlet → Controller
```

Key filters (order matters):

| Filter | Job |
|---|---|
| `UsernamePasswordAuthenticationFilter` | Handles form-login POST, extracts credentials |
| `BasicAuthenticationFilter` | Handles HTTP Basic auth header |
| `BearerTokenAuthenticationFilter` | Handles `Authorization: Bearer <jwt>` |
| `ExceptionTranslationFilter` | Converts security exceptions → 401/403 responses |
| `FilterSecurityInterceptor` / `AuthorizationFilter` | Final authorization decision (allow/deny) |

### 1.2 Minimal Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Adding this dependency ALONE locks down every endpoint with HTTP Basic auth and a generated
password (printed in the console log) — a classic "gotcha" interview question: *"I added Spring
Security and now all my APIs return 401 — why?"*

### 1.3 SecurityFilterChain Configuration (Spring Security 6 / Boot 3 style)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                       // disable for stateless APIs
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/students/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();   // NEVER store plain-text passwords
    }
}
```

> **Note:** `@EnableWebSecurity` + a `SecurityFilterChain` `@Bean` is the Spring Security 6 (Boot 3)
> style. The older `WebSecurityConfigurerAdapter` class-extension style is **deprecated/removed** —
> a common trap for candidates who learned Security from older tutorials.

### 1.4 JWT Authentication — The Full Flow

```
1. POST /api/auth/login {username, password}
2. AuthenticationManager validates credentials against UserDetailsService
3. On success: generate a signed JWT (contains userId, roles, expiry) and return it
4. Client stores JWT (localStorage/memory) and sends it as:
   Authorization: Bearer <jwt>  on every subsequent request
5. A custom JwtAuthenticationFilter (added via addFilterBefore) intercepts each request:
   - Extracts the token
   - Validates signature + expiry
   - Loads user details, builds an Authentication object
   - Sets it into SecurityContextHolder
6. Downstream: @PreAuthorize / requestMatchers can check roles from that Authentication
```

```java
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {

        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            if (jwtUtil.isValid(token)) {
                String username = jwtUtil.extractUsername(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(req, res);
    }
}
```

### 1.5 Method-Level Security

```java
@EnableMethodSecurity   // Spring Security 6 (replaces @EnableGlobalMethodSecurity)
@Configuration
public class MethodSecurityConfig {}

@Service
public class PatientService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deletePatient(Long id) { ... }

    @PreAuthorize("#patientId == authentication.principal.id or hasRole('ADMIN')")
    public Patient getPatient(Long patientId) { ... }   // user can only fetch their OWN record

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Document getDocument(Long id) { ... }        // check AFTER method runs, on the result
}
```

### 1.6 OAuth2 / OIDC (Login with Google/GitHub, or as a Resource Server)

**As an OAuth2 Client** (login via Google):
```properties
spring.security.oauth2.client.registration.google.client-id=xxx
spring.security.oauth2.client.registration.google.client-secret=xxx
spring.security.oauth2.client.registration.google.scope=openid,profile,email
```

**As a Resource Server** (validating JWTs issued by an external Identity Provider like Auth0/Keycloak):
```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://your-issuer.auth0.com/
```
```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```
No custom `JwtAuthFilter` needed here — Spring Security auto-validates tokens against the issuer's
public keys (JWKS endpoint).

### 1.7 Common Interview Questions

**Q: Why is CSRF protection disabled for REST APIs but NOT for server-rendered (Thymeleaf) apps?**
> CSRF exploits rely on the browser AUTOMATICALLY attaching cookies/session credentials to
> cross-site requests. Stateless JWT-based APIs (token sent explicitly in an `Authorization` header,
> not auto-attached by the browser) aren't vulnerable to CSRF the same way, so it's commonly
> disabled. Session-cookie-based apps (traditional server-rendered) remain vulnerable and need
> CSRF tokens enabled.

**Q: What's the difference between Authentication and Authorization?**
> Authentication = "who are you?" (verifying identity — login). Authorization = "what are you
> allowed to do?" (permission checks — roles/scopes on specific endpoints/methods).

**Q: Why use `BCryptPasswordEncoder` instead of MD5/SHA-256 for passwords?**
> BCrypt is deliberately SLOW and includes a built-in random SALT per hash, making brute-force and
> rainbow-table attacks impractical. MD5/SHA-256 are FAST general-purpose hashes — exactly the
> wrong property for password storage (fast = attacker can try billions of guesses/second).

---

## Section 2 — Microservices & Spring Cloud

### 2.1 Why Microservices (and the trade-offs)

| Monolith | Microservices |
|---|---|
| Single deployable unit | Many independently deployable services |
| Simple transactions (ACID) | Distributed transactions (Saga, eventual consistency) |
| Easy local debugging | Complex — needs distributed tracing |
| Scale the whole app | Scale individual services independently |
| One tech stack | Can mix languages/frameworks per service |

> **Interview trap:** "Should we always use microservices?" — No. Start with a well-structured
> monolith (a "modular monolith") and split into services only when a genuine scaling/team/deploy
> boundary problem justifies the added operational complexity.

### 2.2 Service Discovery — Eureka

In a dynamic environment (containers, auto-scaling), services don't have fixed IP addresses.
**Eureka** (Netflix, still widely used) lets services register themselves and discover each other by name.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
```properties
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
spring.application.name=patient-service
```

Calling another service by NAME instead of hardcoded host:port:
```java
@Bean
@LoadBalanced   // enables client-side load balancing across instances
public RestClient.Builder restClientBuilder() { return RestClient.builder(); }

// Calling "billing-service" by logical name — Eureka + LoadBalancer resolve the actual instance
restClient.get().uri("http://billing-service/api/invoices/{id}", id)...
```

> **Modern alternative:** Kubernetes' own built-in Service Discovery (DNS-based `Service` objects)
> often REPLACES Eureka entirely when deploying to K8s — a very common follow-up interview question.

### 2.3 API Gateway

A single entry point that routes requests to the right downstream service, and centralizes
cross-cutting concerns (auth, rate limiting, logging) instead of duplicating them in every service.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: patient-service
          uri: lb://patient-service
          predicates:
            - Path=/api/patients/**
        - id: billing-service
          uri: lb://billing-service
          predicates:
            - Path=/api/billing/**
```

### 2.4 Circuit Breaker — Resilience4j

When Service A calls Service B and B is slow/down, WITHOUT protection, A's threads pile up waiting
→ A also becomes unresponsive → **cascading failure** across the whole system.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```java
@CircuitBreaker(name = "billingService", fallbackMethod = "fallbackGetInvoice")
public Invoice getInvoice(Long patientId) {
    return billingClient.getInvoice(patientId);   // calls another microservice
}

public Invoice fallbackGetInvoice(Long patientId, Throwable t) {
    return Invoice.empty();   // graceful degradation instead of a hanging/failing request
}
```

**Circuit breaker states:**

| State | Behavior |
|---|---|
| `CLOSED` | Normal — requests pass through; failures are counted |
| `OPEN` | Failure threshold exceeded — requests immediately fail-fast (call the fallback) WITHOUT hitting the real service |
| `HALF_OPEN` | After a wait period, allows a few trial requests through to see if the downstream recovered |

Related resilience patterns often asked together: **Retry** (with exponential backoff),
**Rate Limiter**, **Bulkhead** (isolating thread pools per downstream dependency so one slow
dependency can't exhaust ALL threads).

### 2.5 Feign Client — Declarative HTTP Calls

```java
@FeignClient(name = "billing-service")
public interface BillingClient {
    @GetMapping("/api/billing/{patientId}")
    Invoice getInvoice(@PathVariable Long patientId);
}

// Usage — looks like a local method call, but it's a real HTTP request under the hood:
@Autowired private BillingClient billingClient;
Invoice invoice = billingClient.getInvoice(42L);
```

### 2.6 Distributed Configuration — Config Server

Instead of every microservice keeping its own `application.properties` (hard to update
consistently across 20 services), a **Config Server** centralizes configuration in a Git repo,
and services pull their config at startup (and optionally refresh at runtime via `/actuator/refresh`).

### 2.7 Saga Pattern — Distributed Transactions

Since a single ACID transaction can't span multiple databases/services, the **Saga pattern**
breaks a business transaction into a sequence of local transactions, each publishing an event
that triggers the next step — with explicit **compensating transactions** to undo prior steps if
a later one fails.

```
OrderService: create order (PENDING) → publish OrderCreatedEvent
PaymentService: charge card → publish PaymentSucceededEvent (or PaymentFailedEvent)
  if failed → OrderService compensates: cancel order (compensating transaction)
InventoryService: reserve stock → publish StockReservedEvent (or StockUnavailableEvent)
  if failed → PaymentService compensates: refund payment; OrderService compensates: cancel order
```

Two flavors: **Choreography** (services react to each other's events, no central coordinator) vs
**Orchestration** (a central saga orchestrator explicitly calls each step and handles failures).

---

## Section 3 — Reactive Programming (Spring WebFlux)

### 3.1 Why Reactive?

Traditional Spring MVC is **thread-per-request, blocking**: one thread is occupied for the ENTIRE
duration of a request, including waiting on slow I/O (DB, external APIs). Under high concurrency,
you need huge thread pools, each consuming memory (~1MB stack per thread).

**Spring WebFlux** is **non-blocking/reactive**, built on Project Reactor: a small number of
event-loop threads handle THOUSANDS of concurrent requests by never blocking on I/O — when a
request is waiting on the DB, the thread is freed to handle other requests.

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Model | Thread-per-request, blocking | Event-loop, non-blocking |
| Server | Tomcat/Jetty (Servlet) | Netty (default) |
| Return types | `T`, `List<T>`, `ResponseEntity<T>` | `Mono<T>` (0-1 result), `Flux<T>` (0-N results) |
| Best for | CPU-bound work, simpler mental model, most CRUD apps | High-concurrency I/O-bound workloads (streaming, many downstream calls) |
| Learning curve | Lower | Higher (functional/operator-chain style, harder debugging) |

### 3.2 Mono and Flux Basics

```java
@RestController
public class ReactivePatientController {

    @GetMapping("/patients/{id}")
    public Mono<PatientDto> getPatient(@PathVariable Long id) {
        return patientRepository.findById(id)          // ReactiveCrudRepository
                .map(p -> modelMapper.map(p, PatientDto.class))
                .switchIfEmpty(Mono.error(new NotFoundException()));
    }

    @GetMapping("/patients")
    public Flux<PatientDto> getAllPatients() {
        return patientRepository.findAll()
                .map(p -> modelMapper.map(p, PatientDto.class));
    }
}
```

> **Critical rule:** Never call `.block()` inside a reactive pipeline in production code — it
> defeats the ENTIRE purpose of non-blocking I/O by tying up the (scarce) event-loop thread. This
> is the #1 practical mistake made by teams migrating from MVC to WebFlux.

### 3.3 WebClient vs RestTemplate

`RestTemplate` is now in maintenance mode. `WebClient` is the modern replacement — non-blocking,
usable from BOTH WebFlux and traditional MVC apps (you can call `.block()` in an MVC app since
MVC is already blocking anyway).

```java
WebClient client = WebClient.builder().baseUrl("http://billing-service").build();

Mono<Invoice> invoice = client.get()
    .uri("/api/billing/{id}", patientId)
    .retrieve()
    .bodyToMono(Invoice.class);
```

### 3.4 R2DBC — Reactive Database Access

JDBC is fundamentally BLOCKING (the driver blocks the thread waiting on the DB). For a truly
non-blocking pipeline end-to-end, you need **R2DBC** (Reactive Relational Database Connectivity)
instead of JPA/JDBC — `ReactiveCrudRepository` instead of `JpaRepository`. Trade-off: R2DBC lacks
many mature JPA features (no automatic dirty checking, limited cascading), so it's a deliberate
architectural choice, not a drop-in replacement.

### 3.5 When to Actually Use WebFlux (Interview Answer)

> Use WebFlux for I/O-heavy services making MANY concurrent downstream calls (API gateways,
> aggregation services, streaming/SSE endpoints) where thread-per-request would need an
> impractically large thread pool. For typical CRUD APIs backed by a relational DB with moderate
> concurrency, Spring MVC remains simpler to write, debug, and reason about — reactive adds real
> complexity (harder stack traces, operator chains, mandatory non-blocking libraries everywhere in
> the call chain) that must be justified by an actual concurrency bottleneck, not "reactive sounds
> more modern."

---

## Section 4 — Messaging & Event-Driven Architecture

### 4.1 Why Asynchronous Messaging?

Synchronous REST calls between services create tight coupling (both services must be up
simultaneously) and cascading latency (A waits for B waits for C). **Message brokers** decouple
producers and consumers — the producer publishes an event and moves on; consumers process it
independently, at their own pace, even if temporarily offline (message stays queued).

### 4.2 Apache Kafka with Spring

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

**Producer:**
```java
@Service
@RequiredArgsConstructor
public class AppointmentEventPublisher {
    private final KafkaTemplate<String, AppointmentCreatedEvent> kafkaTemplate;

    public void publish(AppointmentCreatedEvent event) {
        kafkaTemplate.send("appointment-events", event.getPatientId().toString(), event);
    }
}
```

**Consumer:**
```java
@Component
public class NotificationConsumer {

    @KafkaListener(topics = "appointment-events", groupId = "notification-service")
    public void handle(AppointmentCreatedEvent event) {
        // send SMS/email — runs independently of the producing service
    }
}
```

### 4.3 Key Kafka Concepts (Frequently Asked)

| Concept | Meaning |
|---|---|
| **Topic** | A named stream of events (e.g., `appointment-events`) |
| **Partition** | A topic is split into partitions for parallelism; ORDER is guaranteed only WITHIN a partition |
| **Consumer Group** | A set of consumers sharing the work of a topic — each partition is consumed by exactly ONE consumer within a group |
| **Offset** | A consumer's position/checkpoint in a partition — enables replay/resume |
| **Key** | Determines WHICH partition a message goes to (same key → same partition → guaranteed order for that key) |

**Q: Why send `patientId` as the Kafka message KEY in the example above?**
> All events for the SAME patient land in the SAME partition, guaranteeing they're processed IN
> ORDER relative to each other (Kafka only guarantees ordering within a partition, not across the
> whole topic).

### 4.4 The Outbox Pattern — Solving the Dual-Write Problem

**Problem:** If a service both (1) saves to its DB AND (2) publishes a Kafka event as two SEPARATE
operations, a crash between them causes inconsistency (DB committed but event never published, or
vice versa).

**Solution — Transactional Outbox:** Within the SAME DB transaction as the business write, also
insert a row into an `outbox` table describing the event. A separate poller/CDC process (e.g.,
Debezium) reads the outbox table and reliably publishes those events to Kafka, then marks them
sent. Since the business write and the outbox insert are in ONE atomic DB transaction, they can
never be inconsistent with each other.

### 4.5 RabbitMQ (Quick Comparison to Kafka)

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed commit log (pull-based, retains history) | Traditional message queue (push-based) |
| Best for | High-throughput event streaming, replay, analytics pipelines | Task queues, complex routing (exchanges), RPC-style messaging |
| Message retention | Configurable, can replay old messages | Typically deleted once consumed/acknowledged |
| Ordering | Per-partition | Per-queue (with caveats under multiple consumers) |

---

## Section 5 — Scheduling & Asynchronous Execution

### 5.1 @Scheduled — Cron Jobs Inside Spring Boot

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {}

@Component
public class ReportJob {

    @Scheduled(cron = "0 0 2 * * *")           // every day at 2 AM
    public void generateDailyReport() { ... }

    @Scheduled(fixedRate = 60000)              // every 60 seconds, from start of previous execution
    public void pollExternalApi() { ... }

    @Scheduled(fixedDelay = 60000)             // 60 seconds AFTER previous execution finishes
    public void cleanupTempFiles() { ... }
}
```

> **Interview trap:** By default, ALL `@Scheduled` methods share a SINGLE thread — if one task
> runs long, it delays every other scheduled task. Fix: configure a dedicated `TaskScheduler` bean
> with a thread pool (`ThreadPoolTaskScheduler`).

### 5.2 @Async — Fire-and-Forget / Background Execution

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class EmailService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendWelcomeEmail(String to) {
        // runs on a background thread; caller doesn't wait
        return CompletableFuture.completedFuture(null);
    }
}
```

> **Same self-invocation trap as `@Transactional`:** `@Async` is AOP-proxy-based too — calling an
> `@Async` method from WITHIN the same class (`this.sendWelcomeEmail(...)`) bypasses the proxy and
> runs SYNCHRONOUSLY on the caller's thread, silently defeating the async behavior.

### 5.3 CompletableFuture for Parallel Composition

```java
public CompletableFuture<PatientFullProfile> getFullProfile(Long id) {
    CompletableFuture<Patient> patientFuture = CompletableFuture.supplyAsync(() -> patientService.get(id));
    CompletableFuture<List<Appointment>> apptFuture = CompletableFuture.supplyAsync(() -> apptService.getFor(id));

    return patientFuture.thenCombine(apptFuture, (patient, appts) ->
        new PatientFullProfile(patient, appts));   // both calls run IN PARALLEL, combined when both finish
}
```

---

## Section 6 — Spring Data JPA Auditing

Automatically track WHO created/modified a record and WHEN — essential for almost every real
production entity (compliance, debugging, "who changed this?").

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                .map(SecurityContext::getAuthentication)
                .map(Authentication::getName);   // pulls current logged-in username
    }
}

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass   // fields inherited by entities, no separate table
public abstract class Auditable {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

@Entity
public class Patient extends Auditable {
    // id, name, etc. — createdAt/updatedAt/createdBy/updatedBy are automatic
}
```

**Q: Why `@MappedSuperclass` instead of just putting these 4 fields directly on every entity?**
> DRY — avoids repeating the same 4 fields (and their annotations) on every single entity.
> `@MappedSuperclass` is NOT itself an `@Entity` (no separate table); its fields are simply
> inherited into each subclass's own table as regular columns.

---

## Section 7 — API Documentation (OpenAPI/Swagger)

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

Adding this ALONE auto-generates interactive API docs at `http://localhost:8080/swagger-ui.html`
by scanning your `@RestController`s, `@RequestMapping`s, and DTOs — zero manual YAML needed for a
baseline.

```java
@Operation(summary = "Get a patient by ID", description = "Returns patient details for a given ID")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Found"),
    @ApiResponse(responseCode = "404", description = "Patient not found")
})
@GetMapping("/patients/{id}")
public ResponseEntity<PatientDto> getPatient(
        @Parameter(description = "Patient ID") @PathVariable Long id) { ... }
```

```java
public class PatientDto {
    @Schema(description = "Unique patient identifier", example = "42")
    private Long id;

    @Schema(description = "Full name", example = "Anuj Sharma")
    private String name;
}
```

> **Interview angle:** OpenAPI specs (the generated `openapi.json`) aren't just documentation —
> they're machine-readable CONTRACTS used to auto-generate client SDKs, run contract tests
> (consumer-driven contracts), and drive API gateway configuration.

---

## Section 8 — Design Patterns Spring Uses (and That You Should Recognize)

Interviewers love asking "what design patterns does Spring use internally?" — recognizing these
demonstrates you understand the framework, not just its annotations.

| Pattern | Where Spring Uses It |
|---|---|
| **Singleton** | Default bean scope — one shared instance per `ApplicationContext` |
| **Factory Method** | `BeanFactory`/`ApplicationContext` — beans are created via factory methods, not `new` |
| **Proxy** | AOP (`@Transactional`, `@Async`, `@Cacheable` all work via JDK/CGLIB proxies wrapping your bean) |
| **Template Method** | `JdbcTemplate`, `RestTemplate`, `TransactionTemplate` — fixed skeleton algorithm, you fill in the specific step (callback) |
| **Strategy** | `PlatformTransactionManager` has multiple implementations (JPA, JDBC, JTA) selected at runtime; `PasswordEncoder` implementations |
| **Observer** | `ApplicationEventPublisher` + `@EventListener` — publish-subscribe within the app context |
| **Chain of Responsibility** | The Servlet `Filter` chain, and Spring Security's `SecurityFilterChain` |
| **Decorator** | `HandlerInterceptor`s wrapping controller invocation; `BeanPostProcessor`s wrapping bean creation |
| **Builder** | Lombok's `@Builder`, `UriComponentsBuilder`, `ResponseEntity.BodyBuilder` |
| **Dependency Injection (a pattern in its own right)** | The entire IoC container |

### Application Events (Observer Pattern) — Practical Example

```java
public class PatientRegisteredEvent extends ApplicationEvent {
    private final Patient patient;
    public PatientRegisteredEvent(Object source, Patient patient) {
        super(source);
        this.patient = patient;
    }
}

@Service
public class PatientService {
    @Autowired private ApplicationEventPublisher publisher;

    public Patient register(Patient patient) {
        Patient saved = patientRepository.save(patient);
        publisher.publishEvent(new PatientRegisteredEvent(this, saved));  // fire and forget
        return saved;
    }
}

@Component
public class WelcomeEmailListener {
    @EventListener
    @Async   // combine with async to NOT block the original request
    public void onPatientRegistered(PatientRegisteredEvent event) {
        // send welcome email — fully decoupled from PatientService
    }
}
```

> This is a lightweight, IN-PROCESS alternative to Kafka/RabbitMQ — use it when decoupling is only
> needed WITHIN a single application, not across services.

---

## Section 9 — Practical Caching (@Cacheable + Redis)

The Interview Supplement covered Hibernate's L2 cache (JPA-specific). This section covers Spring's
GENERAL-PURPOSE caching abstraction, usable for ANY method, not just entity lookups.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```java
@Configuration
@EnableCaching
public class CacheConfig {}

@Service
public class PatientService {

    @Cacheable(value = "patients", key = "#id")
    public PatientDto getPatientById(Long id) {
        System.out.println("Hitting the DB..."); // only prints on a cache MISS
        return modelMapper.map(patientRepository.findById(id).orElseThrow(), PatientDto.class);
    }

    @CachePut(value = "patients", key = "#result.id")
    public PatientDto updatePatient(Long id, UpdateDto dto) {
        // always executes the method AND updates the cache with the fresh result
        ...
    }

    @CacheEvict(value = "patients", key = "#id")
    public void deletePatient(Long id) {
        // removes the stale entry from cache
        patientRepository.deleteById(id);
    }

    @CacheEvict(value = "patients", allEntries = true)
    public void clearAllPatientCache() { ... }
}
```

```properties
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.redis.time-to-live=600000   # 10 minutes TTL
```

**Q: What's the classic bug with `@Cacheable` when the method THROWS an exception?**
> By default, exceptions are NOT cached — every failing call still hits the real method. This is
> usually desired, but for expensive "known failure" lookups you may want `sync = true` (prevents a
> cache-stampede — many concurrent requests all missing the cache simultaneously and hammering the
> DB at once) or a custom `CacheErrorHandler`.

**Q: Why Redis over the in-memory Caffeine cache discussed in the Interview Supplement?**
> Caffeine is per-JVM-instance (each app instance has its OWN cache — inconsistent across a
> multi-instance deployment, and lost on restart). Redis is a SHARED, external cache — all
> instances see the same cached data, survives individual app restarts, and can be sized/scaled
> independently of the app.

---

## Section 10 — CORS (Cross-Origin Resource Sharing) Deep Dive

Browsers block JavaScript from calling an API on a DIFFERENT origin (domain/port/protocol) than
the page itself, UNLESS the server explicitly allows it via CORS headers.

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://myfrontend.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);   // browser caches the preflight response for 1 hour
    }
}
```

**Q: What is a CORS "preflight" request, and when does the browser send one?**
> For "non-simple" requests (e.g., `Content-Type: application/json`, custom headers, or methods
> other than GET/HEAD/POST-with-simple-content-type), the browser AUTOMATICALLY sends an `OPTIONS`
> request FIRST, asking the server "are you OK with this actual request?" (checking
> `Access-Control-Allow-Origin`/`-Methods`/`-Headers`). Only if the server approves does the browser
> send the REAL request. This is entirely a browser-enforced security mechanism — server-to-server
> calls (Postman, another backend) are NEVER subject to CORS at all.

**Q: `allowedOrigins("*")` combined with `allowCredentials(true)` — what happens?**
> This is explicitly REJECTED by browsers/the CORS spec (a security safeguard) — you cannot use a
> wildcard origin together with credentials (cookies/auth headers). You must list explicit allowed
> origins if credentials are involved.

---

## Section 11 — Rate Limiting

Protects your API from abuse/overload by capping how many requests a client can make in a time window.

### Application-Level (Bucket4j)

```java
Bucket bucket = Bucket.builder()
    .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
    .build();   // 100 requests per minute, refilled all at once each interval

@Component
public class RateLimitFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        if (bucket.tryConsume(1)) {
            chain.doFilter(req, res);
        } else {
            res.setStatus(429); // Too Many Requests
        }
    }
}
```

### Algorithms (Common Interview Topic)

| Algorithm | How it Works | Trade-off |
|---|---|---|
| **Fixed Window** | Count requests in a fixed clock window (e.g., per-minute) | Simple but allows bursts at window boundaries (2x limit possible right at the edge) |
| **Sliding Window** | Smooths the fixed-window boundary issue by weighting the previous window | More accurate, slightly more complex |
| **Token Bucket** | Tokens refill at a steady rate; each request consumes a token; allows controlled bursts up to bucket capacity | Most widely used (AWS, Stripe APIs) — smooth AND burst-tolerant |
| **Leaky Bucket** | Requests queue and are processed at a constant output rate | Smooths bursts completely but adds latency |

> In production, rate limiting is usually done at the **API Gateway** layer (Section 2.3) rather
> than in each individual service — centralizing the policy instead of duplicating it everywhere.

---

## Section 12 — HATEOAS

**H**ypermedia **A**s **T**he **E**ngine **O**f **A**pplication **S**tate — REST responses include
links to related actions/resources, so clients don't need to hardcode URL structures.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@GetMapping("/patients/{id}")
public EntityModel<PatientDto> getPatient(@PathVariable Long id) {
    PatientDto dto = patientService.getById(id);
    return EntityModel.of(dto,
        linkTo(methodOn(PatientController.class).getPatient(id)).withSelfRel(),
        linkTo(methodOn(AppointmentController.class).getForPatient(id)).withRel("appointments"));
}
```
```json
{
  "id": 42, "name": "Anuj",
  "_links": {
    "self": { "href": "/api/patients/42" },
    "appointments": { "href": "/api/patients/42/appointments" }
  }
}
```

> **Reality check for interviews:** HATEOAS is asked about conceptually far more often than it's
> actually implemented in real industry APIs (most REST APIs are NOT truly HATEOAS-compliant — they
> use documented, hardcoded URL conventions instead). Know the concept and be able to discuss WHY
> it's theoretically valuable (self-describing, evolvable APIs) even if you rarely deploy it.

---

## Section 13 — Logging & Observability

### 13.1 SLF4J + Logback Basics

```java
@Slf4j   // Lombok — generates: private static final Logger log = LoggerFactory.getLogger(ThisClass.class);
@Service
public class PatientService {
    public Patient getById(Long id) {
        log.info("Fetching patient with id={}", id);     // use {} placeholders, NOT string concatenation
        try {
            return patientRepository.findById(id).orElseThrow();
        } catch (Exception e) {
            log.error("Failed to fetch patient id={}", id, e);   // pass exception as LAST arg for stack trace
            throw e;
        }
    }
}
```

**Q: Why `log.info("id={}", id)` instead of `log.info("id=" + id)`?**
> String concatenation ALWAYS builds the string, even if the log level is disabled (e.g., DEBUG
> logs suppressed in production but the concatenation still runs, wasting CPU). The `{}`
> placeholder form only formats the string if the log level is actually enabled — a real
> performance difference at scale.

### 13.2 MDC (Mapped Diagnostic Context) — Correlating Logs Per Request

```java
@Component
public class RequestIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);   // attached to EVERY log line on this thread until cleared
        try {
            res.setHeader("X-Request-Id", requestId);
            chain.doFilter(req, res);
        } finally {
            MDC.clear();   // MUST clear — thread pools reuse threads across requests!
        }
    }
}
```
```xml
<!-- logback pattern including the MDC value -->
<pattern>%d{HH:mm:ss} [%X{requestId}] %-5level %logger{36} - %msg%n</pattern>
```

This lets you `grep` ALL log lines for one specific request across an entire distributed system
(if the `requestId` is propagated in outgoing HTTP headers to downstream services too).

### 13.3 Distributed Tracing (Micrometer Tracing + Zipkin/OpenTelemetry)

In microservices, ONE user request can span 5+ services. **Distributed tracing** assigns a single
`traceId` to the whole request (propagated across service calls) and a unique `spanId` per
hop/operation, visualized as a timeline/waterfall in a tool like Zipkin or Jaeger.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```
```properties
management.tracing.sampling.probability=1.0   # trace 100% of requests (lower in production for volume)
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```

Spring Boot 3's **Micrometer Tracing** (replacing the older Spring Cloud Sleuth) auto-instruments
`RestTemplate`/`WebClient` calls, DB queries, and `@KafkaListener`s with almost zero manual code.

### 13.4 Metrics (Micrometer + Prometheus + Grafana)

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
```properties
management.endpoints.web.exposure.include=prometheus,health,metrics
```

Exposes `/actuator/prometheus` in a format Prometheus scrapes on a schedule; Grafana then
visualizes dashboards (request rate, p95/p99 latency, error rate, JVM heap/GC) built on top of
that time-series data — the "three pillars of observability" (**logs, metrics, traces**) are
Sections 13.1-13.4 together.

```java
@Timed(value = "patient.lookup.time", description = "Time to fetch a patient")
public Patient getById(Long id) { ... }

// Custom counter
@Autowired MeterRegistry registry;
Counter.builder("patient.registrations").register(registry).increment();
```

---

## Section 14 — Containerization (Docker)

### 14.1 Dockerfile for a Spring Boot App (Multi-Stage Build)

```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests

# Stage 2: run (much smaller final image — no build tools/source code)
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Q: Why multi-stage builds?** The FINAL image only contains the JRE + your fat JAR — none of
> the Maven build tooling, source code, or intermediate artifacts, drastically reducing image size
> and attack surface.

### 14.2 docker-compose for Local Dev (App + DB Together)

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/hospital_db
    depends_on: [db]

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: hospital_db
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

volumes:
  pgdata:
```

> **Q: Why `db` as the hostname in `SPRING_DATASOURCE_URL`, not `localhost`?** Inside Docker's
> internal network, each service is reachable by its docker-compose SERVICE NAME (`db`) rather than
> `localhost` — `localhost` inside the `app` container refers to the container itself, not the
> sibling `db` container.

### 14.3 Buildpacks — Spring Boot's Built-in Docker Image Builder

```bash
./mvnw spring-boot:build-image
```
Spring Boot's Maven/Gradle plugin can build an OPTIMIZED container image directly (using Cloud
Native Buildpacks) WITHOUT writing a Dockerfile at all — automatically layering dependencies vs
application code for better Docker layer caching (dependencies change rarely, so that layer is
cached across builds; only your changed application code layer gets rebuilt).

---

## Section 15 — CI/CD Basics (GitHub Actions Example)

```yaml
# .github/workflows/build.yml
name: Build and Test
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Build and Test
        run: ./mvnw clean verify
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u myuser --password-stdin
          docker push myapp:${{ github.sha }}
```

**Core CI/CD concepts to know for interviews:**
- **CI (Continuous Integration):** every push/PR automatically builds + runs tests, catching
  breakage early, BEFORE merge.
- **CD (Continuous Delivery/Deployment):** automatically deploy a passing build to
  staging/production — "Delivery" = deploy is a manual click; "Deployment" = fully automatic.
- **Pipeline stages:** typically build → unit test → integration test (often with Testcontainers,
  Section 18) → static analysis/security scan (SonarQube, Snyk) → build image → deploy.
- **Blue-Green / Canary Deployments:** strategies to roll out new versions with zero downtime and
  limited blast radius if something's wrong (canary: route 5% of traffic to the new version first).

---

## Section 16 — System Design Questions Using Spring Boot

Interviewers at "good companies" often combine Spring Boot knowledge with a light system-design
question. Some patterns to be ready for:

### 16.1 "Design a URL Shortener"

- `POST /shorten {longUrl}` → generate a short code (Base62-encoded auto-increment ID, or a hash)
  → `@Entity UrlMapping(id, shortCode, longUrl, createdAt)` with a UNIQUE index on `shortCode`.
- `GET /{shortCode}` → look up and `302 Found` redirect (use `ResponseEntity.status(FOUND).location(uri).build()`).
- Discuss: caching hot short-codes in Redis to avoid a DB hit on every redirect; collision handling
  if hash-based; horizontal ID generation at scale (Snowflake-style IDs instead of a single DB
  auto-increment becoming a bottleneck).

### 16.2 "Design an Idempotent Payment API"

- Client sends an `Idempotency-Key` header (client-generated UUID) with every payment request.
- Server: `@Entity IdempotencyRecord(key UNIQUE, requestHash, responseBody, status)`.
- On request: if the key already exists AND matches the request hash → return the STORED response
  immediately (don't reprocess payment). If it's a NEW key → process normally, then store the
  result keyed by that idempotency key, all within one transaction.
- Prevents duplicate charges from network retries/client bugs.

### 16.3 "Design a Notification Service" (Email/SMS/Push)

- `NotificationService` publishes a `NotificationRequestedEvent` to Kafka (Section 4) instead of
  sending synchronously — decouples the caller from slow third-party providers (Twilio, SendGrid).
- Separate consumers per channel (`EmailConsumer`, `SmsConsumer`) allow independent scaling and
  isolate failures (an SMS provider outage doesn't block email).
- Discuss: retry with exponential backoff + dead-letter queue for permanently failing
  notifications; rate limiting per provider's API limits.

### 16.4 "Design a Rate Limiter as a Service" (used BY other services)

- Centralized rate-limiting service backed by Redis (`INCR` + `EXPIRE` for fixed-window, or a Lua
  script for atomic token-bucket operations to avoid race conditions across multiple app instances).
- Exposes `POST /check {clientId, resource}` → allow/deny decision, called from each service's
  Gateway filter (Section 11).

> **What interviewers are really testing:** Not perfect textbook answers, but whether you can
> connect Spring Boot BUILDING BLOCKS (entities, events, caching, idempotency, transactions) to a
> larger architectural problem — showing you can design, not just annotate.

---

## Section 17 — Concurrency & Java 21 Virtual Threads

### 17.1 Classic Concurrency Primitives (Still Asked)

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
Future<String> future = executor.submit(() -> expensiveComputation());
String result = future.get(5, TimeUnit.SECONDS);   // blocks with a timeout

CompletableFuture<Void> pipeline = CompletableFuture
    .supplyAsync(() -> fetchData())
    .thenApplyAsync(data -> transform(data))
    .thenAcceptAsync(result -> save(result))
    .exceptionally(ex -> { log.error("Pipeline failed", ex); return null; });
```

**Q: What is a race condition, and how does `synchronized` (or `AtomicInteger`) prevent it?**
> A race condition occurs when multiple threads read-modify-write shared state without
> coordination, causing lost updates (e.g., two threads both read `count=5`, both compute `6`, both
> write `6` — one increment is lost). `synchronized` enforces mutual exclusion (only one thread in
> the critical section at a time); `AtomicInteger`/`AtomicLong` use lock-free CAS
> (compare-and-swap) hardware instructions for the SAME safety with less contention overhead.

### 17.2 Java 21 Virtual Threads (Project Loom) — Huge Interview Topic in 2025/2026

Traditional Java threads are OS threads — expensive (≈1MB stack each), limiting you to a few
thousand concurrent threads before running out of memory. **Virtual threads** (finalized in Java
21) are lightweight, JVM-managed threads — you can spawn MILLIONS of them, each mapped onto a
small pool of OS "carrier" threads, automatically "parking" the carrier thread whenever the virtual
thread blocks on I/O (so the carrier thread is freed to run other virtual threads).

```java
// Enable virtual threads for the entire Spring MVC request-handling thread pool:
```
```properties
spring.threads.virtual.enabled=true
```

**This is a genuinely huge deal for Spring Boot specifically:** it means a TRADITIONAL, BLOCKING,
easy-to-write Spring MVC application (JDBC, `RestTemplate`, simple imperative code — none of
WebFlux's reactive complexity) can now handle WebFlux-like levels of I/O concurrency, just by
flipping ONE property, because each blocked request now parks a cheap virtual thread instead of an
expensive platform thread.

**Q: Does this mean Spring WebFlux is now obsolete?**
> Not entirely — WebFlux still has an edge for extreme throughput scenarios and back-pressure
> handling (controlling how fast a fast producer overwhelms a slow consumer), which virtual threads
> alone don't solve. But for the VAST majority of "I/O-bound, moderate-to-high concurrency" services
> that previously might have reached for WebFlux JUST for thread efficiency, virtual threads +
> plain Spring MVC is now a much simpler path to similar scalability — a genuinely hot 2025/2026
> Spring interview topic.

**Q: What's a "pinned" virtual thread, and why can it hurt performance?**
> If a virtual thread executes a `synchronized` block (or calls certain native/blocking code) while
> blocked, it can get "pinned" to its carrier OS thread — unable to unmount/yield the way normal
> virtual thread blocking does, temporarily behaving like an expensive platform thread and reducing
> the scalability benefit. Java 21 mitigates many common cases (e.g., `ReentrantLock` is
> preferred over `synchronized` in virtual-thread-heavy code) but it's an important nuance to
> mention in interviews.

---

## Section 18 — Testcontainers (Real-Database Integration Testing)

The course/supplement's `@DataJpaTest` uses an in-memory H2 database — fast, but H2 doesn't behave
IDENTICALLY to PostgreSQL (different SQL dialects, missing PG-specific features like `JSONB`,
different constraint error messages). **Testcontainers** spins up a REAL PostgreSQL instance in
Docker just for the test run, then tears it down — giving you production-parity integration tests.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class PatientRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("test_db");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired PatientRepository patientRepository;

    @Test
    void savesAndFindsPatient() {
        Patient saved = patientRepository.save(new Patient(null, "Anuj", "anuj@gmail.com"));
        assertThat(patientRepository.findById(saved.getId())).isPresent();
    }
}
```

> **Interview angle:** "Why not just use H2 for everything?" — H2's SQL dialect is close but NOT
> identical to production databases; subtle bugs (a native query using a Postgres-only function, a
> constraint violation message you pattern-match on) can PASS on H2 and FAIL in production.
> Testcontainers eliminates that entire class of "works in tests, breaks in prod" bugs, at the cost
> of slower test startup (spinning up a real container).

---

## Section 19 — Java 21 / Spring Boot 3 Modern Features

Given this workspace already targets **Spring Boot 3 + Java 21**, interviewers at good companies
will assume familiarity with what's NEW compared to the (still extremely common) Spring Boot 2 /
Java 8-11 world most tutorials are written for.

### 19.1 Jakarta EE Namespace Migration (`javax.*` → `jakarta.*`)

Spring Boot 3 moved from Java EE (`javax.persistence.Entity`) to Jakarta EE
(`jakarta.persistence.Entity`) — this is why all the entity annotations in this course use
`jakarta.persistence.*`, not `javax.persistence.*`. A very common "gotcha" when copy-pasting code
from older (Spring Boot 2 / Java 8) tutorials/StackOverflow answers — they won't compile without
updating the imports.

### 19.2 Records as DTOs

```java
public record PatientDto(Long id, String name, String email) {}
// Immutable, auto-generates constructor, accessors (name() not getName()), equals/hashCode/toString
// in ONE line — a modern, more concise alternative to Lombok @Data DTOs for simple, immutable shapes.
```

> **Caveat:** Records work great as READ-ONLY DTOs (response shapes). They're generally a POOR fit
> as JPA `@Entity` classes (entities need mutability for dirty-checking, a no-arg constructor
> Hibernate can use, and proxy-ability for lazy loading — all of which conflict with a Record's
> immutable, `final` nature).

### 19.3 Problem Details (RFC 7807) — Standardized Error Responses

Spring Boot 3 has BUILT-IN support for RFC 7807 "Problem Details" — a standardized JSON error
shape, replacing bespoke custom `ApiError` classes (like the one in `INTERVIEW-SUPPLEMENT.md`
Section 6) with a framework-native convention.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Patient Not Found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```
```json
{
  "type": "about:blank",
  "title": "Patient Not Found",
  "status": 404,
  "detail": "Patient not found with id: 42",
  "timestamp": "2026-07-06T10:15:30Z"
}
```

### 19.4 Sealed Classes + Pattern Matching (Java 21) in Domain Modeling

```java
public sealed interface PaymentResult permits Success, Failure {}
public record Success(String transactionId) implements PaymentResult {}
public record Failure(String reason) implements PaymentResult {}

// Exhaustive pattern matching — compiler ENFORCES you handle every case, no default needed:
String message = switch (result) {
    case Success s -> "Payment succeeded: " + s.transactionId();
    case Failure f -> "Payment failed: " + f.reason();
};
```
`sealed` restricts which classes can implement an interface (a closed, known set), and modern
`switch` pattern matching lets the compiler VERIFY you've handled every permitted subtype — a
powerful alternative to traditional exception-based or nullable-based error handling for domain
logic.

### 19.5 GraalVM Native Image (Recap from Section 1's Ch01 answer, expanded)

```bash
./mvnw -Pnative native:compile
./target/myapp   # starts in ~30-50ms instead of ~1-2 seconds on the JVM
```
Trade-offs to discuss in interviews: much faster startup and lower memory (ideal for
serverless/auto-scaled containers), but longer BUILD times, and reflection-heavy libraries
(including some ORM/proxy-based Spring features) need explicit `native-image` hints — Spring Boot 3
auto-generates most of these via its "AOT" (Ahead-Of-Time) processing engine, but custom reflection
usage in your own code may still need manual `RuntimeHints` registration.

### 19.6 Structured Concurrency (Preview) — Where Virtual Threads Are Heading

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var patientFuture = scope.fork(() -> patientService.get(id));
    var apptsFuture = scope.fork(() -> appointmentService.getFor(id));
    scope.join();           // wait for both
    scope.throwIfFailed();  // propagate first failure, cancel the other automatically
    return new Profile(patientFuture.get(), apptsFuture.get());
}
```
A preview feature (check current JDK version support) that treats a group of related concurrent
subtasks as a SINGLE unit of work — if one fails, siblings are automatically cancelled, avoiding
the manual bookkeeping `CompletableFuture` composition requires. Good to be AWARE of even if not
yet used in production code, as an indicator you follow the platform's direction.

---

## Section 20 — GraphQL with Spring (Brief Overview)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

```graphql
# schema.graphqls
type Patient {
    id: ID!
    name: String!
    appointments: [Appointment!]!
}
type Query {
    patient(id: ID!): Patient
}
```

```java
@Controller
public class PatientGraphQLController {

    @QueryMapping
    public Patient patient(@Argument Long id) {
        return patientService.getById(id);
    }

    @SchemaMapping   // resolves the "appointments" field ONLY if the client actually asks for it
    public List<Appointment> appointments(Patient patient) {
        return appointmentService.getForPatient(patient.getId());
    }
}
```

**REST vs GraphQL (Common Comparison Question):**

| | REST | GraphQL |
|---|---|---|
| Data fetching | Fixed response shape per endpoint (over/under-fetching common) | Client specifies EXACTLY the fields it needs |
| Endpoints | Many (`/patients`, `/patients/{id}/appointments`) | Usually ONE endpoint (`/graphql`) |
| Versioning | Often via URL (`/v2/patients`) | Usually via additive schema evolution (deprecate fields, don't remove) |
| Caching | Easy (standard HTTP caching, CDNs) | Harder (POST-based, needs custom caching strategies) |
| Best for | Simple, well-known resource shapes; public APIs; caching-heavy use cases | Complex/nested data with varying client needs (e.g., mobile vs web needing different field subsets) |

> **Note:** `@SchemaMapping`-resolved fields are lazily invoked ONLY when the client's query
> requests them — a built-in structural defense against over-fetching, but naive resolver
> implementations can reintroduce N+1 (Chapter 10's problem) at the GraphQL layer if each resolver
> independently queries the DB per parent; the **DataLoader** pattern (batching resolver calls) is
> GraphQL's equivalent fix to JPA's `JOIN FETCH`/`@BatchSize`.

---

## Section 21 — Final Battle-Readiness Checklist

Use this as a self-assessment before a real interview. If you can explain EVERY row below in your
own words (ideally with a code example), you are genuinely interview-ready for a strong Spring
Boot / backend engineering role.

### Core (Course + Supplement — should already be solid)
- [ ] IoC, DI, Bean lifecycle, scopes, `@Primary`/`@Qualifier`
- [ ] Spring MVC request flow end-to-end (Tomcat → DispatcherServlet → Controller → Jackson)
- [ ] JPA entity lifecycle, dirty checking, `@Transactional`, PersistenceContext
- [ ] All JPA mapping types + owning/inverse side + cascade/orphanRemoval
- [ ] N+1 problem, `JOIN FETCH`, pagination, projections
- [ ] AOP fundamentals, transaction propagation/isolation, optimistic/pessimistic locking
- [ ] Testing pyramid: `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`, Mockito

### This Roadmap (New — the "master-level" additions)
- [ ] Spring Security filter chain, JWT flow, method security, OAuth2 basics
- [ ] Microservices trade-offs, service discovery, circuit breakers, Saga pattern
- [ ] When to use WebFlux vs MVC — and why virtual threads changed that calculus
- [ ] Kafka producer/consumer basics, partitions/consumer groups, the Outbox pattern
- [ ] `@Scheduled`/`@Async` gotchas (self-invocation, shared thread pool)
- [ ] JPA Auditing (`@CreatedDate`/`@CreatedBy`)
- [ ] OpenAPI/Swagger auto-documentation
- [ ] Naming Spring's internal design patterns (Proxy, Template Method, Observer, etc.)
- [ ] Practical `@Cacheable`/Redis, and why Redis vs local Caffeine
- [ ] CORS preflight mechanics; rate-limiting algorithms (token bucket, etc.)
- [ ] Logs/Metrics/Traces (the 3 observability pillars) — MDC, Micrometer, Zipkin
- [ ] Dockerizing a Spring Boot app (multi-stage builds) + a basic CI/CD pipeline
- [ ] Ability to sketch ONE system-design answer end-to-end (URL shortener/idempotent payments)
- [ ] Java 21 virtual threads — what problem they solve and their one caveat (pinning)
- [ ] Testcontainers vs H2 trade-off
- [ ] Spring Boot 3-specific changes: `jakarta.*` namespace, Problem Details (RFC 7807), Records as DTOs

### Suggested Study Order (If Starting Fresh From Here)
1. Spring Security (Section 1) — almost universally asked.
2. Practical Caching + CORS + Rate Limiting (Sections 9-11) — common in API-design questions.
3. Observability (Section 13) — increasingly asked at "good companies" as a maturity signal.
4. Concurrency / Virtual Threads (Section 17) — very current, high signal-to-noise for 2025/2026 interviews.
5. Messaging + Microservices (Sections 2 & 4) — if targeting mid-to-large product companies.
6. Reactive/WebFlux (Section 3) — only deep-dive if the target company's job description mentions it.
7. System Design practice (Section 16) — do this LAST, after the building blocks above are solid,
   since these questions require combining everything.

````
