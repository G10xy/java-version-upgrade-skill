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

### Version string format changed (Java 9, JEP 223)

The `java.version` system property no longer starts with `1.`:

| JDK | `java.version` | `java.specification.version` |
|---|---|---|
| 8 | `1.8.0_392` | `1.8` |
| 11 | `11.0.22` | `11` |

**Impact**: hand-rolled version checks silently misbehave. `"11.0.22".startsWith("1.8")`
is `false` (fine), but `version.substring(0, 3)` yields `"11."`, and code that parses the
minor version out of `1.X` reads `0` instead of `11` — often concluding it is running on
"Java 0" or falling into a Java 5 compatibility branch.

**Search pattern**: `System.getProperty("java.version")`, `java.specification.version`,
`startsWith("1.")`, `substring(0, 3)` on a version string.

```diff
- String v = System.getProperty("java.version");
- int major = Integer.parseInt(v.split("\\.")[1]);   // breaks: "11.0.22" -> 0
+ int major = Runtime.version().feature();            // 11 — JEP 223 API, Java 10+
```

(`Runtime.version()` itself arrived in Java 9, where the accessor was named `major()`;
`feature()` replaced it in Java 10. If the code must still compile on Java 8 during the
transition, parse `java.specification.version` instead — it is `1.8` on 8 and `11` on 11.)

Also check shell scripts that grep `java -version` output, and old copies of libraries
that do their own detection (older Lombok, ASM, Groovy, and Jetty are common offenders —
upgrade them rather than patching around it).

### The application class loader is no longer a `URLClassLoader` (Java 9)

In Java 8 the system/app class loader was a `URLClassLoader`; since Java 9 it is an
internal `AppClassLoader` type that does not extend it. Casting throws
`ClassCastException` at runtime.

**Search pattern**: `(URLClassLoader)`, `ClassLoader.getSystemClassLoader()`,
`addURL`, code that injects JARs onto the classpath at runtime.

```diff
- URLClassLoader cl = (URLClassLoader) ClassLoader.getSystemClassLoader();
- Method m = URLClassLoader.class.getDeclaredMethod("addURL", URL.class);
- m.setAccessible(true);
- m.invoke(cl, jarUrl);   // ClassCastException on Java 9+
+ // Create your own loader for the extra JARs instead of mutating the system one
+ URLClassLoader pluginLoader = new URLClassLoader(new URL[] { jarUrl },
+         ClassLoader.getSystemClassLoader());
```

To enumerate the classpath, read the `java.class.path` system property rather than
casting to `URLClassLoader` and calling `getURLs()`.

### `sun.misc.BASE64Encoder` / `BASE64Decoder` removed (Java 9)

These undocumented internal classes are gone.

```diff
- import sun.misc.BASE64Encoder;
- import sun.misc.BASE64Decoder;
- String encoded = new BASE64Encoder().encode(bytes);
- byte[] decoded = new BASE64Decoder().decodeBuffer(encoded);
+ import java.util.Base64;
+ String encoded = Base64.getEncoder().encodeToString(bytes);
+ byte[] decoded = Base64.getDecoder().decode(encoded);
```

Note `BASE64Encoder.encode()` inserted line breaks every 76 characters; if you must
preserve that, use `Base64.getMimeEncoder()` instead of `getEncoder()`.

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

If the project previously used JAXB (common), you must add it as an explicit dependency.
**Choose a namespace first** — this decision determines whether you touch source files:

**Option A — `jakarta.*` namespace (recommended, forward-looking).** Requires rewriting
every `javax.xml.bind.*` import, but is the only option still receiving updates and the
only one compatible with Jakarta EE 9+ / Spring Boot 3+:

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

**Option B — keep the `javax.*` namespace (zero source changes).** Use the 2.3.x line,
which still uses `javax.xml.bind`. Useful when you want to de-risk the 8 → 11 hop and
defer the namespace migration to a later change:

```xml
<dependencies>
  <dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>2.3.3</version> <!-- 2.3.x still uses the javax.xml.bind package -->
  </dependency>
  <dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.9</version>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

> Do not mix the two. Having both `javax.xml.bind` and `jakarta.xml.bind` API jars on the
> classpath produces confusing `ClassNotFoundException` / `JAXBException: Implementation
> of JAXB-API has not been found` errors at runtime. Pick one and verify with
> `mvn dependency:tree` that no transitive dependency drags in the other.

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
-     new HashMap<String, Integer>() {{ put("alice", 10); put("bob", 20); }});
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
