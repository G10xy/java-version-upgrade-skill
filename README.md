# Java Version Upgrade — Agent Skill

An [Agent Skill](https://agentskills.io) that guides upgrading Java codebases between LTS versions (8 → 11 → 17 → 21 → 25).

Give it a `pom.xml`, `build.gradle`, or source files and it will detect the current Java version, audit for removed/deprecated APIs, produce concrete code diffs, update build configuration, flag incompatible dependencies, and deliver a migration checklist — so you can upgrade with confidence instead of guesswork.

## Supported migration paths

| From | To | Key themes |
|---|---|---|
| 8 | 11 | Module system (JPMS), removed Java EE/CORBA modules, new APIs |
| 11 | 17 | Records, sealed classes, pattern matching, text blocks, switch expressions |
| 17 | 21 | Virtual threads, sequenced collections, UTF-8 default, finalize removal |
| 21 | 25 | SecurityManager removed, Unsafe warnings, scoped values, stream gatherers, AOT |
| 8 | 17/21/25 | Multi-hop — all relevant guides applied cumulatively |

## What it does

1. **Detects** current and target Java version from build files
2. **Audits** source code for removed APIs, JPMS issues, and deprecated patterns
3. **Updates** Maven and Gradle build configuration with diff blocks
4. **Refactors** code with before/after examples (records, pattern matching, virtual threads, text blocks, etc.)
5. **Checks dependencies** against a compatibility matrix of 30+ popular libraries
6. **Produces** a migration summary checklist (blockers → recommended → verification)

## Installation

This skill follows the [Agent Skills open standard](https://agentskills.io/specification) and works with any compatible agent.

### Claude Code

```bash
# Global (all projects)
cp -r java-version-upgrade ~/.claude/skills/

# Per-project
cp -r java-version-upgrade your-project/.claude/skills/
```

### GitHub Copilot (VS Code / CLI / coding agent)

```bash
cp -r java-version-upgrade your-project/.github/skills/
```

### Cursor

```bash
cp -r java-version-upgrade your-project/.cursor/skills/
```

### OpenAI Codex

```bash
cp -r java-version-upgrade your-project/.codex/skills/
```

### Claude.ai

Zip the `java-version-upgrade` folder and upload it in **Settings → Customize → Skills**.

## Usage

The skill triggers automatically when you ask your agent something like:

- *"Upgrade this project from Java 8 to 21"*
- *"We're still on Java 8, what breaks if we move to 17?"*
- *"Audit this codebase for Java 11 to 21 migration issues"*
- *"Bump the JDK version to 21 and update the pom.xml"*

Or invoke it explicitly with `/java-version-upgrade` (in agents that support slash commands).

## Skill structure

```
java-version-upgrade/
├── SKILL.md                          # Main skill instructions
├── LICENSE                           # Apache 2.0
├── README.md
└── resources/
    ├── migration-matrix.md           # Overview of all migration paths
    ├── java8-to-11.md                # Blockers, build config, code examples
    ├── java11-to-17.md               # Blockers, build config, code examples
    ├── java17-to-21.md               # Blockers, build config, code examples
    ├── java21-to-25.md               # Blockers, build config, code examples
    └── common-dependencies.md        # Compat matrix for 30+ popular libraries
```

## Example output

When given a Maven project on Java 8 targeting Java 17, the skill produces:

- **Build config diffs** — updated `pom.xml` with compiler release, plugin versions, and new dependencies (e.g., Jakarta JAXB replacing `javax.xml.bind`)
- **Dependency flags** — Lombok 1.18.12 → 1.18.34, Mockito 3.x → 4.x, Jackson 2.11 → 2.14+, etc.
- **Code refactoring** — pattern matching for `instanceof`, switch expressions, text blocks, `List.of()`, `String.isBlank()`
- **Migration checklist** — blockers, recommended improvements, and verification steps

## Contributing

Contributions are welcome! Some ideas:

- Add coverage for future LTS versions as they are released
- Expand the dependency compatibility matrix
- Add more code refactoring examples for specific frameworks
- Improve triggering for edge-case prompts

## License

[Apache 2.0](LICENSE)
