# 07 — The SqlCallTransformer pattern

The single most useful pattern in Coral. Every backend uses it; most PRs touch one. If you understand this, you can review half of Coral.

## The interface

- `SqlCallTransformer` — abstract. Two methods: `condition(SqlCall)` (should I fire?) and `transform(SqlCall)` (rewrite it).
- `SqlCallTransformers` — list-of-transformers composer. Applies in order; each transformer's output feeds the next.

## Concrete patterns

- `OperatorRenameSqlCallTransformer` — same operands, new name (`NVL → COALESCE`).
- `JsonTransformSqlCallTransformer` — replace a call with a JSON-templated rewrite (e.g., `RAND(n) → RANDOM([n])`).
- `SourceOperatorMatchSqlCallTransformer` — custom logic for matched operators.
- Dialect-specific transformers: `FromUtcTimestampOperatorTransformer`, `NamedStructToCastTransformer`, `SubstrOperatorTransformer`, ...

## Composition example

`CoralToTrinoSqlCallConverter` (chapter 09) wires ~20 transformers into a single `SqlCallTransformers.of(...)`. Order matters: a rename followed by a type adjustment behaves differently than the reverse.

## Anti-patterns

- Silent type narrowing inside `transform()` without preserving the original return type.
- Missing `condition()` precision — fires on too many calls.
- Mutating the input `SqlCall` instead of building a new one.
- Forgetting to override `unparse` on a custom `SqlOperator` (the rewrite works in tests but generates broken SQL).

## Files this chapter discusses

- `coral-common/src/main/java/com/linkedin/coral/common/transformers/SqlCallTransformer.java`
- `coral-common/src/main/java/com/linkedin/coral/common/transformers/SqlCallTransformers.java`
- `coral-common/src/main/java/com/linkedin/coral/common/transformers/OperatorRenameSqlCallTransformer.java`
- `coral-common/src/main/java/com/linkedin/coral/common/transformers/JsonTransformSqlCallTransformer.java`
- `coral-common/src/main/java/com/linkedin/coral/common/transformers/SourceOperatorMatchSqlCallTransformer.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/`
- `coral-spark/src/main/java/com/linkedin/coral/spark/transformers/`

## Read next

- Chapter 09 — coral-trino (the largest transformer collection).
- Chapter 16 — PR review companion (transformer-specific review checklist).
