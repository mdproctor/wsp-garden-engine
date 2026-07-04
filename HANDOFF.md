# Hortora engine — Project Handoff

---

## What Just Shipped

Branch `issue-33-s-xs-batch` closed. Federation now propagates type and tags filters to upstream/peer nodes (#30). Five lines of production code — the session's real value was the adversarial design review on #37, which caught cross-layer coupling and improved the architecture before any code was written. Landed as `fc0bf6d` on main.

## Immediate Next Step

Pick up #37 (ColBERT sequence validation) in a cross-repo session — the design review produced a clean spec: add `maxSequenceLength()` to `MultiModalEmbedder` in inference-api, configurable `max-multivector-floats` in `RagConfig`. Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #37 | ColBERT sequence length validation — maxSequenceLength on MultiModalEmbedder | S | Med | **Priority 1** — reviewed spec ready; needs cross-repo (inference-api, inference-bge-m3, rag, engine) |
| #33 | Convex Combination fusion test — CC (α=0.5) vs RRF | S | Low | Needs configurable fusion strategy in casehub-neocortex-rag |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | Efficiency optimisation, 160ms latency headroom |
| #38 | Filter .git/ files from FlatCorpusStore.list() | XS | Low | Needs change in casehub-neocortex-corpus |
| #30 | Federation type/tags propagation | — | — | **CLOSED** — landed as fc0bf6d |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| #37 reviewed spec | `docs/superpowers/specs/2026-07-04-federation-filter-and-colbert-validation-design.md` |
| Design review workspace | `~/adr/hortora-engine/federation-filter-colbert-validation-*` |
| BGE-M3 benchmark report | `docs/comparison/bge-m3-benchmark.md` |
