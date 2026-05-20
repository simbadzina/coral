# 14 — Other modules

The remaining modules. Smaller surface, lower review frequency, but each pulls its weight.

## coral-pig

Hive → Pig Latin. Mostly legacy. `RelToPigLatinConverter` is the entry. Pig-specific RelNodes live under `rel/`. Uses LinkedIn-specific `dali.data.pig.DaliStorage` for table loading.

## coral-dbt

Applies Coral transformations to dbt models. Used in incremental view maintenance pipelines.

## coral-incremental

Rewrites a view definition into an incremental form. The transformation replaces `T` with `T_delta ∪ T` and for joins emits all combinations of delta and original. Entry: `RelNodeIncrementalTransformer.convertRelIncremental(RelNode)`.

## coral-visualization

Renders SqlNode and RelNode trees as graphviz SVGs. Useful for debugging gnarly transformations. Wrapped by `coral-service`.

## coral-service

Spring Boot REST service. Endpoints for translation and catalog ops. Two run modes: local Hive metastore (embedded Derby) or remote (real HMS). Has a UI under a separate frontend project.

## coral-spark-plan

Reverse direction: a Spark physical plan (JSON from `EXPLAIN`) → Coral RelNode. Useful for analyzing how Spark optimized a query.

## Files this chapter discusses

- `coral-pig/src/main/java/com/linkedin/coral/pig/`
- `coral-dbt/src/main/java/`
- `coral-incremental/src/main/java/`
- `coral-visualization/src/main/java/`
- `coral-service/src/main/java/`
- `coral-spark-plan/src/main/java/`

## Read next

- Chapter 16 — PR review companion (per-module checklists for these modules).
