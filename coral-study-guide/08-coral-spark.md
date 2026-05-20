# 08 — coral-spark: Hive → Spark SQL

`coral-spark` generates Spark SQL from Coral IR. It also returns metadata about UDFs Spark needs to register and JARs Spark needs on the classpath.

## Entry point

- `CoralSpark.create(RelNode, HiveMetastoreClient)` — main API. Returns a `CoralSpark` value with `getSparkSql()`, `getBaseTables()`, `getSparkUDFInfoList()`.
- `CoralSpark.create(RelNode, Schema, HiveMetastoreClient)` — variant that aligns output column names to an Avro schema (uses `AddExplicitAlias`).

## Pipeline

- `IRRelToSparkRelTransformer.transform(RelNode)` — IR-level RelNode rewrites returning `SparkRelInfo` (transformed `RelNode` + collected `SparkUDFInfo`s).
- `CoralRelToSqlNodeConverter` (from coral-common) — generic `RelNode` → `SqlNode`.
- `CoralSqlNodeToSparkSqlNodeConverter` — Spark-specific SqlNode rewrites.
- `CoralToSparkSqlCallConverter` — `SqlCallTransformers` chain.
- `DataTypeDerivedSqlCallConverter` — type-aware adjustments.
- `SparkSqlRewriter` + `SparkSqlDialect` — final SQL serialization.

## Spark-specific transformers

- `HiveUDFTransformer` — Hive UDF class names → Spark function names.
- `TransportUDFTransformer` — LinkedIn Transport UDFs; extracts Artifactory coordinates and emits `SparkUDFInfo` with classifier (Scala 2.11 / 2.12).
- `ExtractUnionFunctionTransformer`, `FuzzyUnionGenericProjectTransformer` — union/schema-evolution helpers.

## UDF metadata

- `SparkUDFInfo` carries everything Spark needs to register a UDF at runtime: class name, function name, artifactory URLs, type tag.
- Caller is expected to invoke `spark.sql("ADD JAR ...")` and register the function.

## Files this chapter discusses

- `coral-spark/src/main/java/com/linkedin/coral/spark/CoralSpark.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/IRRelToSparkRelTransformer.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/CoralSqlNodeToSparkSqlNodeConverter.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/CoralToSparkSqlCallConverter.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/DataTypeDerivedSqlCallConverter.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/SparkSqlRewriter.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/AddExplicitAlias.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/transformers/`
- `coral-spark/src/main/java/com/linkedin/coral/spark/containers/SparkUDFInfo.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/containers/SparkRelInfo.java`

## Read next

- Chapter 11 — coral-spark-catalog (runtime Spark integration).
- Chapter 15 — LinkedIn specifics (Transport UDFs in depth).
