# Java 17 → 21 Migration Guide

## Table of contents
1. Removed / changed behavior (blockers)
2. Build config changes
3. Code refactoring examples
4. New features to adopt (recommended)
5. JVM flags & runtime notes

---

## 1. Removed / changed behavior (blockers)

### UTF-8 by default (Java 18, JEP 400)

Starting in Java 18, `Charset.defaultCharset()` returns UTF-8 on **all** operating
systems, including Windows (which previously defaulted to the system locale encoding
like `windows-1252`).

**Impact**: file I/O or string encoding that relied on `Charset.defaultCharset()`
or omitted charset arguments may produce different results on Windows.

**Search pattern**: `new FileReader(`, `new FileWriter(`, `new InputStreamReader(` (without explicit charset), `Charset.defaultCharset()`, `new String(bytes)` (without charset), `.getBytes()` (without charset).

**Fix**: explicitly specify `StandardCharsets.UTF_8` (or the intended charset) in
all I/O operations. If the code already does this, no change is needed.

### `finalize()` deprecated for removal (Java 18, JEP 421)

The `Object.finalize()` method is deprecated for removal. Classes that override it
will trigger warnings, and future JDKs will remove the method entirely.

**Search pattern**: `protected void finalize()`, `@Override.*finalize`.

**Fix**: replace with `try-with-resources` (for `Closeable` resources) or the
`java.lang.ref.Cleaner` API for custom cleanup.

### Thread control methods throw unconditionally

`Thread.stop()`, `Thread.suspend()`, and `Thread.resume()` now throw
`UnsupportedOperationException` unconditionally. They are no longer just deprecated.

**Search pattern**: `.stop()`, `.suspend()`, `.resume()` on Thread instances.

**Fix**: redesign using cooperative interruption (`Thread.interrupt()` + checking
`Thread.isInterrupted()`), or use `ExecutorService` with proper shutdown.

### `java.net.URL` constructors deprecated (Java 20)

`new URL(...)` constructors are deprecated. Use `URI.create(...).toURL()` instead.

**Search pattern**: `new URL(`.

```diff
- URL url = new URL("https://example.com/api");
+ URL url = URI.create("https://example.com/api").toURL();
```

### SecurityManager further degraded

Some `SecurityManager` APIs were removed or degraded further. If you still have
Security Manager code from Java 17, it is increasingly non-functional.

---

## 2. Build config changes

### Maven

```xml
<properties>
  <maven.compiler.release>21</maven.compiler.release>
</properties>

<build>
  <plugins>
    <!-- Minimum 3.11+ for Java 21 -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.12.1</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
    </plugin>
  </plugins>
</build>
```

### Gradle (Groovy DSL)

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

### Gradle (Kotlin DSL)

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}
```

Ensure Gradle wrapper is at least **8.4** (for Java 21 support); **8.5+** recommended.

---

## 3. Code refactoring examples

### Replacing `new URL(...)` with URI

```diff
- import java.net.URL;
+ import java.net.URI;
+ import java.net.URL;

- URL endpoint = new URL("https://api.example.com/v2/data");
+ URL endpoint = URI.create("https://api.example.com/v2/data").toURL();
```

### Replacing `finalize()` with Cleaner

```diff
- public class ResourceHolder {
-     private long nativePtr;
-
-     @Override
-     protected void finalize() throws Throwable {
-         try {
-             releaseNative(nativePtr);
-         } finally {
-             super.finalize();
-         }
-     }
- }
+ public class ResourceHolder implements AutoCloseable {
+     private static final Cleaner CLEANER = Cleaner.create();
+     private final long nativePtr;
+     private final Cleaner.Cleanable cleanable;
+
+     public ResourceHolder(long ptr) {
+         this.nativePtr = ptr;
+         // capture only the ptr, not 'this', to avoid preventing GC
+         this.cleanable = CLEANER.register(this, () -> releaseNative(ptr));
+     }
+
+     @Override
+     public void close() {
+         cleanable.clean();
+     }
+ }
```

### Sequenced collections

```diff
- // Getting first and last elements
- String first = list.get(0);
- String last = list.get(list.size() - 1);
+ String first = list.getFirst();
+ String last = list.getLast();

