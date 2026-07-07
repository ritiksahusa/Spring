# Chapter 2 — Q&A, MCQ & Practice Questions

> **Topic:** Internal Working of Spring Framework
> Review these after reading `notes.md`. Answers are at the bottom.

---

## Part A — Short Answer Questions

<a id="q1"></a>**Q1.** What is a Spring Bean? How is it different from a regular Java object? [↓ Answer](#a1)

<a id="q2"></a>**Q2.** What is Inversion of Control (IoC)? Why is it useful? [↓ Answer](#a2)

<a id="q3"></a>**Q3.** What is Dependency Injection, and what problem does it solve? [↓ Answer](#a3)

<a id="q4"></a>**Q4.** What is the difference between Constructor Injection and Field Injection? Which is preferred and why? [↓ Answer](#a4)

<a id="q5"></a>**Q5.** What does `@Component` do? Name three other annotations that behave the same way. [↓ Answer](#a5)

<a id="q6"></a>**Q6.** What is Component Scanning? Where does it start in a Spring Boot application? [↓ Answer](#a6)

<a id="q7"></a>**Q7.** What is Auto-Configuration? How does it make Spring Boot different from plain Spring Framework? [↓ Answer](#a7)

<a id="q8"></a>**Q8.** What is the `ApplicationContext`? What does it store? [↓ Answer](#a8)

<a id="q9"></a>**Q9.** What error does Spring throw when two beans of the same type exist and you request one to be injected? [↓ Answer](#a9)

<a id="q10"></a>**Q10.** What is `@ConditionalOnProperty` and when would you use it? [↓ Answer](#a10)

---

## Part B — Multiple Choice Questions (MCQ)

<a id="q11"></a>**Q11.** What does it mean for code to be "tightly coupled"? [↓ Answer](#a11)

- A) The code has many unit tests
- B) The code is directly dependent on a specific implementation class
- C) The code uses only interfaces
- D) The code runs faster than loosely coupled code

<a id="q12"></a>**Q12.** Which annotation is the parent/meta annotation of `@Service`, `@Repository`, and `@Controller`? [↓ Answer](#a12)

- A) `@Autowired`
- B) `@Bean`
- C) `@Component`
- D) `@SpringBootApplication`

<a id="q13"></a>**Q13.** When is a Spring bean actually instantiated (created as an object)? [↓ Answer](#a13)

- A) When `@Component` is added to the class
- B) At compile time
- C) When the `ApplicationContext` is refreshed during startup
- D) When the bean's method is first called

<a id="q14"></a>**Q14.** Which of the following is NOT an advantage of Constructor Injection over Field Injection? [↓ Answer](#a14)

- A) Dependencies can be declared as `private final`
- B) Dependencies are visible and explicit
- C) It requires less code to write
- D) Easier to write unit tests

<a id="q15"></a>**Q15.** `@SpringBootApplication` is a composed annotation. Which three annotations does it combine? [↓ Answer](#a15)

- A) `@Component`, `@Autowired`, `@Bean`
- B) `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`
- C) `@Service`, `@Repository`, `@Controller`
- D) `@Configuration`, `@Import`, `@DependsOn`

<a id="q16"></a>**Q16.** What happens at startup if two beans implement the same interface and you inject that interface into another bean without specifying which one? [↓ Answer](#a16)

- A) Spring picks the first one alphabetically
- B) Spring creates both beans and injects both
- C) Spring throws `NoUniqueBeanDefinitionException`
- D) Spring injects `null`

<a id="q17"></a>**Q17.** Where is the list of Spring Boot auto-configurations stored? [↓ Answer](#a17)

- A) `application.properties`
- B) `pom.xml`
- C) `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- D) `src/main/resources/beans.xml`

<a id="q18"></a>**Q18.** You add `spring-boot-starter-data-jpa` to your project. What triggers the automatic creation of JPA beans? [↓ Answer](#a18)

- A) You manually add `@EnableJpa`
- B) Auto-configuration detects JPA classes on the classpath and conditionally creates beans
- C) Spring Boot always creates JPA beans regardless of dependencies
- D) You have to explicitly list JPA beans in `application.properties`

<a id="q19"></a>**Q19.** A class is in package `com.other.tools` but the main class is in `com.codingshuttle.app`. Will `@Component` on the class in `com.other.tools` get picked up? [↓ Answer](#a19)

- A) Yes, Spring scans all packages
- B) No, Component Scanning only covers the root package and its sub-packages
- C) Yes, if you add `@Autowired`
- D) Only if `debug=true` is set

<a id="q20"></a>**Q20.** What does the `CommandLineRunner` interface provide, and when does its `run()` method execute? [↓ Answer](#a20)

- A) A way to run shell commands; executes at startup
- B) A hook that runs after the entire Spring context is ready
- C) A replacement for `main()` method
- D) A method that runs every 60 seconds

---

## Part C — Scenario / Code-Reading Questions

<a id="q21"></a>**Q21.** Given this code:

```java
@Component
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

