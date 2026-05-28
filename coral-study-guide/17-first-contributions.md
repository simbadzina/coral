# 17 — First contributions

Six concrete, low-risk PR ideas. Each is grounded in a real recent template PR you can read first, names the files you'll touch, and the test class to extend. Pick one, ship it, then come back for the second. After the six ideas, a list of areas to leave alone for now, the submission process, and a 90-day plan that gets you from reader to contributor.

> **Reading time** ~11 min  ·  **Prerequisites** [chapter 16](16-pr-review-companion.md)
>
> **Key takeaways**
> - The lowest-risk first PRs register a missing Trino UDF, write a `SqlCallTransformer` subclass, expand the `coral-benchmark` corpus, document an undocumented module, clear a Javadoc warning batch, or port a transformer between `coral-trino` and `coral-spark`.
> - Avoid `CoralCatalog`, `MergeCoralSchemaWithAvro`, the `coral-data-generation` symbolic solver, `HiveTypeSystem` precision rules, and the `shading/coral-trino-parser` config for a first PR — these are in flux or carry high scrutiny.
> - The submission process from `CONTRIBUTING.md` is fixed: open an issue first for non-trivial changes, branch off `master`, run `./gradlew clean build` before opening, write an imperative-mood title that Shipkit turns into a release note, and accept the CLA.

## Easy first PRs

### 1. Register a missing Hive UDF for Trino

**Template PRs**: #552 "Register `ExtractCollectionUdf` as Trino UDF" and #576 "Register a UDF".

Pattern: Coral knows about a Hive UDF (it's in `StaticHiveFunctionRegistry`), but `coral-trino` does not yet map it to the Trino equivalent. The fix is one of two shapes — either the names match and you add a `CoralRegistryOperatorRenameSqlCallTransformer` entry, or the signature differs and you write a small subclass (see idea 2).

**Files to touch**:
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/CoralRegistryOperatorRenameSqlCallTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/CoralRegistryOperatorRenameSqlCallTransformer.java) — the central registry of name-only renames. Most additions are a single line in the wiring class that builds the transformer list.
- Add the test in [`coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java`](../coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java). The existing tests are excellent specs; mirror their `assertEquals(expected, translated)` shape.

**How to pick a UDF**: scan `StaticHiveFunctionRegistry` for a Hive function that doesn't appear anywhere under `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/`. Cross-check by writing a failing test first — translate a query that calls the UDF and watch Coral emit the wrong Trino name.

**Caveat**: confirm the Trino UDF exists and has the same semantics. Trino's date/time and string functions diverge subtly from Hive's. If signatures differ, this is idea 2, not idea 1.

### 2. Add a Hive → Trino function transformer

When names match but signatures don't (different argument order, default parameters, different return types), a rename isn't enough — you need a `SqlCallTransformer` subclass that rewrites the call shape.

**Template files** (read first):
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/FromUtcTimestampOperatorTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/FromUtcTimestampOperatorTransformer.java) — example where the Trino equivalent takes different argument order.
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/SubstrOperatorTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/SubstrOperatorTransformer.java) — example with index-base translation (Hive 1-based vs Trino 1-based, with edge cases around zero).

**Files to touch**:
- New class under `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/`.
- Register it in the list passed to `SqlCallTransformers.of(...)` in `RelToTrinoConverter`. Order matters — [chapter 07](07-transformers-pattern.md) covers this.
- Test in `HiveToTrinoConverterTest`.

**Caveat**: read [chapter 07](07-transformers-pattern.md) (transformer pattern) before starting. The order of transformers in `SqlCallTransformers.of(...)` controls firing precedence, and a misplaced new transformer can swallow calls another transformer expected to see.

### 3. Expand the coral-benchmark query corpus

The benchmark module landed in #599 and the test corpus is small. Adding realistic queries pays compounding interest: each one becomes a regression test for every future translation PR.

**What to add**: ten queries covering UNION ALL, CTEs, window functions (`ROW_NUMBER`, `RANK`), `LATERAL VIEW EXPLODE` over arrays and structs, correlated subqueries, and aggregate-with-having patterns. Pull shapes from `HiveToTrinoConverterTest` or `HiveToRelConverterTest` if you need inspiration — they are essentially the spec for what Coral supports today.

**Files to touch**:
- Queries go under `coral-benchmark/src/test/resources/` (read [chapter 12](12-coral-benchmark.md) for the layout that ships with the module).
- A test entry that runs each query through `TranslationTestSuite` at the `TRANSLATION` verification level. `EXPLAIN` and `RESULT_SET` require an engine — start with `TRANSLATION` only.

