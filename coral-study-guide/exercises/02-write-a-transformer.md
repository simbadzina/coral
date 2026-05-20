# Exercise 02 — Write a no-op transformer and observe ordering

## Goal

Build the simplest possible `SqlCallTransformer` — one that logs every call and returns it unchanged — and wire it into `CoralToTrinoSqlCallConverter` at different positions in the chain. Observe what each position can see. [Chapter 07](../07-transformers-pattern.md) explains why ordering matters; this exercise makes the consequences impossible to miss. After running it once you will reflexively check chain ordering on every transformer PR you review.

## Prerequisites

- Read [chapter 07](../07-transformers-pattern.md) (the `SqlCallTransformer` pattern) and skim the `CoralToTrinoSqlCallConverter` constructor in [`coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/CoralToTrinoSqlCallConverter.java`](../../coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/CoralToTrinoSqlCallConverter.java). You should be able to name the three concrete transformer subclasses in `coral-common` before starting.
- IntelliJ on the repo, with `coral-trino` test sources importable.
- A test class to run against. We will use `HiveToTrinoConverterTest` in [`coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java`](../../coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java) — pick any short test, for example one that exercises `pmod` or `nvl` so the chain has at least one real rewrite firing.

## Steps

1. **Write the logging transformer.** Create a new file `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/transformers/LoggingSqlCallTransformer.java`:

   ```java
   package com.linkedin.coral.trino.rel2trino.transformers;

   import org.apache.calcite.sql.SqlCall;
   import com.linkedin.coral.common.transformers.SqlCallTransformer;

   public class LoggingSqlCallTransformer extends SqlCallTransformer {
     private final String tag;

     public LoggingSqlCallTransformer(String tag) {
       this.tag = tag;
     }

     @Override
     protected boolean condition(SqlCall sqlCall) {
       return true;
     }

     @Override
     protected SqlCall transform(SqlCall sqlCall) {
       System.err.println("[" + tag + "] " + sqlCall.getOperator().getName()
           + "(" + sqlCall.getOperandList().size() + " operands): " + sqlCall);
       return sqlCall;
     }
   }
   ```

   `condition` returns `true` so every `SqlCall` fires the transformer. `transform` logs and returns the input unchanged — pure observation. The `tag` field will distinguish head and tail instances when both are in the chain.

2. **Wire it at the head of the chain.** Open `CoralToTrinoSqlCallConverter.java`. In the constructor, `SqlCallTransformers.of(...)` opens with `new SqlSelectAliasAppenderTransformer()`. Insert your transformer as the very first argument:

   ```java
   this.sqlCallTransformers = SqlCallTransformers.of(
       new LoggingSqlCallTransformer("HEAD"),
       new SqlSelectAliasAppenderTransformer(),
       new CoralRegistryOperatorRenameSqlCallTransformer("nvl", 2, "coalesce"),
       ...
   ```

3. **Pick a target test.** In `HiveToTrinoConverterTest`, find a method that produces a small, distinct output. A good candidate is any test whose SQL contains `nvl(x, y)` or `pmod(a, b)` — both have visible rewrites in the chain. Run it: right-click `@Test` → "Debug" (or "Run"). The console will fill with `[HEAD] ...` lines, one per `SqlCall` the shuttle visits.

4. **Read the head output.** You should see lines like `[HEAD] nvl(2 operands): NVL(`x`, `y`)`. That is the call shape *before* any transformer in the chain has rewritten anything. The operator name is still `nvl`, the call structure matches the parsed input.

5. **Move the transformer to the tail.** Cut your line and paste it as the *last* argument inside `SqlCallTransformers.of(...)` (after `SubstrIndexTransformer`). Keep the tag string `"HEAD"` for the moment so the diff in output is unambiguous.

6. **Re-run the same test.** The new log lines show calls *after* every other transformer has run. For the same `nvl(x, y)` input you should now see `[HEAD] coalesce(2 operands): COALESCE(`x`, `y`)` — the `CoralRegistryOperatorRenameSqlCallTransformer("nvl", 2, "coalesce")` has already fired. The tail position sees the chain's output, not the chain's input.

