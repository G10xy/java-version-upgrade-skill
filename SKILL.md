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
  "scoped values", "SecurityManager removal", or "sun.misc.Unsafe". Casual phrasing
  like "we're still on Java 8" or "can we move to 25?" should also trigger this skill.
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
| Maven | `<java.version>`, `<maven.compiler.source>`, `<maven.compiler.target>`, `<maven.compiler.release>` in `pom.xml` |
| Gradle (Groovy) | `sourceCompatibility`, `targetCompatibility`, or `java { toolchain { languageVersion } }` in `build.gradle` |
| Gradle (Kotlin DSL) | Same properties in `build.gradle.kts` |

If the version is ambiguous or absent, ask the user to confirm both the **current** and
**target** Java version before proceeding. Do not guess.

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

Scan for patterns described in the loaded migration guide(s). Organize findings into
these categories:

1. **Removed or deprecated APIs** — e.g., `sun.*` internals, Java EE modules removed in 11, `finalize()` deprecated for removal in 18+
2. **Module system issues** — illegal reflective access, need for `--add-opens` (prefer library upgrades over workarounds)
3. **JVM flag and GC changes** — removed flags, changed defaults (CMS → G1, ZGC generational)
4. **Language/API modernization opportunities** — records, sealed classes, pattern matching, virtual threads, text blocks, new collection factories
5. **Third-party dependency incompatibilities** — consult `resources/common-dependencies.md`

For each finding, note the file/pattern affected and the severity (blocker vs. recommended).

### 4. Update build configuration

Produce concrete diffs to update build files:

**Maven** (`pom.xml`):
- Update `<maven.compiler.release>` (preferred) or both `<maven.compiler.source>` and `<maven.compiler.target>`
- Bump `maven-compiler-plugin` to a version that supports the target JDK
- Bump `maven-surefire-plugin` and `maven-failsafe-plugin` if needed
- Add any newly required dependencies (e.g., JAXB for 8→11)

**Gradle** (`build.gradle` / `build.gradle.kts`):
- Update `sourceCompatibility` / `targetCompatibility`, or preferably use the `java { toolchain {} }` block for JDK 11+
- Ensure the Gradle wrapper version supports the target JDK

**CI/CD**:
- Update JDK version in `.github/workflows/*.yml`, `Dockerfile`, `Jenkinsfile`, `.gitlab-ci.yml`, etc.
- If using Docker, suggest an appropriate base image (e.g., `eclipse-temurin:21-jdk-alpine`)

Always present changes as ` ```diff ` blocks so the user can review them visually.

### 5. Refactor source code

For each issue identified in step 3, provide:

1. **Why** — a short explanation of why the code must change (or why adopting the new feature is beneficial)
2. **Search pattern** — what to look for in the codebase (regex or plain text)
3. **Replacement** — the concrete rewrite
4. **Example** — a before/after code snippet in a diff block

Prioritize blockers (code that will fail to compile or crash at runtime) first,
then recommended modernizations second. Group related changes together.

### 6. Dependency compatibility check

Consult `resources/common-dependencies.md` and flag third-party libraries that are
known to be incompatible with the target version. For each flagged dependency:

- State the minimum compatible version
- Provide the updated Maven/Gradle coordinate
- Note any API changes in the library that the upgrade introduces

If the user's dependency is not listed in the reference, search Maven Central or
release notes if web access is available. If not, warn the user to verify compatibility
and suggest checking the library's changelog.

### 7. Produce a migration summary

End every migration response with a checklist in this format:

```markdown
## Migration Summary: Java {SOURCE} → {TARGET}

### Blockers (must fix)
- [ ] Build config updated (`pom.xml` / `build.gradle`)
- [ ] Removed/deprecated API usages addressed
- [ ] Module-system issues resolved (or `--add-opens` documented as temporary)
- [ ] Incompatible dependencies upgraded

### Recommended (should fix)
- [ ] Language modernization applied (records, pattern matching, text blocks, etc.)
- [ ] JVM flags reviewed and updated
- [ ] CI/CD pipeline JDK version updated
- [ ] Docker base image updated

### Verification
- [ ] Project compiles with `javac`/build tool at target version
- [ ] Full test suite passes
- [ ] Application starts and smoke tests pass
- [ ] Performance baseline compared (GC behavior, startup time)
```

Adjust the checklist items based on what was actually found — omit categories that
had no issues, and add project-specific items as needed.

---

## Output format

- Use Markdown with clear headings per issue category.
- For code changes, always use diff blocks (` ```diff `).
- Keep explanations concise — the "why" matters, but avoid lengthy prose when a
  code example speaks for itself.
- Present the migration summary checklist at the end.
- If the full codebase is not provided, frame the output as a migration guide
  (what to search for, what to replace) rather than direct file edits.
