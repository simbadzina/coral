# 04 — coral-common: the foundation

Every other module depends on `coral-common`. It is the abstract base, the schema layer, the Calcite glue, and the transformer plumbing.

## The abstract converter base

- `ToRelConverter` — template method that wires Calcite's `FrameworkConfig`, exposes `convertSql()` and `convertView()`, and delegates dialect specifics to abstract methods (`getConvertletTable`, `getSqlValidator`, `getOperatorTable`, `getSqlToRelConverter`, `toSqlNode`).
- Three constructors: legacy `HiveMetastoreClient`, modern `CoralCatalog`, and `localMetaStore` for tests.

## The schema layer

- `HiveSchema` / `LocalMetastoreHiveSchema` — Calcite `Schema` implementations backed by HMS or an in-memory map.
- `CoralRootSchema` / `CoralDatabaseSchema` — newer schema implementations backed by `CoralCatalog`.
- Adapters: `HiveCalciteTableAdapter`, `HiveCalciteViewAdapter`, `IcebergCalciteTableAdapter`.

## Calcite extensions

- `HiveRelBuilder` — `RelBuilder` subclass that handles Hive's UNNEST + struct-array quirks.
- `HiveRexBuilder` — custom RexBuilder.
- `HiveTypeSystem` — type precision and scale rules matching Hive semantics.
- `HiveUncollect` — Coral's variant of `Uncollect` used for `LATERAL VIEW OUTER EXPLODE`.

## Cross-cutting machinery

- `FuzzyUnionSqlRewriter` — reconciles union branches with heterogeneous struct schemas (covered in chapter 15).
- `functions/` package — base `Function`, `GenericProjectFunction`, `FunctionFieldReferenceOperator`, `CoralSqlUnnestOperator`.
- `transformers/` package — `SqlCallTransformer`, `SqlCallTransformers`, base implementations. Full treatment in chapter 07.
- `catalog/` package — `CoralCatalog`, `CoralTable`. Full treatment in chapter 05.
- `types/` package — `CoralDataType` hierarchy. Full treatment in chapter 05.

## Files this chapter discusses

- `coral-common/src/main/java/com/linkedin/coral/common/ToRelConverter.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveSchema.java`
- `coral-common/src/main/java/com/linkedin/coral/common/CoralRootSchema.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveRelBuilder.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveTypeSystem.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveUncollect.java`
- `coral-common/src/main/java/com/linkedin/coral/common/FuzzyUnionSqlRewriter.java`

## Read next

- Chapter 05 — type system and CoralCatalog.
- Chapter 07 — the transformer pattern.
