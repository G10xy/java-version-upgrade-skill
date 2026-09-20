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
(JEP 471) and now emit **runtime warnings** by default in Java 24. In a future release
(JDK 26 at the earliest) they will throw exceptions by default, and later be removed
entirely.

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

### Dynamic agent loading warns (Java 21, JEP 451)

Dynamically attaching a Java agent to a running JVM (via the Attach API) now prints a
warning to standard error. This is a **warning, not a failure** — the agent still
loads. JEP 451 is explicitly preparing users for a *future* release that will
disallow dynamic loading by default.

```
WARNING: A {Java,JVM TI} agent has been loaded dynamically (...)
WARNING: Dynamic loading of agents will be disallowed by default in a future release
```

**Search pattern**: agents attached at runtime (profilers, APM tools, `VirtualMachine.loadAgent`,
Byte Buddy Agent `ByteBuddyAgent.install()`, Mockito inline mock maker).

**Fix**: pass `-XX:+EnableDynamicAgentLoading` to suppress the warning and to
future-proof the startup command. Agents specified at startup with `-javaagent:` are
not affected and never warn. Serviceability tools such as `jcmd` and `jconsole`
also continue to work without warnings.

> Note: this landed in **Java 21**, so it may already apply before you start this hop.
> It is listed here because the warning becomes hard to ignore as you move toward the
> release that flips the default to "disallowed".

### Non-generational ZGC mode removed (Java 24, JEP 490)

The non-generational mode of ZGC has been removed and the `ZGenerational` option is
now **obsolete**. Behavior in Java 24/25:

| Startup flags | Result |
|---|---|
| `-XX:+UseZGC` | Generational ZGC, no warning |
| `-XX:+UseZGC -XX:+ZGenerational` | Generational ZGC + obsolete-option warning |
| `-XX:+UseZGC -XX:-ZGenerational` | Generational ZGC + obsolete-option warning (the request to disable is ignored) |

**Search pattern**: `-XX:+ZGenerational`, `-XX:-ZGenerational`, `-XX:+UseZGC` in startup scripts.

**Fix**: remove both `ZGenerational` variants. `-XX:+UseZGC` alone implies generational
mode. The option is only *obsolete* today (warning), but JEP 490 states it "will expire
in a future release, at which point it will not be recognized by the HotSpot JVM, which
will refuse to start" — so treat removal as mandatory, not cosmetic.

Workloads switching from non-generational ZGC may also see differences in GC log output
and in data exposed through the serviceability/management APIs — update any log parsing
or dashboards that depend on them.

### JNI usage warnings (Java 24, JEP 472)

Uses of the Java Native Interface (JNI) and the Foreign Function & Memory API now
issue warnings to prepare for a future "integrity by default" mode.

**Search pattern**: `System.loadLibrary(`, `System.load(`, JNI `native` method
declarations, `Linker.nativeLinker()`.

**Fix**: for now, these are warnings only. If your application uses JNI, ensure
you are aware that future Java versions may require explicit opt-in flags
(`--enable-native-access`). The FFM API already requires this flag for unrestricted
native access.

### Legacy `COMPAT` locale data removed (Java 23)

The legacy JRE locale data (`-Djava.locale.providers=COMPAT` or `JRE`) has been
**removed**. It was deprecated with a startup warning in Java 21; specifying `COMPAT`
or `JRE` in `java.locale.providers` now has no effect at all — the JVM silently uses
CLDR data instead.

**Search pattern**: `java.locale.providers` in startup scripts, `JAVA_TOOL_OPTIONS`,
Dockerfiles, `application.properties`, CI configs.

**Fix**: remove the property and migrate to CLDR locale data. If your application still
depends on JDK 8-era formatted output, replace localized `FormatStyle` formatters with
explicit `DateTimeFormatter.ofPattern(...)` patterns so the output is under your control
rather than the locale database's. See the CLDR 42 notes in the 17 → 21 guide — this is
the same problem surfacing a second time, now with no escape hatch.

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

Ensure the Gradle wrapper is at least **9.1.0** — that is the first release with Java 25
toolchain support *and* the first that can run on a Java 25 JVM. Gradle 8.14 only reaches
Java 24. Note that Gradle 9 itself requires Java 17+ to run.

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
+     ScopedValue.where(CURRENT_USER, user).run(this::processRequest);
+     // automatically unbound when run() exits — no cleanup needed
+ }
+
+ public User getCurrentUser() {
+     return CURRENT_USER.get(); // throws NoSuchElementException if not bound
+ }
```

**API shape warning**: the `ScopedValue.runWhere(...)` / `ScopedValue.callWhere(...)`
static methods that appeared in the Java 21–23 *preview* API were **removed in Java 24**
(JEP 487). The finalized Java 25 API is entirely fluent:

| Need | Java 25 API |
|---|---|
| Run a `Runnable` | `ScopedValue.where(KEY, value).run(runnable)` |
| Call and return a value | `ScopedValue.where(KEY, value).call(op)` |
| Bind several values at once | `ScopedValue.where(K1, v1).where(K2, v2).run(...)` |
| Read with a default | `KEY.orElse(fallback)` — note: `orElse` no longer accepts `null` in Java 25 |

If you adopted scoped values while they were in preview, this is a **compile-time
breaking change** you must fix during this hop.

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
| Compact Object Headers | Java 25 (JEP 519) | Object headers reduced from 96–128 bits to 64 bits — smaller heap footprint. **Opt-in** via `-XX:+UseCompactObjectHeaders` |
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

Java 25 promotes compact object headers from an *experimental* to a *product* feature,
reducing object header size from 96–128 bits to 64 bits on 64-bit architectures. This
reduces heap usage and improves cache locality.

**It is NOT enabled by default.** JEP 519 explicitly states that making it the default
is a non-goal. You must opt in:

```bash
# Java 24 (experimental — both flags required)
java -XX:+UnlockExperimentalVMOptions -XX:+UseCompactObjectHeaders -jar app.jar

# Java 25 (product feature — no unlock flag needed)
java -XX:+UseCompactObjectHeaders -jar app.jar
```

Measure before and after: the benefit is largest for heaps dominated by many small
objects. Code that assumes a specific object layout (rare, but present in some
serialization or off-heap libraries) should be tested with the flag enabled.

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

| Value | Behavior | Status |
|---|---|---|
| `allow` | No warnings | Default in Java 23 |
| `warn` | Warning on first use | Default in Java 24–25 |
| `debug` | Warning + stack trace on every use | Opt-in |
| `deny` | Throws `UnsupportedOperationException` | Opt-in today; planned to become the default in a future release (JDK 26 at the earliest) |

Run with `deny` to proactively find all Unsafe usages before they become hard errors.

### Dynamic agent loading

If you use APM tools or profilers that attach at runtime (rather than via `-javaagent:`),
add to your JVM flags:

```
-XX:+EnableDynamicAgentLoading
```

This suppresses the JEP 451 warning today and future-proofs the command line for the
release that disallows dynamic loading by default. Agents loaded at startup via
`-javaagent:` are unaffected, as are `jcmd`/`jconsole`. See the blocker section above
for details.

### Generational Shenandoah (JEP 521)

If you use Shenandoah GC, the generational mode is now a production feature.
Enable with `-XX:+UseShenandoahGC` and optionally `-XX:ShenandoahGCMode=generational`.
