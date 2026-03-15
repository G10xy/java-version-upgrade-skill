# Common Dependency Compatibility Reference

This file lists widely-used Java libraries and the minimum versions required for
each Java LTS target. Use it during step 6 (dependency compatibility check) of the
migration workflow.

## How to use

1. Check the user's dependency versions against the table below
2. Flag any dependency below the minimum version for the target JDK
3. Suggest the recommended version (or a range) and note any API changes

If a dependency is not listed here, check its release notes or Maven Central page.
Most well-maintained libraries document JDK compatibility in their changelogs.

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
| Lombok | 1.18.12+ | 1.18.22+ | 1.18.30+ | 1.18.36+ | Extremely sensitive to JDK internals; always use latest |

## Testing

| Library | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| JUnit 5 | 5.5+ | 5.8+ | 5.10+ | 5.11+ | |
| JUnit 4 | 4.13+ | 4.13.2 | 4.13.2 | 4.13.2 | Consider migrating to JUnit 5 |
| Mockito | 3.0+ | 4.0+ | 5.5+ | 5.14+ | Mockito 5 uses ByteBuddy versions that must support target JDK |
| AssertJ | 3.14+ | 3.22+ | 3.24+ | 3.27+ | |
| Testcontainers | 1.12+ | 1.16+ | 1.19+ | 1.20+ | |

## Build plugins (Maven)

| Plugin | Min for Java 11 | Min for Java 17 | Min for Java 21 | Min for Java 25 | Notes |
|---|---|---|---|---|---|
| maven-compiler-plugin | 3.8+ | 3.8+ | 3.11+ | 3.13+ | |
| maven-surefire-plugin | 3.0.0-M5+ | 3.0.0-M5+ | 3.1+ | 3.5+ | |
| maven-failsafe-plugin | 3.0.0-M5+ | 3.0.0-M5+ | 3.1+ | 3.5+ | |
| maven-jar-plugin | 3.2+ | 3.3+ | 3.3+ | 3.4+ | |
| jacoco-maven-plugin | 0.8.5+ | 0.8.7+ | 0.8.11+ | 0.8.12+ | |

## Build tools (Gradle)

| Gradle version | Java 11 | Java 17 | Java 21 | Java 25 |
|---|---|---|---|---|
| Minimum | 5.0 | 7.3 | 8.4 | 8.12 |
| Recommended | 6.7+ | 7.6+ | 8.5+ | 8.14+ |

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
| ASM (bytecode) | 7.1+ | 9.1+ | 9.6+ | 9.8+ | Critical — must support target JDK bytecode version |
| ByteBuddy | 1.10+ | 1.12+ | 1.14.10+ | 1.15+ | Used by Mockito, Hibernate, Spring — often the first to break |

---

## Tips

- **Lombok** is particularly sensitive to JDK internals changes. Always update Lombok first when upgrading Java, and use the absolute latest release.
- **Mockito** major versions align roughly with Java LTS: Mockito 4 for Java 17, Mockito 5 for Java 21+.
- **ByteBuddy and ASM** are bytecode manipulation libraries used transitively by many frameworks. If you see `ClassFormatError` or `UnsupportedClassVersionError`, these are almost always the root cause. Upgrade them first.
- **Netty** uses `sun.misc.Unsafe` internally for performance. Newer versions (4.1.116+) have migrated to VarHandle/FFM alternatives, but older versions will trigger Unsafe warnings on Java 24+.
- When upgrading Spring Boot from 3.x to 4.0 (needed for full Java 25 support), the major change is the move to Spring Framework 7 and Jakarta EE 11 baseline. This is a Spring Boot upgrade concern, not a Java version concern per se, but they often happen together.
- If a library uses bytecode manipulation (ASM, ByteBuddy, Javassist, cglib), it is almost always the first thing to break on a new JDK. Prioritize updating these.
