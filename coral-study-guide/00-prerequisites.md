# 00 — Prerequisites

Everything you need on your machine to read and exercise this guide.

> **Reading time** ~2 min  ·  **Prerequisites** none
>
> **Key takeaways**
> - The repo builds against Java 8 and uses the bundled Gradle wrapper, not a globally installed Gradle.
> - ANTLR-generated parser sources land under `coral-hive/build/generated/sources/` and must be marked as generated for IDE navigation to work.

## Toolchain

- Java 8 (the repo currently builds against Java 8). Set `JAVA_HOME` or pass `-Dorg.gradle.java.home=` to Gradle.
- Gradle wrapper ships with the repo (`./gradlew`). Don't install Gradle globally; use the wrapper.
- Python 3 is needed by some build scripts.

## Building

- `./gradlew clean build` — full build + tests.
- `./gradlew :coral-hive:test --tests HiveToRelConverterTest` — single test class.
- `./gradlew spotlessApply` — auto-format before commits.

## IDE

- IntelliJ IDEA Community is fine. Import as a Gradle project.
- ANTLR-generated parser sources land under `coral-hive/build/generated/sources/`. Mark that directory as "Generated Sources" so navigation works.
- Enable Lombok plugin if your IDE complains (the repo uses some Lombok).

## What this chapter discusses

- Top-level `README.md`, `build.gradle`, `settings.gradle`.
- `gradle/dependencies.gradle` (Calcite, Hive, Avro, Trino versions).

## Self-check

1. Which Gradle command runs a single test class, and which one auto-formats before a commit?

## Read next

- [Chapter 01](01-the-big-picture.md) — the big picture.
