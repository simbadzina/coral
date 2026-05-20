# 05 — Type system and CoralCatalog

Two intertwined abstractions: a dialect-neutral type system, and a unified catalog that hides whether tables come from Hive or Iceberg.

## CoralDataType hierarchy

- `CoralDataType` — base.
- Primitives: `PrimitiveType`, `CharType`, `VarcharType`, `BinaryType`, `DecimalType`, `TimestampType`.
- Composites: `ArrayType`, `MapType`, `StructType` + `StructField`.
- Enum dispatch: `CoralTypeKind`.

The point of `CoralDataType` is to reason about types without committing to Calcite's `RelDataType` (which is loaded with framework state) or Hive's `TypeInfo` (which is loaded with Hadoop dependencies).

## Type system conversion

- `HiveToCoralTypeConverter` — `TypeInfo` → `CoralDataType`.
- `IcebergToCoralTypeConverter` — Iceberg `Schema.Type` → `CoralDataType`.
- `CoralTypeToRelDataTypeConverter` — `CoralDataType` → Calcite `RelDataType`.
- `HiveTypeSystem` — Calcite `RelDataTypeSystem` defining precision/scale rules that match Hive (e.g., DECIMAL max precision = 38).

## CoralCatalog and CoralTable

- `CoralCatalog` — interface. `getTable(db, name)`, `getAllTables`, `getAllNamespaces`, `namespaceExists`.
- `CoralTable` — interface. `name()`, `tableType()`, `getSchema()`, `properties()`.
- Implementations: `HiveTable` (wraps HMS `Table`), `IcebergTable`, plus `IcebergHiveTableConverter` (bridge for legacy code that still wants HMS `Table`).
- `TableType` enum.

## Migration off HiveMetastoreClient

`HiveMetastoreClient` is `@Deprecated`. New code accepts `CoralCatalog`. The two `ToRelConverter` constructors exist side-by-side to support this migration. PRs touching catalog plumbing should prefer the `CoralCatalog` path; flag the deprecated path in review.

## Files this chapter discusses

- `coral-common/src/main/java/com/linkedin/coral/common/catalog/CoralCatalog.java`
- `coral-common/src/main/java/com/linkedin/coral/common/catalog/CoralTable.java`
- `coral-common/src/main/java/com/linkedin/coral/common/catalog/HiveTable.java`
- `coral-common/src/main/java/com/linkedin/coral/common/catalog/IcebergTable.java`
- `coral-common/src/main/java/com/linkedin/coral/common/catalog/IcebergHiveTableConverter.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveTypeSystem.java`
- `coral-common/src/main/java/com/linkedin/coral/common/HiveToCoralTypeConverter.java`
- `coral-common/src/main/java/com/linkedin/coral/common/IcebergToCoralTypeConverter.java`
- `coral-common/src/main/java/com/linkedin/coral/common/types/*`

## Read next

- Chapter 06 — coral-hive (the first concrete user of these abstractions).
- Chapter 10 — coral-schema (Avro derivation builds on the type system).
