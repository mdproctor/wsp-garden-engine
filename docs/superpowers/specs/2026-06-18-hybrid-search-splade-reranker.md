# Hybrid Search — SPLADE Sparse Embeddings + Cross-Encoder Reranker

**Issue:** Hortora/engine#9
**Date:** 2026-06-18
**Status:** Approved

---

## Overview

Phase 2 of the Hortora engine search pipeline. The engine already delegates all Qdrant interaction to `casehub-rag` (neural-text). The rag module's `RagBeanProducer` uses `Instance<SparseEmbedder>` and `Instance<CrossEncoderReranker>` — optional CDI injection. When these beans are resolvable, hybrid dense+sparse retrieval with RRF fusion and cross-encoder reranking activates automatically.

This spec adds the ONNX inference layer to the engine so it can provide those beans.

## Approach

**Engine-owned CDI producer with OnnxInferenceModel.** Add `casehub-inference-runtime` (ONNX Runtime JVM + DJL HuggingFace Tokenizers JNI). Write a producer that loads ONNX models from operator-configured paths, wraps them in `SparseEmbedder` and `CrossEncoderReranker`, and produces them as CDI beans. casehub-rag auto-detects them.

Alternatives rejected:
- **inference-quarkus extension** — heavier dependency, still needs wrapping code, native image unconfirmed. No architectural benefit over direct wiring.
- **External inference service** — network hop and operational complexity for no gain at garden corpus scale (hundreds of entries).

## Dependencies

| Artifact | Scope | Why |
|----------|-------|-----|
| `casehub-inference-runtime` | compile | `OnnxInferenceModel`, `ModelConfig` — ONNX model loading |
| `casehub-inference-splade` | compile | `SparseEmbedder` — make explicit (transitive via rag) |
| `casehub-inference-tasks` | compile | `CrossEncoderReranker` — make explicit (transitive via rag) |
| `casehub-inference-inmem` | test | `InMemoryInferenceModel` — deterministic stubs, no JNI |

## Configuration

New config interface `InferenceConfig` at `io.hortora.garden.inference`:

```java
@ConfigMapping(prefix = "hortora.inference")
public interface InferenceConfig {
    Optional<SpladeConfig> splade();
    Optional<RerankerConfig> reranker();

    interface SpladeConfig {
        Path modelPath();
        Path tokenizerPath();
        @WithDefault("256")
        int maxSequenceLength();
    }

    interface RerankerConfig {
        Path modelPath();
        Path tokenizerPath();
        @WithDefault("512")
        int maxSequenceLength();
    }
}
```

Both are `Optional<>`. When no model path is configured, that leg is absent — the engine starts dense-only (current behaviour). Existing deployments keep working until the operator downloads models and sets the config.

### Model Selection

- **SPLADE:** `prithivida/Splade_PP_en_v1` — permissive license, equivalent MRR@10 to the naver model which is CC NonCommercial (GE-20260614-b94048)
- **Cross-encoder:** `cross-encoder/ms-marco-MiniLM-L-6-v2` — standard choice, fast, 22M params

Both require ONNX export as an operator setup step (not baked into the app).

### Existing Config Changes

```properties
# application.properties (production)
casehub.rag.retrieval.rerank-enabled=true   # flip from false

# application.properties (test) — unchanged
casehub.rag.retrieval.rerank-enabled=false  # stays false in test
```

The `casehub.rag.retrieval.*` config (denseTopK, sparseTopK, rrfK, rerankTopN) remains at defaults. `rerank-enabled=true` is inert without a `CrossEncoderReranker` bean.

## CDI Producer

`InferenceModelProducer` at `io.hortora.garden.inference`:

- `@ApplicationScoped` with `@Observes StartupEvent` for eager model loading
- Creates `OnnxInferenceModel` instances from config when present
- Wraps in `SparseEmbedder` / `CrossEncoderReranker`
- `@Produces @Singleton` methods return the instance or null
- `@Observes ShutdownEvent` closes all `OnnxInferenceModel` instances

**CDI null-bean behaviour:** `RagBeanProducer` calls `Instance<SparseEmbedder>.isResolvable()` — returns true when a producer exists (even if it returns null). Then `.get()` returns null. The null propagates through `RagBeanProducer` into `HybridCaseRetriever` which checks `this.sparseEmbedder != null`. Dense-only fallback is functionally correct.

## Collection Migration

Existing dense-only Qdrant collections lack a sparse vector space. `QdrantEmbeddingIngestor.ensureCollection()` creates new collections with the right schema but doesn't alter existing ones.

Strategy: force a full re-index on first hybrid-enabled startup.

`CollectionMigration` — a `@Startup` observer that runs before `CorpusIngestionService.onStart()`:
1. Checks whether `SparseEmbedder` is newly present
2. Deletes the existing collection via `EmbeddingIngestor.deleteCorpus(corpusRef)`
3. Resets the ingestion cursor via `CursorStore.save(bindingName, "")` — `FileCursorStore.load()` treats empty content as `Optional.empty()`, which causes `CorpusIngestionService.doProcessBinding()` to bootstrap with `fullScan()` and re-ingest all entries with both dense + sparse vectors

One-time migration. After re-index, incremental changes flow through the normal pipeline — which already writes both dense and sparse embeddings when `SparseEmbedder` is present.

No backward-compat concern — single-deployment service. Removing SPLADE config later falls back to dense-only gracefully (sparse vectors in Qdrant are ignored by the dense-only query path).

## Testing

**InferenceModelProducer test** — verifies:
- Non-null `SparseEmbedder` when SPLADE config is set
- Null `SparseEmbedder` when SPLADE config is absent
- Same for `CrossEncoderReranker`
- Models closed on shutdown

Uses `casehub-inference-inmem` (`InMemoryInferenceModel`) — deterministic stubs, no ONNX JNI, safe in `@QuarkusTest`.

**Existing tests unchanged** — `SearchResourceTest`, `FederationIntegrationTest`, etc. continue using `TestCaseRetriever` / `TestEmbeddingIngestor` stubs. These test engine search/federation logic, not the retrieval pipeline.

**No integration test with real ONNX models in CI** — models are ~400MB+ and require JNI. CI runs without them. Producer gracefully returns null without config. Real hybrid search verified manually in dev mode.

## File Inventory

| File | Action |
|------|--------|
| `pom.xml` | Add 4 deps (inference-runtime, inference-splade, inference-tasks compile; inference-inmem test) |
| `src/main/java/io/hortora/garden/inference/InferenceConfig.java` | New — config mapping |
| `src/main/java/io/hortora/garden/inference/InferenceModelProducer.java` | New — CDI producer + lifecycle |
| `src/main/java/io/hortora/garden/inference/CollectionMigration.java` | New — one-time re-index on hybrid enable |
| `src/test/java/io/hortora/garden/inference/InferenceModelProducerTest.java` | New — producer wiring test |
| `src/main/resources/application.properties` | Flip `rerank-enabled` to true |

## Garden Entries Referenced

- **GE-20260614-b94048** — SPLADE model licensing: use `Splade_PP_en_v1`, not naver (CC NonCommercial)
- **GE-20260609-2abdfd** — Qdrant client ≥1.10.0 required for Query API (already satisfied by casehub-rag)
- **GE-20260609-c1998e** — `QueryFactory.rrf()` vs `fusion()` for configurable k (already used by `HybridCaseRetriever`)
