# 15 — LinkedIn-specific concepts

Terms you can't fully understand by reading Coral source alone. They come from LinkedIn's surrounding data platform.

## Dali

LinkedIn's logical dataset platform. Provides versioned views, function registries, and a unified namespace over Hive/Iceberg storage. Coral is the engine that translates Dali views into target-dialect SQL at query time.

## Transport UDFs

A LinkedIn framework for writing UDFs once and running them on multiple engines (Hive, Spark, Trino, Presto). Each Transport UDF ships as an Artifactory artifact with engine-specific JARs. Coral detects Transport UDFs during translation and emits `SparkUDFInfo` records so callers know which JARs to load and which functions to register.

## ViewShift

Dynamic policy enforcement layer over data lakes. Uses Coral IR rewriting to inject access control, redaction, and similar policies into views at translation time. Talk: `docs/talks/coral-viewshift.pdf`.

## Fuzzy Union

A Coral mechanism for reconciling UNION branches with heterogeneous struct schemas — e.g., a view that unions two source tables where one added a struct field. `FuzzyUnionSqlRewriter` introduces the necessary CAST + null-padding so the union has a consistent type.

## Espresso

LinkedIn's distributed key-value store. Consumes Avro schemas. One of the downstream reasons coral-schema cares about Avro fidelity.

## Incremental View Maintenance (IVM)

LinkedIn uses Coral to rewrite views into incremental form, computing deltas instead of full recomputes. See chapter 14 (coral-incremental) and the IVM talk under `docs/talks/`.

## Read next

- Chapter 16 — PR review companion.
- Chapter 19 — glossary.
