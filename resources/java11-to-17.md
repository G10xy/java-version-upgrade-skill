# Java 11 → 17 Migration Guide

## Table of contents
1. Removed / changed behavior (blockers)
2. Build config changes
3. Code refactoring examples
4. New features to adopt (recommended)
5. JVM flags & runtime notes

---

## 1. Removed / changed behavior (blockers)

### Stronger encapsulation of JDK internals (Java 16/17)

Starting in Java 16, "illegal reflective access" warnings become **hard errors** by
default. Code or libraries that access `sun.*`, `jdk.internal.*`, or use deep
reflection into `java.base` will fail at runtime.

**Search pattern**: `import sun.`, `import jdk.internal.`, calls to `setAccessible(true)` on JDK-internal classes.

**Preferred fix**: upgrade the offending library. Most frameworks released
JPMS-compatible versions by 2021.

**The `--illegal-access` escape hatch is gone.** In JDK 9–16 you could relax
encapsulation wholesale with `--illegal-access=permit|warn|debug`. JEP 403 makes this
option **obsolete in JDK 17**: it is ignored and merely prints a warning. If your
startup scripts rely on it, it is silently no longer doing anything — you must replace
it with targeted flags:

```diff
- --illegal-access=permit
+ --add-opens java.base/java.lang=ALL-UNNAMED
+ --add-opens java.base/java.util=ALL-UNNAMED
```

**Temporary workaround**: add `--add-opens` scoped to the minimum necessary packages.
Document each one with a deadline to remove. Note that `--add-opens` can also be baked
into a jar's `MANIFEST.MF` via `Add-Opens:`, and into Surefire/Failsafe via
`<argLine>`, so tests and production match.

### Security Manager deprecated for removal (Java 17)

`System.setSecurityManager(...)` and `java.security.Policy` are deprecated for removal.
If your application uses a Security Manager for sandboxing, plan migration to
container-level isolation or application-level authorization.

**Search pattern**: `SecurityManager`, `setSecurityManager`, `checkPermission`, `.policy` files.

### Nashorn removed (Java 15)

The Nashorn JavaScript engine was fully removed. If you deferred migration from Java 11,
this is now a hard blocker.

**Replacement**: GraalVM JS (`org.graalvm.js:js`), or redesign away from embedded JS.

### RMI Activation removed (Java 17)

`rmid` and activation groups are gone. If you use RMI Activation, replace with
standard RMI or a modern remoting approach.

**Search pattern**: `import java.rmi.activation.`, `rmid`, `ActivationGroup`.

### Applet API deprecated for removal (Java 17)

`java.applet.*` is deprecated for removal. Unlikely to affect server-side code, but
audit if present.

---

## 2. Build config changes

### Maven

```xml
<properties>
  <maven.compiler.release>17</maven.compiler.release>
</properties>

<build>
  <plugins>
    <!-- Minimum 3.8+ for Java 17; 3.11+ recommended -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
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
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

### Gradle (Kotlin DSL)

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}
```

Ensure Gradle wrapper is at least **7.3** (for Java 17 support); **7.6+** recommended.

---

## 3. Code refactoring examples

### Pattern matching for instanceof

```diff
- if (obj instanceof String) {
-     String s = (String) obj;
-     System.out.println(s.length());
- }
+ if (obj instanceof String s) {
+     System.out.println(s.length());
+ }
```

### Records replacing DTOs

```diff
- public final class Point {
-     private final int x;
-     private final int y;
-
-     public Point(int x, int y) {
-         this.x = x;
-         this.y = y;
-     }
-
-     public int getX() { return x; }
-     public int getY() { return y; }
-
-     @Override
-     public boolean equals(Object o) { /* boilerplate */ }
-     @Override
-     public int hashCode() { return Objects.hash(x, y); }
-     @Override
-     public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
- }
+ public record Point(int x, int y) {}
```

Note: records are not a universal DTO replacement. They work best for simple
immutable data carriers. If your class has mutable state, inheritance, or complex
builder patterns, keep it as a regular class.

### Switch expressions

```diff
- String label;
- switch (status) {
-     case ACTIVE:
-         label = "Active";
-         break;
-     case INACTIVE:
-         label = "Inactive";
-         break;
-     case PENDING:
-         label = "Pending Review";
-         break;
-     default:
-         label = "Unknown";
-         break;
- }
+ String label = switch (status) {
+     case ACTIVE   -> "Active";
+     case INACTIVE -> "Inactive";
+     case PENDING  -> "Pending Review";
+     default       -> "Unknown";
+ };
```

### Text blocks

```diff
- String json = "{\n"
-     + "  \"name\": \"Alice\",\n"
-     + "  \"age\": 30\n"
-     + "}";
+ String json = """
+     {
+       "name": "Alice",
+       "age": 30
+     }
+     """;
```

### Sealed classes

```diff
- // Before: open hierarchy, anyone can extend
- public abstract class Shape { }
- public class Circle extends Shape { /* ... */ }
- public class Rectangle extends Shape { /* ... */ }
+ // After: closed hierarchy, compiler knows all subtypes
+ public sealed class Shape permits Circle, Rectangle { }
+ public final class Circle extends Shape { /* ... */ }
+ public final class Rectangle extends Shape { /* ... */ }
```

---

## 4. New features to adopt (recommended)

- **Records** (Java 16): replace verbose immutable DTOs with `record`. See code example above.
- **Sealed Classes / Interfaces** (Java 17): model closed hierarchies with `sealed` + `permits`. Enables exhaustive pattern matching in later versions.
- **Pattern Matching for `instanceof`** (Java 16): eliminate redundant casts after type checks.
- **Switch Expressions** (Java 14): replace fall-through switch blocks with expression form.
- **Text Blocks** (Java 15): replace heavily escaped multi-line strings (JSON, XML, SQL, HTML).
- **`Stream.toList()`** (Java 16): replaces `.collect(Collectors.toList())` — shorter, and returns an unmodifiable list. The behavioral difference matters: if callers mutate the result, keep `Collectors.toList()` or use `Collectors.toCollection(ArrayList::new)`.
- **Helpful NullPointerExceptions** (Java 14): enabled by default; stack traces pinpoint the exact null dereference.
- **New `RandomGenerator` APIs** (Java 17): prefer `java.util.random.RandomGenerator` for explicit algorithm choices.
- **Other small API wins**: `String.formatted()`, `stripIndent()`, `translateEscapes()` (Java 15); `Files.mismatch()` (Java 12); `Collectors.teeing()` (Java 12); `Stream.mapMulti()` (Java 16); `HexFormat` (Java 17); `Objects.requireNonNullElse()` (Java 9).

```diff
- List<String> names = people.stream()
-     .map(Person::name)
-     .collect(Collectors.toList());
+ List<String> names = people.stream()
+     .map(Person::name)
+     .toList();
```

---

## 5. JVM flags & runtime notes

- **Re-baseline GC tuning** after upgrading. Even with the same collector, JDK improvements change GC behavior between versions. Don't carry forward legacy flags blindly.
- **Remove unrecognized flags**: the JVM will refuse to start on unknown `-XX:` flags. Audit your startup scripts.
- **Classpath vs. module path**: most applications should keep using the classpath unless you explicitly modularize. JPMS module-info is optional for typical services.
- **`--add-opens` audit**: if you added workarounds during the 8→11 migration, revisit them now. Many can be removed if libraries have been upgraded.