7. **Put a logger in both positions.** Change the constructor lines to add two instances:

   ```java
   new LoggingSqlCallTransformer("HEAD"),
   ... // existing chain
   new LoggingSqlCallTransformer("TAIL"));
   ```

   Re-run. The log will interleave `HEAD` and `TAIL` lines as the shuttle recurses, but for each `SqlCall` the `HEAD` line shows the pre-chain operator and the `TAIL` line shows the post-chain operator. For Calcite/Hive builtins that the chain rewrites, the two lines will differ. For builtins the chain does not rewrite (`>`, `AND`, `*`), they will match.

8. **Add a real transformation between them.** Replace the head logger with a transformer that actually mutates: rename `>` to `GREATER_THAN` (a fake name nothing downstream knows about). Use `OperatorRenameSqlCallTransformer`:

   ```java
   new OperatorRenameSqlCallTransformer(SqlStdOperatorTable.GREATER_THAN, 2, "GREATER_THAN"),
   ```

   Re-run with the tail logger still present. The tail will now print `[TAIL] GREATER_THAN(...)` for every `>` in the query. This is the exercise's payload: the tail-position transformer sees the output of every transformer above it, not the original parse output.

## Checkpoints

- After step 4 (head only): expect lines like `[HEAD] nvl(2 operands): NVL(...)`. Operator names match what `ParseTreeBuilder` emitted.
- After step 6 (tail only, with `nvl` in the query): expect `[HEAD] coalesce(2 operands): COALESCE(...)`. The rename has already fired.
- After step 7 (both): each visited `SqlCall` produces a `HEAD` and a `TAIL` line. For `nvl`, the head line says `nvl` and the tail line says `coalesce`. For `>`, both say `>` (the unmodified chain does not rewrite it).
- After step 8: the tail line for `>` now reads `GREATER_THAN`.

A common gotcha: if you put the head logger *before* `SqlSelectAliasAppenderTransformer`, you might see slightly more calls than the tail position does, because the alias appender adds `AS` calls that subsequent transformers also visit. That is the shuttle recursing into the synthetic node, not a bug. Note it; the difference is small for most queries.

## What you learn

The `SqlCallTransformers` chain is not a set; it is an ordered sequence, and each transformer sees the output of its predecessors. Three concrete consequences:

1. **A late transformer cannot match on the original operator name** if an earlier transformer renamed it. This is why `HiveUDFTransformer` lives *after* the named renames in `CoralToTrinoSqlCallConverter` — it must not preempt a built-in's rename, and it must catch only the calls no earlier rename touched.
2. **A type-aware transformer must see the canonical operator.** `ReturnTypeAdjustmentTransformer` runs after the operator renames so it can adjust the type of the *renamed* operator. Putting it first would adjust the original operator and the rename would then erase the cast.
3. **`DataTypeDerivedSqlCallConverter` is structurally a second shuttle for the same reason.** The transformers in it need `TypeDerivationUtil`, which is only meaningful once the prior shuttle has stabilized the operator names. Two shuttles is not a quirk — it is the cleanest way to enforce phase ordering when the second phase depends on the type system seeing the post-rename names.

For PR review, this maps directly to [chapter 16](../16-pr-review-companion.md): any new transformer PR must specify where in the chain it goes and why. "I added it at the end" is not an answer; the question is whether the new transformer needs to see the renamed or the original operator, and whether downstream transformers in the chain rely on its output.

## Optional extensions

- **Mirror to the Spark side.** Drop a `LoggingSqlCallTransformer` into [`coral-spark/src/main/java/com/linkedin/coral/spark/CoralToSparkSqlCallConverter.java`](../../coral-spark/src/main/java/com/linkedin/coral/spark/CoralToSparkSqlCallConverter.java) and run a `HiveToSparkSqlConverterTest`. The Spark chain is shorter (the file's own javadoc spells out the head-priority rule for `TransportUDFTransformer` vs. `HiveUDFTransformer`) — observing one fire ahead of the other is a clean demonstration of the rule.
- **Break the order on purpose.** Reorder two transformers in `CoralToTrinoSqlCallConverter` that you suspect depend on each other (try `HiveUDFTransformer` before the renames). Run the full test suite. Categorize the failures by which transformer's invariant got violated. This is the most efficient way to learn which orderings are load-bearing.

When you are done, revert every change in this exercise. The transformer file and the wiring edits are exploratory — none of it should land in a PR.
