# Ramp paths

Time-boxed routes through this guide. Pick the one that matches the time you have. Every chapter carries a reading-time estimate in its front-matter block, so you can re-budget as you go.

The fastest way to become a useful reviewer is **not** to read the book front to back. It's to learn the pipeline, the one pattern that dominates PRs, and the review checklist — then ship a toy change. The reviewer ramp below does exactly that in 8–10 hours.

## Path 1 — Speed-run (~1 hour)

Enough to follow a code-review conversation and not be lost.

| Read | Time |
|---|---|
| [README](README.md) | 5 min |
| [01 — The big picture](01-the-big-picture.md) | 8 min |
| [03 — Pipeline deep dive](03-pipeline-deep-dive.md) | 9 min |
| [16 — PR review companion](16-pr-review-companion.md) (skim the per-module checklists) | 20 min |

End state: you can sketch the SQL → IR → SQL pipeline and name the review preflight checks.

## Path 2 — Reviewer ramp (8–10 hours)

The recommended path for ramping in private time. Five sessions; do them in order. End state: you can review a PR in any of the high-traffic modules and ship a first contribution.

| Session | Time | What to read / do |
|---|---|---|
| **1. Orientation** | ~2h | [01](01-the-big-picture.md) + [03](03-pipeline-deep-dive.md). If Calcite is rusty, skim [02](02-calcite-primer.md). Sketch the pipeline from memory before moving on. |
| **2. The pattern** | ~2h | [07 — transformers](07-transformers-pattern.md) + [exercise 01](exercises/01-trace-a-query.md) (trace a query in a debugger). |
| **3. Backend skim** | ~1.5h | [08 — coral-spark](08-coral-spark.md) and [09 — coral-trino](09-coral-trino.md): read the openings, the transformer-zoo overview, and the reviewer cheat sheets. Skip the per-transformer prose. |
| **4. PR review** | ~2h | [16 — PR review companion](16-pr-review-companion.md) in full, plus [04 — coral-common](04-coral-common.md) and [06 — coral-hive](06-coral-hive.md). |
| **5. First PR** | ~1.5h | [17 — first contributions](17-first-contributions.md) + [exercise 03](exercises/03-add-a-function.md) (add a toy function locally). |

**Skip until a PR sends you there:** [05](05-type-system-and-catalog.md), [10](10-coral-schema.md), [11](11-coral-spark-catalog.md), [12](12-coral-benchmark.md), [13](13-coral-data-generation.md), [14](14-other-modules.md), [15](15-linkedin-specifics.md), [18](18-engagement-and-community.md), [19](19-glossary.md). They're on-demand reference, not ramp material. (Keep [19 — glossary](19-glossary.md) open in a tab for term lookups.)

## Path 3 — Full mastery (~20+ hours)

Sequential, every chapter and every exercise. The route to committer-level depth.

- Read [00](00-prerequisites.md) → [19](19-glossary.md) in order. ~5 hours of reading.
- Do all four [exercises](exercises/) with a debugger and a real source tree. ~1–3 hours each — this is where the model actually lands.

## Highest leverage per hour

If you only optimize for review readiness, this is the priority order:

```
16 (review checklist)  >  07 (the transformer pattern)  >  03 (pipeline)  >
01 (big picture)  >  exercise 01 (trace a query)  >  08/09 skim  >  exercise 03 (add a function)
```

## Reading faster

- **Open the source file the chapter links, and read the prose beside the code.** The mental model lands in roughly half the time versus reading prose alone.
- **Redraw each Mermaid diagram from memory** before moving on. Forces synthesis; cheap.
- **Skip the "Files this chapter discusses" sections while learning.** They're navigation appendices, not study material.
- **Don't memorize file paths.** Recognize them when a PR touches them later. The links exist for jumping, not for recall.
- **Use the front-matter "Key takeaways"** as a pre-read (what to look for) and the **"Self-check"** questions as a post-read (did it land).

## Spaced-repetition schedule

Retention beats coverage when time is short. Space the material:

- **Day 1:** Speed-run (Path 1).
- **Week 1:** Reviewer-ramp sessions 1–4 (chapters 01, 03, 07, 08, 09, 04, 06, 16) + exercise 01.
- **Week 2–3:** Session 5 + the skipped chapters as PRs require + exercises 02 and 04.
- **Revisit the Self-check questions** at 1 day, 1 week, and 1 month after first reading a chapter. If you can answer all of them cold, that chapter is durable.

## Read next

- [README](README.md) — the table of contents.
- [01 — The big picture](01-the-big-picture.md) — start here regardless of path.
