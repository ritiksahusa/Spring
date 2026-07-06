# Chapter 1: Setup a Spring Boot Project in IntelliJ IDEA

> **Video time:** `0:01` → `21:47`

---

## 1. IntelliJ IDEA — The IDE

**IntelliJ IDEA** is an **Integrated Development Environment (IDE)** — one place to write, compile, run, debug, and test your code.

| Edition | Cost | Who uses it |
|---------|------|-------------|
| **Community** | Free | Students, personal projects |
| **Ultimate** | Paid (30-day trial) | Professional developers, companies |

> Industry tip: Most companies (including Amazon) provide the Ultimate edition to employees. Community edition is sufficient for learning.

**Download:** [jetbrains.com/idea](https://www.jetbrains.com/idea/)  
Choose the right installer for your OS:
- **Windows** → `.exe`
- **macOS (Apple Silicon)** → ARM `.dmg`
- **macOS (Intel chip)** → Intel `.dmg`
- **Linux** → `.tar.gz` (extract and run)

---

## 2. Spring Initializr — Project Bootstrapper

URL: **[start.spring.io](https://start.spring.io)**

Spring Initializr generates a **boilerplate project** — the standard folder structure and minimum config files every Spring Boot project needs, so you don't start from a blank `Main.java`.

### Configuration options explained

| Field | What to choose | Why |
|-------|---------------|-----|
| **Project (Build tool)** | Maven | Maven is used in most legacy enterprise codebases. Gradle is faster but newer. |
| **Language** | Java | Production standard; Kotlin is an excellent alternative |
| **Spring Boot version** | Latest stable | Always pick the latest non-SNAPSHOT |
| **Group** | `com.yourname` | Reverse domain name convention (like `com.codingshuttle`) |
| **Artifact / Name** | `learning-spring-boot` | This becomes the JAR file name |
| **Package name** | Auto-filled as `Group + Artifact` | This is your root Java package |
| **Packaging** | JAR | WAR is for web server deployments with HTML/static files; pure Java backends use JAR |
| **Java version** | 21 | LTS version — pick 21 or above |

### Adding Dependencies

Dependencies are pre-built libraries your project needs. Added via the **"Add Dependencies"** button in Initializr.

Example — clicking **"Spring Web"** adds:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`spring-boot-starter-web` is a **starter** — it bundles multiple related dependencies so you don't manage them individually:

- `spring-boot-starter` (core auto-configuration)
- `spring-web` (Spring MVC)
- `spring-webmvc` (DispatcherServlet, controllers)
- `jackson-databind` (JSON serialization)
- `tomcat-embed-core` (**embedded Apache Tomcat server**)

---

## 3. Project Structure

After unzipping and opening in IntelliJ IDEA:

```
learning-spring-boot/
├── .idea/                        ← IntelliJ config (ignore)
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties   ← Maven wrapper config
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/codingshuttle/youtube/
│   │   │       └── LearningSpringBootApplication.java   ← ENTRY POINT
│   │   └── resources/
│   │       └── application.properties   ← Configuration file
│   └── test/
│       └── java/
│           └── com/codingshuttle/youtube/
│               └── LearningSpringBootApplicationTests.java
└── pom.xml                       ← Maven project descriptor
```

### Key files

#### `LearningSpringBootApplication.java` — Entry Point

```java
@SpringBootApplication
public class LearningSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(LearningSpringBootApplication.class, args);
    }
}
```

- The `main()` method is the entry point for the entire Spring Boot application.
- `SpringApplication.run(...)` starts the Spring context, auto-configures everything, and starts the embedded Tomcat server.

#### `pom.xml` — Maven Project Object Model

Manages:
- Dependencies (what libraries to download)
- Build plugins
- Java version (`<java.version>21</java.version>`)
- Packaging (JAR/WAR)

To change the JDK version used, edit `<properties>`:
```xml
<properties>
    <java.version>21</java.version>
</properties>
```

#### `application.properties` — Runtime Configuration

Central place to configure your app:
```properties
# Change server port (default 8080)
server.port=9090

# Database connection
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
```

---

## 4. Setting Up the JDK in IntelliJ

1. Press **⌘ + ;** (Mac) or **Ctrl + ;** (Windows/Linux)
2. Go to **Project Settings → SDK**
3. If no JDK is listed, click **"Download JDK"**
4. Choose version **21** (or any 21+), vendor: **Azul Zulu Community** (free, popular)
5. Click **OK** → IntelliJ re-indexes the project

> Rule: The JDK version in IntelliJ must be **≥** the `<java.version>` in `pom.xml`.

---

## 5. Running the Application

Click the **▶ Run button** next to `main()` or use **Shift + F10** (Windows) / **Ctrl + R** (Mac).

**Startup log output:**
```
Tomcat started on port 8080 (http) with context path '/'
Started LearningSpringBootApplication in 1.234 seconds
```

The app is now live at `http://localhost:8080`.

**Default behavior without any endpoint:**
- Visit `http://localhost:8080/` → No mapping → redirects to `/error`
- Spring Boot shows its default Whitelabel Error Page

---

## 6. Writing Your First API Endpoint

```java
@RestController
public class HelloController {

    @GetMapping("/")
    public String hello() {
        return "Hello World from Anuj!";
    }
}
```

**What's happening:**
- `@RestController` — marks this class as a Spring MVC controller that returns data directly (not a view template)
- `@GetMapping("/")` — maps HTTP GET requests to the `/` URL to this method
- The return value (`String`) is written directly to the HTTP response body

Restart the app → visit `http://localhost:8080/` → you see `Hello World from Anuj!`

---

## 7. How a REST Response is Serialized

When a method returns an **object** (not a String), Spring automatically converts it to **JSON** using **Jackson** (`jackson-databind`) — included automatically with `spring-boot-starter-web`.

```java
@GetMapping("/user")
public User getUser() {
    return new User(1, "Ritik", "ritik@example.com");
    // → {"id":1,"name":"Ritik","email":"ritik@example.com"}
}
```

---

## 8. Switching Embedded Servers (Advanced)

Default embedded server is **Apache Tomcat**. To switch to **Jetty**:

```xml
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
```

---

## 9. Maven vs Gradle — Quick Comparison

| | Maven | Gradle |
|--|-------|--------|
| Config file | `pom.xml` (XML) | `build.gradle` (Groovy/Kotlin DSL) |
| Maturity | Older, very stable | Newer, faster builds |
| Industry prevalence | Dominant in enterprise legacy code | Growing in new projects |
| Learning curve | Lower for beginners | Slightly steeper |

**Rule of thumb:** Use Maven for this course. Most enterprise jobs use Maven.

---

## 10. Key Concepts Summary

| Concept | Meaning |
|---------|---------|
| **IDE** | Integrated Development Environment (IntelliJ IDEA) |
| **Spring Initializr** | Web tool to generate project boilerplate |
| **Boilerplate** | Standard starter code every project needs |
| **Build tool** | Maven/Gradle — compile, test, package, manage dependencies |
| **Dependency** | A third-party library your project needs |
| **Starter** | A bundled set of related dependencies (e.g., `spring-boot-starter-web`) |
| **Embedded server** | Tomcat/Jetty bundled inside your JAR — no separate server install needed |
| **pom.xml** | Maven's project descriptor — defines dependencies, plugins, Java version |
| **application.properties** | Runtime config file for Spring Boot |
| **@SpringBootApplication** | Meta-annotation that enables component scan, auto-config, and more |
| **@RestController** | Marks a class as a web controller returning data |
| **@GetMapping** | Maps HTTP GET to a method |
| **JAR vs WAR** | JAR = standalone Java app; WAR = web archive for servlet containers |
