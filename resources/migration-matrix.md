# Java LTS Migration Matrix

## Supported migration paths

| From | To  | Key themes                                                        | Guides to read                       |
|------|-----|-------------------------------------------------------------------|--------------------------------------|
| 8    | 11  | Module system (JPMS), removed Java EE/CORBA modules, new APIs     | `java8-to-11.md`                     |
| 11   | 17  | Sealed classes, records, pattern matching, text blocks, switch expr| `java11-to-17.md`                    |
| 17   | 21  | Virtual threads, sequenced collections, finalize removal, UTF-8   | `java17-to-21.md`                    |
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
| 8 → 11  | Medium–High          | Days to weeks        | Removed Java EE modules, JPMS reflection breakage     |
| 11 → 17 | Low–Medium           | Hours to days        | Stronger internal encapsulation, Security Manager dep. |
| 17 → 21 | Low                  | Hours to days        | UTF-8 default charset, `finalize()` removal, URL dep. |
| 21 → 25 | Low–Medium           | Hours to days        | SecurityManager permanently gone, Unsafe warnings, dynamic agent restrictions |

## Multi-hop strategy

When jumping multiple LTS versions (e.g., 8 → 25), there are two approaches:

1. **Direct jump** (recommended for most projects): target the final version directly, applying all migration guides cumulatively. This avoids intermediate build changes.
2. **Stepwise** (for very large codebases or when you want incremental CI verification): migrate to each LTS in sequence, verifying the build and tests at each stop.

Either way, read the guides in ascending order so that earlier fixes aren't contradicted by later ones.
