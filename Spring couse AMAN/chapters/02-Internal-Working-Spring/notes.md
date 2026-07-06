# Chapter 2: Internal Working of Spring Framework

> **Video time:** `21:54` → `59:04`

---

## 1. The Problem — Tight Coupling

Consider a plain Java class that needs to make a payment:

```java
// Tight coupling — hardcoded to RazorPay
public class App implements CommandLineRunner {
    private RazorPayPaymentService paymentService = new RazorPayPaymentService();

    @Override
    public void run(String... args) {
        String result = paymentService.pay();
        System.out.println("Payment done: " + result);
    }
}
```

**Problems:**
- If you want to switch from RazorPay → Stripe → PhonePe, you must **change code** every time.
- Every change requires **recompilation and redeployment**.
- Code is rigidly coupled to one specific implementation.

---

## 2. The Solution — Beans + Dependency Injection

### What is a Bean?

> **Bean** = A Java object whose **lifecycle is managed by the Spring IoC container** (creation, dependency injection, destruction).

You declare a class as a bean by annotating it with `@Component` (or any stereotype annotation).

```java
@Component
public class RazorPayPaymentService implements PaymentService {
    public String pay() {
        System.out.println("Payment from RazorPay");
        return "RazorPay Payment";
    }
}
```

When Spring starts, it finds this annotation, **creates one instance**, and stores it in the **Application Context**.

---

## 3. Dependency Injection

Dependency Injection (DI) means Spring automatically **provides (injects) the required beans** wherever they are needed — you never call `new` yourself.

### Type 1: Constructor Injection ✅ (Preferred)

```java
@Component
public class App implements CommandLineRunner {
    private final PaymentService paymentService; // final = immutable after construction

    // Spring sees this constructor needs a PaymentService bean
    public App(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @Override
    public void run(String... args) {
        String result = paymentService.pay();
        System.out.println("Payment done: " + result);
    }
}
```

**Why constructor injection is preferred:**
- Allows `private final` — the dependency is **immutable** after construction
- Makes dependencies **explicit** — you can see what the class needs just by looking at the constructor
- Easier to test (you can pass mock objects in tests)

### Type 2: Field Injection (Discouraged)

```java
@Component
public class App implements CommandLineRunner {
    @Autowired
    private PaymentService paymentService; // Spring injects this via reflection

    @Override
    public void run(String... args) {
        paymentService.pay();
    }
}
```

**Drawbacks:**
- Cannot be `final` — dependency could be changed later
- Dependencies are hidden — not visible from constructor
- Harder to unit test without a Spring context

---

## 4. Loose Coupling with Interfaces

The real power: inject an **interface**, not a concrete class.

```java
// Step 1: Define the interface
public interface PaymentService {
    String pay();
}

// Step 2: Multiple implementations
@Component
public class RazorPayPaymentService implements PaymentService {
    public String pay() { return "Payment via RazorPay"; }
}

@Component
public class StripePaymentService implements PaymentService {
    public String pay() { return "Payment via Stripe"; }
}

// Step 3: Use the interface — the App doesn't care which implementation!
@Component
public class App implements CommandLineRunner {
    private final PaymentService paymentService; // just the interface

    public App(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Problem:** If both `RazorPayPaymentService` and `StripePaymentService` have `@Component`, Spring sees **two beans of type PaymentService** and throws:

```
Parameter 0 of constructor required a single bean, but 2 were found
```

**Solutions:**
1. Remove `@Component` from the one you don't want
2. Use `@ConditionalOnProperty` (most powerful — see below)
3. Use `@Primary` to mark one bean as the default

---

## 5. Conditional Bean Creation — `@ConditionalOnProperty`

```properties
# application.properties
payment.provider=razorpay
```

```java
@Component
@ConditionalOnProperty(name = "payment.provider", havingValue = "razorpay")
public class RazorPayPaymentService implements PaymentService {
    public String pay() { return "Payment via RazorPay"; }
}

@Component
@ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
public class StripePaymentService implements PaymentService {
    public String pay() { return "Payment via Stripe"; }
}
```

Now you can switch payment providers just by changing `application.properties` — **no code change, no recompilation!**

This is the essence of **Loose Coupling**.

### Other Conditional annotations:

| Annotation | Condition |
|---|---|
| `@ConditionalOnProperty` | A property has a specific value |
| `@ConditionalOnClass` | A class is present on the classpath |
| `@ConditionalOnMissingBean` | A bean of that type does NOT exist yet |
| `@ConditionalOnExpression` | A SpEL expression evaluates to `true` |
| `@ConditionalOnResource` | A resource file is present |

---

## 6. Stereotype Annotations — All Are `@Component` Under the Hood

These are all specialized versions of `@Component`:

| Annotation | Layer | Extra behavior |
|---|---|---|
| `@Component` | Any | Generic bean marker |
| `@Service` | Business logic layer | Semantic hint (no extra behavior) |
| `@Repository` | Data access layer | Enables Spring exception translation |
| `@Controller` | Web MVC controller | Marks as request handler (returns views) |
| `@RestController` | REST API controller | `@Controller` + `@ResponseBody` |

All of them tell Spring: **"Manage an instance of this class as a bean."**

```java
// These are all equivalent for basic bean registration:
@Component
@Service
@Repository
@Controller
```

---

## 7. IoC Container / Application Context

**Inversion of Control (IoC)** means the control of object creation is **inverted** — instead of the developer creating objects (`new SomeClass()`), the **Spring framework** creates and manages them.

### The IoC Container (Application Context):

```
POJO Classes + External Configuration
         ↓
   IoC Container (Spring)
         ↓  (component scanning + auto-configuration + conditionals)
   Application Context
     = bunch of ready-to-use beans
         ↓
   Injects beans wherever needed (Dependency Injection)
