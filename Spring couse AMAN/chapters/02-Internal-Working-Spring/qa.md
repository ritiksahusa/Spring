# Chapter 2 — Q&A, MCQ & Practice Questions

> **Topic:** Internal Working of Spring Framework
> Review these after reading `notes.md`. Answers are at the bottom.

---

## Part A — Short Answer Questions

**Q1.** What is a Spring Bean? How is it different from a regular Java object?

**Q2.** What is Inversion of Control (IoC)? Why is it useful?

**Q3.** What is Dependency Injection, and what problem does it solve?

**Q4.** What is the difference between Constructor Injection and Field Injection? Which is preferred and why?

**Q5.** What does `@Component` do? Name three other annotations that behave the same way.

**Q6.** What is Component Scanning? Where does it start in a Spring Boot application?

**Q7.** What is Auto-Configuration? How does it make Spring Boot different from plain Spring Framework?

**Q8.** What is the `ApplicationContext`? What does it store?

**Q9.** What error does Spring throw when two beans of the same type exist and you request one to be injected?

**Q10.** What is `@ConditionalOnProperty` and when would you use it?

---

## Part B — Multiple Choice Questions (MCQ)

**Q11.** What does it mean for code to be "tightly coupled"?

- A) The code has many unit tests
- B) The code is directly dependent on a specific implementation class
- C) The code uses only interfaces
- D) The code runs faster than loosely coupled code

**Q12.** Which annotation is the parent/meta annotation of `@Service`, `@Repository`, and `@Controller`?

- A) `@Autowired`
- B) `@Bean`
- C) `@Component`
- D) `@SpringBootApplication`

**Q13.** When is a Spring bean actually instantiated (created as an object)?

- A) When `@Component` is added to the class
- B) At compile time
- C) When the `ApplicationContext` is refreshed during startup
- D) When the bean's method is first called

**Q14.** Which of the following is NOT an advantage of Constructor Injection over Field Injection?

- A) Dependencies can be declared as `private final`
- B) Dependencies are visible and explicit
- C) It requires less code to write
- D) Easier to write unit tests

**Q15.** `@SpringBootApplication` is a composed annotation. Which three annotations does it combine?

- A) `@Component`, `@Autowired`, `@Bean`
- B) `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`
- C) `@Service`, `@Repository`, `@Controller`
- D) `@Configuration`, `@Import`, `@DependsOn`

**Q16.** What happens at startup if two beans implement the same interface and you inject that interface into another bean without specifying which one?

- A) Spring picks the first one alphabetically
- B) Spring creates both beans and injects both
- C) Spring throws `NoUniqueBeanDefinitionException`
- D) Spring injects `null`

**Q17.** Where is the list of Spring Boot auto-configurations stored?

