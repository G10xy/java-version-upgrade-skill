---
name: "java-version-upgrade"
license: Apache-2.0
description: >
  Guides upgrading a Java codebase between LTS versions (8→11, 11→17, 17→21, 21→25,
  or multi-hop like 8→25). Detects removed/deprecated APIs, rewrites code with diffs,
  updates Maven/Gradle build config, checks dependency compatibility, and produces a
  migration checklist. Use whenever the user mentions upgrading, migrating, or bumping
  a Java/JDK version, asks what breaks between versions, provides a pom.xml or
  build.gradle targeting a newer JDK, or mentions keywords like "illegal reflective
  access", "JPMS", "javax to jakarta", "virtual threads", "records", "sealed classes",
  "scoped values", "SecurityManager removal", or "sun.misc.Unsafe". Also trigger on
  version-mismatch symptoms such as "UnsupportedClassVersionError", "Unsupported class
  file major version", "InaccessibleObjectException", "class file version 61/65/69",
  or unexplained charset/date-format differences after a JDK change.
  Casual phrasing like "we're still on Java 8" or "can we move to 25?" should also
  trigger this skill, as should questions about which version to target or whether to
  adopt a non-LTS release.
  Do NOT use for Spring Boot version upgrades, adding new Java features without a
  version change, or general refactoring unrelated to version migration.
---

# Java Version Upgrade Skill

## When to use

Trigger this skill when:
- The user asks to "upgrade Java to X", "migrate from Java X to Y", "bump the JDK version"
- The user provides a `pom.xml`, `build.gradle(.kts)`, or source files and asks what needs to change for a newer Java
- The user wants a compatibility audit before upgrading the JVM
- The user mentions symptoms of a version mismatch (e.g., "illegal reflective access", missing `javax.xml.bind`)

Do NOT use this skill for:
- Spring Boot major-version upgrades (use the `spring-boot-upgrade` skill if available)
- Adding new Java language features without changing the target version
- General code refactoring unrelated to version migration

---

## Step-by-step workflow

Follow these steps in order. If the user has not provided the full codebase, produce a
*guide* (what to search for and how to fix it) rather than direct edits.

### 1. Detect the current and target versions

Look for version indicators in build files:

| Build tool | Where to look |
|---|---|
| Maven | `<java.version>`, `<maven.compiler.source>`, `<maven.compiler.target>`, `<maven.compiler.release>` in `pom.xml` — check the **parent** pom too, and `maven-enforcer-plugin`'s `<requireJavaVersion>` |
| Gradle (Groovy) | `sourceCompatibility`, `targetCompatibility`, or `java { toolchain { languageVersion } }` in `build.gradle` |
| Gradle (Kotlin DSL) | Same properties in `build.gradle.kts` |
| Any | `.sdkmanrc`, `.tool-versions` (asdf/mise), `.java-version` (jenv) |
| Runtime | `FROM` line in `Dockerfile`, `JAVA_HOME` in deploy scripts, `.mvn/jvm.config`, `JAVA_TOOL_OPTIONS` |
| CI | `actions/setup-java` `java-version` in `.github/workflows/*.yml`, `jdk` in `Jenkinsfile`, `image` in `.gitlab-ci.yml` |

Two distinct versions matter and they are frequently different — record both:

- the **language level** the code is compiled to (`release` / `sourceCompatibility`)
- the **runtime JDK** the build and the application actually execute on

A project compiling with `--release 8` while running on a JDK 17 toolchain is already
half-migrated; its remaining work is entirely different from a project still running
JDK 8. In a multi-module build, check every module — mixed levels are common.

If the version is ambiguous or absent, ask the user to confirm both the **current** and
**target** Java version before proceeding. Do not guess.

#### Choosing the target version

Java ships a feature release every six months, but only some are **LTS**. Since Java 17,
a new LTS arrives every two years:

| | Releases |
|---|---|
| **LTS** | 8, 11, 17, 21, 25 — then every two years (29 next, September 2027) |
| **Non-LTS** | everything else (9–10, 12–16, 18–20, 22–24, 26–28, …) |

A non-LTS release receives updates only until the next feature release ships six months
later, at which point it is considered superseded. **Default to the newest LTS** the
project's dependency ecosystem supports, and treat that as the recommendation unless the
user says otherwise.

If the user names a **non-LTS target** (e.g. "let's go to 27"), do not refuse and do not
silently retarget. Instead:

1. State the tradeoff plainly — a non-LTS release stops receiving updates in about six
   months, so the project is committing to a six-month upgrade treadmill.
2. Ask whether they want the newest LTS instead. Legitimate reasons to pick non-LTS do
   exist: needing a feature that finalized after the last LTS, validating readiness for
   the *next* LTS early, or library maintainers testing compatibility across releases.
3. If they confirm the non-LTS target, apply the guide for the **LTS at or below it**,
   then explicitly check the release notes of each intervening version for changes this
   skill does not cover. Say which versions you could not cover from the bundled guides.