And both these exist:

```java
@Component
public class RazorPayService implements PaymentService { ... }

@Component
public class StripeService implements PaymentService { ... }
```

What will happen when the Spring application starts? What are two ways to fix it? [↓ Answer](#a21)

---

<a id="q22"></a>**Q22.** A developer writes this:

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User findUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

The app starts fine. But a colleague says this is bad practice. What specifically is bad here, and how would you refactor it? [↓ Answer](#a22)

---

<a id="q23"></a>**Q23.** Explain what happens step-by-step when a developer clicks "Run" on their Spring Boot application in IntelliJ IDEA. List at least 6 steps. [↓ Answer](#a23)

---

<a id="q24"></a>**Q24.** You want your app to use a different database connection bean based on whether the environment is `dev` or `prod`. You control this via `app.env=dev` in `application.properties`. How would you implement this using conditional annotations? [↓ Answer](#a24)

---

<a id="q25"></a>**Q25.** A developer sets `debug=true` in `application.properties` and sees this in the startup log:

```
StripePaymentService: matched (ConditionalOnProperty payment.provider=stripe)
RazorPayPaymentService: did not match (ConditionalOnProperty payment.provider=stripe)
```

What does this tell you about the current configuration and which bean will be injected? [↓ Answer](#a25)

---

## Part D — Fill in the Blanks

<a id="q26"></a>**Q26.** The IoC container in Spring stores beans in an object called the ________. [↓ Answer](#a26)

<a id="q27"></a>**Q27.** When you annotate a class with `@Component`, Spring creates a ________ and stores it in the ApplicationContext. [↓ Answer](#a27)

<a id="q28"></a>**Q28.** The process by which Spring scans packages to find `@Component` annotated classes is called ________. [↓ Answer](#a28)

<a id="q29"></a>**Q29.** `@ConditionalOnProperty` creates a bean only if the specified ________ has the required ________. [↓ Answer](#a29)

<a id="q30"></a>**Q30.** `@RestController` is equivalent to `@Controller` + ________. [↓ Answer](#a30)

---

## Part E — True or False

<a id="q31"></a>**Q31.** A class annotated with `@Service` behaves differently from `@Component` at the bean registration level. _(True/False)_ [↓ Answer](#a31)

<a id="q32"></a>**Q32.** Field Injection (`@Autowired` on a field) allows you to declare the field as `private final`. _(True/False)_ [↓ Answer](#a32)

<a id="q33"></a>**Q33.** `@SpringBootApplication` triggers component scanning of all packages in the entire JVM classpath. _(True/False)_ [↓ Answer](#a33)

<a id="q34"></a>**Q34.** Auto-configuration beans are created unconditionally when Spring Boot starts. _(True/False)_ [↓ Answer](#a34)

<a id="q35"></a>**Q35.** Constructor injection is preferred over field injection in production Spring Boot code. _(True/False)_ [↓ Answer](#a35)

---

## Answers

### Part A

<a id="a1"></a>**1.** A Spring Bean is a Java object whose instantiation, configuration, and lifecycle is managed by the Spring IoC container. A regular Java object is created by the developer using `new`. A bean is created by Spring and stored in the ApplicationContext. [↑ Question](#q1)

<a id="a2"></a>**2.** IoC means the responsibility of creating objects is inverted from the developer to the framework. Benefit: developers don't manage object creation, Spring handles it — leading to loose coupling and easier testing. [↑ Question](#q2)

<a id="a3"></a>**3.** DI means Spring automatically provides (injects) required dependencies to a class. It solves the tight coupling problem: instead of a class creating its own dependencies, it declares what it needs and Spring provides them. [↑ Question](#q3)

<a id="a4"></a>**4.** Constructor injection uses a constructor parameter to receive the dependency (allows `private final`, explicit). Field injection uses `@Autowired` on a field (cannot be `final`, hidden). Constructor injection is preferred: safer, testable, explicit. [↑ Question](#q4)

<a id="a5"></a>**5.** `@Component` marks a class so Spring registers it as a bean. Three equivalent annotations: `@Service`, `@Repository`, `@Controller`. [↑ Question](#q5)

<a id="a6"></a>**6.** Component Scanning is Spring's process of scanning packages to find classes annotated with `@Component` (and derivatives) and register them as beans. It starts from the package of the class annotated with `@SpringBootApplication`. [↑ Question](#q6)

<a id="a7"></a>**7.** Auto-Configuration automatically configures beans based on what's on the classpath and what properties are set. Spring Framework required manual XML config; Spring Boot adds auto-config on top to eliminate boilerplate. [↑ Question](#q7)

<a id="a8"></a>**8.** ApplicationContext is the Spring IoC container — an object that holds all registered beans and provides them wherever needed via dependency injection. [↑ Question](#q8)

<a id="a9"></a>**9.** `NoUniqueBeanDefinitionException` (or "required a single bean, but 2 were found"). [↑ Question](#q9)

<a id="a10"></a>**10.** `@ConditionalOnProperty` creates a bean only when a specific property in `application.properties` (or environment) has a particular value. Use it when you want to switch between implementations by configuration without code changes. [↑ Question](#q10)

### Part B

<a id="a11"></a>**11.** **B** — Code is directly dependent on a specific implementation class [↑ Question](#q11)

<a id="a12"></a>**12.** **C** — `@Component` [↑ Question](#q12)

<a id="a13"></a>**13.** **C** — When the ApplicationContext is refreshed during startup [↑ Question](#q13)

<a id="a14"></a>**14.** **C** — Requires less code [↑ Question](#q14)

<a id="a15"></a>**15.** **B** — `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan` [↑ Question](#q15)

<a id="a16"></a>**16.** **C** — `NoUniqueBeanDefinitionException` [↑ Question](#q16)

<a id="a17"></a>**17.** **C** — `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` [↑ Question](#q17)

<a id="a18"></a>**18.** **B** — Auto-configuration detects JPA classes on the classpath and conditionally creates beans [↑ Question](#q18)

<a id="a19"></a>**19.** **B** — No, Component Scanning only covers the root package and its sub-packages [↑ Question](#q19)

<a id="a20"></a>**20.** **B** — A hook that runs after the entire Spring context is ready [↑ Question](#q20)

### Part C

<a id="a21"></a>**21.** Spring will throw `NoUniqueBeanDefinitionException` — it finds two beans implementing `PaymentService` and doesn't know which to inject into `OrderService`. Two fixes: (1) Remove `@Component` from one of them. (2) Add `@ConditionalOnProperty` to each so only one is created based on a property. [↑ Question](#q21)

<a id="a22"></a>**22.** Bad practice: Field injection (`@Autowired` on field). The field cannot be `final`, making it mutable. Refactored: [↑ Question](#q22)
```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    ...
}
```

<a id="a23"></a>**23.** Steps when "Run" is clicked: [↑ Question](#q23)
1. JVM starts and calls `main()`
2. `SpringApplication.run(...)` is invoked
3. Spring detects `@SpringBootApplication`
4. Spring creates a `SpringApplication` object
5. Environment is prepared (reads `application.properties`, env vars)
6. `ApplicationContext` is created
7. Component Scanning scans the root package recursively for beans
8. Auto-Configuration evaluates ~150+ conditional configs
9. ApplicationContext is refreshed — beans are instantiated, dependencies injected
10. Embedded Tomcat server starts on port 8080
11. `CommandLineRunner` beans execute
12. Application is ready to serve requests

<a id="a24"></a>**24.** [↑ Question](#q24)
```java
@Component
@ConditionalOnProperty(name = "app.env", havingValue = "dev")
public class DevDatabaseConfig implements DatabaseConfig { ... }

@Component
@ConditionalOnProperty(name = "app.env", havingValue = "prod")
public class ProdDatabaseConfig implements DatabaseConfig { ... }
```
Set `app.env=dev` or `app.env=prod` in `application.properties` to switch.

<a id="a25"></a>**25.** The property `payment.provider=stripe` is set. `StripePaymentService` matched and will be created as a bean. `RazorPayPaymentService` did not match and will NOT be created. Therefore, wherever `PaymentService` is injected, the `StripePaymentService` bean will be provided. [↑ Question](#q25)

### Part D

<a id="a26"></a>**26.** ApplicationContext [↑ Question](#q26)

<a id="a27"></a>**27.** bean (an instance / a singleton object) [↑ Question](#q27)

<a id="a28"></a>**28.** Component Scanning [↑ Question](#q28)

<a id="a29"></a>**29.** property / value [↑ Question](#q29)

<a id="a30"></a>**30.** `@ResponseBody` [↑ Question](#q30)

### Part E

<a id="a31"></a>**31.** **False** — `@Service` has the same bean registration behavior as `@Component`. The difference is semantic (better readability, and `@Repository` adds exception translation). [↑ Question](#q31)

<a id="a32"></a>**32.** **False** — Field injection does NOT allow `private final`. [↑ Question](#q32)

<a id="a33"></a>**33.** **False** — It only scans the root package and its sub-packages. [↑ Question](#q33)

<a id="a34"></a>**34.** **False** — Auto-configuration beans are created **conditionally** — only when their conditions are met. [↑ Question](#q34)

<a id="a35"></a>**35.** **True** — Constructor injection is the recommended and preferred approach. [↑ Question](#q35)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** How does Spring resolve circular dependencies between two singleton beans using field/setter injection, and why does the SAME circular dependency fail with constructor injection? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** Spring uses a 3-level cache during singleton creation: `singletonObjects` (fully initialized beans), `earlySingletonObjects` (raw, not-yet-populated instances), and `singletonFactories` (`ObjectFactory` references for beans currently under construction). When Bean A needs Bean B mid-construction, Spring exposes an early, not-fully-populated reference of A to satisfy B, then finishes populating A afterward. This only works because the bean instance already exists (via a no-arg constructor) before dependencies are set. Constructor injection requires ALL dependencies to exist BEFORE the object itself can be instantiated — there is no "early reference" to hand out, so Spring throws `BeanCurrentlyInCreationException`. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** What is the real difference between `BeanFactory` and `ApplicationContext`, and why does Spring Boot always use the latter? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** `BeanFactory` is the minimal root IoC container — lazy instantiation, no built-in AOP/events/i18n/environment support. `ApplicationContext` extends it and adds eager singleton pre-instantiation (fail-fast at startup), `ApplicationEventPublisher`, `MessageSource` (i18n), the `Environment` abstraction (profiles/properties), and automatic `BeanPostProcessor` registration. Spring Boot's auto-configuration, profiles, and embedded-server lifecycle all depend on these features, so it always bootstraps a concrete `ApplicationContext`. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** Why can't a `@Transactional` method calling ANOTHER `@Transactional` method on `this` (self-invocation) actually trigger a new transaction, and how does this relate to JDK dynamic proxies vs CGLIB? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** Spring AOP is proxy-based — Spring wraps your bean in a proxy (JDK dynamic proxy if it implements an interface, otherwise a CGLIB subclass). The proxy intercepts calls made FROM OUTSIDE through the injected reference (which is actually the proxy). A self-invocation (`this.otherMethod()`) bypasses the proxy entirely, so no AOP advice (transactions, caching, security) applies. Fix: inject the bean into itself via `@Lazy` self-injection, or move the method to a separate collaborator bean. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** What determines whether Spring uses a JDK dynamic proxy or CGLIB for a `@Service`, and what silent failure can occur with CGLIB? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** By default, Spring uses a JDK dynamic proxy if the target implements an interface, otherwise CGLIB. Spring Boot's default (`spring.aop.proxy-target-class=true`) actually forces CGLIB even for interface-implementing beans, for consistency. CGLIB proxies cannot subclass `final` classes or override `final` methods — a common silent-failure gotcha where `@Transactional` on a `final` method does absolutely nothing, with no compile or startup error. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** When would you use `@Lazy` on a bean, and what guarantee do you lose by using it? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** `@Lazy` defers instantiation until first use — useful for breaking circular-dependency deadlocks, reducing startup cost for rarely-used heavy beans, or optional feature-flagged integrations. The trade-off: you lose Spring Boot's "fail fast" guarantee. A misconfigured lazy bean's error only surfaces on first use (potentially mid-request in production) instead of at deploy time. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** What's the actual difference between a Spring "bean definition" and a "bean instance"? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** A `BeanDefinition` is metadata (class name, scope, constructor args, init/destroy method names) registered during component scanning / `@Configuration` processing — no objects exist yet. A bean instance is the actual object later created from that definition. This is why `BeanFactoryPostProcessor`s (which modify `BeanDefinition`s, e.g., property placeholder resolution) run BEFORE any bean is instantiated, while `BeanPostProcessor`s operate on already-created objects. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** If you inject `PaymentService` as `List<PaymentService>` instead of a single bean when two implementations exist, what happens — and why is this often the BETTER design choice? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** No exception — Spring injects ALL matching beans into the list (ordered via `@Order`/`Ordered` if present). This is the recommended Strategy-pattern approach: iterate the list and pick the implementation whose `supports(type)` returns true, instead of juggling `@Qualifier` strings scattered across the codebase whenever a new payment provider is added. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** A teammate adds `@ConditionalOnMissingBean` expecting their custom bean to act as an overridable "default." What's the classic ordering bug that can break this? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** `@ConditionalOnMissingBean` only evaluates against beans ALREADY registered at the point that configuration class is processed. Between TWO auto-configurations (or auto-config vs. user config) racing to register the "default" bean, processing order determines the outcome. Spring Boot guarantees user `@Configuration` classes are generally evaluated before auto-configurations matching `@ConditionalOnMissingBean`, but between multiple auto-configurations, explicit `@AutoConfigureOrder`/`@AutoConfiguration(before/after = ...)` is required, or bean selection becomes non-deterministic. [↑ Question](#qf8)

````
