# 16 — PR review companion

The chapter you live in as a reviewer. Per-module checklists, red flags, and reusable comment templates. After this chapter you can review a PR in any Coral module systematically — without re-deriving the per-module conventions every time, and without re-reading chapter 04-14 from scratch. This is the reference. Keep it open while you review.

Coral PRs are usually small. The merged history is dominated by single-module changes that add a function, fix a transformer, or extend a registry. Big architectural moves (the type system in #558/#563, the `CoralCatalog` migration in #604, the `coral-benchmark` module in #599) happen, but they are the exception. Most reviews are a single transformer file plus a test diff. The cheat sheets below are sized accordingly.

## Universal preflight

Before reading any code in the diff, run a five-minute meta-check. These are headline review comments — if any fail, that comment belongs in your first pass.

- **Is the build green?** The PR's CI check. A red build means the PR is not ready. Don't waste cycles on logic review until it goes green.
- **Was `./gradlew spotlessApply` run?** Spotless failures show as the first CI failure. The fix is one command. If the contributor missed it, the first comment is "run `./gradlew spotlessApply` and force-push." `CONTRIBUTING.md` calls this out explicitly under tip #2.
- **Are there tests?** `CONTRIBUTING.md` tip #1: "Make sure all new features are tested." Tip #3: "Bug fixes must include a test case demonstrating the error that it fixes." A PR fixing parser behavior with no `HiveToRelConverterTest` addition is structurally incomplete.
- **Does the commit message match the project style?** Run `git log --oneline -30` against the branch and compare. The convention is `[Coral-Module] Imperative subject (#PR)` for module-scoped changes (`[Coral-Hive] Fix Coral parse failure for non-reserved keywords used as table aliases`), or a plain imperative for cross-cutting changes (`Add coral-benchmark module for cross-dialect translation testing`). Body explains why; the diff explains what.
- **Is there a linked issue for non-trivial change?** `CONTRIBUTING.md` tip #4: "Open an issue first and seek advice for your change before submitting a pull request." Three-line bug fixes don't need an issue. New features, public API changes, and module restructures do. If the description has no issue link and the PR is more than a small fix, ask for one.
- **New dependencies in any `build.gradle`?** `git diff master -- '**/build.gradle' '**/*.gradle'` shows them. New deps need justification — Coral's classpath is already tight (Calcite's ANTLR v3 vs. Trino's ANTLR v4 is the canonical example, see chapter 09's shading discussion). A line added to `dependencies { }` without context is a comment-worthy event.
- **Is the PR sized reasonably?** Most merged Coral PRs are under 300 lines of non-test diff. A 2000-line PR is not automatically bad, but it should justify the size. Restructuring + behavior change in one PR is a flag.

If preflight passes, you can start reading the diff.

## Reading the test diff first

Always read the test diff before the production diff. The integration test classes — `HiveToRelConverterTest` for the Hive frontend, `HiveToTrinoConverterTest` for Trino emission, `CoralSparkTest` for Spark emission, `MergeCoralSchemaWithAvroTests` for the Avro merge engine, `CoralSparkViewCatalogTest` for end-to-end Spark catalog integration — are *specifications*. The corpus method names read like requirement sentences: `shouldUseFieldNamesFromCoral`, `testViewQueryWithUDF`, `shouldRetainDefaultValuesFromAvro`. If a PR's test diff is small, missing, or only adds a single happy-path assertion, that is the headline comment.

Specifically check:

- **New behavior has a test in the integration class**, not just a unit test. Chapter 09's reviewer cheat sheet flags this explicitly for Trino transformers: a transformer-only unit test misses ordering interactions with the rest of the chain.
- **Bug fixes have a regression test that fails without the fix.** Make the contributor commit the test before the fix and run it red, or at least describe how to verify the test would have failed.
- **Test corpus changes match the production change shape.** If a PR adds a new operator to `StaticHiveFunctionRegistry`, the test should call that operator. If it adds a transformer, the test should exercise the rewrite end-to-end.

