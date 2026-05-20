# Exercise 02 — Write a no-op transformer and observe it fire

**Goal:** internalize the `SqlCallTransformer` pattern by writing one.

## Steps

1. Create a `LoggingSqlCallTransformer` that extends `SqlCallTransformer`. `condition()` returns true for every `SqlCall`. `transform()` logs the call and returns it unchanged.
2. Register it in `CoralToTrinoSqlCallConverter` at the head of the `SqlCallTransformers.of(...)` list.
3. Run a test in `HiveToTrinoConverterTest`. Observe the log.
4. Move your transformer to the tail of the list. Compare logs.

## What you'll learn

Order matters. Transformers earlier in the chain see less-modified trees; later transformers see what previous transformers produced. This is the most common source of subtle review-stage bugs (chapter 16).

Discard the changes — this is exploratory, not a PR.
