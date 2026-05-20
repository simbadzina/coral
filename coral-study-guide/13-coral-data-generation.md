# 13 — coral-data-generation: the symbolic solver

The most academically interesting module in the repo. Given a SQL predicate, generate concrete inputs that satisfy it. Symbolic, not random.

## The shift in perspective

Forward: input → output. Backward: output constraint → input domain. Coral inverts SQL expressions symbolically so that test data generation produces only valid rows.

## Domains

- `Domain` — base abstraction.
- `RegexDomain` — automaton-backed string constraints. Backed by `dk.brics.automaton`. Supports intersect, union, emptiness check, deterministic sampling.
- `IntegerDomain` — disjoint-interval representation `[10,20] ∪ [50,60]`. Closed under arithmetic.

## Transformers

Each transformer inverts one SQL operator:

- `LowerRegexTransformer` — `LOWER(x) = 'abc'` → x must match `^[aA][bB][cC]$`.
- `SubstringRegexTransformer` — embeds the pattern at the correct offset.
- `CastRegexTransformer` — string ↔ integer cross-domain.
- `PlusIntegerTransformer`, `TimesIntegerTransformer` — arithmetic inversion.

## RelNode preprocessing

Before inverting, the plan is normalized:

- `ProjectPullUpController` — fixed-point pull projections up.
- `CanonicalPredicateExtractor` — pull predicates out and index fields globally.
- `DnfRewriter` — disjunctive normal form so OR branches solve independently.

## Solver

- `DomainInferenceProgram` — top-down recursive inversion. Returns a `Domain` per source field.

## Why this matters

The framework is extensible to new transformers and new domains. It also detects contradictory queries early (empty intersection). Pairs with `coral-benchmark` (chapter 12).

## Files this chapter discusses

- `coral-data-generation/README.md` (extensive — read first)
- `coral-data-generation/src/main/java/com/linkedin/coral/datagen/domain/`
- `coral-data-generation/src/main/java/com/linkedin/coral/datagen/domain/transformer/`
- `coral-data-generation/src/main/java/com/linkedin/coral/datagen/rel/`

## Read next

- Chapter 12 — coral-benchmark (consumer of inferred domains).
