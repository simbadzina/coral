# Exercise 03 — Add a toy Hive function

**Goal:** practice the registry + test + transformer flow that most easy PRs follow.

## Steps

1. Pick a toy function name that doesn't conflict with existing Hive built-ins (e.g., `coral_double`).
2. Register it in `StaticHiveFunctionRegistry` (one-line addition).
3. Add a unit test in `HiveToRelConverterTest` asserting that `SELECT coral_double(x) FROM ...` parses and converts to a sensible RelNode.
4. Add a transformer in `CoralToTrinoSqlCallConverter` that renames it to `coral_double_trino`. Add a test in `HiveToTrinoConverterTest`.
5. Run both tests.

## What you'll learn

The full add-a-function loop. Recent PR #552 is roughly this shape, end to end.

Discard or keep in a personal branch — this is exploratory.