A test class read this way also tells you what *not* to demand: if the existing tests don't cover edge cases X and Y, this PR is not the place to require them — file a follow-up.

## Per-module checklists

Each subsection below is sized for the review surface that module presents. Read the one that matches the PR's module, then drop down to the cross-cutting section.

### coral-common

The foundation. Most PRs here are architectural and ripple outward.

**Common change shapes.** Touches to `ToRelConverter` (constructor changes for the `CoralCatalog` migration, view-expansion logic), additions to `HiveTypeSystem` precision rules, new generic `SqlCallTransformer` subclasses in `transformers/`, additions to the `catalog/` package (new `CoralTable` subclass, new `CoralCatalog` method), additions to the `types/` package (new `CoralDataType` variant), Calcite extension tweaks (`HiveRelBuilder`, `HiveUncollect`, `CoralJavaTypeFactoryImpl`).

**What to verify.** Both `ToRelConverter` constructors (deprecated `HiveMetastoreClient` plus modern `CoralCatalog`) still work — chapter 05 covers the migration discipline. The schema-layer dispatch in `CoralDatabaseSchema.getTable()` still branches `HiveTable` vs. `IcebergTable` cleanly. Precision changes in `HiveTypeSystem` are reflected by tests on at least two backends (Spark and Trino) because the type factory feeds every emitter — see d1d5b1ea (#501, "Add missing numeric precisions in Hive type system") as the template for what good coverage looks like. New entries in `functions/` must be paired with whatever validator hook needs them (a new `SqlOperator` without a `convertlet` registration in `CoralConvertletTable` will silently fall through to `StandardConvertletTable`).

**Red flags.** A `coral-common` change that touches only one downstream module's tests — the foundation should not become Spark-specific. New use of `org.apache.hadoop.hive.metastore.api.Table` in a class outside `catalog/` or `HiveMscAdapter` is moving in the wrong direction. New `HiveMetastoreClient`-typed public surface (the interface is `@Deprecated`). Removal of the legacy schema path (`HiveSchema`, `HiveDbSchema`) without a coordinated multi-PR plan — it is still on the wire for backward compatibility.

### coral-hive

The reference frontend. Most PRs are either function-registry additions or parser tweaks.

**Common change shapes.** Adding a Hive built-in to `StaticHiveFunctionRegistry` (`addFunctionEntry`, `createAddUserDefinedFunction`, `createAddUserDefinedTableFunction`). ANTLR grammar changes in `.g` files under `src/main/antlr/`. `ParseTreeBuilder` visitor additions for new Hive constructs. `HiveViewExpander` recursion changes. `CoralConvertletTable` cast-folding tweaks. Examples from recent history: 96e24e01 (#519, "Add function in static hive registry"), 61727335 (#533, "Add EXTRACT_COLLECTION UDF to registry"), b5077611 (#539, "Always allow unquoted keywords as column names"), f958b09c (#602, parser fix for non-reserved keywords).

**What to verify.** A new function added to `StaticHiveFunctionRegistry` has a corresponding test in `HiveToRelConverterTest` — the registry has no autoloading, no annotation scan; the test confirms `ParseTreeBuilder` actually resolves the call. The chosen `SqlReturnTypeInference` matches Hive semantics (`FunctionReturnTypes.STRING` vs. `ReturnTypes.VARCHAR_2000` is not stylistic — see chapter 06). UDTFs additionally register their return field names in `UDTF_RETURN_FIELD_NAME_MAP`, or `LATERAL VIEW` parsing throws. Grammar `.g` file changes trigger ANTLR regeneration in Gradle (CI will fail otherwise, but call it out so the contributor doesn't push a broken build). `HiveViewExpander` recursion changes must be stress-tested with at least three nested levels — chapter 06's reviewer cheat sheet calls this out and it matters because LinkedIn's Dali views frequently nest three or more deep.

**Red flags.** A grammar change with no accompanying generated-source refresh. A `StaticHiveFunctionRegistry` add without a `HiveToRelConverterTest` entry. A `ParseTreeBuilder` visitor change that uses regex on the raw SQL text instead of structural matching on the `ASTNode` tree. Touches to `CoralConvertletTable.convertCast` without a justification — that override is load-bearing across every backend (see chapter 06).

### coral-spark

Hive → Spark SQL backend. Smaller transformer chain than Trino.

**Common change shapes.** New entry in the `TransportUDFTransformer` set in `CoralToSparkSqlCallConverter`'s constructor (one per LinkedIn UDF mapping with Scala 2.11 and Scala 2.12 Ivy URLs). Tweaks to `SparkSqlRewriter` (final AST cleanups the dialect can't express, see chapter 08 — recent example f9b78246 #593 added the `VARBINARY → BINARY` rewrite). Changes to `SparkSqlDialect` (identifier quoting, `unparseCall` overrides, reserved-keyword list). Tweaks to `IRRelToSparkRelTransformer`'s `SparkRexConverter` (operating at the `RelNode`/`RexNode` layer; the 1-based to 0-based array index conversion is the canonical example). New `SparkUDFInfo` fields if a new registration mechanism is needed.

**What to verify.** A new `TransportUDFTransformer` entry has both Scala 2.11 and Scala 2.12 Ivy URLs (or has `null` for the unsupported one with `UnsupportedUDFException` semantics clearly intentional). The transformer sits *before* `HiveUDFTransformer` in the chain — chapter 08 explains why this priority matters. The schema-aware `create(RelNode, Schema, HiveMetastoreClient)` overload still produces aliased output via `AddExplicitAlias` (regression here breaks `coral-spark-catalog`'s schema lineage). `IRRelToSparkRelTransformer` changes preserve row types across the standard logical operators. `SparkSqlRewriter` additions justify why the rewrite can't live in `SparkSqlDialect.unparseCall` (chapter 08's reviewer cheat sheet: the rewriter is for cases the dialect API can't reach).

**Red flags.** A new Spark-side change that also applies to Trino but isn't mirrored — dialect parity is a recurring miss. A change to `SparkSqlDialect`'s reserved-keyword list without a test that verifies the new keyword actually triggers quoting. A `SparkSqlRewriter` rewrite that could have been a `SqlCallTransformer` (rewriters are for surface-syntax fixes; transformers are for semantic ones).

### coral-trino

Largest transformer collection in the codebase. Most PRs here add or tweak a transformer.

**Common change shapes.** New `SqlCallTransformer` subclass in `coral-trino/.../rel2trino/transformers/`. New entry in the `SqlCallTransformers.of(...)` chain in `CoralToTrinoSqlCallConverter` or `DataTypeDerivedSqlCallConverter`. Tweaks to `RelToTrinoConverter`'s Calcite overrides (`visit(Project)`, `visit(Uncollect)`, `visit(TableScan)`, etc.). Changes to `TrinoSqlDialect.unparseCall` for new operator forms. UDF mapping additions in `HiveUDFTransformer`. Recent examples: 7908fdbe (#525, "Modify wrong Trino UDF name"), 9e79488a (#524, "Map some UDFs to Trino UDF"), 9f8dfce7 (#552, "Register ExtractCollectionUdf as Trino UDF"), 0d5dd3f3 (#499, "Fix substring start index issue").

**What to verify.** Which pass does the transformer belong in? Type-aware (`DataTypeDerivedSqlCallConverter`) only if it calls `deriveRelDatatype()`; otherwise structural (`CoralToTrinoSqlCallConverter`). Order in the chain — chapter 07 has the punchline, but the per-PR check is: read the immediate neighbors and confirm the new transformer's `condition()` doesn't intersect either of them. If a custom `SqlOperator` is introduced (under `functions/`), it must override `unparse(SqlWriter, SqlCall, int, int)` or the final SQL is garbled. Integration coverage exists in `HiveToTrinoConverterTest` (transformer-only unit tests miss chain interactions).

**Red flags.** Adding a transformer to the structural pass that needs `deriveRelDatatype()` (will throw at runtime — `TypeDerivationUtil` is null). A new `RelToTrinoConverter.visit()` override without confirming Calcite's default would actually fail — overriding for taste leaves the next reviewer guessing. A `JsonTransformSqlCallTransformer` JSON spec that doesn't round-trip through the `HiveToTrinoConverterTest` corpus. A change that visibly applies to Spark as well — chapter 07's fifth anti-pattern, "missing the mirror."

### coral-schema

Avro schema derivation. The most metadata-sensitive module.

**Common change shapes.** Nullability rule tweaks in `RelToAvroSchemaConverter` (operator visit overrides, `SchemaUtilities.isFieldNullable`). New operator handling (`LogicalSort`, `LogicalIntersect`, etc., are still TODOs — chapter 10). Merge-engine changes in `MergeCoralSchemaWithAvro` (the new path) or `MergeHiveSchemaWithAvro` (the legacy path). New `coral-schema` overloads for `CoralCatalog` (54ac5592 #604 is the recent template). Type-level fixes in `RelDataTypeToAvroType`. Recent examples: 988ca6d0 (#597, DECIMAL Jackson1Utils fix), 73c798a3 (#561, namespace disambiguation), 8786af3d (#510, repeated field reference fix), 5b8f8177 (#600, new `MergeCoralSchemaWithAvro` merge engine).

**What to verify.** Base-table nullability is preserved for pass-through columns — that is rule #1 in chapter 10's nullability section. Iceberg-related changes prefer `MergeCoralSchemaWithAvro`; touching `MergeHiveSchemaWithAvro` is acceptable for legacy-path fixes but should be flagged as not the long-term destination. Field matching, name resolution, and nullability changes have a regression test in `MergeCoralSchemaWithAvroTests` — that file reads like a contract sheet. New `ViewToAvroSchemaConverter` overloads accept `CoralCatalog`, not `HiveMetastoreClient` (the migration is additive — both still work, but new public surface lands on the modern side).

**Red flags.** A change in `MergeHiveSchemaWithAvro` without a matching consideration for `MergeCoralSchemaWithAvro` (legacy fix that should be paired). Changes to `SchemaUtilities.isFieldNullable` without a `RelToAvroSchemaConverter` test for each new nullability branch — that helper is consulted for every `RexCall`. Public overloads that add a `HiveMetastoreClient` parameter when a `CoralCatalog` overload exists.

### coral-spark-catalog

Spark 3.5 `CatalogExtension` integration. Newest module; SPI surface is the contract.

**Common change shapes.** Changes to `CoralSparkViewCatalog.loadView` (the only method that runs Coral). UDF registration tweaks in `registerUDFs` / `isSparkUdf`. Test additions in `CoralSparkViewCatalogTest` or `TrinoToSparkCatalogTest`. Bumps to the embedded metastore version (e.g., cacdbfba #585, Hive 1.2.2 → 2.3.9). README updates. Recent examples: dc90da11 (#584, initial module landing), e7108bfe (#594, README), 0bad7eda (#595, README link fix).

**What to verify.** View name casing — Spark and Hive disagree on case-folding, and `loadView` builds the `Identifier`-to-`db.tbl` path through both worlds. Test assertions on `spark.table("default.<view>").schema()` should match the exact casing the test expects (chapter 11 calls this out). UDF registration dispatches correctly on `udfType` (`TRANSPORTABLE_UDF` → `register(String)` reflection; `HIVE_CUSTOM_UDF` with Spark-native class → `functionRegistry().registerFunction`; `HIVE_CUSTOM_UDF` falling back → `sessionState().catalog().registerFunction`). The `synchronized (sessionCatalog)` block around `loadFunctionResources` stays in place — `SessionCatalog` is not thread-safe. Methods that intentionally throw `UnsupportedOperationException` (`createView`, `alterView`, `dropView`, `renameView`) keep throwing — the SPI deliberately doesn't proxy view writes.

**Red flags.** Removing the `synchronized` block. Adding a method that proxies a view write. A change to `registerUDFs` that doesn't update both `isSparkUdf` and the dispatch table. Test additions that don't actually exercise `loadView` (a test that creates a view via `spark.sql` but reads back without `spark.table(...)` doesn't hit the catalog).

### coral-benchmark

Cross-dialect testing framework. Brand new, SPI is the contract.

**Common change shapes.** New corpus query `.sql` file under `coral-benchmark/src/test/resources/queries/<source-dialect>/`. Additions to the SPI (`DialectPlugin`, `EnginePlugin`, `VerificationLevel`). New plugin implementations (`DialectPluginProvider` for a specific dialect). `ComparisonConfig` knobs. `ResultSetComparator` implementation work (chapter 12 notes the comparator currently throws `UnsupportedOperationException`; finishing that is the next high-leverage PR).

**What to verify.** New SPI methods are backward-compatible — `DialectPlugin`, `DialectPluginProvider`, `EnginePlugin` are shipped as interfaces consumers will implement; adding a non-default method breaks every implementer. New `Dialect` enum values are coordinated with at least one plugin implementation in the same PR (an enum value with no provider produces `IllegalStateException` at `resolveProvider`). Test resource queries actually parse with `HiveToRelConverter` or `TrinoToRelConverter` on the source side and emit valid SQL on the target side — a corpus add is the cheapest review category in the project, but only when the query actually round-trips.

**Red flags.** SPI changes without a default implementation (forces every consumer to recompile). A new `Dialect` value with no provider. A corpus add that references tables not registered in any `InMemoryCatalog` (will throw at translation, not catch a bug). Bypassing the `VerificationLevel` ordering (treating ordinals as not load-bearing — chapter 12 calls them load-bearing).

### coral-data-generation

Symbolic constraint solver. Easy to introduce unsound behavior.

**Common change shapes.** New `DomainTransformer` for a previously-unhandled SQL operator (`UPPER`, `CONCAT`, `MOD`, `DIVIDE`, date functions). New `Domain` subtype. Changes to `RegexDomain` (the brics-automaton-backed string domain) or `IntegerDomain`. Relational preprocessing tweaks (`ProjectPullUpController`, `CanonicalPredicateExtractor`, `DnfRewriter`). PR #564 (f56226f0) is the only entry in the module's history; it landed the entire framework, so every PR here is an extension.

**What to verify.** Soundness — every value the new transformer's domain produces must satisfy the predicate the transformer inverted. Falsely-narrow domains (false positives) silently corrupt the test corpus and produce confusing benchmark failures. Falsely-wide domains are tolerable but reduce the framework's value. Read the `DomainTransformer`'s `refineInputDomain` and ask: if the predicate is `op(x, lit) = output`, is the returned domain `D` such that every `v ∈ D` produces an output in `output`? `isVariableOperandPositionValid` correctly rejects multi-variable cases (`x + y = z` is not single-pass invertible — chapter 13 covers this). Cross-domain transformers (`CastRegexTransformer`) catch `NonConvertibleDomainException` and fall back rather than throwing.

**Red flags.** A new transformer added to `DomainInferenceProgram.withDefaultTransformers()` ordering without verifying its `canHandle` doesn't overlap an earlier transformer's. A transformer that silently passes through a domain when it can't precisely invert (`LowerRegexTransformer`'s non-literal fallback is the canonical safe pass-through; new transformers should match that discipline). A `RegexDomain` change that breaks `sample(int)`'s cycle-prevention via `StateWithPrefix`.

### coral-pig, coral-dbt, coral-visualization, coral-service, coral-spark-plan

These five modules have a small review surface (chapter 14 covers them combined). Most PRs are localized: a Pig backend tweak in `RelToPigLatinConverter`, a dbt model transform fix, a `RelNodeIncrementalTransformer` delta-union change in `coral-incremental`, a Spring Boot endpoint addition in `coral-service`, or a Spark plan JSON parser tweak in `coral-spark-plan`. Verify the change doesn't leak module-specific assumptions back into `coral-common` — these modules are downstream consumers and should not require core changes for backend-local needs. The `coral-incremental` correctness check is specifically about the delta-union rewrite: the transformation replaces `T` with `T_delta ∪ T` and for joins emits all combinations of delta and original (chapter 14), so any change to that algorithm needs a test that confirms the union-of-deltas matches the original full recompute. The `coral-spark-plan` module has effectively zero recent PR traffic — review carefully because most of its surface is from before the type-system migration.

## Cross-cutting concerns

These hit multiple modules at once. Any PR that touches one of these areas needs a wider check than the per-module section above.

**Type system invariants.** `HiveTypeSystem` precision rules (chapter 05) govern `DECIMAL`, `VARCHAR`, `CHAR`, `BINARY`, `TIMESTAMP`, default integer precisions, and the `deriveSumType` / `deriveDecimalDivideType` / `deriveDecimalMultiplyType` overrides. Every backend feeds its `RelDataTypeFactory` with `HiveTypeSystem`, so changes here ripple to every emitter. The d1d5b1ea (#501, "Add missing numeric precisions in Hive type system") template is correct: add tests in at least two backend test suites.

**Transformer ordering.** Chapter 07's punchline: order is everything. A rename after a structural rewrite will not see the original operator name. A type-aware transformer that needs `deriveRelDatatype()` belongs in the type-aware pass (`DataTypeDerivedSqlCallConverter` in Trino, the analogous wrapper in Spark). The two examples from chapter 07: `HiveUDFTransformer` runs *after* the named renames so the rename catches Hive built-ins before the catch-all UDF handler fires; `ReturnTypeAdjustmentTransformer` runs *after* the operator renames so the type adjustment sees the renamed Trino operator's return type. Reviewing a transformer add without considering chain order is incomplete review.

**Function-registry vs. operator-table dual registration.** A new Hive function added to `StaticHiveFunctionRegistry` without a corresponding test in `HiveToRelConverterTest` is structurally incomplete (chapter 06). Resolution happens at parse time through `HiveFunctionResolver`, and the validator goes through `DaliOperatorTable.lookupOperatorOverloads` which ultimately consults the same resolver — but a registry entry without a parse-time test means nobody verified the resolver path actually fires. The test is cheap and the assurance is real.

**Schema/Avro coupling.** PRs in `coral-schema` must not break `coral-spark`'s schema-aware overload (`CoralSpark.create(RelNode, Schema, HiveMetastoreClient)`). The overload uses Avro field names via `AddExplicitAlias` to align Spark output column names with the Avro contract `coral-schema` produces — they share the schema, so a `coral-schema` change that alters field names or namespaces propagates to Spark output and from there to `coral-spark-catalog`'s `View.schema()`. The chain `ViewToAvroSchemaConverter → AddExplicitAlias → CoralSparkViewCatalog.loadView` is load-bearing; reviewers should pull the chain through at least once mentally on any `coral-schema` PR.

**Iceberg vs. Hive table paths.** Prefer `CoralCatalog` and `CoralTable` — chapter 05 covers the migration discipline. Flag direct uses of `org.apache.hadoop.hive.metastore.api.Table` outside the catalog adapters (`HiveCalciteTableAdapter`, `HiveTable.getHiveTable()`, `IcebergHiveTableConverter`). The bridge `IcebergHiveTableConverter.toHiveTable(...)` is tracked for removal by issue #575; new call sites are moving in the wrong direction. The deprecated `HiveMetastoreClient` interface should not see new methods.

**Dialect parity.** A fix in `coral-trino` that also applies to `coral-spark` should ideally mirror — and vice versa. Chapter 07 names this the fifth anti-pattern: "adding a transformer to one backend and forgetting the other." Examples: `from_utc_timestamp`, `named_struct`, `pmod`, `nvl → coalesce`. The dialect-specific subpackages do not share code; the mirror is applied by hand. A reviewer's job is to ask the question explicitly.

## Red flags catalog

Patterns to push back on, in roughly decreasing order of frequency:

- `// TODO` in test corpus or in shipped code. TODOs that lose context are landmines.
- Swallowed exceptions in transformers (`catch (Exception e) {}` or `catch (Exception e) { return sqlCall; }` without rationale). The transformer pattern's anti-pattern #2: condition-too-broad. Silently failing transformers cause downstream surprises.
- Regex-based SQL fixups where SqlNode rewrites would be cleaner. The whole point of the IR is to avoid string-level munging. A `String.replace("FOO", "BAR")` on the SQL output should almost always be a `SqlCallTransformer` or `SparkSqlRewriter` shuttle instead.
- Silent type narrowing: `VARCHAR(255) → VARCHAR`, `INT → BIGINT` without explicit cast. Chapter 07's third transformer anti-pattern. Downstream type derivation drifts.
- New direct uses of `org.apache.hadoop.hive.metastore.api.Table` outside catalog adapters. Should go through `CoralCatalog` / `CoralTable`. See chapter 05.
- A PR that adds a function on the Hive side without touching Spark and Trino sides (or, conversely, adds a Trino transformer without considering the Spark equivalent). Dialect parity; chapter 07's anti-pattern #5.
- Missing `unparse` override on a new custom `SqlOperator`. The operator unparses fine in the AST round-trip but produces broken SQL when serialized. Chapter 07's fourth anti-pattern.
- Changes to `HiveTypeSystem` precision rules without backend regression coverage. The cross-cutting note above lists the rule: tests on at least two backends.
- Transformer added without considering chain ordering. A new entry in `SqlCallTransformers.of(...)` placed at the bottom because it was the convenient spot rather than the correct spot.
- New SPI methods in `coral-benchmark` (`DialectPlugin`, `DialectPluginProvider`, `EnginePlugin`) that aren't backward-compatible. The module ships interfaces consumers will implement; non-default new methods are breaking changes.
- Re-using a deprecated constructor (`HiveMetastoreClient`, `HiveMscAdapter`) in new code when the `CoralCatalog` constructor exists.
- Mutating the input `SqlCall` in a transformer. Chapter 07's first anti-pattern. `SqlCall` is treated as immutable; `transform` builds a fresh call via `operator.createCall(...)`.
- `getHiveTable()` / `getIcebergTable()` called from anywhere that isn't a documented bridge utility. These are escape hatches with `@deprecated INTERNAL API` tags.
- Adding a Spark dialect or Trino dialect change without running the `MergeCoralSchemaWithAvroTests` regression — the Avro contract change is silent until a consumer breaks.

## Reviewer comment templates

Copy-paste-able comment phrasing for the patterns that recur. Adapt to PR specifics; keep the tone direct.

**Test gap.**

```
Could you add a test in HiveToTrinoConverterTest covering [the edge case
this PR touches]? The current change passes [the basic case] but it's not
clear from the diff how [the edge case] behaves with the rewrite. The
transformer-only unit tests don't exercise chain interactions — the
integration test is the spec for this surface.
```

**Spotless failure.**

```
The Spotless check is failing in CI. Run `./gradlew spotlessApply` and
push the formatting changes. Per CONTRIBUTING.md, this is the first thing
to fix before logic review.
```

**Order matters (transformer chain).**

```
This transformer is added at position N in SqlCallTransformers.of(...) in
[CoralToTrinoSqlCallConverter | CoralToSparkSqlCallConverter | other].
Order matters: it currently fires after `XTransformer`, which means [Y].
Did you intend that ordering, or should it precede X? Chapter 07 of the
study guide and the class-level javadoc on the chain both discuss the
ordering discipline.
```

**Dialect parity.**

```
This looks like a Trino-only rewrite. The same Hive semantics divergence
applies to Spark too (see [coral-spark/.../transformers/X.java] for the
analogous transformer). Should we add the equivalent transformer in
coral-spark to keep dialect parity, or is there a reason the Spark side
already handles this case differently?
```

**Deprecated constructor.**

```
This new public method/constructor takes a `HiveMetastoreClient`. That
interface is `@Deprecated` and the `CoralCatalog` overload exists for
new code (see chapter 05). Could we accept `CoralCatalog` here instead,
or add a `CoralCatalog` overload alongside the `HiveMetastoreClient` one?
```

**Type narrowing.**

```
The transformer emits [Trino operator | Spark operator] which has return
type [T1], but the source operator returns [T2]. Downstream type
derivation will drift. The fix is either to pass
`sourceOperator.getReturnTypeInference()` into `createSqlOperator` (the
`OperatorRenameSqlCallTransformer` pattern) or to wrap the result in an
explicit CAST (the `FromUtcTimestampOperatorTransformer` pattern).
```

**Missing test in MergeCoralSchemaWithAvroTests.**

```
This changes [nullability | field matching | merge contract] in
[MergeCoralSchemaWithAvro | RelToAvroSchemaConverter]. Could you add a
test case in MergeCoralSchemaWithAvroTests modeling the new behavior?
The existing test names read like a spec sheet — a regression there is
what downstream Iceberg consumers hit first.
```

## Files this chapter discusses

- `CONTRIBUTING.md`
- `coral-hive/src/test/java/com/linkedin/coral/hive/hive2rel/HiveToRelConverterTest.java`
- `coral-trino/src/test/java/com/linkedin/coral/trino/rel2trino/HiveToTrinoConverterTest.java`
- `coral-spark/src/test/java/com/linkedin/coral/spark/CoralSparkTest.java`
- `coral-schema/src/test/java/com/linkedin/coral/schema/avro/MergeCoralSchemaWithAvroTests.java`
- `coral-spark-catalog/src/test/java/com/linkedin/coral/spark/CoralSparkViewCatalogTest.java`
- `coral-spark-catalog/src/test/java/com/linkedin/coral/spark/TrinoToSparkCatalogTest.java`
- `coral-hive/src/main/java/com/linkedin/coral/hive/hive2rel/functions/StaticHiveFunctionRegistry.java`
- `coral-common/src/main/java/com/linkedin/coral/common/ToRelConverter.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveTypeSystem.java`
- `coral-common/src/main/java/com/linkedin/coral/common/transformers/SqlCallTransformers.java`
- `coral-spark/src/main/java/com/linkedin/coral/spark/CoralToSparkSqlCallConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/CoralToTrinoSqlCallConverter.java`
- `coral-trino/src/main/java/com/linkedin/coral/trino/rel2trino/DataTypeDerivedSqlCallConverter.java`

## Read next

- Chapter 17 — first contributions. The offensive counterpart to this defensive chapter: where to land your first PR.
- Chapter 07 — the `SqlCallTransformer` pattern. Re-read the anti-patterns section if a transformer PR is open.
- The module chapter that matches the PR you're reviewing (04 for `coral-common`, 06 for `coral-hive`, 08 for `coral-spark`, 09 for `coral-trino`, 10 for `coral-schema`, 11 for `coral-spark-catalog`, 12 for `coral-benchmark`, 13 for `coral-data-generation`, 14 for the rest).
