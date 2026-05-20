# 19 — Glossary

Alphabetical. Each entry one sentence; longer treatment in the linked chapter.

- **ANTLR** — parser generator. Coral uses ANTLR for Hive grammars (chapter 06).
- **AST** — abstract syntax tree. In Coral, often refers to `SqlNode` (chapter 02).
- **Avro** — schema-driven serialization format; LinkedIn's standard for data interchange (chapter 10).
- **Calcite** — Apache Calcite. The framework Coral builds on (chapter 02).
- **Convertlet** — Calcite's `SqlRexConvertlet` — a rule for converting a `SqlCall` into a `RexNode` (chapter 02).
- **CoralCatalog** — unified table-metadata abstraction over Hive and Iceberg (chapter 05).
- **Coral IR** — the two-layer intermediate representation: `SqlNode` AST and `RelNode` logical plan (chapter 01).
- **Dali** — LinkedIn's logical dataset platform (chapter 15).
- **Dialect** — a SQL variant (Hive, Trino, Spark, Pig). Coral has frontends and backends per dialect.
- **EXPLAIN level** — a coral-benchmark verification level that runs target-engine `EXPLAIN` on translated SQL (chapter 12).
- **Espresso** — LinkedIn's distributed key-value store (chapter 15).
- **Fuzzy Union** — Coral's reconciliation of UNION branches with heterogeneous struct schemas (chapter 15).
- **HMS** — Hive Metastore. The legacy table-metadata service (chapter 04).
- **Iceberg** — table format newer than Hive; supported via `CoralCatalog` (chapter 05).
- **IR Round-trip** — source SQL → IR → target SQL → IR — a sanity check used by coral-benchmark (chapter 12).
- **IVM** — Incremental View Maintenance. Rewriting a view to compute deltas (chapter 14).
- **Logical plan** — Calcite's `RelNode` tree. The lower layer of Coral IR (chapter 02).
- **RelNode** — Calcite's logical plan node (chapter 02).
- **RexNode** — Calcite's row-expression node, the children inside `RelNode` operators (chapter 02).
- **RESULT_SET level** — coral-benchmark verification that runs both engines and compares result sets (chapter 12).
- **Shaded parser** — `coral-trino-parser`, Trino's parser repackaged to avoid ANTLR version conflicts (chapter 09).
- **SqlCall** — a `SqlNode` representing a function or operator call (chapter 02).
- **SqlCallTransformer** — Coral's pattern for rewriting `SqlCall` nodes (chapter 07).
- **SqlNode** — Calcite's AST node. The upper layer of Coral IR (chapter 02).
- **StaticHiveFunctionRegistry** — Coral's hardcoded table of Hive built-in functions (chapter 06).
- **Symbolic solver** — `coral-data-generation`'s domain inference (chapter 13).
- **Transport UDF** — LinkedIn's portable UDF framework (chapter 15).
- **TRANSLATION level** — coral-benchmark verification that only checks round-trip translation (chapter 12).
- **ViewShift** — LinkedIn's dynamic policy enforcement on data lakes, powered by Coral rewrites (chapter 15).
