# 16 — PR review companion

The chapter you live in as a reviewer. Per-module checklists, red flags, and reusable comment templates.

## Universal preflight

Before reading the diff:

- Does the build pass? (CI check)
- Was `./gradlew spotlessApply` run? (`Spotless` failure = first comment)
- Are there tests for the change?
- Does the commit message style match `git log` (imperative subject, body explains why)?
- For non-trivial changes: is there a linked issue per `CONTRIBUTING.md`?

## Per-module quick reference

### coral-common
- Touches to `ToRelConverter`: do both constructors (deprecated MSC + modern CoralCatalog) still work?
- Touches to `HiveTypeSystem`: precision changes can ripple to every backend — flag for regression testing.

### coral-hive
- New Hive function: must be added to `StaticHiveFunctionRegistry` AND have a test.
- Parser changes (`.g` files): regenerate sources locally; CI should fail otherwise.
- `HiveViewExpander` recursion changes: stress-test with deeply nested views.

### coral-trino, coral-spark
- New transformer: where does it sit in `SqlCallTransformers.of(...)`? Order matters — verify by reading neighbors.
- Make sure `unparse` is overridden if a custom `SqlOperator` is introduced.
- Type-aware changes: did `DataTypeDerivedSqlCallConverter` get updated too?

### coral-schema
- Avro nullability: did the change preserve base-table nullability for pass-through columns?
- Iceberg changes: prefer `MergeCoralSchemaWithAvro` over `MergeHiveSchemaWithAvro`.

### coral-spark-catalog
- View name casing: Spark and Hive disagree. Test asserts must check exact casing.

### coral-benchmark, coral-data-generation
- New SPI methods: are they breaking changes for existing plugin implementers?
- New transformers in coral-data-generation: are they sound (no false positives)?

## Red flags

- `// TODO` in test corpus.
- Swallowed exceptions in transformers (`catch (Exception e) {}`).
- Regex-based SQL fixups in places where SqlNode rewrites would be cleaner.
- Silent type narrowing (`VARCHAR(255)` → `VARCHAR`).
- New direct uses of `org.apache.hadoop.hive.metastore.api.Table` outside the catalog adapters — should go through `CoralCatalog` / `CoralTable`.
- A PR that adds a function on the Hive side without touching the Spark or Trino side (likely incomplete).

## Reading the test diff first

`HiveToRelConverterTest`, `HiveToTrinoConverterTest`, `CoralSparkTest` are excellent specs. If a PR's test diff is small or absent, that's the headline comment.

## Reusable comment templates

```
Could you add a test in `HiveToTrinoConverterTest` covering [edge case]?
The current change passes [basic case] but I'm not sure about [edge].
```

```
This transformer is added at position N in `SqlCallTransformers.of(...)`.
Order matters: it currently fires after `XTransformer`. Did you intend
that ordering, or should it precede X?
```

```
Spotless is failing. Run `./gradlew spotlessApply` and amend.
```

```
This looks Hive-only. Should we also add the equivalent transformer in
coral-trino / coral-spark to keep dialect parity?
```

## Files this chapter discusses

This chapter is meta — it discusses every other module. Use the per-module sections above as jump points.

## Read next

- Chapter 17 — first contributions (the offensive counterpart to this defensive chapter).
