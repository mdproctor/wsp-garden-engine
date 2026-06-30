# Hortora engine — Project Handoff

---

## What's In Flight

Nothing — main is clean and pushed.

## Immediate Next Step

Pick the next retrieval improvement: BGE-M3 adoption (single model replacing nomic-embed-text + SPLADE + cross-encoder) or convex combination fusion test (CC α=0.5 vs RRF — config change only, no code). BGE-M3 is the bigger win but requires neural-text #30; CC is a quick experiment.

## What Changed This Session

- **All 224 three-leg benchmark entries scored** — 89.3% relevant precision, 64.3% highly relevant, 10.7% noise. Three-leg gardenSearch now clearly beats grep (was 6-6-2 tied in #27).
- **Adaptive result extension** — gardenSearch over-fetches 2x, extends through dense score clusters (gap < 0.05), returns total-count metadata signal. Addresses the fixed 8-result cap that was grep's remaining structural advantage.
- **DESIGN.md synced** — precision figure updated, adaptive extension documented.
- **1 unpushed commit from previous session landed** — `2e6e682` docs update now on origin/main.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | BGE-M3 adoption (neural-text #30) | L | Med | Single model replaces nomic-embed-text + SPLADE + cross-encoder; ColBERT as reranker |
| — | Convex Combination fusion test | S | Low | CC (α=0.5) vs RRF — config change, no code |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Known Issues

- In-process `BM25Index` in neural-text is dead code on retrieval path (Qdrant-native BM25 is primary)
- neural-text #51/#52 (ONNX model validation) workaround'd in download-models.sh, not fixed upstream

## Key References

| Resource | Location |
|---|---|
| Three-leg benchmark results | `scripts/benchmark/results/three-leg.json` |
| Retrieval research + roadmap | `docs/comparison/retrieval-research.md` |
| Hybrid benchmark report | `docs/comparison/hybrid-benchmark.md` |
| Blog — scoring then fixing | `blog/2026-06-30-mdp03-scoring-then-fixing.md` |
| Engine design | `docs/DESIGN.md` |
