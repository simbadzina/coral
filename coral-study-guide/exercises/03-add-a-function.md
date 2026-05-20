# Exercise 03 — Add a toy Hive function

## Goal

Walk the full lifecycle of adding a new Hive built-in: register it in `StaticHiveFunctionRegistry`, prove it resolves in `HiveToRelConverterTest`, then add a Trino-side rename transformer and a `HiveToTrinoConverterTest`. This is the exact pattern most "add function X" PRs follow — PR #552 was approximately this shape. After running this exercise you have done the work of a small real PR in a sandbox where you cannot break anyone.

## Prerequisites

- Read chapter 06 (`coral-hive`) sections "Function resolution" and "StaticHiveFunctionRegistry."
- Read chapter 07 (the transformer pattern) at least through `OperatorRenameSqlCallTransformer`.
- You have completed exercise 02, so you understand where transformers attach in `CoralToTrinoSqlCallConverter`.

## Steps

1. **Pick a non-colliding name.** Use `coral_double` — the namespace prefix `coral_` is not a real Hive function root, so the static registry will not have anything under it. Verify with a quick `grep -n "coral_double" coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java`; it should print nothing.

2. **Register it in `StaticHiveFunctionRegistry`.** Open `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java`. The file's `static {}` block holds every registration. Find a math-function block (the area around `pmod`, `pow`, `sqrt`) and add one line that follows the surrounding pattern:

   ```java
   createAddUserDefinedFunction("coral_double", DOUBLE_NULLABLE, NUMERIC);
   ```

   The `createAddUserDefinedFunction(name, returnType, operandTypeChecker)` helper builds a `SqlUserDefinedFunction`. `DOUBLE_NULLABLE` is the return-type inference used by the surrounding math functions (`exp`, `ln`, `sqrt`); `NUMERIC` is the operand-type checker (single numeric arg). These map directly to the static imports already present at the top of the file (`org.apache.calcite.sql.type.ReturnTypes.*`, `org.apache.calcite.sql.type.OperandTypes.*`).

   You do not need a `try/catch`, you do not need an annotation, you do not need to touch any other file in `coral-hive`. The registry is one line per function.

3. **Add a Hive-side test.** Open `coral-hive/src/test/java/com/linkedin/coral/hive/hive2rel/HiveToRelConverterTest.java`. Add at the bottom:

   ```java
   @Test
   public void testCoralDouble() {
     String sql = "SELECT coral_double(c) FROM foo";
     RelNode rel = converter.convertSql(sql);
     String relString = RelOptUtil.toString(rel);
     String expected =
         "LogicalProject(EXPR$0=[coral_double($2)])\n"
             + "  LogicalTableScan(table=[[hive, default, foo]])\n";
     assertEquals(relString, expected);
   }
   ```

   `foo` is the test table set up in `TestUtils.java#163` as `foo(a int, b varchar(30), c double)`; `c` is a `double`, which matches the `NUMERIC` operand checker. Run the test:

   ```bash
   ./gradlew :coral-hive:test --tests \
     "com.linkedin.coral.hive.hive2rel.HiveToRelConverterTest.testCoralDouble"
   ```

   It should pass. If it fails with `UnknownSqlFunctionException`, your registration is wrong (most likely a typo); if it fails with `Cannot apply 'coral_double' to arguments of type ...`, your operand checker does not accept the column's type.

4. **Add a Trino-side rename transformer.** Open `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/CoralToTrinoSqlCallConverter.java`. The constructor's `SqlCallTransformers.of(...)` already has examples of the rename pattern — search for `nvl` and you will see `new CoralRegistryOperatorRenameSqlCallTransformer("nvl", 2, "coalesce")`. Add a parallel line in the math-function region of the chain:

   ```java
   new CoralRegistryOperatorRenameSqlCallTransformer("coral_double", 1, "coral_double_trino"),
   ```

   `CoralRegistryOperatorRenameSqlCallTransformer` takes `(sourceName, arity, targetName)`, looks up the source operator in `StaticHiveFunctionRegistry`, and emits a new `SqlUserDefinedFunction` with the target name and the source's return-type inference. The source operator's `DOUBLE_NULLABLE` carries through, so downstream type derivation will keep `coral_double_trino` returning `DOUBLE`.

