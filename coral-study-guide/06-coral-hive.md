# 06 — coral-hive: the reference frontend

`coral-hive` is the most-used frontend and the template every other parser follows. It also accepts Spark SQL since Spark is largely Hive-compatible.

## ANTLR grammar

- `src/main/antlr/roots/` — root grammar (lexer + parser).
- `src/main/antlr/imports/` — `FromClauseParser.g`, `SelectClauseParser.g`, `IdentifiersParser.g`.
- Output: ANTLR-generated parser produces `ASTNode` trees.

## ParseTreeBuilder

- Extends `AbstractASTVisitor<SqlNode, ParseContext>`.
- Walks `ASTNode` tree, builds Calcite `SqlNode` tree.
- Resolves function names via `HiveFunctionResolver` during the walk.
- Returns a `SqlNode` ready for validation.

## Function resolution

- `StaticHiveFunctionRegistry` — hardcoded mapping of ~100 Hive built-ins. New functions are added here; PRs that add a Hive-side function but skip the registry break translation.
- `HiveFunctionResolver` — pluggable resolver that also handles Dali UDFs at view-load time.
- `DaliOperatorTable` — chained onto `SqlStdOperatorTable` so the validator sees Dali functions.

## SqlNode normalization

- `HiveSqlNodeToCoralSqlNodeConverter extends SqlShuttle` — applies normalization passes (e.g., the array-indexing shift). The full transformer treatment is chapter 07.

## SqlNode → RelNode

- `HiveSqlToRelConverter extends SqlToRelConverter` — overrides `convertFrom`, `convertUnnest`, and table resolution to honor Hive semantics.
- `CoralConvertletTable` — RexNode-level expression conversion rules. Notably overrides `convertCast` to keep abstract casts (preventing Calcite from optimizing casts away in ways that disagree with Hive).

## View expansion + fuzzy union

- `HiveViewExpander` — recursively expands view definitions during conversion.
- `FuzzyUnionSqlRewriter` — applied after the parse tree for views, reconciles UNION branches whose struct columns drifted apart (see chapter 15).

## Files this chapter discusses

- `coral-hive/src/main/antlr/`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveToRelConverter.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/parsetree/ParseTreeBuilder.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlToRelConverter.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlNodeToCoralSqlNodeConverter.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlValidator.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/CoralConvertletTable.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveViewExpander.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/DaliOperatorTable.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/HiveFunctionResolver.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java`
- `coral-common/src/main/java/com/linkedin/coral/common/FuzzyUnionSqlRewriter.java`

## Read next

- Chapter 07 — the `SqlCallTransformer` pattern (the most-touched code surface).
- Chapter 08 — coral-spark (the most-used backend).
