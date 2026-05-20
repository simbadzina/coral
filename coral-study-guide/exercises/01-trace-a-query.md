# Exercise 01 — Trace a query end to end

**Goal:** see every IR transition for one query.

## Setup

1. Open IntelliJ on the repo, run `./gradlew :coral-hive:compileTestJava` once to make sure ANTLR sources are generated.
2. Pick a small test in `HiveToRelConverterTest` (e.g., one that uses a base table and a `SELECT a, lower(b) FROM ...` shape).

## Breakpoints

- `ParseTreeBuilder.process(String sql, Table hiveView)` — see the `ASTNode` come in and the `SqlNode` come out.
- `HiveSqlNodeToCoralSqlNodeConverter` — observe normalization passes.
- `HiveSqlValidator.validate(SqlNode)` — see types annotated on the tree.
- `HiveSqlToRelConverter.convertQuery(SqlNode, ...)` — see the `RelNode` emerge.

## Deliverable

A note in your own copy (not committed) with:

- The string form of the SqlNode after `ParseTreeBuilder`.
- The string form of the SqlNode after `HiveSqlNodeToCoralSqlNodeConverter`.
- The `RelOptUtil.toString(rel)` of the final RelNode.

## What you'll learn

The exact shape of the IR at each stage. Once you've seen this for one query, you can imagine it for any future PR you review.
