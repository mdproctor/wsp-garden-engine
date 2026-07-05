# Hortora engine — Project Handoff

---

## What Just Shipped

Branch `issue-38-git-filter-and-quantization` closed. Matryoshka truncation wiring (#34) and ColBERT sequence validation (#37) landed as `4b244d8` on main. Cross-repo: `maxSequenceLength()` added to `MultiModalEmbedder` and `maxMultivectorFloats()` added to `RagConfig` in neocortex (`3ef904b`). Also moved engine#38 to neocortex#99 with full root-cause spec.

## Immediate Next Step

Run Matryoshka benchmark — set `casehub.rag.matryoshka.dimension=512` in dev config, restart (triggers re-indexing via CollectionMigration dimension mismatch), run benchmark harness. Compare 512-dim precision against 1024-dim baseline (87%).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| neocortex#99 | Filter hidden paths (.git/) from FlatCorpusStore.list() | XS | Low | Full spec in issue — FlatCorpusStore + FlatChangeSource |
| neocortex#100 | ColBERT scalar quantization config in RagConfig | S | Low | Enables ColBERT int8 quantization evaluation |
| #33 | Convex Combination fusion test — CC (α=0.5) vs RRF | S | Low | Needs configurable fusion strategy in casehub-neocortex-rag |
| #34 | Matryoshka benchmark evaluation | — | — | **CLOSED** — wiring landed; benchmark run pending |
| #37 | ColBERT sequence validation | — | — | **CLOSED** — landed as 4b244d8 |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| Matryoshka + ColBERT spec | `docs/superpowers/specs/2026-07-04-federation-filter-and-colbert-validation-design.md` |
| BGE-M3 benchmark report | `docs/comparison/bge-m3-benchmark.md` |
| neocortex#99 (hidden path filtering) | casehubio/neocortex#99 |
| neocortex#100 (ColBERT quantization) | casehubio/neocortex#100 |
