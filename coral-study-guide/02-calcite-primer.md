# 02 — A Calcite primer for Coral readers

Just enough Apache Calcite to read Coral without constantly tab-switching to the Calcite docs.

## The two layers

- `SqlNode` — abstract syntax tree. Roughly 1:1 with the surface SQL. Subclasses: `SqlCall`, `SqlIdentifier`, `SqlLiteral`, `SqlNodeList`, `SqlSelect`, `SqlJoin`, ...
- `RelNode` — logical plan. Operators: `LogicalProject`, `LogicalFilter`, `LogicalJoin`, `LogicalAggregate`, `TableScan`, ...
- `RexNode` — row expressions inside a `RelNode` (the expression on a project, the predicate on a filter).

## The extension points Coral uses

- `SqlOperator` / `SqlOperatorTable` — function and operator definitions. Coral chains a `DaliOperatorTable` onto `SqlStdOperatorTable`.
- `SqlValidator` — type checks and resolves identifiers. Coral subclasses to `HiveSqlValidator`.
- `SqlToRelConverter` — turns validated `SqlNode` into `RelNode`. Coral subclasses to `HiveSqlToRelConverter`.
- `SqlRexConvertletTable` — rules for converting `SqlCall` expressions to `RexNode`. Coral's is `CoralConvertletTable`.
- `RelBuilder` / `RelDataTypeFactory` — fluent builders. Coral overrides via `HiveRelBuilder`, `HiveRexBuilder`, `HiveTypeSystem`.

## The FrameworkConfig handshake

`ToRelConverter` constructs a Calcite `FrameworkConfig` wiring all of the above together. Reading this chapter then `ToRelConverter` makes the relationship clear.

## What this chapter discusses

- Apache Calcite's public interfaces (no Coral code in this chapter).
- Pointers to where each interface is extended in Coral.

## Read next

- Chapter 03 — pipeline deep dive (the next chapter applies all of this).
- Chapter 04 — coral-common (where the extension points live).
