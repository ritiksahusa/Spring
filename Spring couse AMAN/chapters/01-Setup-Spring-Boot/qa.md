# Chapter 1 — Q&A, MCQ & Practice Questions

> **Topic:** Setup a Spring Boot Project in IntelliJ IDEA
> Review these questions after reading `notes.md`. All answers are at the bottom.

---

## Part A — Short Answer Questions

**Q1.** What does IDE stand for, and name three things you can do inside an IDE?

**Q2.** What is the URL of the Spring project bootstrapper tool, and what does it generate?

**Q3.** What is the difference between IntelliJ IDEA Community and Ultimate editions?

**Q4.** What is a "starter dependency" in Spring Boot? Give one example.

**Q5.** What is the purpose of the `pom.xml` file?

**Q6.** What is `application.properties` used for? Give two examples of what you can configure there.

**Q7.** What happens when you run the Spring Boot application and visit `http://localhost:8080/` without creating any controller?

**Q8.** What is the default embedded web server in `spring-boot-starter-web`?

**Q9.** What is the minimum JDK version you need if your `pom.xml` has `<java.version>21</java.version>`?

**Q10.** What annotation marks a class as a REST controller, and what annotation maps an HTTP GET request to a method?

---

## Part B — Multiple Choice Questions (MCQ)

**Q11.** Which file manages dependencies and build configuration in a Maven-based Spring Boot project?

- A) `application.properties`
- B) `settings.gradle`
- C) `pom.xml`
- D) `build.gradle`

**Q12.** What does `spring-boot-starter-web` dependency include by default?

- A) Spring Data JPA + Hibernate
- B) Embedded Tomcat + Spring MVC + Jackson
- C) Spring Security + JWT
- D) Spring WebFlux + Reactor

**Q13.** Which packaging format is typically used for a pure Java backend Spring Boot application?

- A) WAR
- B) EAR
- C) ZIP
- D) JAR

**Q14.** What is the default port Spring Boot's embedded Tomcat listens on?

- A) 80
- B) 443
- C) 8080
- D) 3000

**Q15.** At Spring Initializr, the "Package Name" is auto-generated as a concatenation of which two fields?

- A) Name + Description
- B) Group + Artifact
- C) Language + Version
- D) Artifact + Java version

**Q16.** Which annotation in the main class triggers Spring Boot's auto-configuration and component scanning?

- A) `@Controller`
- B) `@EnableAutoConfiguration`
- C) `@SpringBootApplication`
- D) `@ComponentScan`

**Q17.** You want to switch from Tomcat to Jetty. What is the correct approach in `pom.xml`?

- A) Just add `spring-boot-starter-jetty` without removing anything
- B) Remove `spring-boot-starter-web` entirely and add Jetty
- C) Exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`, then add `spring-boot-starter-jetty`
- D) Set `server.engine=jetty` in `application.properties`

**Q18.** Where is your main business logic (Java source code) placed in a Maven Spring Boot project?

- A) `src/test/java/`
- B) `src/main/resources/`
- C) `src/main/java/`
- D) `resources/java/`

**Q19.** What library does Spring Boot use behind the scenes to convert Java objects to JSON automatically?

- A) Gson
- B) FastJSON
- C) Jackson (`jackson-databind`)
- D) JAXB

**Q20.** In which edition of IntelliJ IDEA would you typically find the Spring Boot-specific plugins used in large companies?

- A) Community Edition (free)
- B) Ultimate Edition (paid)
- C) Android Studio
- D) Eclipse

---

## Part C — Scenario / Code-Reading Questions

**Q21.** Look at this code:

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

What does `SpringApplication.run(...)` do? List at least 3 things it triggers internally.

---

**Q22.** A developer writes this controller:

```java
@RestController
public class GreetController {

    @GetMapping("/greet")
    public String greet() {
        return "Welcome!";
    }
}
```

When the app starts and someone visits `http://localhost:8080/`, what response do they get, and why?

---

**Q23.** A developer has `<java.version>17</java.version>` in `pom.xml` but has JDK 11 selected in IntelliJ IDEA's SDK settings. What problem will occur?

---

