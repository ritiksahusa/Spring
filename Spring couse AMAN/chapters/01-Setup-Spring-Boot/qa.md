# Chapter 1 — Q&A, MCQ & Practice Questions

> **Topic:** Setup a Spring Boot Project in IntelliJ IDEA
> Review these questions after reading `notes.md`. All answers are at the bottom.

---

## Part A — Short Answer Questions

<a id="q1"></a>**Q1.** What does IDE stand for, and name three things you can do inside an IDE? [↓ Answer](#a1)

<a id="q2"></a>**Q2.** What is the URL of the Spring project bootstrapper tool, and what does it generate? [↓ Answer](#a2)

<a id="q3"></a>**Q3.** What is the difference between IntelliJ IDEA Community and Ultimate editions? [↓ Answer](#a3)

<a id="q4"></a>**Q4.** What is a "starter dependency" in Spring Boot? Give one example. [↓ Answer](#a4)

<a id="q5"></a>**Q5.** What is the purpose of the `pom.xml` file? [↓ Answer](#a5)

<a id="q6"></a>**Q6.** What is `application.properties` used for? Give two examples of what you can configure there. [↓ Answer](#a6)

<a id="q7"></a>**Q7.** What happens when you run the Spring Boot application and visit `http://localhost:8080/` without creating any controller? [↓ Answer](#a7)

<a id="q8"></a>**Q8.** What is the default embedded web server in `spring-boot-starter-web`? [↓ Answer](#a8)

<a id="q9"></a>**Q9.** What is the minimum JDK version you need if your `pom.xml` has `<java.version>21</java.version>`? [↓ Answer](#a9)

<a id="q10"></a>**Q10.** What annotation marks a class as a REST controller, and what annotation maps an HTTP GET request to a method? [↓ Answer](#a10)

---

## Part B — Multiple Choice Questions (MCQ)

<a id="q11"></a>**Q11.** Which file manages dependencies and build configuration in a Maven-based Spring Boot project? [↓ Answer](#a11)

- A) `application.properties`
- B) `settings.gradle`
- C) `pom.xml`
- D) `build.gradle`

<a id="q12"></a>**Q12.** What does `spring-boot-starter-web` dependency include by default? [↓ Answer](#a12)

- A) Spring Data JPA + Hibernate
- B) Embedded Tomcat + Spring MVC + Jackson
- C) Spring Security + JWT
- D) Spring WebFlux + Reactor

<a id="q13"></a>**Q13.** Which packaging format is typically used for a pure Java backend Spring Boot application? [↓ Answer](#a13)

- A) WAR
- B) EAR
- C) ZIP
- D) JAR

<a id="q14"></a>**Q14.** What is the default port Spring Boot's embedded Tomcat listens on? [↓ Answer](#a14)

- A) 80
- B) 443
- C) 8080
- D) 3000

<a id="q15"></a>**Q15.** At Spring Initializr, the "Package Name" is auto-generated as a concatenation of which two fields? [↓ Answer](#a15)

- A) Name + Description
- B) Group + Artifact
- C) Language + Version
- D) Artifact + Java version

<a id="q16"></a>**Q16.** Which annotation in the main class triggers Spring Boot's auto-configuration and component scanning? [↓ Answer](#a16)

- A) `@Controller`
- B) `@EnableAutoConfiguration`
- C) `@SpringBootApplication`
- D) `@ComponentScan`

