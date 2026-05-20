# 11 — coral-spark-catalog: runtime view translation

A Spark 3.5 `CatalogExtension` that translates Hive views to Spark SQL on the fly when Spark looks them up. Users don't have to pre-translate.

## How it plugs in

- Spark config: `spark.sql.catalog.spark_catalog=com.linkedin.coral.spark.CoralSparkViewCatalog`.
- Spark calls into `CoralSparkViewCatalog.loadView(Identifier)` whenever a query references a view.

## What happens on loadView

1. Resolve the view from the underlying Hive metastore.
2. Extract its Hive SQL definition.
3. Run it through `CoralSpark.create(...)` (chapter 08).
4. Wrap the Spark SQL in a Spark `View` object and return.

## Delegation

- `createView`, `alterView`, `dropView` delegate to the underlying `SessionCatalog`.
- Function operations delegate to `SessionCatalog`.
- The class only intercepts the read path.

## Test surface

- `CoralSparkViewCatalogTest` — runs a real `SparkSession`, creates views, runs queries, asserts results. The closest thing to an end-to-end test in the repo.

## Files this chapter discusses

- `coral-spark-catalog/src/main/java/com/linkedin/coral/spark/CoralSparkViewCatalog.java`
- `coral-spark-catalog/src/test/java/com/linkedin/coral/spark/CoralSparkViewCatalogTest.java`
- `coral-spark-catalog/README.md`

## Read next

- Chapter 08 — coral-spark (the translator this module wraps).
- Chapter 12 — coral-benchmark (broader testing surface).
