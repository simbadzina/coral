# 19 — Glossary

Alphabetical reference. Each entry is one sentence — enough to anchor recall, short enough to scan. For the load-bearing terms, follow the chapter pointer.

> **Reading time** ~11 min  ·  **Prerequisites** none
>
> **Key takeaways**
> - Each entry is a one-sentence definition followed by a chapter pointer; read the entry to anchor recall, then jump to the cited chapter when a term is load-bearing.
> - Entries are alphabetical, so scan rather than read top to bottom; class names like `ToRelConverter`, `FuzzyUnionSqlRewriter`, and `StaticHiveFunctionRegistry` appear under their own names.
> - Use this as the cross-chapter index for LinkedIn-specific terms (`Dali`, `Espresso`, `ViewShift`, `Transport UDF`) and Calcite terms (`SqlNode`, `RelNode`, `RexNode`, convertlet) that recur throughout the guide.

- **ANTLR** — parser generator. Coral uses ANTLR v3 for the Hive grammar (`coral-hive`) and a shaded ANTLR v4 for the Trino grammar (`shading/coral-trino-parser`); the version mismatch is why shading exists ([chapter 06](06-coral-hive.md), [chapter 09](09-coral-trino.md)).
- **AST** — abstract syntax tree. In Coral this means the `SqlNode` tree, close to surface SQL and produced by every frontend ([chapter 02](02-calcite-primer.md)).
- **Avro** — schema-driven serialization format; LinkedIn's standard for Kafka, Espresso, and event pipelines, and the output format of `coral-schema` ([chapter 10](10-coral-schema.md)).
- **Calcite (Apache)** — the SQL framework Coral builds on, providing `SqlNode`, `RelNode`, `RexNode`, the validator, and the convertlet machinery ([chapter 02](02-calcite-primer.md)).
- **CatalogExtension (Spark SPI)** — the Spark 3.5 plug-in interface that `CoralSparkViewCatalog` implements to intercept view resolution and translate Hive views into Spark SQL at query time ([chapter 11](11-coral-spark-catalog.md)).
- **Convertlet** — Calcite's `SqlRexConvertlet`, a rule mapping a `SqlCall` to a `RexNode` during the `SqlNode → RelNode` lowering; Coral extends the default convertlet table with Hive-flavored entries ([chapter 02](02-calcite-primer.md)).
- **CoralCatalog** — the unified table-metadata abstraction over Hive Metastore and Iceberg catalogs, replacing the deprecated `HiveMetastoreClient` interface; migration tracked in issue #575 ([chapter 05](05-type-system-and-catalog.md)).
- **CoralDataType** — Coral's wrapper over Calcite's `RelDataType` that carries Hive-specific precision/scale defaults and nullability semantics ([chapter 05](05-type-system-and-catalog.md)).
- **Coral IR** — Coral's intermediate representation: the two-layer `SqlNode` AST + `RelNode` logical plan, isomorphic and convertible to each other ([chapter 01](01-the-big-picture.md)).
- **Dali** — LinkedIn's logical dataset platform; provides versioned views and a unified namespace, and is the historical reason Coral exists ([chapter 15](15-linkedin-specifics.md)).
- **Dialect** — a SQL variant (Hive, Trino, Spark, Pig). Coral has one frontend per source dialect and one backend per target dialect ([chapter 01](01-the-big-picture.md)).
- **DialectPlugin (coral-benchmark)** — the SPI a benchmark consumer implements to declare how a given dialect is parsed and emitted; lives at [`coral-benchmark/src/main/java/com/linkedin/coral/benchmark/spi/DialectPlugin.java`](../coral-benchmark/src/main/java/com/linkedin/coral/benchmark/spi/DialectPlugin.java) ([chapter 12](12-coral-benchmark.md)).
- **DNF (Disjunctive Normal Form)** — `(A AND B) OR (C AND D)` form; `DnfRewriter` in `coral-data-generation` rewrites predicates into DNF so each disjunct can be solved independently ([chapter 13](13-coral-data-generation.md)).
- **Domain (coral-data-generation)** — an interface describing the set of values a variable can take; subclasses include `IntegerDomain`, `RegexDomain`, and others ([chapter 13](13-coral-data-generation.md)).
- **DomainInferenceProgram** — the entry point in `coral-data-generation` that walks a `RelNode` predicate and emits domain constraints per column ([chapter 13](13-coral-data-generation.md)).
- **EnginePlugin (coral-benchmark)** — the SPI that declares how to actually execute SQL against a target engine for `EXPLAIN` or `RESULT_SET` verification ([chapter 12](12-coral-benchmark.md)).
- **EXPLAIN level (coral-benchmark)** — verification level that runs the target engine's `EXPLAIN` on the translated SQL; passes if the engine accepts the translation, even without executing it ([chapter 12](12-coral-benchmark.md)).
- **Espresso** — LinkedIn's distributed key-value store, an Avro-schema consumer downstream of `coral-schema`-derived schemas ([chapter 15](15-linkedin-specifics.md)).
- **FrameworkConfig (Calcite)** — Calcite's bundle of `SqlParserConfig`, `SchemaPlus`, `SqlOperatorTable`, and convertlet table; Coral builds one inside `ToRelConverter` for use by the planner ([chapter 04](04-coral-common.md)).
- **Fuzzy Union** — Coral's reconciliation of `UNION` branches whose nested-struct schemas don't line up exactly; implemented in `FuzzyUnionSqlRewriter` ([chapter 15](15-linkedin-specifics.md)).
- **HiveCalciteTableAdapter** — the Calcite `Table` implementation that wraps an HMS `Table` so the Calcite planner can see Hive-managed tables ([chapter 05](05-type-system-and-catalog.md)).
- **HiveMetastoreClient (deprecated)** — Coral's original interface for fetching table metadata, being phased out in favor of `CoralCatalog` ([chapter 05](05-type-system-and-catalog.md), issue #575).
- **HMS (Hive Metastore)** — the legacy table-metadata service; still the primary metadata source for most LinkedIn datasets ([chapter 04](04-coral-common.md)).
- **HiveTypeSystem** — Coral's Hive-flavored override of Calcite's type system, defining precision rules for DECIMAL, default lengths for VARCHAR, and the rest — touching it ripples to every backend ([chapter 05](05-type-system-and-catalog.md)).
- **HiveUncollect** — Coral's RelNode operator for Hive's `LATERAL VIEW EXPLODE` semantics, which differ from Calcite's stock `Uncollect` around null and array handling ([chapter 06](06-coral-hive.md)).
- **Iceberg** — table format newer than Hive; supported in Coral via the `IcebergCalciteTableAdapter` and the Iceberg path through `CoralCatalog` ([chapter 05](05-type-system-and-catalog.md)).
- **IcebergHiveTableConverter** — the legacy adapter under `coral-common/.../catalog/IcebergHiveTableConverter.java` that wraps an Iceberg table in a "minimal Hive Table" for code paths that still expect HMS shapes ([chapter 05](05-type-system-and-catalog.md)).
- **IntegerDomain (coral-data-generation)** — the `Domain` implementation for integer-valued columns, represented as a union of intervals ([chapter 13](13-coral-data-generation.md)).
- **IR Round-trip** — `source SQL → IR → target SQL → IR` translated back, used by `coral-benchmark` at the `TRANSLATION` verification level as a cheap correctness sanity check ([chapter 12](12-coral-benchmark.md)).
- **IVM (Incremental View Maintenance)** — rewriting a view definition so it computes deltas against changed inputs instead of full recompute; implemented in `coral-incremental` ([chapter 14](14-other-modules.md)).
- **LATERAL VIEW** — Hive's syntax for unnesting an array column into rows; converted by Coral into a `Correlate` + `HiveUncollect` shape in the RelNode plan ([chapter 06](06-coral-hive.md)).
- **Logical plan** — Calcite's `RelNode` tree; the lower, canonical layer of Coral IR ([chapter 02](02-calcite-primer.md)).
- **LogicalProject / Filter / Join / Aggregate (RelNode)** — the four most common Calcite `RelNode` operators, corresponding to SQL `SELECT` projections, `WHERE`, `JOIN`, and `GROUP BY` clauses respectively ([chapter 02](02-calcite-primer.md)).
- **MergeCoralSchemaWithAvro vs MergeHiveSchemaWithAvro** — two engines in `coral-schema` for merging a view's derived schema with the Avro schemas of base tables; `MergeCoralSchema...` is the newer (#600) path that goes through `CoralCatalog`, `MergeHiveSchema...` is the legacy path ([chapter 10](10-coral-schema.md)).
- **ParseTreeBuilder** — the visitor that walks the ANTLR-produced parse tree and emits Calcite `SqlNode`s; one implementation per source dialect ([chapter 06](06-coral-hive.md), [chapter 09](09-coral-trino.md)).
- **RegexDomain** — `coral-data-generation`'s `Domain` implementation for string columns, expressed as a regular expression and convertible to/from `IntegerDomain` when arithmetic is involved ([chapter 13](13-coral-data-generation.md)).
- **RelBuilder** — Calcite's fluent API for constructing `RelNode` trees; Coral subclasses it as `HiveRelBuilder` to override unnest and a few other ops ([chapter 02](02-calcite-primer.md), [chapter 04](04-coral-common.md)).
- **RelDataType** — Calcite's type interface; `HiveTypeSystem` configures the factory that produces `RelDataType` instances with Hive-correct precision ([chapter 05](05-type-system-and-catalog.md)).
- **RelNode** — Calcite's logical relational plan node; the bottom layer of Coral IR ([chapter 02](02-calcite-primer.md)).
- **RexNode** — Calcite's row-expression node, the children inside `RelNode` operators — function calls, column references, literals ([chapter 02](02-calcite-primer.md)).
- **RESULT_SET level** — `coral-benchmark`'s strictest verification level: runs the query on both engines, compares actual result sets ([chapter 12](12-coral-benchmark.md)).
- **Shaded parser (coral-trino-parser)** — the repackaged copy of Trino's parser plus ANTLR v4 plus Airlift relocated under `coral.shading.*`, isolating it from Calcite's ANTLR v3 ([chapter 09](09-coral-trino.md)).
- **ShipKit** — the release-automation tool that auto-generates Coral's release notes from merged PR titles; explains why PR title hygiene matters ([chapter 18](18-engagement-and-community.md)).
- **SparkUDFInfo** — the metadata bundle `coral-spark` returns alongside translated SQL, listing the UDFs Spark needs to register (artifact URLs, class names, registration mode) — lives at [`coral-spark/src/main/java/com/linkedin/coral/spark/containers/SparkUDFInfo.java`](../coral-spark/src/main/java/com/linkedin/coral/spark/containers/SparkUDFInfo.java) ([chapter 08](08-coral-spark.md)).
- **SqlCall** — a `SqlNode` representing a function or operator invocation; the most common subtype manipulated by transformers ([chapter 02](02-calcite-primer.md)).
- **SqlCallTransformer** — Coral's pattern (an abstract class in `coral-common/.../transformers/`) for matching and rewriting `SqlCall` nodes during backend emission ([chapter 07](07-transformers-pattern.md)).
- **SqlCallTransformers** — the utility class (`coral-common/.../transformers/SqlCallTransformers.java`) that composes an ordered list of `SqlCallTransformer`s into a single traversal; order matters ([chapter 07](07-transformers-pattern.md)).
- **SqlNode** — Calcite's AST node; the top layer of Coral IR ([chapter 02](02-calcite-primer.md)).
- **SqlOperator** — Calcite's representation of a function or operator (e.g. `=`, `SUBSTR`, `CAST`); registered in a `SqlOperatorTable` and resolved during validation ([chapter 02](02-calcite-primer.md)).
- **SqlOperatorTable** — Calcite's registry of available `SqlOperator`s; Coral provides `DaliOperatorTable` for LinkedIn-specific functions plus the static Hive registry ([chapter 06](06-coral-hive.md)).
- **SqlRexConvertletTable** — Calcite's registry of `SqlRexConvertlet`s; Coral overrides this to inject Hive-specific lowering rules during `SqlNode → RelNode` ([chapter 02](02-calcite-primer.md)).
- **SqlValidator** — Calcite's semantic validator; runs after parsing to check column references, resolve operators, and infer types — Coral uses `HiveSqlValidator` for Hive-flavored resolution ([chapter 02](02-calcite-primer.md), [chapter 06](06-coral-hive.md)).
- **StaticHiveFunctionRegistry** — Coral's hardcoded table mapping Hive built-in function names to `SqlOperator`s with return-type inference; the single source of truth for "does Coral know about this UDF" ([chapter 06](06-coral-hive.md)).
- **StructType / StructField** — Calcite's `RelDataType` for nested records and their fields; used heavily by `coral-schema` when deriving Avro records ([chapter 10](10-coral-schema.md)).
- **Symbolic solver** — the `coral-data-generation` engine that inverts SQL expressions to derive input-column constraints — see `DomainInferenceProgram` and the `domain.transformer` package ([chapter 13](13-coral-data-generation.md)).
- **ToRelConverter** — the abstract base class in `coral-common` that every frontend extends; orchestrates parse → validate → convert into `RelNode` ([chapter 04](04-coral-common.md)).
- **Transport UDF** — LinkedIn's portable-UDF framework, with a single source written once and emitted as engine-specific JARs; Coral detects them via `TransportUDFTransformer` ([chapter 15](15-linkedin-specifics.md)).
- **TranslationTestSuite** — the entry point in `coral-benchmark` that runs a query corpus through translation + verification at the configured level ([chapter 12](12-coral-benchmark.md)).
- **TRANSLATION level** — `coral-benchmark`'s cheapest verification level — only checks that the round-trip translation produces well-formed target SQL ([chapter 12](12-coral-benchmark.md)).
- **ViewShift** — LinkedIn's dynamic policy-enforcement layer for data lakes, powered by Coral IR rewrites that inject access-control predicates into views at translation time ([chapter 15](15-linkedin-specifics.md)).
- **VolcanoPlanner** — Calcite's cost-based planner; Coral uses it for the rewrite passes in `coral-incremental` and a few `coral-common` rewrites, but most Coral conversions are not cost-driven ([chapter 02](02-calcite-primer.md)).

## Read next

- [Chapter 01](01-the-big-picture.md) — the big picture (the map that contextualizes most of these terms).
- [Chapter 02](02-calcite-primer.md) — Calcite primer (for the dense cluster of Calcite terms above).
- [Chapter 15](15-linkedin-specifics.md) — LinkedIn specifics (for Dali, Espresso, ViewShift, Transport UDF).