**Q24.** Study this `pom.xml` snippet:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>
</dependencies>
```

What embedded server will this application use? Explain why.

---

**Q25.** A teammate complains: "I added the `spring-boot-starter-web` dependency but I have no idea why Tomcat started. I only wanted MVC controllers!" 

Explain why Tomcat starts automatically, and what they could do if they only want the web layer without Tomcat.

---

## Part D — Fill in the Blanks

**Q26.** The `src/test/java` folder is used for ________ code, and it is separate from the business logic in `src/main/java`.

**Q27.** `spring-boot-starter-web` is called a ________ because it bundles several individual dependencies together.

**Q28.** The default error page shown by Spring Boot when no mapping exists is called the ________ Error Page.

**Q29.** The `application.properties` file is located under `src/main/________`.

**Q30.** To download JDK from within IntelliJ IDEA, you press ________ (Mac) or ________ (Windows/Linux) to open Project Structure.

---

## Answers

### Part A
1. IDE = Integrated Development Environment. Write, compile, run, debug, test — all in one place.
2. `start.spring.io`. It generates a boilerplate (starter) Spring Boot project zip file.
3. Community is free with basic Java support. Ultimate is paid (30-day trial) and includes Spring-specific plugins, database tools, profiling, etc.
4. A starter bundles multiple related dependencies. Example: `spring-boot-starter-web` includes Tomcat, Spring MVC, Jackson.
5. `pom.xml` is Maven's project descriptor — it defines dependencies, Java version, plugins, and packaging type.
6. Server port (`server.port`), database URL/credentials (`spring.datasource.*`), logging levels, profile settings, etc.
7. Spring Boot shows the default "Whitelabel Error Page" because no mapping exists for `/` or `/error`.
8. Apache Tomcat (embedded).
9. JDK 21 or higher (must be ≥ the version declared in pom.xml).
10. `@RestController` marks the class; `@GetMapping("/path")` maps GET HTTP requests.

### Part B
11. **C** — `pom.xml`
12. **B** — Embedded Tomcat + Spring MVC + Jackson
13. **D** — JAR
14. **C** — 8080
15. **B** — Group + Artifact
16. **C** — `@SpringBootApplication`
17. **C** — Exclude Tomcat, add Jetty starter
18. **C** — `src/main/java/`
19. **C** — Jackson (`jackson-databind`)
20. **B** — Ultimate Edition

### Part C
21. `SpringApplication.run(...)` triggers: (1) Creates the Spring ApplicationContext, (2) Starts embedded Tomcat server, (3) Performs auto-configuration (scans classpath, configures beans), (4) Runs component scan to discover `@Component`/`@Service`/`@Repository`/`@Controller` classes, (5) Prints the Spring Boot banner.

22. They get the **Whitelabel Error Page**. The controller only maps `/greet`, not `/`. When no mapping is found for `/`, Spring redirects to `/error`. Since no `/error` mapping exists either, Spring Boot returns its default error page.

23. The build will fail or produce errors because the JDK in IntelliJ (v11) is older than the required version (v17). Java maintains backward compatibility but NOT forward compatibility — you cannot compile Java 17 code with a Java 11 JDK.

24. The application will use **Jetty**. By excluding `spring-boot-starter-tomcat` from `spring-boot-starter-web`, Tomcat's auto-configuration is removed. Adding `spring-boot-starter-jetty` provides Jetty, and Spring Boot's auto-configuration detects Jetty on the classpath and starts it instead.

25. Tomcat starts automatically because `spring-boot-starter-web` includes `spring-boot-starter-tomcat` as a transitive dependency, and Spring Boot auto-configures it. To keep the web layer without Tomcat, the developer should exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`. They could also use Spring WebFlux (`spring-boot-starter-webflux`) if they want a reactive, non-servlet setup.

### Part D
26. testing / unit testing / integration testing
27. starter dependency
28. Whitelabel
29. resources
30. **⌘ + ;** (Mac) / **Ctrl + ;** (Windows/Linux)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

**F1.** Spring Boot produces a "fat/uber JAR". How does its internal structure differ from a normal JAR, and how does the JVM actually run classes nested inside another JAR (which a plain `java -cp` cannot do)?