<a id="q17"></a>**Q17.** You want to switch from Tomcat to Jetty. What is the correct approach in `pom.xml`? [↓ Answer](#a17)

- A) Just add `spring-boot-starter-jetty` without removing anything
- B) Remove `spring-boot-starter-web` entirely and add Jetty
- C) Exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`, then add `spring-boot-starter-jetty`
- D) Set `server.engine=jetty` in `application.properties`

<a id="q18"></a>**Q18.** Where is your main business logic (Java source code) placed in a Maven Spring Boot project? [↓ Answer](#a18)

- A) `src/test/java/`
- B) `src/main/resources/`
- C) `src/main/java/`
- D) `resources/java/`

<a id="q19"></a>**Q19.** What library does Spring Boot use behind the scenes to convert Java objects to JSON automatically? [↓ Answer](#a19)

- A) Gson
- B) FastJSON
- C) Jackson (`jackson-databind`)
- D) JAXB

<a id="q20"></a>**Q20.** In which edition of IntelliJ IDEA would you typically find the Spring Boot-specific plugins used in large companies? [↓ Answer](#a20)

- A) Community Edition (free)
- B) Ultimate Edition (paid)
- C) Android Studio
- D) Eclipse

---

## Part C — Scenario / Code-Reading Questions

<a id="q21"></a>**Q21.** Look at this code:

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

What does `SpringApplication.run(...)` do? List at least 3 things it triggers internally. [↓ Answer](#a21)

---

<a id="q22"></a>**Q22.** A developer writes this controller:

```java
@RestController
public class GreetController {

    @GetMapping("/greet")
    public String greet() {
        return "Welcome!";
    }
}
```

When the app starts and someone visits `http://localhost:8080/`, what response do they get, and why? [↓ Answer](#a22)

---

<a id="q23"></a>**Q23.** A developer has `<java.version>17</java.version>` in `pom.xml` but has JDK 11 selected in IntelliJ IDEA's SDK settings. What problem will occur? [↓ Answer](#a23)

---

<a id="q24"></a>**Q24.** Study this `pom.xml` snippet:

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

