# Common Dependency Compatibility Reference

This file lists widely-used Java libraries and the minimum versions required for
each Java LTS target. Use it during step 6 (dependency compatibility check) of the
migration workflow.

> **Treat this as a starting point, not an authority.** Library release cadences move
> faster than this document. Always confirm against the library's own release notes
> before telling the user a version is safe, and say so when you have not verified.

## How to use

1. Check the user's dependency versions against the table below
2. Flag any dependency below the minimum version for the target JDK
3. Suggest the recommended version (or a range) and note any API changes

If a dependency is not listed here, check its release notes or Maven Central page.
Most well-maintained libraries document JDK compatibility in their changelogs.

### The release-date rule (use when you cannot find explicit documentation)

A library cannot support a JDK that did not exist when it was released. Any library
that **reads or generates bytecode** must be updated for each new class-file version,
so its minimum version for a target JDK is one published *after* that JDK's GA:

| Java LTS | Class-file version | GA date |
|---|---|---|
| 8 | 52 | March 2014 |
| 11 | 55 | September 2018 |
| 17 | 61 | September 2021 |
| 21 | 65 | September 2023 |
| 25 | 69 | September 2025 |

If a suggested version predates the target JDK's GA date, it is wrong — bump to the
latest release and verify. This applies most sharply to ASM, ByteBuddy, cglib,
Javassist, Lombok, JaCoCo, Groovy, Kotlin, and anything that mocks or proxies classes.

---

## Frameworks

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Spring Framework | 5.1+ | 5.3.15+ (or 6.0+) | 6.1+ | 7.0+ | Spring 7 uses Jakarta EE 11, recommends JDK 25 |
| Spring Boot | 2.1+ | 2.5.8+ (or 3.0+) | 3.2+ | 3.5.5+ (or 4.0+) | Boot 4.0 built on Spring 7, first-class Java 25 support |
| Quarkus | 1.0+ | 2.7+ | 3.5+ | 3.17+ | |
| Micronaut | 1.0+ | 3.2+ | 4.2+ | 4.7+ | |
| Jakarta EE (formerly Java EE) | 8 | 9+ | 10+ | 11+ | EE 11: Servlet 6.1, JPA 3.2, BV 3.1 |

## ORM / Data access

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Hibernate ORM | 5.4+ | 5.6.5+ (or 6.0+) | 6.3+ | 6.6+ (or 7.0+) | Hibernate 7 uses jakarta.persistence 3.2 |
| MyBatis | 3.5+ | 3.5.9+ | 3.5.14+ | 3.5.17+ | |
| JOOQ | 3.12+ | 3.16+ | 3.18+ | 3.19+ | |
| Flyway | 6.0+ | 8.2+ | 9.16+ (or 10.0+) | 10.6+ (or 11.0+) | |
| Liquibase | 3.8+ | 4.5+ | 4.24+ | 4.30+ | |

## Serialization / Mapping

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Jackson | 2.10+ | 2.13+ | 2.15+ | 2.18+ | Ensure all jackson modules match versions |
| Gson | 2.8.6+ | 2.9+ | 2.10+ | 2.11+ | |
| MapStruct | 1.3+ | 1.5+ | 1.5.5+ | 1.6+ | Annotation processor — update compiler plugin too |
| Lombok | 1.18.12+ | 1.18.22+ | 1.18.30+ | **1.18.40+** (1.18.44+ recommended) | Needs a release per JDK — see note below |

## Testing

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| JUnit 5 | 5.5+ | 5.8+ | 5.10+ | 5.11+ | |
| JUnit 4 | 4.13+ | 4.13.2 | 4.13.2 | 4.13.2 | Consider migrating to JUnit 5 |
| Mockito | 3.0+ | 4.0+ | 5.5+ | ⚠️ use latest 5.x | Delegates to ByteBuddy, which must support the target JDK |
| AssertJ | 3.14+ | 3.22+ | 3.24+ | 3.27+ | |
| Testcontainers | 1.12+ | 1.16+ | 1.19+ | 1.20+ | |
| PowerMock | 2.0+ | ⚠️ blocker | ⚠️ blocker | ⚠️ blocker | See warning below — plan removal, not an upgrade |

## Build plugins (Maven)

| Plugin | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| maven-compiler-plugin | 3.8+ | 3.8+ | 3.11+ | 3.13+ | |
| maven-surefire-plugin | 3.0.0-M5+ | 3.0.0-M5+ | 3.1+ | 3.5+ | |
| maven-failsafe-plugin | 3.0.0-M5+ | 3.0.0-M5+ | 3.1+ | 3.5+ | |
| maven-jar-plugin | 3.2+ | 3.3+ | 3.3+ | 3.4+ | |
| jacoco-maven-plugin | 0.8.5+ | 0.8.7+ | 0.8.11+ | **0.8.14+** | 0.8.14 is the first release with official Java 25 support |

## Build tools (Gradle)

| Gradle version | Java 17 | Java 21 | Java 24 | Java 25 |
|---|---|---|---|---|
| Toolchain support (compile/test) | 7.3 | 8.4 | 8.14 | **9.1.0** |
| Can run Gradle itself | 7.3 | 8.5 | 8.14 | **9.1.0** |

Gradle 8.14.x is the last line that runs on Java 8–16; **Gradle 9 requires Java 17+ to
run**. A Java 25 migration therefore usually means a Gradle 9 upgrade too — budget for
it, since Gradle 9 removes long-deprecated APIs and may break custom build logic.
Use `./gradlew wrapper --gradle-version 9.1.0` to upgrade the wrapper.

