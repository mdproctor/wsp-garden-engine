# Design Journal — issue-36-bge-m3-benchmark

## 2026-07-02 — Benchmark code complete, blocked on neocortex ingestion

### Design decisions

- **Approach B chosen**: new `analyze_bge_m3.py` importing utilities from existing `analyze.py`, separate report generator. Old pipeline untouched — `hybrid-benchmark.md` is a committed artifact for #28.
- **Pipeline-level go/no-go gate**: spec explicitly scoped as a pipeline comparison (not model-vs-model). Signal attribution deferred to #33. Clarified after adversarial design review (3 rounds, 15 issues, $9.25).
- **Export fix**: `torch.onnx.export` needs `external_data=True` (not `use_external_data_format` — renamed in torch 2.12 without deprecation warning). Produces `model.onnx` (graph) + `model.onnx.data` (weights). Atomic move ordering: data first, then graph.
- **`maxSequenceLength` is not a valid config property** — removed from `%dev` properties. Only `model-path` and `tokenizer-path` are recognized by `casehub-inference-quarkus`.
- **`casehub.corpus.corpora.garden.source` must NOT be added** — the engine's `GardenBindingProducer` is the intended corpus binding. Adding the config-driven property creates a duplicate binding via neocortex's `CorpusBindingProducer`.

### Blocker

neocortex `CorpusIngestionService` fullScan produces zero chunks in dev mode (casehubio/neocortex#67). Cursor is saved but nothing reaches Qdrant. No errors logged. All benchmark code is complete — only the actual run is blocked.
