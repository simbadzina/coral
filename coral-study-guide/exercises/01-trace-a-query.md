# Exercise 01 — Trace a query end-to-end

## Goal

Watch one Hive query move through every IR stage by stepping a live debugger from `ParseTreeBuilder` to `CoralSpark.create`. Chapter 03 narrates the pipeline; this exercise puts your hands on it. At the end you should be able to predict, for any future PR, which stage the change lands in by reading the diff.

## Prerequisites

- Read chapter 03 (the pipeline deep dive). Keep the diagram open in a second window.
- IntelliJ IDEA with the `coral` project imported as a Gradle project.
- The ANTLR-generated parser sources must exist on disk; run `./gradlew :coral-hive:compileTestJava` once before opening any parser file in the IDE. Without this the `com.linkedin.coral.hive.hive2rel.parsetree.parser.HiveParser` class will not resolve and breakpoints in `ParseTreeBuilder` will look broken.

The example query, used throughout the exercise:

```sql
SELECT a, lower(b) AS lower_b FROM default.foo WHERE c > 1
```

The `foo` table is set up by `coral-hive/src/test/java/com/linkedin/coral/hive/hive2rel/TestUtils.java#163` as `foo(a int, b varchar(30), c double)`, so the query is well-typed against the test catalog.

## Steps

1. **Add a scratch test.** Open `coral-hive/src/test/java/com/linkedin/coral/hive/hive2rel/HiveToRelConverterTest.java`. Add a `@Test` method at the bottom of the class:

   ```java
   @Test
   public void testTrace() {
     String sql = "SELECT a, lower(b) AS lower_b FROM default.foo WHERE c > 1";
     RelNode rel = converter.convertSql(sql);
     System.out.println(RelOptUtil.toString(rel));
   }
   ```

   The `converter` field is the shared `HiveToRelConverter` from `ToRelConverterTestUtils`. Right-click the method in IntelliJ and choose "Debug 'testTrace'."

2. **Breakpoint 1 — parse entry.** Set a breakpoint on `CoralParseDriver.parse(String)` in `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/parsetree/parser/CoralParseDriver.java`. When it hits, evaluate the `command` parameter; that is the SQL string entering the pipeline. Step over until the method returns and inspect the returned `ASTNode` — note that every node has a numeric `type` field (a `HiveParser.TOK_*` constant) and a `text` string. There is no polymorphism. Then resume.

3. **Breakpoint 2 — ASTNode to SqlNode.** Set a breakpoint on `ParseTreeBuilder.process(String, Table)` in `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/parsetree/ParseTreeBuilder.java`. When it hits, step into `processAST` and then `visit(node, ctx)`. The dispatcher in `AbstractASTVisitor` switches on `node.getType()`; for our query the root is `TOK_QUERY`, which routes to `visitQuery`, which descends into `visitSelect`. When `process` returns, evaluate the returned `SqlNode` with `node.toString()` in the Evaluate window — you should see:

   ```sql
   SELECT `a`, `lower`(`b`) AS `lower_b`
   FROM `default`.`foo`
   WHERE `c` > 1
   ```

   This is the post-`ParseTreeBuilder` tree: structured `SqlNode`s, unvalidated, untyped, with `lower` already bound to `SqlStdOperatorTable.LOWER` via `HiveFunctionResolver`. Set a sub-breakpoint inside `ParseTreeBuilder.visitFunctionInternal` if you want to watch the function-resolution step described in chapter 06.

4. **Breakpoint 3 — normalization shuttle.** Set a breakpoint on the `HiveSqlNodeToCoralSqlNodeConverter` constructor in `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlNodeToCoralSqlNodeConverter.java`. When it hits, step until `sqlNode.accept(this)` returns inside `HiveToRelConverter.toSqlNode`. For this query the shuttle's only registered transformer (`ShiftArrayIndexTransformer`) does not fire — confirm by evaluating the returned `SqlNode` and seeing it unchanged from step 3.

