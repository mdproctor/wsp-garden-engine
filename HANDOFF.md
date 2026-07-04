# Hortora engine — Project Handoff

---

## What Just Shipped

Branch `issue-36-bge-m3-benchmark` closed. BGE-M3 four-signal benchmark complete: 87% precision at 50ms median latency (vs 90%/240ms three-leg baseline). Ghost cursor bug fixed (CollectionMigration now clears stale cursors on fresh Qdrant). Stale `quarkus.index-dependency` artifact ID fixed after neocortex rename. Two garden entries submitted (Qdrant ColBERT multi-vector limit, macOS /var/folders glob). Landed as `b44add8` on main.

## Immediate Next Step

Pick up #33 (Convex Combination fusion) — this is the most likely path to recover the 3pp precision gap vs three-leg. Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Convex Combination fusion test — CC (α=0.5) vs RRF | S | Low | **Priority 1** — tuning signal weights may recover SEMANTIC_WIN gap |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | **Priority 2** — efficiency optimisation, 160ms latency headroom available |
| — | Qdrant ColBERT multi-vector limit workaround | S | Med | Truncate ColBERT tokens or raise limit — caps sequence length at ~1023 |
| — | Filter `.git/` files from `FlatCorpusStore.list()` | XS | Low | Efficiency — cuts ingestion from 11K to 2K entries, halves startup time |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| BGE-M3 benchmark report | `docs/comparison/bge-m3-benchmark.md` |
| Benchmark design spec | `docs/superpowers/specs/2026-07-02-bge-m3-benchmark-design.md` |
| Retrieval research | `docs/comparison/retrieval-research.md` |
| Qdrant ColBERT limit | GE-20260704-ba9911 |