**Caveat**: every query you add must actually translate cleanly. If you find one that doesn't, that's not a corpus PR — file an issue and consider it for idea 1 or 2 instead.

### 4. Document an undocumented module

[`coral-spark-catalog/README.md`](../coral-spark-catalog/README.md) (#594) and [`coral-data-generation/README.md`](../coral-data-generation/README.md) (introduced with #564) exist. `coral-incremental/` and `coral-pig/` have no module README; `coral-spark-plan` and `coral-visualization` lack rich introductions too. A short, accurate README modeled on [`coral-spark-catalog/README.md`](../coral-spark-catalog/README.md) is a high-signal first PR.

**Files to touch**:
- Create `coral-incremental/README.md` (or `coral-pig/README.md`). Read [`coral-incremental/src/main/java/com/linkedin/coral/incremental/RelNodeIncrementalTransformer.java`](../coral-incremental/src/main/java/com/linkedin/coral/incremental/RelNodeIncrementalTransformer.java) — that's the whole module, one class — and document what it does and how callers use it.
- Mirror the structure of [`coral-spark-catalog/README.md`](../coral-spark-catalog/README.md): one paragraph thesis, "How it works" with numbered steps, "Configuration" or "Usage", "Dependencies".

**Caveat**: don't invent capabilities. If the module does one thing, say one thing.

### 5. Clean up a Spotless or Javadoc warning batch

`./gradlew check` surfaces them. PRs that fix Javadoc broken links, missing `@param` tags, or stale Javadoc references are usually accepted quickly — they're easy to review and uncontroversial.

**How to find them**: run `./gradlew javadoc` and scan stderr for warnings. Or `./gradlew spotlessCheck` followed by `./gradlew spotlessApply` — but spotless is almost always run by maintainers, so this is rarer. Javadoc cleanup is the better bet.

**Caveat**: keep the PR scoped. "Fix all Javadoc in coral-common" is a sensible scope; "Fix all Javadoc in the repo" is a 1500-line diff nobody wants to review. One module per PR.

### 6. Port a transformer between coral-trino and coral-spark

When the same Hive surface gets fixed for one backend but not the other. Example: a Hive UDF that's been registered for Trino but not Spark, or a SQL shape that `coral-trino` handles but `coral-spark` does not.

**How to find them**: pick a recent `coral-trino` transformer PR (search `git log -- coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers`), then check whether `coral-spark/src/main/java/com/linkedin/coral/spark/transformers/` has a parallel. If it doesn't, that's your PR.

**Files to touch**:
- New transformer class in `coral-spark/src/main/java/com/linkedin/coral/spark/transformers/`.
- Test in [`coral-spark/src/test/java/com/linkedin/coral/spark/CoralSparkTest.java`](../coral-spark/src/test/java/com/linkedin/coral/spark/CoralSparkTest.java).

**Caveat**: `coral-spark` and `coral-trino` use different operator tables. The transformer playbook is identical ([chapter 07](07-transformers-pattern.md)), but the target operator names and dialect quirks differ. Read both `RelToTrinoConverter` and `CoralSpark` to see how their transformer lists are assembled before porting.

## What to avoid as a first PR

- **Anything touching `CoralCatalog`** — issue #575 covers an active migration off the deprecated `HiveMetastoreClient`. The surface is changing under your feet. Read [chapter 05](05-type-system-and-catalog.md) to understand the migration before going near it.
- **`MergeCoralSchemaWithAvro`** — landed in #600, paired with #604, and the team is likely still tuning. Wait for the dust to settle.
- **The symbolic solver in `coral-data-generation`** — `DomainInferenceProgram`, `DnfRewriter`, `RegexToIntegerDomainConverter` are deep semantics. Soundness bugs here are subtle and very embarrassing. Read [chapter 13](13-coral-data-generation.md) first; even then, don't make this your first PR.
- **`HiveTypeSystem` precision rules** — precision changes ripple to every backend. [Chapter 16](16-pr-review-companion.md) lists this as a red flag for reviewers; that means reviewers will scrutinize hard.
- **Anything in `shading/coral-trino-parser`** — the shading config is delicate (ANTLR v3 vs v4 conflict avoidance) and not where you want to spend a first PR.

## Process

The submission process, condensed from `CONTRIBUTING.md`:

1. **Open an issue first** for anything non-trivial. `CONTRIBUTING.md` is explicit: "Large features which have never been discussed are unlikely to be accepted." Even small PRs benefit from a five-line issue describing intent — it gives a maintainer a chance to redirect before you spend time.
2. **Branch off `master`**, push to your fork. Coral uses GitHub's standard fork-and-PR model.
3. **Run `./gradlew clean build` locally before opening.** This is non-negotiable — CI will reject you otherwise. If Spotless complains, run `./gradlew spotlessApply` and amend.
4. **Open the PR, link the issue.** Use a clear, imperative-mood title — Shipkit generates release notes from PR titles, so a bad title becomes a bad release note ([chapter 18](18-engagement-and-community.md) covers this).
5. **Sign the CLA** (the contribution agreement at the top of `CONTRIBUTING.md`) — submitting a PR is implicit acceptance.
6. **CI must pass.** Re-request review after addressing feedback. Don't force-push during review unless you're rebasing on master to resolve a conflict.

## A first-90-days plan

A month-by-month plan to go from reading the codebase to being a recognized contributor.

**Month 1 — orient.**
- Read chapters 01–05 of this study guide in week 1.
- Run `./gradlew clean build` in week 1 — confirm the toolchain works on your machine.
- Work through [`coral-study-guide/exercises/01-trace-a-query.md`](../coral-study-guide/exercises/01-trace-a-query.md) and `02-write-a-transformer.md` in weeks 2–3.
- Open an issue in week 4 describing one piece of the codebase you found confusing. Even if you don't propose a fix, this puts your name in the issue tracker and starts a conversation.

**Month 2 — first PRs.**
- Ship idea 4 (a missing module README) in week 5. Low-stakes, high-signal, gets you through the CLA, CI, and review loop for the first time.
- Ship idea 1 or 2 (a Trino UDF registration or transformer) in weeks 6–8. This is your first real code contribution. Reviewers will be more thorough; that's the point.
- Start watching the issue tracker — subscribe to notifications on issues touching modules you've read.

**Month 3 — engage.**
- Ship idea 3 (benchmark corpus expansion) — by now you understand which queries are interesting, and the corpus PR doubles as documentation of what you've learned.
- Review one PR a week, even if just to ask a clarifying question. [Chapter 16](16-pr-review-companion.md) is the checklist.
- Comment on a design discussion (issue #575 and similar are good places — see [chapter 18](18-engagement-and-community.md)). Offer a perspective, not just a question.
- By the end of month 3, you should have 3–4 merged PRs and a few thoughtful review comments. That's the threshold where maintainers start cc'ing you on PRs in your areas.

## Self-check

1. For a missing Trino UDF, what distinguishes idea 1 (a `CoralRegistryOperatorRenameSqlCallTransformer` entry) from idea 2 (a `SqlCallTransformer` subclass)?
2. The process section lists `CONTRIBUTING.md`'s steps — which one is violated most often, and why does skipping it waste your time?
3. Why does the chapter steer first-time contributors away from `CoralCatalog` and `HiveTypeSystem`, and which chapters explain the migration and the cross-backend ripple (see [chapter 16](16-pr-review-companion.md))?

## Files this chapter discusses

- `CONTRIBUTING.md`
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/CoralRegistryOperatorRenameSqlCallTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/CoralRegistryOperatorRenameSqlCallTransformer.java)
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/FromUtcTimestampOperatorTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/FromUtcTimestampOperatorTransformer.java)
- [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/SubstrOperatorTransformer.java`](../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/SubstrOperatorTransformer.java)
- [`coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java`](../coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java)
- `coral-spark/src/main/java/com/linkedin/coral/spark/transformers/`
- [`coral-spark/src/test/java/com/linkedin/coral/spark/CoralSparkTest.java`](../coral-spark/src/test/java/com/linkedin/coral/spark/CoralSparkTest.java)
- [`coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java`](../coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java)
- [`coral-incremental/src/main/java/com/linkedin/coral/incremental/RelNodeIncrementalTransformer.java`](../coral-incremental/src/main/java/com/linkedin/coral/incremental/RelNodeIncrementalTransformer.java)
- [`coral-spark-catalog/README.md`](../coral-spark-catalog/README.md)
- [`coral-data-generation/README.md`](../coral-data-generation/README.md)
- [`coral-benchmark/src/main/java/com/linkedin/coral/benchmark/suite/TranslationTestSuite.java`](../coral-benchmark/src/main/java/com/linkedin/coral/benchmark/suite/TranslationTestSuite.java)

## Read next

- [Chapter 18](18-engagement-and-community.md) — engagement and community (where you'll talk about the PR you just opened).
- [Chapter 16](16-pr-review-companion.md) — PR review companion (review with the same checklist you'll be reviewed against).
- [Chapter 07](07-transformers-pattern.md) — the transformer pattern (required reading before ideas 1, 2, and 6).
