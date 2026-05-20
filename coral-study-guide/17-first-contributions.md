# 17 — First contributions

Concrete, low-risk PR ideas grounded in patterns observed in the recent commit history.

## Easy first PRs

### 1. Register a missing Hive UDF for Trino

Recent template PRs: #552 (Register `ExtractCollectionUdf` as Trino UDF), #576.

Pick a Hive UDF that has no Trino mapping. Add the mapping in `coral-trino/.../rel2trino/transformers/` (often a one-liner using `CoralRegistryOperatorRenameSqlCallTransformer`). Add a test in `HiveToTrinoConverterTest`.

### 2. Add a Hive → Trino function transformer

When a function name matches but signatures differ. Mirror `FromUtcTimestampOperatorTransformer` or `SubstrOperatorTransformer` as a template. Single-class addition + test.

### 3. Expand the coral-benchmark query corpus

The benchmark module is new and the corpus is small. Add 10 representative queries spanning UNION, CTE, window functions, and EXPLODE. Inherently useful, low blast radius.

### 4. Document an undocumented module

`coral-spark-catalog/README.md` and `coral-data-generation/README.md` exist; `coral-incremental` and `coral-pig` do not have one. Write a short README modeled on the existing ones.

### 5. Clean up a Spotless or Javadoc warning batch

`./gradlew check` surfaces them. PRs that clean these up are usually welcomed.

### 6. Port a coral-trino transformer pattern to coral-spark

When a parallel mismatch exists but only one backend fixed it. The transformer playbook is identical; the operator table differs.

## What to avoid as a first PR

- Anything touching `CoralCatalog` (active migration, breaking surface).
- `MergeCoralSchemaWithAvro` (recent, likely still iterating).
- The symbolic solver (`coral-data-generation`) — deep semantics, easy to get wrong.
- `HiveTypeSystem` precision rules — ripples to every backend.

## Process

1. Open an issue before a non-trivial PR — `CONTRIBUTING.md` makes this explicit.
2. Branch off `master`, push to your fork.
3. Run `./gradlew clean build` locally before opening.
4. Open PR, link the issue.
5. CI must pass. Re-request review after fixing feedback.

## Read next

- Chapter 18 — engagement and community (who reviews, where conversations happen).
- Chapter 16 — PR review companion (review with the same checklist you'll be reviewed against).