> **Answer:** A Spring Boot fat JAR contains `BOOT-INF/classes` (your compiled classes), `BOOT-INF/lib` (all dependency JARs, un-extracted), and `org/springframework/boot/loader/*` (Spring Boot's own loader classes). A standard JVM classloader cannot load classes from a JAR nested inside another JAR, so Spring Boot bundles a custom `LaunchedURLClassLoader` and a `JarLauncher` (declared as `Main-Class` in `MANIFEST.MF`). Running `java -jar app.jar` actually starts `JarLauncher` first, which builds a classloader capable of reading nested JARs, and only then invokes your real `main()` method (recorded as `Start-Class` in the manifest).

**F2.** Two Spring Boot apps try to bind to port 8080 on the same machine. What actually happens, and how do you make the port dynamic for parallel integration tests?

> **Answer:** The second app fails at startup with `BindException: Address already in use` while embedded Tomcat opens its server socket — this is a fatal startup exception that prevents the `ApplicationContext` from refreshing, not a runtime error. For dynamic ports (common in tests), set `server.port=0` so the OS assigns a free ephemeral port, retrievable via `@LocalServerPort private int port;` inside a `@SpringBootTest(webEnvironment = RANDOM_PORT)` test.

**F3.** Why is the PostgreSQL JDBC driver dependency marked with `<scope>runtime</scope>`, and what does that scope actually change?

> **Answer:** `runtime` scope means the dependency is needed to RUN the app but not to COMPILE it — your code never directly calls `org.postgresql.Driver`; JDBC's `DriverManager` loads it reflectively via the Service Provider Interface. Marking it `runtime` keeps the compile-time classpath leaner and documents intent: it's a pluggable implementation detail, not an API your code depends on directly.

**F4.** What is the actual precedence order when the same property (e.g., `spring.datasource.url`) is set in `application.properties`, an OS environment variable, AND a command-line argument?

> **Answer:** Command-line arguments win first, followed by environment variables, followed by profile-specific `application-{profile}.properties`, followed by the base `application.properties`. This layered precedence is exactly what lets a packaged JAR be reused unmodified across environments — e.g., Kubernetes overrides `SPRING_DATASOURCE_URL` as an env var without rebuilding the image.

**F5.** If you exclude `spring-boot-starter-tomcat` but forget to add any other web server starter (Jetty/Undertow), what happens when you run the app?

> **Answer:** Spring Boot still detects `spring-boot-starter-web` on the classpath and expects a Servlet web `ApplicationContext`, but finds no `ServletWebServerFactory` bean (no embedded container implementation present). It throws `ApplicationContextException: Unable to start ServletWebServerApplicationContext due to missing ServletWebServerFactory bean` at startup — a fail-fast error, not a silent fallback.

**F6.** How does `SpringApplication.run()` decide whether to bootstrap a Servlet, Reactive, or plain (non-web) `ApplicationContext`?

> **Answer:** It inspects the classpath via `WebApplicationType.deduceFromClasspath()`: if `DispatcherHandler` (WebFlux) is present AND `DispatcherServlet` (Servlet MVC) is absent → REACTIVE; if neither is present → NONE (plain CLI app); otherwise → SERVLET. This is why having BOTH `spring-boot-starter-web` and `spring-boot-starter-webflux` on the classpath together defaults to a Servlet app unless you explicitly call `SpringApplication.setWebApplicationType(...)`.

**F7.** Why isn't `mvn clean package` alone enough to produce a runnable Spring Boot JAR, and what plugin actually makes it runnable?

> **Answer:** Plain `mvn package` produces a *thin* JAR — just your compiled classes, like any ordinary Java library, with no `Main-Class`/`Start-Class` manifest wiring and no bundled dependencies. The `spring-boot-maven-plugin`'s `repackage` goal (bound to the `package` phase by the Spring Initializr POM) repackages that thin JAR into the runnable fat/uber JAR with the nested `BOOT-INF` structure and Spring Boot's `Launcher` classes. Skip/exclude this plugin and `java -jar` fails with `NoClassDefFoundError`.

**F8.** Your team wants sub-100ms cold starts for a Spring Boot 3 service on Kubernetes. What two Java 21 / Spring Boot 3 features would you evaluate, and what's the trade-off of each?

> **Answer:** (1) **GraalVM Native Image** (via `spring-boot:build-image` / `native-maven-plugin`) compiles ahead-of-time to a native binary — near-instant startup (tens of ms) and lower memory, at the cost of longer build times and reflection/proxy configuration overhead (`reflect-config.json`). (2) **AppCDS/CDS**, which Spring Boot 3.3+ can auto-generate, caches JVM class metadata to speed up normal JIT-mode startup without going fully native. Native image gives the biggest win but the steepest compatibility tax; CDS is a safer incremental improvement.

````