The `resources/` guides are LTS-to-LTS by design. When the target is not an LTS, say so
rather than implying full coverage.

### 2. Load the relevant migration guide

Read the appropriate file(s) from `resources/` based on the migration path:

| Migration path | File(s) to read |
|---|---|
| 8 → 11 | `resources/java8-to-11.md` |
| 11 → 17 | `resources/java11-to-17.md` |
| 17 → 21 | `resources/java17-to-21.md` |
| 21 → 25 | `resources/java21-to-25.md` |
| 8 → 17 | `resources/java8-to-11.md` then `resources/java11-to-17.md` |
| 8 → 21 | `java8-to-11.md`, `java11-to-17.md`, `java17-to-21.md` in order |
| 8 → 25 | All four files, in order |
| 11 → 21 | `resources/java11-to-17.md` then `resources/java17-to-21.md` |
| 11 → 25 | `java11-to-17.md`, `java17-to-21.md`, `java21-to-25.md` in order |
| 17 → 25 | `resources/java17-to-21.md` then `resources/java21-to-25.md` |

Also read `resources/migration-matrix.md` for a quick overview of key themes.

**Multi-hop migrations**: when crossing multiple LTS boundaries, apply changes
cumulatively. Start from the lowest hop and layer upward. If two guides give
conflicting advice (rare), the higher-version guide wins because it reflects the
final target state. For example, if 8→11 says "add JAXB as a dependency" and
11→17 says nothing about removing it, keep it.

### 3. Audit the codebase

#### 3a. Run the JDK's own analysis tools first

Before hand-searching for patterns, run the tools that ship with the JDK — they find
issues no grep can. Run them with the **target** JDK installed:

```bash
# Find uses of internal/encapsulated JDK APIs (sun.*, jdk.internal.*) and suggest replacements
jdeps --jdk-internals --multi-release <target> -cp "libs/*" target/classes

# Find uses of APIs deprecated as of the target release (scans your code AND your jars)
jdeprscan --release <target> target/classes
jdeprscan --release <target> --for-removal target/classes   # highest priority

# List which JDK modules your code actually needs
jdeps --print-module-deps -cp "libs/*" target/classes
```

`jdeps --jdk-internals` is the single highest-value command in this workflow: it reports
internal-API usage in **third-party jars** too, which is where most upgrade failures
actually originate.

Then compile and run the test suite on the target JDK to surface
`UnsupportedClassVersionError`, `ClassFormatError`, `NoClassDefFoundError`, and
`InaccessibleObjectException` — these almost always point at an outdated bytecode
library (ASM, ByteBuddy, cglib, Javassist) rather than at the user's own code.

#### 3b. Scan for source patterns

Scan for patterns described in the loaded migration guide(s). Organize findings into
these categories:

1. **Removed or deprecated APIs** — e.g., `sun.*` internals, Java EE modules removed in 11, `finalize()` deprecated for removal in 18+
2. **Module system issues** — illegal reflective access, need for `--add-opens` (prefer library upgrades over workarounds)
3. **JVM flag and GC changes** — removed flags, changed defaults (CMS → G1, ZGC generational)
4. **Language/API modernization opportunities** — records, sealed classes, pattern matching, virtual threads, text blocks, new collection factories
5. **Third-party dependency incompatibilities** — consult `resources/common-dependencies.md`

For each finding, note the file/pattern affected and the severity (blocker vs. recommended).

### 4. Update build configuration

**Sequence the change in two phases.** Bumping the runtime JDK and the language level at
the same moment makes every failure ambiguous. Instead:

1. **Phase 1 — run on the new JDK, keep the old language level.** Point the build and
   tests at the target JDK but leave `<maven.compiler.release>` / `sourceCompatibility`
   at the current value. Everything that breaks here is a *runtime/dependency* problem:
   outdated bytecode libraries, reflective access, removed JDK modules, dead JVM flags.
   Fix those first — they are the blockers.
2. **Phase 2 — raise the language level.** Only once Phase 1 is green, bump `release` to
   the target. Failures now are *source* problems: removed APIs, new restricted
   keywords, changed defaults. This phase is usually much smaller.

Mention this split to the user when the migration crosses two or more LTS versions or
the project is large; skip it for small, single-hop projects.

Produce concrete diffs to update build files:

**Maven** (`pom.xml`):
- Update `<maven.compiler.release>` (preferred) or both `<maven.compiler.source>` and `<maven.compiler.target>`
- Bump `maven-compiler-plugin` to a version that supports the target JDK
- Bump `maven-surefire-plugin` and `maven-failsafe-plugin` if needed
- Add any newly required dependencies (e.g., JAXB for 8→11)

**Gradle** (`build.gradle` / `build.gradle.kts`):
- Update `sourceCompatibility` / `targetCompatibility`, or preferably use the `java { toolchain {} }` block for JDK 11+
- Ensure the Gradle wrapper version supports the target JDK — check the table in `resources/common-dependencies.md`. Targeting Java 25 requires Gradle 9.x, which is itself a non-trivial upgrade

