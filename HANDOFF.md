# Hortora engine — Project Handoff

---

## What's In Flight

Nothing — both repos on main, clean, pushed.

## Immediate Next Step

Run the retrieval benchmark against the real BGE-M3 model. The ONNX export is at `~/.hortora/models/bge-m3/model.onnx` — start the engine in dev mode (`quarkus:dev` with `%dev` properties uncommented), index the garden, then run `scripts/benchmark/run_queries.py` against the 14 real-world scenarios. Compare against the 94% dense-only baseline from #27. This feeds #33 (CC vs RRF) and #34 (ColBERT quantization).

## What Changed This Session

- **BGE-M3 ONNX export landed** — engine `12e4027` (squashed). Three-head PyTorch wrapper adapted from aapot/bge-m3-onnx with sparse scatter baked into the ONNX graph, ColBERT including CLS, export via `torch.onnx.export` (opset 18). Validated against PyTorch. `download-models.sh` converted to verification-only.
- **Neocortex rename** — engine `c2383bc`. Neural-text → neocortex import renames (parallel session work, Refs #628).
- **Design review:** adversarial, 4 rounds, $14.19, 19 issues raised, 16 verified, 0 unresolved.
- **1 garden entry:** GE-20260701-f7e1d5 (ColBERT CLS token must be included for batch inference).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Run BGE-M3 benchmark | S | Low | Model exported — just run the harness |
| — | SeparateModelEmbedder for non-BGE-M3 deployments | S | Low | File neocortex issue |
| #33 | Convex Combination fusion test | S | Low | CC α=0.5 vs RRF — config change only |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | After baseline established |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| ONNX export design spec | `docs/superpowers/specs/2026-07-01-bge-m3-onnx-export-design.md` |
| BGE-M3 adoption design spec | `docs/superpowers/specs/2026-06-30-bge-m3-adoption-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-07-01-bge-m3-onnx-export.md` |
| Retrieval research | `docs/comparison/retrieval-research.md` |
