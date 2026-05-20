# 09 — coral-trino: bidirectional translation

`coral-trino` is the only backend that's also a frontend. It generates Trino SQL from Coral IR and parses Trino SQL into Coral IR.

## Hive → Trino pipeline

- `HiveToTrinoConverter.toTrino(view)` — convenience that composes the two stages.
- Stage 1: `HiveToRelConverter` (from coral-hive) produces `RelNode`.
- Stage 2: `RelToTrinoConverter` (extends Calcite `RelToSqlConverter`) produces a Trino SqlNode tree.
- Then: `CoralToTrinoSqlCallConverter` + `DataTypeDerivedSqlCallConverter` + `TrinoSqlRewriter` + `TrinoSqlDialect`.

## The transformer zoo

20+ transformers under `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/`. The interesting ones:

- `FromUtcTimestampOperatorTransformer` — `from_utc_timestamp(ts, tz)` is rewritten to `CAST(at_timezone(from_unixtime_nanos(... * 1e6), $canonicalize_hive_timezone_id(tz)) AS TIMESTAMP(3))`.
- `NamedStructToCastTransformer` — `named_struct(...)` → row constructor + cast.
- `SubstrOperatorTransformer`, `SubstrIndexTransformer`, `ConcatOperatorTransformer` — string semantics differences.
- `UnnestOperatorTransformer` — Trino's `UNNEST` differs from Hive's `LATERAL VIEW`.
- `NullOrderingTransformer` — `NULLS FIRST` / `NULLS LAST` defaults differ.
- `CoralRegistryOperatorRenameSqlCallTransformer` — bulk renames (`NVL → COALESCE`, `RAND → RANDOM`).

## Trino → Coral IR

- `TrinoToRelConverter` — uses `TrinoParserDriver` (shaded; see `shading/coral-trino-parser`).
- `Trino2CoralOperatorConverter` + `Trino2CoralOperatorTransformerMap` — reverse operator mapping.
- POC quality: views are assumed Hive-defined; only base tables in Trino queries are fully supported.

## Shaded parser

- `shading/coral-trino-parser/` repackages Trino's parser + Airlift + ANTLR v4 into `coral.shading.io.trino.*` to avoid classpath conflicts with Calcite (which uses ANTLR v3).

## Files this chapter discusses

- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/RelToTrinoConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/CoralToTrinoSqlCallConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/DataTypeDerivedSqlCallConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/TrinoSqlRewriter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/TrinoSqlDialect.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/`
- `coral-trino/src/main/java/com/linkedin/coral/trino/trino2rel/TrinoToRelConverter.java`
- `shading/coral-trino-parser/`

## Read next

- Chapter 12 — coral-benchmark (cross-dialect correctness — the validation surface for trino changes).
- Chapter 16 — PR review companion (trino-specific checklist).