**CI/CD**:
- Update JDK version in `.github/workflows/*.yml`, `Dockerfile`, `Jenkinsfile`, `.gitlab-ci.yml`, etc.
- Keep the CI JDK and the build's toolchain JDK in sync — a mismatch produces failures that reproduce only in CI
- If using Docker, use a JDK image to build and a **JRE** image to run (e.g. build on `eclipse-temurin:21-jdk`, run on `eclipse-temurin:21-jre`). Note that `-alpine` variants are musl-based: they are smaller but can break native libraries that expect glibc

Always present changes as ` ```diff ` blocks so the user can review them visually.

### 5. Refactor source code

For each issue identified in step 3, provide:

1. **Why** — a short explanation of why the code must change (or why adopting the new feature is beneficial)
2. **Search pattern** — what to look for in the codebase (regex or plain text)
3. **Replacement** — the concrete rewrite
4. **Example** — a before/after code snippet in a diff block

Prioritize blockers (code that will fail to compile or crash at runtime) first,
then recommended modernizations second. Group related changes together.

**Automated refactoring**: for large codebases, mention OpenRewrite as an accelerator —
it has maintained recipes for exactly these migrations (e.g.
`org.openrewrite.java.migrate.UpgradeToJava21`), runnable via
`mvn org.openrewrite.maven:rewrite-maven-plugin:run`. Treat its output as a reviewable
first pass, not a finished result: it handles mechanical rewrites well, but does not
understand behavioral changes like the charset and locale defaults. Never suggest it as
a substitute for running the test suite.

### 6. Dependency compatibility check

Consult `resources/common-dependencies.md` and flag third-party libraries that are
known to be incompatible with the target version. For each flagged dependency:

- State the minimum compatible version
- Provide the updated Maven/Gradle coordinate
- Note any API changes in the library that the upgrade introduces

Apply the **release-date rule** from `common-dependencies.md`: a library released before
the target JDK's GA date cannot support it. This matters most for anything that reads or
generates bytecode (ASM, ByteBuddy, cglib, Javassist, Lombok, JaCoCo, mocking
frameworks) — these break first and produce the most confusing errors.

Useful commands for finding what is actually on the classpath:

```bash
mvn dependency:tree -Dverbose            # includes transitive/omitted versions
mvn versions:display-dependency-updates  # what newer versions exist
gradle dependencies --configuration runtimeClasspath
```

If the user's dependency is not listed in the reference, search Maven Central or
release notes if web access is available. If not, warn the user to verify compatibility
and suggest checking the library's changelog. **Say explicitly when a version number is
unverified** rather than presenting a guess as fact.

### 7. Produce a migration summary

End every migration response with a checklist in this format:

```markdown
## Migration Summary: Java {SOURCE} → {TARGET}

### Blockers (must fix)
- [ ] Build config updated (`pom.xml` / `build.gradle`), all modules
- [ ] Build tool itself supports the target JDK (Maven plugins / Gradle wrapper)
- [ ] Removed/deprecated API usages addressed
- [ ] Module-system issues resolved (or `--add-opens` documented as temporary)
- [ ] Incompatible dependencies upgraded (bytecode libraries first)
- [ ] Removed/obsolete JVM flags deleted from startup scripts

### Silent behavior changes (verify — these do NOT fail the build)
- [ ] Default charset: explicit `StandardCharsets` in all I/O (UTF-8 default since 18)
- [ ] Locale-dependent formatting re-checked (CLDR data changes every release)
- [ ] Serialized/persisted data and wire formats still round-trip
- [ ] Time zone and date parsing behavior unchanged

### Recommended (should fix)
- [ ] Language modernization applied (records, pattern matching, text blocks, etc.)
- [ ] JVM flags reviewed and re-baselined (GC especially)
- [ ] CI/CD pipeline JDK version updated
- [ ] Docker base image updated

### Verification
- [ ] `jdeps --jdk-internals` reports no internal API usage
- [ ] `jdeprscan --for-removal` reports no blocking deprecations
- [ ] Project compiles with `javac`/build tool at target version
- [ ] Full test suite passes — including on the CI runner, not just locally
- [ ] Application starts with no warnings on stderr (Unsafe, agents, obsolete flags)
- [ ] Smoke tests pass against a real deployment
- [ ] Performance baseline compared (GC behavior, memory, startup time)
```

Adjust the checklist items based on what was actually found — omit categories that
had no issues, and add project-specific items as needed.

**Be honest about uncertainty.** If you could not inspect the full codebase, or a
dependency version could not be verified, state that plainly in the summary instead of
implying the migration is fully covered. A missed blocker discovered in production costs
far more than an explicit "verify this yourself" note.

---

## Output format

- Use Markdown with clear headings per issue category.
- For code changes, always use diff blocks (` ```diff `).
- Keep explanations concise — the "why" matters, but avoid lengthy prose when a
  code example speaks for itself.
- Present the migration summary checklist at the end.
- If the full codebase is not provided, frame the output as a migration guide
  (what to search for, what to replace) rather than direct file edits.
