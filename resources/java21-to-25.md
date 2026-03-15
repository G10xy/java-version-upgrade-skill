# Java 21 → 25 Migration Guide

## Table of contents
1. Removed / changed behavior (blockers)
2. Build config changes
3. Code refactoring examples
4. New features to adopt (recommended)
5. JVM flags & runtime notes

---

## 1. Removed / changed behavior (blockers)

### Security Manager permanently disabled (Java 24, JEP 486)

The Security Manager, deprecated since Java 17, is now **permanently disabled**.
It can no longer be enabled at startup (`-Djava.security.manager`) or at runtime
(`System.setSecurityManager(...)`). Multiple related APIs now throw unconditionally:

| Method / Class | Behavior in Java 24+ |
|---|---|
| `System.getSecurityManager()` | Always returns `null` |
| `System.setSecurityManager(...)` | Always throws `UnsupportedOperationException` |
| `AccessController.checkPermission(...)` | Always throws `AccessControlException` |
| `Policy.setPolicy(...)` | Always throws `UnsupportedOperationException` |
| `SecurityManager.check*(...)` | Always throws `SecurityException` |

**Search pattern**: `SecurityManager`, `setSecurityManager`, `getSecurityManager`,
`AccessController`, `doPrivileged`, `checkPermission`, `.policy` files,
`-Djava.security.manager` in startup scripts.

**Fix**: remove all Security Manager code. Replace security checks with
application-level authorization or container-level sandboxing. For
`AccessController.doPrivileged(...)` calls, simply inline the action — in Java 24+
these methods execute the action immediately but will be removed in a future release.

```diff
- AccessController.doPrivileged((PrivilegedAction<Void>) () -> {
-     doSensitiveOperation();
-     return null;
- });
+ doSensitiveOperation();
```

### `sun.misc.Unsafe` memory-access methods emit runtime warnings (Java 24, JEP 498)

The memory-access methods in `sun.misc.Unsafe` were deprecated for removal in Java 23
(JEP 471) and now emit **runtime warnings** by default in Java 24. In Java 26+, they
will throw exceptions, and later be removed entirely.

**Search pattern**: `import sun.misc.Unsafe`, `Unsafe.getUnsafe()`,
`UNSAFE.putInt`, `UNSAFE.getLong`, `UNSAFE.allocateMemory`,
`UNSAFE.objectFieldOffset`.

**Fix**: migrate to the standard replacements:
- **On-heap access** (fields, arrays): use `java.lang.invoke.VarHandle` (since JDK 9)
- **Off-heap / native memory**: use `java.lang.foreign.MemorySegment` from the
  Foreign Function & Memory API (since JDK 22)

Most application code does not use `Unsafe` directly, but **libraries** do. Check
your dependencies — if a library triggers Unsafe warnings, upgrade it to a version
that uses VarHandle/FFM. You can audit your application with:

```bash
java --sun-misc-unsafe-memory-access=deny -jar your-app.jar
```

### Dynamic agent loading restricted (Java 21+, JEP 451)

Dynamically attaching Java agents at runtime (via the Attach API) now requires the
explicit flag `-XX:+EnableDynamicAgentLoading`. Without it, the JVM will refuse to
load the agent.

**Search pattern**: agents loaded at runtime (profilers, APM tools, debuggers).

**Fix**: add `-XX:+EnableDynamicAgentLoading` to JVM startup flags if you use
runtime-attached agents (e.g., JProfiler, YourKit, Datadog, New Relic). Agents
specified at startup with `-javaagent:` are not affected.

### Non-generational ZGC mode removed (Java 24, JEP 490)

The non-generational mode of ZGC has been removed. If your startup scripts contain
`-XX:+UseZGC` without `-XX:+ZGenerational`, the JVM now runs generational ZGC
automatically. If your scripts contained `-XX:-ZGenerational` to explicitly disable
generational mode, that flag must be removed — it will cause a startup error.

**Search pattern**: `-XX:-ZGenerational`, `-XX:+UseZGC` in startup scripts.

**Fix**: remove `-XX:-ZGenerational` if present. `-XX:+UseZGC` alone now implies
generational mode. The `-XX:+ZGenerational` flag itself is also deprecated since
Java 23 (it's the default now) — you can remove it for cleanliness.

### JNI usage warnings (Java 24, JEP 472)

Uses of the Java Native Interface (JNI) and the Foreign Function & Memory API now
issue warnings to prepare for a future "integrity by default" mode.

**Search pattern**: `System.loadLibrary(`, `System.load(`, JNI `native` method
declarations, `Linker.nativeLinker()`.

**Fix**: for now, these are warnings only. If your application uses JNI, ensure
you are aware that future Java versions may require explicit opt-in flags
(`--enable-native-access`). The FFM API already requires this flag for unrestricted
native access.

### Legacy API removals

| Removed | Version | Replacement |
|---|---|---|
| `Thread.countStackFrames()` | Java 22 | Use `Thread.getStackTrace()` |
| Old reflection implementation (`-Djdk.reflect.useDirectMethodHandle=false`) | Java 22 | No action — the flag is now a no-op |
| `sun.misc.Unsafe.shouldBeInitialized()` / `ensureClassInitialized()` | Java 22 | Use `MethodHandles.Lookup.ensureInitialized(Class)` |
| Windows 32-bit x86 port | Java 24 | Use 64-bit JDK |
| Non-generational ZGC | Java 24 | Generational ZGC is now the only mode |

### String Templates removed

String Templates (JEP 430, previewed in Java 21–22) have been **withdrawn** and will
not be finalized. If you experimented with `STR."..."` template processors in preview
mode, remove all usages. Use `String.formatted(...)` or `MessageFormat` instead.

---

## 2. Build config changes

### Maven

```xml
<properties>
  <maven.compiler.release>25</maven.compiler.release>
</properties>

<build>
  <plugins>
    <!-- Minimum 3.13+ for Java 25 -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.14.0</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.5.2</version>
    </plugin>
  </plugins>
</build>
```

### Gradle (Groovy DSL)

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}
```

### Gradle (Kotlin DSL)

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}
```

