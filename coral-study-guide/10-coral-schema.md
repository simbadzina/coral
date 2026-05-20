# 10 — coral-schema: Avro schema derivation

LinkedIn's data platform is Avro-centric. Coral views need to produce Avro schemas that downstream systems (Kafka, Espresso, event pipelines) can consume directly.

## Entry point

- `ViewToAvroSchemaConverter.toAvroSchema(db, view)` — orchestrates: `HiveToRelConverter` builds the RelNode; `RelToAvroSchemaConverter` walks it.

## The conversion

- `RelToAvroSchemaConverter` — bottom-up `RelNode` visitor. Each operator contributes to or transforms the schema.
- `RelDataTypeToAvroType` — Calcite `RelDataType` → Avro primitive/composite.
- `TypeInfoToAvroSchemaConverter` — Hive `TypeInfo` → Avro (for base tables).
- `SchemaUtilities` — namespace handling, name compatibility, nullable handling.

## Merge engines

Two strategies for combining a derived schema with a partner Avro schema stored in table metadata (preserving docs, defaults, custom properties):

- `MergeHiveSchemaWithAvro` (legacy) — Hive types drive structure; Avro metadata fills in.
- `MergeCoralSchemaWithAvro` (new) — `CoralDataType` drives structure (Iceberg-first); partner Avro contributes metadata. Case-insensitive matching; output uses Coral's casing.

## Why two

Hive tables and Iceberg tables have different semantics. `MergeHiveSchemaWithAvro` exists for backward compatibility; `MergeCoralSchemaWithAvro` is the path forward as the catalog migrates to `CoralCatalog`.

## Files this chapter discusses

- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/ViewToAvroSchemaConverter.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/RelToAvroSchemaConverter.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/RelDataTypeToAvroType.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/SchemaUtilities.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/MergeHiveSchemaWithAvro.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/MergeCoralSchemaWithAvro.java`
- `coral-schema/src/main/java/com/linkedin/coral/schema/avro/TypeInfoToAvroSchemaConverter.java`

## Read next

- Chapter 05 — type system (Avro derivation builds on it).
- Chapter 15 — LinkedIn specifics (why Avro matters here).
