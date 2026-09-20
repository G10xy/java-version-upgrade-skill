# Java LTS Migration Matrix

## LTS vs. non-LTS

Java ships a feature release every six months; only some are Long-Term Support.

| | Releases | Update window |
|---|---|---|
| **LTS** | 8, 11, 17, 21, 25, then every two years (29 next) | Years — the supported choice for production |
| **Non-LTS** | 9–10, 12–16, 18–20, 22–24, 26–28, … | ~6 months, until the next feature release supersedes it |

This file covers **LTS-to-LTS** hops only. If the target is a non-LTS release, apply the
guide for the LTS at or below it, then review the release notes of each intervening
version separately — and tell the user which versions were not covered.

## Supported migration paths

| From | To  | Key themes                                                        | Guides to read                       |
|------|-----|-------------------------------------------------------------------|--------------------------------------|
| 8    | 11  | Module system (JPMS), removed Java EE/CORBA modules, new APIs     | `java8-to-11.md`                     |
| 11   | 17  | Sealed classes, records, pattern matching, text blocks, switch expr| `java11-to-17.md`                    |
| 17   | 21  | Virtual threads, sequenced collections, UTF-8 + CLDR format changes | `java17-to-21.md`                    |
| 21   | 25  | SecurityManager removed, Unsafe warnings, scoped values, AOT      | `java21-to-25.md`                    |
| 8    | 17  | Cumulative: 8→11 + 11→17                                         | Both guides, in order                |
| 8    | 21  | Cumulative: 8→11 + 11→17 + 17→21                                 | Three guides, in order               |
| 8    | 25  | Cumulative: 8→11 + 11→17 + 17→21 + 21→25                        | All four guides, in order            |
| 11   | 21  | Cumulative: 11→17 + 17→21                                        | Both guides, in order                |
| 11   | 25  | Cumulative: 11→17 + 17→21 + 21→25                                | Three guides, in order               |
| 17   | 25  | Cumulative: 17→21 + 21→25                                        | Both guides, in order                |

## Risk summary by hop

| Hop     | Typical blocker count | Effort estimate      | Highest-risk area                                     |
|---------|-----------------------|----------------------|-------------------------------------------------------|
| 8 → 11  | Medium–High          | Days to weeks        | Removed Java EE modules, JPMS reflection breakage, version-string parsing |
| 11 → 17 | Low–Medium           | Hours to days        | Strong encapsulation becomes enforced (`--illegal-access` no longer works), Security Manager deprecation |
| 17 → 21 | Low                  | Hours to days        | **Silent behavior changes**: UTF-8 default charset, CLDR 42 date/time formats, `finalize()` removal, `URL` deprecation |
| 21 → 25 | Low–Medium           | Hours to days        | SecurityManager permanently disabled, `Unsafe` warnings, `COMPAT` locale data removed, Gradle 9 requirement |

> The 17 → 21 hop is the most commonly underestimated. Its blocker count is low, but its
> failures are *behavioral* rather than compile-time: code builds and starts fine, then
> produces subtly different strings, dates, and encodings. Budget testing time, not
> refactoring time.

## Multi-hop strategy

When jumping multiple LTS versions (e.g., 8 → 25), there are two approaches:

1. **Direct jump** (recommended for most projects): target the final version directly, applying all migration guides cumulatively. This avoids intermediate build changes.
2. **Stepwise** (for very large codebases or when you want incremental CI verification): migrate to each LTS in sequence, verifying the build and tests at each stop.

Either way, read the guides in ascending order so that earlier fixes aren't contradicted by later ones.