5. **Add a Trino-side test.** Open `coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java`. Find the simplest existing test for a renamed function (`nvl` is a good template) and pattern after it:

   ```java
   @Test
   public void testCoralDouble() {
     String hiveSql = "SELECT coral_double(c) FROM foo";
     String expected = "SELECT \"coral_double_trino\"(\"foo\".\"c\")\n"
         + "FROM \"hive\".\"default\".\"foo\"";
     RelToTrinoConverter converter = new RelToTrinoConverter(...);
     // (use the established helper in the test class to convert hiveSql and assert equals)
   }
   ```

   The exact helper varies by test method — copy the shape from an existing `nvl` or `pmod` test in the same file. Run it:

   ```bash
   ./gradlew :coral-trino:test --tests \
     "com.linkedin.coral.trino.rel2trino.HiveToTrinoConverterTest.testCoralDouble"
   ```

6. **Run both modules' full test suites.** This catches the most common second-order bug — that your registration accidentally collided with an existing Hive function name. Run:

   ```bash
   ./gradlew :coral-hive:test :coral-trino:test
   ```

   Both should pass with no regressions. If a previously-passing test now fails, your function name is colliding with something — back out and pick a different prefix.

## Checkpoints

- After step 2: the `static {}` block compiles. The static imports at the top of `StaticHiveFunctionRegistry` already include `ReturnTypes.DOUBLE_NULLABLE` and `OperandTypes.NUMERIC`; if your IDE wants to add imports, your edit is in the wrong place in the file.
- After step 3: the Hive test passes. The `LogicalProject` operator name is the lowercase `coral_double`, which confirms `StaticHiveFunctionRegistry.lookup` is being hit during `ParseTreeBuilder.visitFunctionInternal`.
- After step 4: `:coral-trino:compileJava` succeeds.
- After step 5: the Trino test passes. The unparsed SQL contains `coral_double_trino` instead of `coral_double`, which confirms the transformer fired during stage 6 of the pipeline (chapter 03).
- After step 6: no other tests broke.

## What you learn

You have done the work of a real Hive-function-addition PR end to end. Specifically, you have practiced:

- **Registry hygiene** — picking the correct `SqlReturnTypeInference` and `SqlOperandTypeChecker`. Chapter 06 calls out that this choice is not cosmetic; a wrong return type silently corrupts downstream schema derivation, which a reviewer will not notice until a query crashes in production. Doing it once in a sandbox builds the instinct to check it on PRs.
- **The frontend/backend coupling.** Registration in `coral-hive` makes the function *resolvable*. The transformer in `coral-trino` makes it *translatable* into Trino. Both are required for a usable Hive→Trino translation, and the two PRs are conventionally bundled. A PR that adds the registration but forgets the transformer leaves users with a function that parses but cannot be executed on Trino — a bug that escapes module-level testing because each module's own tests still pass.
- **The test pattern.** A Hive test asserts the RelNode plan shape; a Trino test asserts the unparsed SQL. Both styles are load-bearing — the RelNode test exercises stages 1-5, the SQL test exercises stages 6-7. PR #552's review chain mirrors exactly this two-test discipline.

## Optional extensions

- **Add a Spark-side mirror.** Open `coral-spark/src/main/java/com/linkedin/coral/spark/CoralToSparkSqlCallConverter.java`. The Spark chain does not have `CoralRegistryOperatorRenameSqlCallTransformer` available, so write an inline `OperatorRenameSqlCallTransformer` using the operator you registered. Add a `HiveToSparkSqlConverterTest`. This is the cross-dialect parity reviewers ask about; doing it by hand once teaches you to ask the same question on every transformer PR.
- **Make the function type-dependent.** Change the registration to take two operands of possibly-different numeric types (`NUMERIC_NUMERIC`), then in the Trino chain replace your rename with a `JsonTransformSqlCallTransformer` that rewrites `coral_double(a, b)` into `(a + b) * 2`. You will see why chapter 07 lists `JsonTransformSqlCallTransformer` as the escape hatch for non-trivial structural rewrites.

When you are done, revert all four edits. This exercise is sandbox-only.