Ensure Gradle wrapper is at least **8.12+** for Java 25 support.

---

## 3. Code refactoring examples

### Removing Security Manager / AccessController code

```diff
- import java.security.AccessController;
- import java.security.PrivilegedAction;

- String prop = AccessController.doPrivileged(
-     (PrivilegedAction<String>) () -> System.getProperty("my.config"));
+ String prop = System.getProperty("my.config");

- SecurityManager sm = System.getSecurityManager();
- if (sm != null) {
-     sm.checkRead(filePath);
- }
+ // Security Manager is gone — remove the check entirely.
+ // Use application-level authorization if access control is needed.
```

### Stream Gatherers (Java 24, JEP 485)

Stream Gatherers provide a way to define custom intermediate operations, filling
the gap between the built-in operations (`map`, `filter`, `flatMap`) and terminal
collectors:

```diff
- // Windowing: manually collecting fixed-size groups
- List<List<Integer>> windows = new ArrayList<>();
- for (int i = 0; i < items.size(); i += 3) {
-     windows.add(items.subList(i, Math.min(i + 3, items.size())));
- }
+ // Using built-in gatherer for fixed windows
+ import java.util.stream.Gatherers;
+ List<List<Integer>> windows = items.stream()
+     .gather(Gatherers.windowFixed(3))
+     .toList();
```

### Unnamed variables and patterns (Java 22, JEP 456)

Use `_` for variables you don't need:

```diff
- try {
-     int num = Integer.parseInt(str);
- } catch (NumberFormatException ex) {
-     // ex is never used
-     System.out.println("Not a number");
- }
+ try {
+     int num = Integer.parseInt(str);
+ } catch (NumberFormatException _) {
+     System.out.println("Not a number");
+ }

- map.forEach((key, value) -> {
-     // key is unused
-     process(value);
- });
+ map.forEach((_, value) -> process(value));
```

### Scoped Values replacing ThreadLocal (Java 25, JEP 506)

For immutable, inheritable per-request context (e.g., user identity, trace IDs),
Scoped Values are safer and more performant than `ThreadLocal`, especially with
virtual threads:

```diff
- private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
-
- public void handleRequest(User user) {
-     CURRENT_USER.set(user);
-     try {
-         processRequest();
-     } finally {
-         CURRENT_USER.remove(); // easy to forget → memory leak
-     }
- }
-
- public User getCurrentUser() {
-     return CURRENT_USER.get();
- }
+ private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
+
+ public void handleRequest(User user) {
+     ScopedValue.runWhere(CURRENT_USER, user, this::processRequest);
+     // automatically unbound when runWhere exits — no cleanup needed
+ }
+
+ public User getCurrentUser() {
+     return CURRENT_USER.get(); // throws if not bound
+ }
```

### Flexible constructor bodies (Java 25, JEP 513)

You can now execute statements before the `super(...)` or `this(...)` call, as long
as they don't access the instance being constructed:

```diff
- public class ValidatedOrder extends Order {
-     // Before: had to use a static helper or factory method
-     public ValidatedOrder(int quantity) {
-         super(validateQuantity(quantity)); // workaround
-     }
-     private static int validateQuantity(int q) {
-         if (q <= 0) throw new IllegalArgumentException("quantity must be positive");
-         return q;
-     }
- }
+ public class ValidatedOrder extends Order {
+     public ValidatedOrder(int quantity) {
+         if (quantity <= 0) throw new IllegalArgumentException("quantity must be positive");
+         super(quantity); // statements before super() are now allowed
+     }
+ }
```

### Module import declarations (Java 25, JEP 511)

Import all exported packages from a module with a single declaration:

```diff
- import java.util.List;
- import java.util.Map;
- import java.util.stream.Collectors;
- import java.util.function.Predicate;
+ import module java.base;
```

This is most useful for scripts, prototypes, and compact source files. For larger
production codebases, explicit imports often remain clearer.

### Compact source files and instance main (Java 25, JEP 512)

Simple programs no longer need a class declaration or `static` on `main`:

```diff
- public class HelloWorld {
-     public static void main(String[] args) {
-         System.out.println("Hello, world!");
-     }
- }
+ // HelloWorld.java — compact source file
+ void main() {
+     println("Hello, world!");
+ }
```