## Logging

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Logback | 1.2.3+ | 1.2.11+ (or 1.4+) | 1.4.11+ | 1.5.12+ | Logback 1.4+ uses `jakarta.servlet` |
| SLF4J | 1.7.28+ | 1.7.36+ (or 2.0+) | 2.0.9+ | 2.0.16+ | |
| Log4j 2 | 2.13+ | 2.17.1+ | 2.21+ | 2.24+ | Ensure 2.17.1+ to avoid Log4Shell |

## HTTP / Networking

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Apache HttpClient 4 | 4.5.10+ | 4.5.14 | 4.5.14 | 4.5.14 | EOL — migrate to HttpClient 5 |
| Apache HttpClient 5 | 5.0+ | 5.1+ | 5.2.1+ | 5.4+ | |
| OkHttp | 4.2+ | 4.9+ | 4.12+ | 4.12+ | OkHttp 5 (alpha) requires Java 11+ |
| Netty | 4.1.43+ | 4.1.72+ | 4.1.100+ | 4.1.116+ | Netty is impacted by Unsafe deprecation |
| gRPC-Java | 1.25+ | 1.44+ | 1.58+ | 1.68+ | |

## Other common libraries

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| Guava | 28.0+ | 31.0+ | 33.0+ | 33.4+ | |
| Apache Commons Lang | 3.9+ | 3.12+ | 3.14+ | 3.17+ | |
| Apache Commons IO | 2.6+ | 2.11+ | 2.15+ | 2.18+ | |
| Bouncy Castle | 1.64+ | 1.70+ | 1.76+ | 1.79+ | |
| ASM (bytecode) | 7.1+ | 9.1+ | 9.6+ | ⚠️ use latest 9.x | Critical — must understand the target JDK's class-file version |
| ByteBuddy | 1.10+ | 1.12+ | 1.14.10+ | ⚠️ use latest | Used by Mockito, Hibernate, Spring — often the first thing to break |
| cglib | 3.3+ | ⚠️ verify | ⚠️ verify | ⚠️ verify | Largely superseded by ByteBuddy; Spring ships a repackaged copy |
| Javassist | 3.25+ | 3.28+ | ⚠️ verify | ⚠️ verify | Bytecode library — same class-file constraint as ASM |

> ⚠️ **On the bytecode libraries.** Do not pin Mockito, ByteBuddy, ASM, cglib or
> Javassist from a table like this one for a recently released JDK. Their support for a
> new class-file version lands in a release published *after* that JDK ships, so any
> number recorded here goes stale the moment a new JDK appears. Resolve the current
> version from Maven Central and prefer the latest release. (Concretely: Mockito 5.14,
> ByteBuddy 1.15 and ASM 9.8 all predate JDK 25's September 2025 GA and cannot be valid
> Java 25 minimums.)
>
> ⚠️ **On PowerMock.** PowerMock works by deep bytecode manipulation and reflection into
> JDK internals — exactly what every JDK since 9 has been progressively locking down. It
> has historically lagged far behind JDK releases, and projects on Java 17+ routinely
> find it unfixable. Treat PowerMock as a **migration blocker to be removed**, not a
> dependency to be bumped: rewrite its tests using Mockito's `mockStatic` /
> `mockConstruction` (Mockito 3.4+/4+, inline mock maker), or refactor the static calls
> behind an injectable seam. Budget for this explicitly — on large legacy codebases it
> is often the single largest item in an 8 → 17+ migration.

---

## Tips

- **Lombok** needs a new release for *every* JDK, because it hooks into compiler
  internals. Verified mapping: `1.18.32` → JDK 22, `1.18.36` → JDK 23, `1.18.38` → JDK 24,
  `1.18.40` → JDK 25. Note that `1.18.36` is a **JDK 23** release — it is not sufficient
  for JDK 25, despite being a plausible-looking "recent" version. Update Lombok *first*
  when upgrading Java, and use the latest release.
- **Mockito** major versions align roughly with Java LTS: Mockito 4 for Java 17,
  Mockito 5 for Java 21+. Mockito 5 defaults to the inline mock maker, which loads an
  agent dynamically — see the JEP 451 notes in `java21-to-25.md`.
- **Read the error message carefully — the two common failures mean opposite things:**
  - `UnsupportedClassVersionError: ... has been compiled by a more recent version of the
    Java Runtime` → a *dependency or module* was compiled for a newer JDK than the one
    running. Your runtime is too old, or a library dropped support for your baseline.
  - `IllegalArgumentException: Unsupported class file major version NN` or
    `ClassFormatError` → a *bytecode library* (ASM, ByteBuddy, cglib, Javassist) is too
    old to parse the classes your new JDK produces. Upgrade the bytecode library.
    Major version 61 = Java 17, 65 = Java 21, 69 = Java 25.
- **Netty** uses `sun.misc.Unsafe` internally for performance. Newer versions have
  migrated to VarHandle/FFM alternatives, but older versions will trigger Unsafe
  warnings on Java 24+.
- **JVM languages need their own upgrade.** Groovy, Kotlin, and Scala each compile to
  bytecode and must be bumped alongside the JDK. Groovy is the most common hidden
  offender because it arrives transitively via Spock and Gradle build scripts. Verify
  the required version against the language's own compatibility matrix — do not assume
  the version that worked on the previous LTS will work.
- When upgrading Spring Boot from 3.x to 4.0 (needed for full Java 25 support), the
  major change is the move to Spring Framework 7 and Jakarta EE 11 baseline. This is a
  Spring Boot upgrade concern, not a Java version concern per se, but they often happen
  together.
- If a library uses bytecode manipulation (ASM, ByteBuddy, Javassist, cglib), it is
  almost always the first thing to break on a new JDK. Prioritize updating these.