5. **Breakpoint 4 — validation.** Set a breakpoint on `HiveSqlValidator.validate(SqlNode)` in `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlValidator.java`. When it returns, evaluate `getValidatedNodeType(sqlNode)` — you should get a `RecordType(INTEGER a, VARCHAR(30) lower_b)`. The tree is now fully typed; `lower(b)` infers `VARCHAR(30)` because Hive's `LOWER` is `ARG0_NULLABLE` over a `VARCHAR(30)` column.

6. **Breakpoint 5 — SqlNode to RelNode.** Set a breakpoint on `HiveSqlToRelConverter.convertQuery(SqlNode, boolean, boolean)` in `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/HiveSqlToRelConverter.java`. Step out, then evaluate `RelOptUtil.toString(rel)` on the returned root. You should see:

   ```
   LogicalProject(a=[$0], lower_b=[LOWER($1)])
     LogicalFilter(condition=[>($2, 1)])
       LogicalTableScan(table=[[hive, default, foo]])
   ```

   This is Coral IR. Every backend works from this shape.

7. **Optional — feed it to a backend.** Resume past `convertSql` and add a follow-up in the same test:

   ```java
   CoralSpark spark = CoralSpark.create(rel, null);
   System.out.println(spark.getSparkSql());
   ```

   Set breakpoints on `IRRelToSparkRelTransformer.transform`, `CoralRelToSqlNodeConverter` (entry), `CoralToSparkSqlCallConverter.visit(SqlCall)`, and `SparkSqlRewriter.write`. You will walk stages 6-7 from chapter 03 in the same debugger session.

## Checkpoints

You have done the exercise correctly when, for the example query, your debugger pauses produce these intermediate forms:

- After step 2: `SqlSelect` with select list `[SqlIdentifier(a), SqlCall(LOWER, [SqlIdentifier(b)]) AS lower_b]`, from `SqlIdentifier(default.foo)`, where `SqlCall(>, [SqlIdentifier(c), SqlLiteral(1)])`.
- After step 3: structurally identical — no normalization fires.
- After step 4: same shape, every `SqlNode` annotated with a `RelDataType` retrievable via the validator.
- After step 5: the three-line `LogicalProject / LogicalFilter / LogicalTableScan` plan above.

If the RelNode plan you see does not match, the most common cause is forgetting `default.` — `convertSql` resolves bare table names against `hive.default`, but `foo` and `default.foo` produce the same plan, while `db.tbl` (the chapter 03 example) does not exist in the test catalog and will throw.

## What you learn

You internalize the exact data structure at each stage instead of reasoning about it abstractly. The next time a PR claims to "fix function resolution," you will know that the fix belongs in `ParseTreeBuilder.visitFunctionInternal` or `HiveFunctionResolver.tryResolve` — and the next time someone says "the validator drops the cast," you will know to look at `HiveSqlValidator.expand` or `CoralConvertletTable.convertCast`. The mental model is no longer "Coral converts SQL"; it is "stage 3 normalizes, stage 4 types, stage 5 builds RelNode."

For PR review specifically, this exercise gives you the muscle memory to ask the right "where does this land?" question. A change in `coral-hive/.../parsetree/` is a stage-2 change, must come with a `HiveToRelConverterTest`. A change in `coral-trino/.../transformers/` is a stage-6 change, must come with a `HiveToTrinoConverterTest`. Misplacement is the single most common architectural review finding.

## Optional extensions

- **Trace a view.** Re-run with `converter.convertView("default", "<view-name>")` on a test view registered in `TestUtils`. Set an extra breakpoint on `HiveViewExpander.expandView` and watch the recursion when the view's body itself references another view.
- **Break a function.** Replace `lower(b)` with `not_a_function(b)`. Step into `HiveFunctionResolver.tryResolve` and observe the `UnknownSqlFunctionException` path. This is the failure mode users see as "function not found."