What embedded server will this application use? Explain why. [↓ Answer](#a24)

---

<a id="q25"></a>**Q25.** A teammate complains: "I added the `spring-boot-starter-web` dependency but I have no idea why Tomcat started. I only wanted MVC controllers!" 

Explain why Tomcat starts automatically, and what they could do if they only want the web layer without Tomcat. [↓ Answer](#a25)

---

## Part D — Fill in the Blanks

<a id="q26"></a>**Q26.** The `src/test/java` folder is used for ________ code, and it is separate from the business logic in `src/main/java`. [↓ Answer](#a26)

<a id="q27"></a>**Q27.** `spring-boot-starter-web` is called a ________ because it bundles several individual dependencies together. [↓ Answer](#a27)

<a id="q28"></a>**Q28.** The default error page shown by Spring Boot when no mapping exists is called the ________ Error Page. [↓ Answer](#a28)

<a id="q29"></a>**Q29.** The `application.properties` file is located under `src/main/________`. [↓ Answer](#a29)

<a id="q30"></a>**Q30.** To download JDK from within IntelliJ IDEA, you press ________ (Mac) or ________ (Windows/Linux) to open Project Structure. [↓ Answer](#a30)

---

## Answers

### Part A
<a id="a1"></a>**1.** IDE = Integrated Development Environment. Write, compile, run, debug, test — all in one place. [↑ Question](#q1)

<a id="a2"></a>**2.** `start.spring.io`. It generates a boilerplate (starter) Spring Boot project zip file. [↑ Question](#q2)

<a id="a3"></a>**3.** Community is free with basic Java support. Ultimate is paid (30-day trial) and includes Spring-specific plugins, database tools, profiling, etc. [↑ Question](#q3)

<a id="a4"></a>**4.** A starter bundles multiple related dependencies. Example: `spring-boot-starter-web` includes Tomcat, Spring MVC, Jackson. [↑ Question](#q4)

<a id="a5"></a>**5.** `pom.xml` is Maven's project descriptor — it defines dependencies, Java version, plugins, and packaging type. [↑ Question](#q5)

<a id="a6"></a>**6.** Server port (`server.port`), database URL/credentials (`spring.datasource.*`), logging levels, profile settings, etc. [↑ Question](#q6)

<a id="a7"></a>**7.** Spring Boot shows the default "Whitelabel Error Page" because no mapping exists for `/` or `/error`. [↑ Question](#q7)

<a id="a8"></a>**8.** Apache Tomcat (embedded). [↑ Question](#q8)

<a id="a9"></a>**9.** JDK 21 or higher (must be ≥ the version declared in pom.xml). [↑ Question](#q9)

<a id="a10"></a>**10.** `@RestController` marks the class; `@GetMapping("/path")` maps GET HTTP requests. [↑ Question](#q10)

### Part B
<a id="a11"></a>**11.** **C** — `pom.xml` [↑ Question](#q11)

<a id="a12"></a>**12.** **B** — Embedded Tomcat + Spring MVC + Jackson [↑ Question](#q12)

<a id="a13"></a>**13.** **D** — JAR [↑ Question](#q13)

<a id="a14"></a>**14.** **C** — 8080 [↑ Question](#q14)

<a id="a15"></a>**15.** **B** — Group + Artifact [↑ Question](#q15)

<a id="a16"></a>**16.** **C** — `@SpringBootApplication` [↑ Question](#q16)

<a id="a17"></a>**17.** **C** — Exclude Tomcat, add Jetty starter [↑ Question](#q17)

<a id="a18"></a>**18.** **C** — `src/main/java/` [↑ Question](#q18)

<a id="a19"></a>**19.** **C** — Jackson (`jackson-databind`) [↑ Question](#q19)

<a id="a20"></a>**20.** **B** — Ultimate Edition [↑ Question](#q20)

### Part C
<a id="a21"></a>**21.** `SpringApplication.run(...)` triggers: (1) Creates the Spring ApplicationContext, (2) Starts embedded Tomcat server, (3) Performs auto-configuration (scans classpath, configures beans), (4) Runs component scan to discover `@Component`/`@Service`/`@Repository`/`@Controller` classes, (5) Prints the Spring Boot banner. [↑ Question](#q21)

<a id="a22"></a>**22.** They get the **Whitelabel Error Page**. The controller only maps `/greet`, not `/`. When no mapping is found for `/`, Spring redirects to `/error`. Since no `/error` mapping exists either, Spring Boot returns its default error page. [↑ Question](#q22)

<a id="a23"></a>**23.** The build will fail or produce errors because the JDK in IntelliJ (v11) is older than the required version (v17). Java maintains backward compatibility but NOT forward compatibility — you cannot compile Java 17 code with a Java 11 JDK. [↑ Question](#q23)

<a id="a24"></a>**24.** The application will use **Jetty**. By excluding `spring-boot-starter-tomcat` from `spring-boot-starter-web`, Tomcat's auto-configuration is removed. Adding `spring-boot-starter-jetty` provides Jetty, and Spring Boot's auto-configuration detects Jetty on the classpath and starts it instead. [↑ Question](#q24)

<a id="a25"></a>**25.** Tomcat starts automatically because `spring-boot-starter-web` includes `spring-boot-starter-tomcat` as a transitive dependency, and Spring Boot auto-configures it. To keep the web layer without Tomcat, the developer should exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`. They could also use Spring WebFlux (`spring-boot-starter-webflux`) if they want a reactive, non-servlet setup. [↑ Question](#q25)

### Part D
<a id="a26"></a>**26.** testing / unit testing / integration testing [↑ Question](#q26)

<a id="a27"></a>**27.** starter dependency [↑ Question](#q27)

<a id="a28"></a>**28.** Whitelabel [↑ Question](#q28)

<a id="a29"></a>**29.** resources [↑ Question](#q29)

<a id="a30"></a>**30.** **⌘ + ;** (Mac) / **Ctrl + ;** (Windows/Linux) [↑ Question](#q30)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** Spring Boot produces a "fat/uber JAR". How does its internal structure differ from a normal JAR, and how does the JVM actually run classes nested inside another JAR (which a plain `java -cp` cannot do)? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** A Spring Boot fat JAR contains `BOOT-INF/classes` (your compiled classes), `BOOT-INF/lib` (all dependency JARs, un-extracted), and `org/springframework/boot/loader/*` (Spring Boot's own loader classes). A standard JVM classloader cannot load classes from a JAR nested inside another JAR, so Spring Boot bundles a custom `LaunchedURLClassLoader` and a `JarLauncher` (declared as `Main-Class` in `MANIFEST.MF`). Running `java -jar app.jar` actually starts `JarLauncher` first, which builds a classloader capable of reading nested JARs, and only then invokes your real `main()` method (recorded as `Start-Class` in the manifest). [↑ Question](#qf1)

<a id="qf2"></a>**F2.** Two Spring Boot apps try to bind to port 8080 on the same machine. What actually happens, and how do you make the port dynamic for parallel integration tests? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** The second app fails at startup with `BindException: Address already in use` while embedded Tomcat opens its server socket — this is a fatal startup exception that prevents the `ApplicationContext` from refreshing, not a runtime error. For dynamic ports (common in tests), set `server.port=0` so the OS assigns a free ephemeral port, retrievable via `@LocalServerPort private int port;` inside a `@SpringBootTest(webEnvironment = RANDOM_PORT)` test. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** Why is the PostgreSQL JDBC driver dependency marked with `<scope>runtime</scope>`, and what does that scope actually change? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** `runtime` scope means the dependency is needed to RUN the app but not to COMPILE it — your code never directly calls `org.postgresql.Driver`; JDBC's `DriverManager` loads it reflectively via the Service Provider Interface. Marking it `runtime` keeps the compile-time classpath leaner and documents intent: it's a pluggable implementation detail, not an API your code depends on directly. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** What is the actual precedence order when the same property (e.g., `spring.datasource.url`) is set in `application.properties`, an OS environment variable, AND a command-line argument? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** Command-line arguments win first, followed by environment variables, followed by profile-specific `application-{profile}.properties`, followed by the base `application.properties`. This layered precedence is exactly what lets a packaged JAR be reused unmodified across environments — e.g., Kubernetes overrides `SPRING_DATASOURCE_URL` as an env var without rebuilding the image. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** If you exclude `spring-boot-starter-tomcat` but forget to add any other web server starter (Jetty/Undertow), what happens when you run the app? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** Spring Boot still detects `spring-boot-starter-web` on the classpath and expects a Servlet web `ApplicationContext`, but finds no `ServletWebServerFactory` bean (no embedded container implementation present). It throws `ApplicationContextException: Unable to start ServletWebServerApplicationContext due to missing ServletWebServerFactory bean` at startup — a fail-fast error, not a silent fallback. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** How does `SpringApplication.run()` decide whether to bootstrap a Servlet, Reactive, or plain (non-web) `ApplicationContext`? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** It inspects the classpath via `WebApplicationType.deduceFromClasspath()`: if `DispatcherHandler` (WebFlux) is present AND `DispatcherServlet` (Servlet MVC) is absent → REACTIVE; if neither is present → NONE (plain CLI app); otherwise → SERVLET. This is why having BOTH `spring-boot-starter-web` and `spring-boot-starter-webflux` on the classpath together defaults to a Servlet app unless you explicitly call `SpringApplication.setWebApplicationType(...)`. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** Why isn't `mvn clean package` alone enough to produce a runnable Spring Boot JAR, and what plugin actually makes it runnable? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** Plain `mvn package` produces a *thin* JAR — just your compiled classes, like any ordinary Java library, with no `Main-Class`/`Start-Class` manifest wiring and no bundled dependencies. The `spring-boot-maven-plugin`'s `repackage` goal (bound to the `package` phase by the Spring Initializr POM) repackages that thin JAR into the runnable fat/uber JAR with the nested `BOOT-INF` structure and Spring Boot's `Launcher` classes. Skip/exclude this plugin and `java -jar` fails with `NoClassDefFoundError`. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** Your team wants sub-100ms cold starts for a Spring Boot 3 service on Kubernetes. What two Java 21 / Spring Boot 3 features would you evaluate, and what's the trade-off of each? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** (1) **GraalVM Native Image** (via `spring-boot:build-image` / `native-maven-plugin`) compiles ahead-of-time to a native binary — near-instant startup (tens of ms) and lower memory, at the cost of longer build times and reflection/proxy configuration overhead (`reflect-config.json`). (2) **AppCDS/CDS**, which Spring Boot 3.3+ can auto-generate, caches JVM class metadata to speed up normal JIT-mode startup without going fully native. Native image gives the biggest win but the steepest compatibility tax; CDS is a safer incremental improvement. [↑ Question](#qf8)

````
