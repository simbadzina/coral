# 12 — coral-benchmark: cross-dialect correctness

A framework for verifying that a query translated from source dialect to target dialect produces equivalent results. Three escalating verification levels mean you can validate cheaply or thoroughly.

## The three levels

- **TRANSLATION** — source SQL → IR → target SQL succeeds and matches an expected string. No engines required.
- **EXPLAIN** — target engine successfully parses and plans the translated SQL. Catches syntactic mismatches the IR roundtrip didn't.
- **RESULT_SET** — both source and target engines execute on the same in-memory data; results compare equal. Catches semantic differences.

## SPI

- `DialectPlugin` / `DialectPluginProvider` — wraps a dialect parser/printer. Hive plugin wraps `HiveToRelConverter`; Trino plugin wraps `RelToTrinoConverter`.
- `EnginePlugin` — start engine, create schemas, load data, explain, execute. Implementations live outside this module.
- `VerificationLevel` enum.

## Test orchestration

- `TranslationTestSuite` — iterates SQL files, runs through the chosen verification level, builds a `TestReport`.
- `InMemoryCatalog` / `InMemoryTable` — test-friendly catalog.
- `RowSet` — typed test data.
- `ResultSetComparator` + `ComparisonConfig` — equivalence rules (row ordering, NULL handling, float epsilon, timestamp precision).

## Why it matters

This module is a high-leverage place to contribute. Adding queries to the corpus is low-risk, high-value work that catches regressions across all backends.

## Files this chapter discusses

- `coral-benchmark/coral-benchmark-spec.md` (the design doc — read this first)
- `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/spi/`
- `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/suite/TranslationTestSuite.java`
- `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/catalog/InMemoryCatalog.java`
- `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/comparison/`
- `coral-benchmark/src/main/java/com/linkedin/coral/benchmark/data/`

## Read next

- Chapter 17 — first contributions (this module is in the easy-PR list).
- Chapter 13 — coral-data-generation (test data for benchmark).