- // Iterating in reverse
- ListIterator<String> it = list.listIterator(list.size());
- while (it.hasPrevious()) { /* ... */ }
+ for (String s : list.reversed()) { /* ... */ }
```

### Pattern matching for switch

```diff
- if (obj instanceof Integer i) {
-     handleInt(i);
- } else if (obj instanceof String s) {
-     handleString(s);
- } else if (obj instanceof Double d) {
-     handleDouble(d);
- } else {
-     handleUnknown(obj);
- }
+ switch (obj) {
+     case Integer i -> handleInt(i);
+     case String s  -> handleString(s);
+     case Double d  -> handleDouble(d);
+     default        -> handleUnknown(obj);
+ }
```

### Record patterns (destructuring)

```diff
- if (obj instanceof Point p) {
-     int x = p.x();
-     int y = p.y();
-     System.out.println("x=" + x + ", y=" + y);
- }
+ if (obj instanceof Point(int x, int y)) {
+     System.out.println("x=" + x + ", y=" + y);
+ }
```

### Virtual threads (replacing thread pool for I/O-bound work)

```diff
- // Fixed thread pool for blocking I/O tasks
- ExecutorService executor = Executors.newFixedThreadPool(200);
- for (Request req : requests) {
-     executor.submit(() -> processBlocking(req));
- }
+ // Virtual threads: one per task, no pool sizing needed
+ try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
+     for (Request req : requests) {
+         executor.submit(() -> processBlocking(req));
+     }
+ }
```

Note: virtual threads are most beneficial for **I/O-bound** workloads (HTTP calls,
database queries, file I/O). For CPU-bound tasks, platform threads with a fixed pool
remain appropriate.

### Explicit charset in I/O (fixing UTF-8 default change)

```diff
- // Relied on platform default charset (risky since Java 18)
- BufferedReader reader = new BufferedReader(new FileReader("data.csv"));
+ // Explicit charset — works identically across all JDK versions
+ BufferedReader reader = new BufferedReader(
+     new FileReader("data.csv", StandardCharsets.UTF_8));

- byte[] bytes = text.getBytes();
+ byte[] bytes = text.getBytes(StandardCharsets.UTF_8);
```

---

## 4. New features to adopt (recommended)

- **Virtual Threads** (JEP 444): replace traditional thread pools for I/O-bound workloads. Especially impactful in web servers (Spring Boot 3.2+ has built-in support for Tomcat/Jetty virtual threads).
- **Sequenced Collections** (JEP 431): `getFirst()`, `getLast()`, `reversed()` on `List`, `Set`, `Map` — cleaner than index arithmetic.
- **Pattern Matching for switch** (JEP 441): replace long `instanceof` chains with switch expressions.
- **Record Patterns** (JEP 440): destructure records in `instanceof` and switch for concise data extraction.

**Avoid adopting**:
- **String Templates** (JEP 430): these were previewed in Java 21 but **removed in later versions**. Do not adopt them to avoid future migration debt.

---

## 5. JVM flags & runtime notes

- **Generational ZGC** (JEP 439): if using ZGC, enable the generational mode for significantly better performance: `-XX:+UseZGC -XX:+ZGenerational`. In Java 23+, generational mode becomes the default.
- **`--add-opens` audit**: revisit all `--add-opens` flags. After upgrading libraries over the 11→17 cycle, many workarounds can be removed.
- **Removed JVM flags**: audit startup scripts for flags removed between 17 and 21. The JVM will refuse to start with unrecognized flags.
- **Virtual thread pinning**: be aware that `synchronized` blocks can pin virtual threads to platform threads, reducing their benefit. Prefer `ReentrantLock` in hot paths if using virtual threads extensively.
