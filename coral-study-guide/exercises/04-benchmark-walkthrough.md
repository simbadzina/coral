# Exercise 04 — Walk through coral-benchmark at TRANSLATION level

## Goal

See how `coral-benchmark` runs a corpus of `.sql` queries through Coral's translation pipeline without booting any engine. The module is new (PR #599) and currently ships only the framework — no test corpus, no engine plugins, the orchestrator's `run()` still throws `UnsupportedOperationException`. The exercise is therefore as much a tour of the SPI as a runnable walkthrough. After it you can sketch how a Hive→Trino test case will move through the framework once `run()` lands, and you know exactly where the missing pieces go.

## Prerequisites

- Read chapter 12 (`coral-benchmark: cross-dialect correctness`).
- Read `coral-benchmark/coral-benchmark-spec.md` start to finish — the spec is normative; the code is the spec's shadow until `run()` is implemented.
- The coral repo on `master` at or after commit `8f285d24` (the PR #599 merge).

## Steps

1. **Open the spec.** Read `coral-benchmark/coral-benchmark-spec.md`. Pay particular attention to the three verification levels and the SPI contract sections — those are the load-bearing pieces. The spec is more detailed than chapter 12 because the chapter assumes you have read it.

2. **Map the SPI to files.** Open each in `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/`:

   - `spi/Dialect.java` — three values: `HIVE`, `SPARK`, `TRINO`.
   - `spi/VerificationLevel.java` — `TRANSLATION < EXPLAIN < RESULT_SET`. The ordinal order is load-bearing.
   - `spi/DialectPlugin.java` — `toRelNode(String)` and `toDialectSql(RelNode)`. Two methods. The plugin is the wrapper around an existing Coral converter.
   - `spi/DialectPluginProvider.java` — `dialect()` and `create(CoralCatalog)`. The factory the orchestrator finds via `ServiceLoader`.
   - `spi/EnginePlugin.java` — `start`, `createTable`, `loadData`, `explain`, `execute`, `stop`. Only required for EXPLAIN and RESULT_SET levels.

   Notice that no concrete `DialectPlugin` implementation exists in the module. The spec lists them; the code does not ship them. This is intentional — chapter 12 explains why.

3. **Look at where the test entry point will go.** Run:

   ```bash
   ls /Users/sdzinama/dev/coral/coral-benchmark/src/
   ```

   You will see only `main/`. There is no `src/test/` yet. The `:coral-benchmark:test` task will run but execute zero tests:

   ```bash
   ./gradlew :coral-benchmark:test
   ```

   The output reports `BUILD SUCCESSFUL` with no test classes discovered. This is correct behavior for the current state of the module — the framework is in place, the corpus and the engine plugins are the next contributions.

4. **Read the orchestrator.** Open `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/suite/TranslationTestSuite.java`. The constructor and `Builder` are complete; `build()` validates that the right fields are set for the requested level (the `verificationLevel.ordinal() >= VerificationLevel.EXPLAIN.ordinal()` checks). The `run()` method body currently throws:

   ```java
   throw new UnsupportedOperationException("Not yet implemented");
   ```

   The javadoc on `run()` enumerates the six steps the implementation will perform — discover SQL files, start engines, create tables, translate, verify, stop engines, build report. This is the contract for the next implementation PR.

5. **Trace a single fixture mentally.** Pick a hypothetical query `SELECT count(*) FROM users GROUP BY country` saved as `count_by_country.sql`. The flow once `run()` is implemented:

   - `TranslationTestSuite.run()` discovers `.sql` files under the configured `queryDir` (default convention from the spec: `queries/<source-dialect>/`).
   - For each file, reads the SQL string.
   - Calls `sourcePlugin.toRelNode(sql)` — for Hive source, that wraps `HiveToRelConverter.convertSql`.
   - Calls `targetPlugin.toDialectSql(relNode)` — for Trino target, that wraps `RelToTrinoConverter.convert`.
   - If both succeed and produce non-empty SQL, the per-query result is `PASS` at TRANSLATION level.
   - If either throws, the per-query result is `FAIL` with `FailureCategory.TRANSLATION_ERROR`, capturing the exception.
   - All per-query `QueryTestResult`s accumulate into a `TestReport`.

   The shape of the `TestReport` is visible right now in `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/suite/TestReport.java` and `QueryTestResult.java`. `TestReport` exposes `passCount()`, `failCount()`, `passRate()`, `getFailures()`, and `failureCountsByCategory()`.

6. **Read the catalog and data types.** Open `catalog/InMemoryCatalog.java` and `catalog/InMemoryTable.java`. The catalog uses the existing `CoralCatalog` interface from `coral-common`, populated through a builder that enforces namespace-then-table order. Schemas use `CoralDataType` and its subtypes (`StructType`, `PrimitiveType`, etc.) — no parallel type system. This matters because any future `DialectPlugin` must accept a `CoralCatalog`, and the in-memory implementation is the canonical reference for what the framework expects.

7. **Look at the comparison shim.** Open `comparison/ResultSetComparator.java`. Its `compare()` method also throws `UnsupportedOperationException`. The `ComparisonConfig` next to it is complete and documents the five real-world ways two engines diverge on equivalent queries: row ordering, floating-point tolerance, NULL equivalence, timestamp precision, and type widening. The configuration is the stable contract; the comparison logic is the next implementation step.

## Checkpoints

- After step 3: `./gradlew :coral-benchmark:test` returns `BUILD SUCCESSFUL` with no test classes discovered. That is the correct state today.
- After step 5: you can describe, without looking, the chain `TranslationTestSuite.run → sourcePlugin.toRelNode → targetPlugin.toDialectSql → QueryTestResult → TestReport`.
- After step 6: you can recite the rule that schemas pass through `CoralDataType`, never raw strings.

## What you learn

Three things, in order:

1. **Where queries will live.** The spec's convention is `coral-benchmark/src/test/resources/queries/<source-dialect>/`. Once the test corpus exists, adding a regression test for a Hive→Trino translation gap is one new `.sql` file. No Java, no Gradle, no plugin glue — the framework's value is exactly this low-friction extension surface (chapter 17 lists "expand the benchmark corpus" as a canonical first PR for this reason).
2. **What a `TestReport` actually contains.** Per-query: source SQL, translated SQL, status, optional `ExplainResult`, optional `ComparisonResult`, `FailureCategory`, exception. Aggregate: pass/fail counts and category histograms. When reviewing a benchmark-related PR, you should expect to see assertions against these fields, not against engine-execution side effects.
3. **What is missing.** `TranslationTestSuite.run()` and `ResultSetComparator.compare()` are not yet implemented. No concrete `DialectPlugin` ships. No `META-INF/services` registration. The framework is scaffolding waiting for content. Chapter 17 spells out which of those gaps make good first contributions.

## Optional extensions

- **Mentally translate a translation failure.** Pick a Hive query you know `RelToTrinoConverter` mishandles — chapter 09 lists candidates; `regexp_extract` with a literal pattern is a common one. Walk it through the eventual flow: which level catches it (TRANSLATION? EXPLAIN? RESULT_SET?), which `FailureCategory` it lands under, what the `TestReport` entry would look like. The mental rehearsal makes the value of a corpus tangible.
- **Sketch a `DialectPluginProvider` for Hive.** In a scratch file (do not commit), write what `HiveDialectPluginProvider implements DialectPluginProvider` would look like: `dialect()` returns `Dialect.HIVE`, `create(CoralCatalog)` returns a `HiveDialectPlugin` that wraps a `HiveToRelConverter`. This is one of the canonical first PRs against this module.
