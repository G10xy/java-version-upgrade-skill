# Java 8 → 11 Migration Guide

## Table of contents
1. Removed / changed behavior (blockers)
2. Build config changes
3. Code refactoring examples
4. New features to adopt (recommended)
5. JVM & runtime notes

---

## 1. Removed / changed behavior (blockers)

### Java EE / CORBA modules removed from the JDK (Java 11)

The following modules were removed entirely. Code that imports from these packages
will fail to compile on Java 11:

| Removed module    | Common imports                         | Replacement dependency                         |
|-------------------|----------------------------------------|------------------------------------------------|
| `java.xml.bind`   | `javax.xml.bind.*` (JAXB)             | `jakarta.xml.bind:jakarta.xml.bind-api` + impl (e.g., `org.glassfish.jaxb:jaxb-runtime`) |
| `java.xml.ws`     | `javax.xml.ws.*` (JAX-WS)            | `jakarta.xml.ws:jakarta.xml.ws-api` + impl     |
| `java.activation` | `javax.activation.*`                  | `jakarta.activation:jakarta.activation-api`     |
| `java.corba`      | `org.omg.*`                           | No direct replacement; redesign away from CORBA |
| `java.transaction` | `javax.transaction.*`                 | `jakarta.transaction:jakarta.transaction-api`   |
| `java.xml.ws.annotation` | `javax.annotation.*`           | `jakarta.annotation:jakarta.annotation-api`     |

**Search pattern**: `import javax.xml.bind`, `import javax.xml.ws`, `import javax.activation`, `import org.omg.`, `import javax.annotation.`

### Java Platform Module System — JPMS (Java 9)

Stronger encapsulation can break reflection-heavy code. Symptoms:
- `InaccessibleObjectException` at runtime
- "illegal reflective access" warnings (warnings in 9–15, hard errors from 16+)

**Preferred fix**: upgrade the library doing the reflection. Most major frameworks
(Spring, Hibernate, Jackson) have JPMS-compatible versions.

**Temporary workaround**: add `--add-opens` flags to JVM arguments. Always document
these with a comment and a timeline to remove them:

```
# TEMPORARY — remove once Hibernate is upgraded to 6.x
--add-opens java.base/java.lang=ALL-UNNAMED
```

### JavaFX removed from the JDK (Java 11)

If the project uses `javafx.*` packages, add OpenJFX as an explicit dependency.

### Nashorn JavaScript engine deprecated (Java 11, removed in 15)

If using `javax.script` with Nashorn, plan migration to GraalVM JS or another engine.

### TLS / security defaults

Java 11 uses stricter TLS defaults. Re-test integrations that depend on older TLS
versions or specific cipher suites.

---

## 2. Build config changes

### Maven

```xml
<properties>
  <!-- Prefer 'release' over source+target; it also sets the boot classpath -->
  <maven.compiler.release>11</maven.compiler.release>
</properties>

<build>
  <plugins>
    <!-- Minimum 3.8+ for Java 11 support -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
    </plugin>
    <!-- Minimum 3.0+ for Java 11; 3.1+ recommended -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
    </plugin>
  </plugins>
</build>
```

If the project previously used JAXB (common), add these dependencies:

```xml
<dependencies>
  <dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>4.0.2</version>
  </dependency>
  <dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.5</version>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

### Gradle (Groovy DSL)

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(11)
    }
}

// Or the classic way:
// sourceCompatibility = JavaVersion.VERSION_11
// targetCompatibility = JavaVersion.VERSION_11
```

### Gradle (Kotlin DSL)

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(11))
    }
}
```

Ensure Gradle wrapper is at least **5.0** (for Java 11 support); **6.7+** recommended.

---

## 3. Code refactoring examples

### JAXB removal

```diff
- import javax.xml.bind.JAXBContext;
- import javax.xml.bind.Marshaller;
+ import jakarta.xml.bind.JAXBContext;
+ import jakarta.xml.bind.Marshaller;
```

### Replacing HttpURLConnection with the new HTTP Client

```diff
- import java.net.HttpURLConnection;
- import java.net.URL;
+ import java.net.URI;
+ import java.net.http.HttpClient;
+ import java.net.http.HttpRequest;
+ import java.net.http.HttpResponse;

- URL url = new URL("https://api.example.com/data");
- HttpURLConnection conn = (HttpURLConnection) url.openConnection();
- conn.setRequestMethod("GET");
- int status = conn.getResponseCode();
- // ... read InputStream manually
+ HttpClient client = HttpClient.newHttpClient();
+ HttpRequest request = HttpRequest.newBuilder()
+     .uri(URI.create("https://api.example.com/data"))
+     .GET()
+     .build();
+ HttpResponse<String> response = client.send(request,
+     HttpResponse.BodyHandlers.ofString());
+ int status = response.statusCode();
+ String body = response.body();
```

### Immutable collection factories

```diff
- List<String> items = Collections.unmodifiableList(
-     Arrays.asList("a", "b", "c"));
+ List<String> items = List.of("a", "b", "c");

- Map<String, Integer> scores = Collections.unmodifiableMap(
-     new HashMap<>() {{ put("alice", 10); put("bob", 20); }});
+ Map<String, Integer> scores = Map.of("alice", 10, "bob", 20);
```

### String utility methods

```diff
- if (str == null || str.trim().isEmpty()) {
+ if (str == null || str.isBlank()) {

- String content = new String(Files.readAllBytes(path), StandardCharsets.UTF_8);
+ String content = Files.readString(path);
```

### Optional improvements

```diff
- if (opt.isPresent()) {
-     doSomething(opt.get());
- } else {
-     doFallback();
- }
+ opt.ifPresentOrElse(this::doSomething, this::doFallback);
```

---

## 4. New features to adopt (recommended)

These are not required for the migration to succeed, but improve code quality:

- **Local-variable type inference** (`var`, Java 10): use for obvious types to reduce verbosity, especially with generics. Avoid in public API signatures and when it hurts readability.
- **New HTTP Client** (Java 11): prefer `java.net.http.HttpClient` over `HttpURLConnection` for HTTP/2 support and cleaner async APIs.
- **String improvements** (Java 11): `isBlank()`, `strip()`, `stripLeading()`, `stripTrailing()`, `lines()`, `repeat(n)`.
- **File I/O convenience** (Java 11): `Files.readString(Path)` / `Files.writeString(Path, ...)`.
- **Collections / Streams helpers** (Java 9–11): `List.of(...)`, `Set.of(...)`, `Map.of(...)`, `Optional.ifPresentOrElse(...)`, `Optional.or(...)`, `Optional.isEmpty()`, `Stream.takeWhile(...)`, `dropWhile(...)`, `ofNullable(...)`.
- **Flight Recorder** (Java 11): consider enabling JFR in production for low-overhead profiling.

---

## 5. JVM & runtime notes

- **Default GC is G1GC** (since Java 9). If you tuned ParallelGC or CMS in Java 8, re-baseline with G1. CMS flags are deprecated and will be removed in later JDKs.
- **Container-awareness** improved significantly vs. Java 8, but still validate memory limits and heap sizing in Docker/Kubernetes.
- **Removed JVM flags**: audit startup scripts for flags that no longer exist. The JVM will refuse to start if it encounters an unrecognized `-XX:` flag (use `-XX:+IgnoreUnrecognizedVMOptions` temporarily during migration if needed).
