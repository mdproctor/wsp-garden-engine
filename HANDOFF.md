# Hortora engine — Project Handoff

---

## What's In Flight

Nothing — main is clean and pushed.

## Immediate Next Step

Deliver the BGE-M3 three-head ONNX export script. The SPI and pipeline changes are merged (neural-text `issue-30-bge-m3-multi-modal`, engine `ee7cc47`), but end-to-end retrieval requires the actual ONNX model with dense/sparse/ColBERT output heads baked in. HuggingFace Optimum only exports the backbone — a custom export wrapping BAAI's `modeling.py` forward() is needed. File neural-text issue for the export script, then update `download-models.sh`.

## What Changed This Session

- **BGE-M3 adoption complete** — `InferenceOutput` evolved to multi-output final class, `MultiModalEmbedder` SPI created in `inference-api`, `BgeM3Embedder` module added, full RAG pipeline evolved (HybridCaseRetriever, QdrantEmbeddingIngestor, all reactive variants, bean producers), engine wired to BGE-M3.
- **Ollama removed** — `quarkus-langchain4j-ollama` dependency dropped. Dense embeddings now come from ONNX via the `InferenceModel` SPI.
- **ColBERT MAX_SIM reranking** — replaces client-side cross-encoder. Document ColBERT vectors stored as Qdrant multi-vectors, server-side rescoring via two-stage query.
- **Design review (adversarial, 4 rounds, $15)** — 18 verified improvements including dependency inversion fix, immutability guarantees, sparse post-processing correction, Matryoshka continuity.
- **2 garden entries** — BGE-M3 sparse post-processing gotcha (GE-20260630-db5dce), Java record array immutability gotcha (GE-20260630-73d3c2).

## Cross-Repo: neural-text

Branch `issue-30-bge-m3-multi-modal` has 7 commits (not yet pushed to origin). Changes span inference-api, inference-inmem, inference-runtime, inference-bge-m3 (new), rag, inference-tasks. Full neural-text build green (23 modules).

**Action needed:** push neural-text branch and open PR or merge to main.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | BGE-M3 ONNX export script | M | Med | Custom three-head export from HuggingFace; prerequisite for end-to-end |
| — | Neural-text branch merge | S | Low | Push + merge issue-30-bge-m3-multi-modal to main |
| #33 | Convex Combination fusion test | S | Low | CC α=0.5 vs RRF — config change only |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | Storage optimization after baseline established |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Known Issues

- BGE-M3 three-head ONNX model does not exist yet — all tests use InMemoryInferenceModel stubs
- Neural-text `issue-30-bge-m3-multi-modal` branch is local-only (not pushed to origin)
- `SeparateModelEmbedder` (for casehub deployments not adopting BGE-M3) not yet implemented — file neural-text issue

## Key References

| Resource | Location |
|---|---|
| BGE-M3 design spec | `docs/superpowers/specs/2026-06-30-bge-m3-adoption-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-30-bge-m3-adoption.md` |
| SDD progress ledger | `.superpowers/sdd/progress.md` |
| Retrieval research | `docs/comparison/retrieval-research.md` |
| Engine design | `docs/DESIGN.md` |
