---
inclusion: always
---
# Historical sources policy

## Primary source
General Electric Review, vol. 17, no. 1, January 1914 (public domain). Internet Archive item `generalelectricr17gene`. Abbreviations used in code and docs:
- **S**: Edward Schildhauer, "The Control of Lock Machinery on the Panama Canal", pp. 7-15
- **He**: E. M. Hewlett, "Centralized Control System and Power Station Switchboards for the Panama Canal", pp. 16-18
- **HH**: C. H. Hill and C. T. Hentschel, "Lock Control Boards and Interlocking System for the Panama Canal", pp. 19-30
- **U**: J. W. Upp, "An Instance of Co-operation", pp. 3-4

Page numbers always refer to the original magazine pages.

## Rules for any historical claim (code, UI text, docs, commits)
1. Every interlock rule in the model MUST carry a `source` (article + page, and figure if any) and a `status`: `documented` or `inferred`.
2. NEVER invent an interlock. If a rule is needed for the model but is not in the sources, mark it `inferred` and explain why in one sentence.
3. Do not present a modeling simplification as historical fact. Simplifications are listed in `docs/MODEL.md`.
4. Known traps. These were **NOT** mechanical interlocks in 1914; do not claim they were:
   - Equal water levels before opening a gate. The operator judged this from the level indicators (S p.15, He p.17).
   - The maximum filling rate. The operator controlled it (S p.15).
   - End of travel. Limit switches handled it, not interlocks (S p.8).
5. The interlocks acted on **control handle positions**, not machine positions. The only documented exception is the electrical interlock that requires the gates to be fully closed before the miter-forcing machine closes (S p.13).
6. When unsure, ask the developer instead of guessing.

## Wording for verification results
Say "unreachable within the encoded model". Never say "proven safe" about the real Canal or about real infrastructure.
