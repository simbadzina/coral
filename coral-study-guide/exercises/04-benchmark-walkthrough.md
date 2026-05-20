# Exercise 04 — Walk through coral-benchmark at TRANSLATION level

**Goal:** see the benchmark module work end-to-end without needing real engines.

## Steps

1. Read `coral-benchmark/coral-benchmark-spec.md` once.
2. Locate the test entry point in `coral-benchmark/src/test/`.
3. Run it: `./gradlew :coral-benchmark:test`.
4. Identify how a single SQL fixture flows through `TranslationTestSuite` → `DialectPlugin` (source) → `DialectPlugin` (target) → `TestReport`.

## What you'll learn

- Where queries live in the corpus.
- How to add a new query (this is candidate first-PR territory — see chapter 17).
- What a `TestReport` actually contains.

Optional: after running, scan the `TestReport` for queries that fail at TRANSLATION. Each failure is a translation gap — worth a focused PR.
