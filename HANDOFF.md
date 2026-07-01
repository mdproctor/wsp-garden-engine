# Hortora engine — Project Handoff

---

## What's In Flight

Nothing — both repos on main, clean, pushed.

## Immediate Next Step

Deliver the BGE-M3 three-head ONNX export script. The SPI and pipeline are merged in both repos, but all tests use `InMemoryInferenceModel` stubs. End-to-end retrieval requires an ONNX model with dense/sparse/ColBERT output heads baked in — HuggingFace Optimum only exports the backbone, so a custom script wrapping BAAI's `modeling.py` is needed. Update `download-models.sh` with the real download URL once the export is available.

## What Changed This Session

- **BGE-M3 adoption landed in both repos** — neural-text `50c5a04` (squashed), engine `ee7cc47` (squashed). Both on main, pushed to all remotes.
- **neural-text:** `InferenceOutput` evolved to multi-output final class, `MultiModalEmbedder` SPI, `BgeM3Embedder` module, RAG pipeline migrated from `EmbeddingModel + SparseEmbedder + CrossEncoderReranker` to `MultiModalEmbedder`. ColBERT MAX_SIM reranking. 40 files, +1535/-690.
- **engine:** Ollama removed, `HybridSearchProducer` produces `MultiModalEmbedder` from `@Inference("bge-m3")`, `CollectionMigration` detects dimension mismatch + missing ColBERT.
- **Design review:** adversarial, 4 rounds, $15, 18 verified improvements.
- **2 garden entries:** GE-20260630-db5dce (BGE-M3 sparse post-processing), GE-20260630-73d3c2 (record array immutability).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | BGE-M3 ONNX export script | M | Med | Custom three-head export; prerequisite for end-to-end |
| — | SeparateModelEmbedder for non-BGE-M3 deployments | S | Low | File neural-text issue |
| #33 | Convex Combination fusion test | S | Low | CC α=0.5 vs RRF — config change only |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | After baseline established |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| BGE-M3 design spec | `docs/superpowers/specs/2026-06-30-bge-m3-adoption-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-30-bge-m3-adoption.md` |
| Retrieval research | `docs/comparison/retrieval-research.md` |
