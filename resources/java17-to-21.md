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

### Locale data upgraded to CLDR 42 — date/time formats changed (Java 20)

This is the most frequently missed 17 → 21 blocker. JDK 20 upgraded the CLDR locale
data to version 42, which **changed the output of standard date/time formatters**:

| Change | Before (JDK 17) | After (JDK 20+) |
|---|---|---|
| Space before AM/PM | `1:30 PM` (U+0020 space) | `1:30 PM` (U+202F **narrow no-break space**) |
| Date/time separator | `Jan 1, 2024 at 1:30 PM` | `Jan 1, 2024, 1:30 PM` (`" at "` no longer used) |
| First day of week (CN) | Sunday | Monday |

**Impact**: any code that string-compares, parses, or asserts on formatted dates will
break — and it breaks *silently* in production but *loudly* in test suites, because the
narrow no-break space is visually identical to a normal space in diff output. Symptoms
are baffling assertion failures like `expected:<1:30 PM> but was:<1:30 PM>`.

**Search pattern**: `DateTimeFormatter.ofLocalizedDate`, `ofLocalizedTime`,
`ofLocalizedDateTime`, `FormatStyle.`, `DateFormat.getDateTimeInstance`,
`SimpleDateFormat` with `a` (am/pm) patterns, and any test asserting on formatted
date strings.

**Fix** (in order of preference):

1. **Use explicit patterns** instead of localized styles where the exact output matters
   (e.g., machine-readable output, log lines, file names, API payloads):

   ```diff
   - DateTimeFormatter fmt = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT);
   + DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
   ```

   Better still, use `DateTimeFormatter.ISO_LOCAL_DATE_TIME` for interchange formats.

2. **Normalize before comparing** in tests that must keep localized output:

   ```java
   String normalized = formatted.replace('\u202F', ' ').replace('\u00A0', ' ');
   ```

3. **Do NOT reach for `-Djava.locale.providers=COMPAT`.** It still works in Java 21 but
   prints a deprecation warning, and it was **removed entirely in JDK 23** — so it is a
   dead end that will break again on the very next hop.

> Note: CLDR data is refreshed in *every* JDK release, so localized formatting output is
> never a stable contract. Treat localized formats as display-only.

### `finalize()` deprecated for removal (Java 18, JEP 421)

The `Object.finalize()` method is deprecated for removal. Classes that override it
will trigger warnings, and future JDKs will remove the method entirely.

**Search pattern**: `protected void finalize()`, `@Override.*finalize`.

**Fix**: replace with `try-with-resources` (for `Closeable` resources) or the
`java.lang.ref.Cleaner` API for custom cleanup.

### Thread control methods throw unconditionally (Java 20 / 21)

These long-deprecated methods no longer merely warn — they now throw
`UnsupportedOperationException` on every call:

| Method | Throws unconditionally since |
|---|---|
| `Thread.stop()` | Java 20 |
| `Thread.suspend()` / `Thread.resume()` | Java 21 |
| `ThreadGroup.stop()` / `suspend()` / `resume()` | Java 20–21 |

**Impact**: this is a *runtime* failure, not a compile error — code that called these in
a rarely-exercised shutdown or timeout path will compile cleanly and then throw in
production. Grep for them even if the compiler is silent.

**Search pattern**: `.stop()`, `.suspend()`, `.resume()` on `Thread` or `ThreadGroup`
instances, `ThreadDeath`.

**Fix**: redesign using cooperative interruption (`Thread.interrupt()` + checking
`Thread.isInterrupted()`), or use `ExecutorService` with `shutdownNow()` and
`awaitTermination(...)`.

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