```

**What Application Context does:**
1. Scans all classes for `@Component` (and friends)
2. Reads configuration (`application.properties`, environment variables)
3. Applies conditions (`@ConditionalOn*`)
4. Creates the required beans
5. Injects dependencies where needed
6. Provides beans on demand throughout the app

---

## 8. Component Scanning

Component scanning starts from the class annotated with `@SpringBootApplication`.

```java
@SpringBootApplication  // This annotation triggers everything
public class LearningSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(LearningSpringBootApplication.class, args);
    }
}
```

`@SpringBootApplication` is a composed annotation:
- `@SpringBootConfiguration` — marks as config class
- `@EnableAutoConfiguration` — activates auto-config
- `@ComponentScan` — scans the package and all sub-packages

**Important rule:** Only classes **inside the same package or sub-packages** of the main class get scanned.

```
com.codingshuttle.youtube/
├── LearningSpringBootApplication.java  ← root, scanning starts here
├── payment/
│   ├── RazorPayPaymentService.java     ← ✅ scanned
│   └── StripePaymentService.java      ← ✅ scanned
└── App.java                            ← ✅ scanned

com.other/
└── SomeClass.java                      ← ❌ NOT scanned!
```

---

## 9. Auto-Configuration

Auto-configuration is what differentiates **Spring** from **Spring Boot**.

- **Spring Framework** = powerful but requires lots of manual XML/Java config
- **Spring Boot** = Spring Framework + Auto-Configuration = productivity boost

### How it works:

Spring Boot ships with `spring-boot-autoconfigure.jar`. Inside it, there's a file:
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

This file lists ~150+ auto-configuration classes:
```
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
... (150+ more)
```

Each entry is a config class that creates beans **conditionally**:
```java
@AutoConfiguration
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class })
public class HibernateJpaAutoConfiguration {
    // Creates JPA beans only if JPA classes are on the classpath
}
```

So when you add `spring-boot-starter-data-jpa` to your `pom.xml`, the JPA classes appear on the classpath → auto-configuration triggers → all JPA beans are created **automatically** with sensible defaults.

---

## 10. What Happens When You Click "Run"

**Step-by-step lifecycle:**

```
1. JVM starts → main() is called
2. SpringApplication.run(...) is invoked
3. Spring Boot detects @SpringBootApplication
4. SpringApplication object is created
5. Environment is prepared:
   - Reads application.properties, environment variables, command-line args
6. Application Context (IoC Container) is created
7. Component Scanning starts from root package:
   - Finds all @Component, @Service, @Repository, @Controller classes
8. Auto-Configuration is loaded:
   - spring.factories / AutoConfiguration.imports is read
   - ~150+ conditional configurations are evaluated
   - Matching ones are activated
9. Application Context is REFRESHED:
   - All beans are actually instantiated
   - Dependencies are autowired (injected)
   - @PostConstruct lifecycle methods are called
10. Embedded server (Tomcat) starts on port 8080
11. CommandLineRunner / ApplicationRunner beans are executed
12. Application is fully started → ready to handle requests
```

Enable debug logging to see bean creation in detail:
```properties
debug=true
```

This prints which beans were created, which auto-configurations matched, and which were skipped.

---

## 11. Manual Bean Registration with `@Bean`

Besides `@Component`, you can also create beans manually in a `@Configuration` class:

```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService(); // manual creation
    }
}
```

**When to use `@Bean`:**
- For third-party classes you can't annotate (no source access)
- When you need custom initialization logic
- When creating multiple beans of the same type with different configs

---

## 12. Key Concepts Summary

| Concept | Meaning |
|---|---|
| **Bean** | A Java object created and managed by the Spring IoC container |
| **IoC** | Inversion of Control — Spring manages object creation, not you |
| **DI** | Dependency Injection — Spring provides required objects to dependents |
| **Constructor Injection** | DI via constructor argument (preferred) |
| **Field Injection** | DI via `@Autowired` on a field (discouraged) |
| **ApplicationContext** | The Spring IoC container that holds all beans |
| **Component Scanning** | Spring scans packages for `@Component` and registers beans |
| **Auto-Configuration** | Spring Boot automatically configures beans based on classpath |
| **@Component** | Generic bean marker |
| **@Service / @Repository / @Controller** | Specialized `@Component` for different layers |
| **Loose Coupling** | Code depends on interfaces, not concrete implementations |
| **@ConditionalOnProperty** | Create a bean only if a property matches a value |
| **Tight Coupling** | Code is hardwired to a specific implementation (BAD) |