This is most relevant for scripts, teaching, and quick prototyping. Existing `public
static void main(String[])` entry points continue to work unchanged.

---

## 4. New features to adopt (recommended)

### Finalized features (safe to adopt)

| Feature | Finalized in | Description |
|---|---|---|
| Foreign Function & Memory API | Java 22 (JEP 454) | Safe native interop replacing JNI and `sun.misc.Unsafe` for off-heap memory |
| Unnamed Variables & Patterns | Java 22 (JEP 456) | Use `_` for intentionally unused variables in catches, lambdas, patterns |
| ZGC Generational by Default | Java 23 (JEP 474) | Generational ZGC is now the only mode — better throughput, lower allocation stalls |
| Markdown Javadoc Comments | Java 23 (JEP 467) | Write Javadoc in Markdown (`///` comments) instead of HTML |
| Stream Gatherers | Java 24 (JEP 485) | Custom intermediate stream operations (windowing, folding, scanning) |
| Class-File API | Java 24 (JEP 484) | Standard API for parsing/generating `.class` files — replaces ASM for JDK-internal use |
| Virtual Thread Pinning Fix | Java 24 (JEP 491) | `synchronized` no longer pins virtual threads to platform threads |
| Quantum-Resistant Crypto | Java 24 (JEPs 496, 497) | ML-KEM and ML-DSA post-quantum algorithms available |
| Scoped Values | Java 25 (JEP 506) | Safer, faster alternative to `ThreadLocal` — especially beneficial with virtual threads |
| Flexible Constructor Bodies | Java 25 (JEP 513) | Statements allowed before `super()`/`this()` calls |
| Module Import Declarations | Java 25 (JEP 511) | `import module java.base;` imports all exported packages |
| Compact Source Files | Java 25 (JEP 512) | Simplified `main()` entry points without class boilerplate |
| Compact Object Headers | Java 25 (JEP 519) | Object headers reduced from 96–128 bits to 64 bits — smaller heap footprint |
| Generational Shenandoah | Java 25 (JEP 521) | Shenandoah GC now has a generational mode for improved performance |
| Key Derivation Function API | Java 25 (JEP 510) | Standard API for HKDF and other KDFs — no more custom implementations |
| AOT Class Loading & Profiling | Java 25 (JEPs 514, 515) | Simplified ahead-of-time caches and method profiling for faster startup |

### Still in preview / incubator (do NOT adopt in production)

| Feature | Status in Java 25 | Note |
|---|---|---|
| Primitive Types in Patterns | 3rd Preview (JEP 507) | May change before finalization |
| Structured Concurrency | 5th Preview (JEP 505) | API may still evolve |
| Vector API | 10th Incubator (JEP 508) | Waiting for Valhalla value types |
| PEM Encodings API | Preview (JEP 470) | New in Java 25 preview |
| Stable Values | Preview (JEP 502) | New in Java 25 preview |

---

## 5. JVM flags & runtime notes

### Virtual thread `synchronized` pinning fixed (JEP 491)

This is one of the most impactful changes for virtual thread adopters. Before Java 24,
`synchronized` blocks would **pin** a virtual thread to its carrier platform thread,
reducing the scalability benefit. In Java 24+, this pinning is eliminated. If you
switched from `synchronized` to `ReentrantLock` purely to avoid pinning, you can now
switch back if `synchronized` is more natural for your code. However, `ReentrantLock`
still offers features like `tryLock()` that `synchronized` does not.

### Compact Object Headers (JEP 519)

Java 25 finalizes compact object headers, reducing object header sizes from 96–128 bits
to 64 bits on 64-bit architectures. This reduces heap usage and improves cache locality.
It is enabled by default — no flags needed.

### Ahead-of-Time (AOT) improvements (JEPs 514, 515)

Java 25 simplifies AOT cache creation and adds method profiling from previous runs. This
can significantly improve startup time for long-running applications:

```bash
# Create an AOT cache (simplified in Java 25)
java -XX:AOTCacheOutput=app.aot -cp app.jar com.example.Main
# Run with the AOT cache
java -XX:AOTCache=app.aot -cp app.jar com.example.Main
```

### `--sun-misc-unsafe-memory-access` flag

Use this flag to audit and prepare for Unsafe removal:

| Value | Behavior | Default in |
|---|---|---|
| `allow` | No warnings | Java 23 |
| `warn` | Warning on first use | Java 24–25 |
| `debug` | Warning + stack trace on every use | — |
| `deny` | Throws `UnsupportedOperationException` | Java 26+ |

Run with `deny` to proactively find all Unsafe usages before they become hard errors.

### Dynamic agent loading

If you use APM tools or profilers that attach at runtime, add to your JVM flags:

```
-XX:+EnableDynamicAgentLoading
```

Agents loaded at startup via `-javaagent:` are unaffected.

### Generational Shenandoah (JEP 521)

If you use Shenandoah GC, the generational mode is now a production feature.
Enable with `-XX:+UseShenandoahGC` and optionally `-XX:ShenandoahGCMode=generational`.