- A) `application.properties`
- B) `pom.xml`
- C) `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- D) `src/main/resources/beans.xml`

**Q18.** You add `spring-boot-starter-data-jpa` to your project. What triggers the automatic creation of JPA beans?

- A) You manually add `@EnableJpa`
- B) Auto-configuration detects JPA classes on the classpath and conditionally creates beans
- C) Spring Boot always creates JPA beans regardless of dependencies
- D) You have to explicitly list JPA beans in `application.properties`

**Q19.** A class is in package `com.other.tools` but the main class is in `com.codingshuttle.app`. Will `@Component` on the class in `com.other.tools` get picked up?

- A) Yes, Spring scans all packages
- B) No, Component Scanning only covers the root package and its sub-packages
- C) Yes, if you add `@Autowired`
- D) Only if `debug=true` is set

**Q20.** What does the `CommandLineRunner` interface provide, and when does its `run()` method execute?

- A) A way to run shell commands; executes at startup
- B) A hook that runs after the entire Spring context is ready
- C) A replacement for `main()` method
- D) A method that runs every 60 seconds

---

## Part C — Scenario / Code-Reading Questions

**Q21.** Given this code:

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

What will happen when the Spring application starts? What are two ways to fix it?

---

**Q22.** A developer writes this:

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

The app starts fine. But a colleague says this is bad practice. What specifically is bad here, and how would you refactor it?

---

**Q23.** Explain what happens step-by-step when a developer clicks "Run" on their Spring Boot application in IntelliJ IDEA. List at least 6 steps.

---

**Q24.** You want your app to use a different database connection bean based on whether the environment is `dev` or `prod`. You control this via `app.env=dev` in `application.properties`. How would you implement this using conditional annotations?

---

**Q25.** A developer sets `debug=true` in `application.properties` and sees this in the startup log:

```
StripePaymentService: matched (ConditionalOnProperty payment.provider=stripe)
RazorPayPaymentService: did not match (ConditionalOnProperty payment.provider=stripe)
```

What does this tell you about the current configuration and which bean will be injected?

---

## Part D — Fill in the Blanks

**Q26.** The IoC container in Spring stores beans in an object called the ________.

**Q27.** When you annotate a class with `@Component`, Spring creates a ________ and stores it in the ApplicationContext.

**Q28.** The process by which Spring scans packages to find `@Component` annotated classes is called ________.

**Q29.** `@ConditionalOnProperty` creates a bean only if the specified ________ has the required ________.

**Q30.** `@RestController` is equivalent to `@Controller` + ________.

---

## Part E — True or False

**Q31.** A class annotated with `@Service` behaves differently from `@Component` at the bean registration level. _(True/False)_

**Q32.** Field Injection (`@Autowired` on a field) allows you to declare the field as `private final`. _(True/False)_

**Q33.** `@SpringBootApplication` triggers component scanning of all packages in the entire JVM classpath. _(True/False)_

**Q34.** Auto-configuration beans are created unconditionally when Spring Boot starts. _(True/False)_

**Q35.** Constructor injection is preferred over field injection in production Spring Boot code. _(True/False)_

---

## Answers

### Part A

1. A Spring Bean is a Java object whose instantiation, configuration, and lifecycle is managed by the Spring IoC container. A regular Java object is created by the developer using `new`. A bean is created by Spring and stored in the ApplicationContext.

2. IoC means the responsibility of creating objects is inverted from the developer to the framework. Benefit: developers don't manage object creation, Spring handles it — leading to loose coupling and easier testing.

3. DI means Spring automatically provides (injects) required dependencies to a class. It solves the tight coupling problem: instead of a class creating its own dependencies, it declares what it needs and Spring provides them.

4. Constructor injection uses a constructor parameter to receive the dependency (allows `private final`, explicit). Field injection uses `@Autowired` on a field (cannot be `final`, hidden). Constructor injection is preferred: safer, testable, explicit.

5. `@Component` marks a class so Spring registers it as a bean. Three equivalent annotations: `@Service`, `@Repository`, `@Controller`.

6. Component Scanning is Spring's process of scanning packages to find classes annotated with `@Component` (and derivatives) and register them as beans. It starts from the package of the class annotated with `@SpringBootApplication`.

7. Auto-Configuration automatically configures beans based on what's on the classpath and what properties are set. Spring Framework required manual XML config; Spring Boot adds auto-config on top to eliminate boilerplate.

8. ApplicationContext is the Spring IoC container — an object that holds all registered beans and provides them wherever needed via dependency injection.

9. `NoUniqueBeanDefinitionException` (or "required a single bean, but 2 were found").

10. `@ConditionalOnProperty` creates a bean only when a specific property in `application.properties` (or environment) has a particular value. Use it when you want to switch between implementations by configuration without code changes.

### Part B

11. **B** — Code is directly dependent on a specific implementation class
12. **C** — `@Component`
13. **C** — When the ApplicationContext is refreshed during startup
14. **C** — Requires less code
15. **B** — `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`
16. **C** — `NoUniqueBeanDefinitionException`
17. **C** — `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
18. **B** — Auto-configuration detects JPA classes on the classpath and conditionally creates beans
19. **B** — No, Component Scanning only covers the root package and its sub-packages
20. **B** — A hook that runs after the entire Spring context is ready

### Part C

21. Spring will throw `NoUniqueBeanDefinitionException` — it finds two beans implementing `PaymentService` and doesn't know which to inject into `OrderService`. Two fixes: (1) Remove `@Component` from one of them. (2) Add `@ConditionalOnProperty` to each so only one is created based on a property.

22. Bad practice: Field injection (`@Autowired` on field). The field cannot be `final`, making it mutable. Refactored:
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

23. Steps when "Run" is clicked:
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

24. 
```java
@Component
@ConditionalOnProperty(name = "app.env", havingValue = "dev")
public class DevDatabaseConfig implements DatabaseConfig { ... }

@Component
@ConditionalOnProperty(name = "app.env", havingValue = "prod")
public class ProdDatabaseConfig implements DatabaseConfig { ... }
```
Set `app.env=dev` or `app.env=prod` in `application.properties` to switch.

25. The property `payment.provider=stripe` is set. `StripePaymentService` matched and will be created as a bean. `RazorPayPaymentService` did not match and will NOT be created. Therefore, wherever `PaymentService` is injected, the `StripePaymentService` bean will be provided.

### Part D

26. ApplicationContext
27. bean (an instance / a singleton object)
28. Component Scanning
29. property / value
30. `@ResponseBody`

### Part E

31. **False** — `@Service` has the same bean registration behavior as `@Component`. The difference is semantic (better readability, and `@Repository` adds exception translation).
32. **False** — Field injection does NOT allow `private final`.
33. **False** — It only scans the root package and its sub-packages.
34. **False** — Auto-configuration beans are created **conditionally** — only when their conditions are met.
35. **True** — Constructor injection is the recommended and preferred approach.
