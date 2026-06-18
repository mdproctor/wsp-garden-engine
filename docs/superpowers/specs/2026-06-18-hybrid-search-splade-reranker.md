# Hybrid Search — SPLADE Sparse Embeddings + Cross-Encoder Reranker

**Issue:** Hortora/engine#9
**Date:** 2026-06-18
**Status:** Approved (revised after review)

---

## Overview

Phase 2 of the Hortora engine search pipeline. The engine already delegates all Qdrant interaction to `casehub-rag` (neural-text). The rag module's `RagBeanProducer` uses `Instance<SparseEmbedder>` and `Instance<CrossEncoderReranker>` — optional CDI injection. When these beans are resolvable, hybrid dense+sparse retrieval with RRF fusion and cross-encoder reranking activates automatically.

This spec adds the ONNX inference layer to the engine so it can provide those beans.

## Approach

**Use `casehub-inference-quarkus` + engine-owned bridge producer.** The inference-quarkus Quarkus extension handles ONNX model loading, config mapping, lifecycle management (ConcurrentHashMap pool + ShutdownEvent cleanup), and native image metadata (1578 lines of reachability-metadata.json + `--initialize-at-run-time` flags). The engine adds a bridge producer that injects `@Inference`-qualified `InferenceModel` beans and wraps them in `SparseEmbedder` / `CrossEncoderReranker`.

Alternatives rejected:
- **Engine-owned OnnxInferenceModel producer** — duplicates inference-quarkus's ONNX lifecycle, config mapping, and native image metadata. The engine would own JNI native image metadata forever — a maintenance burden that should live once in the platform.
- **External inference service** — network hop and operational complexity for no gain at garden corpus scale (hundreds of entries).

## Dependencies

| Artifact | Scope | Why |
|----------|-------|-----|
| `casehub-inference-quarkus` | compile | ONNX CDI wiring, config, native image metadata. Transitively pulls `inference-runtime`, `inference-api`, `inference-splade`, `inference-tasks`. |
| `casehub-inference-inmem` | test | `InMemoryInferenceModel` — deterministic stubs, no JNI |

## Configuration

inference-quarkus provides `InferenceModelConfig` at `casehub.inference.models.<name>.*`:

```properties
# SPLADE sparse embedder
casehub.inference.models.splade.model-path=/path/to/splade/model.onnx
casehub.inference.models.splade.tokenizer-path=/path/to/splade/tokenizer.json
casehub.inference.models.splade.max-sequence-length=256

# Cross-encoder reranker
casehub.inference.models.reranker.model-path=/path/to/reranker/model.onnx
casehub.inference.models.reranker.tokenizer-path=/path/to/reranker/tokenizer.json
casehub.inference.models.reranker.max-sequence-length=512
```

Both are optional — when no model path is configured, the engine starts dense-only (current behaviour). Threading config (`intra-op-threads`, `inter-op-threads`) is also available via inference-quarkus for production tuning.

### Model Selection

- **SPLADE:** `prithivida/Splade_PP_en_v1` — permissive license, equivalent MRR@10 to the naver model which is CC NonCommercial (GE-20260614-b94048)
- **Cross-encoder:** `cross-encoder/ms-marco-MiniLM-L-6-v2` — standard choice, fast, 22M params

Both require ONNX export as an operator setup step (not baked into the app).

### Existing Config Changes

```properties
# application.properties (production) — REMOVE the rerank-enabled=false override.
# RagConfig defaults rerankEnabled to true. The override was suppressing the default;
# removing it lets the default stand. Reranking is inert without a CrossEncoderReranker bean.

# application.properties (test) — REMOVE the rerank-enabled=false override.
# Tests use @Mock stubs (TestCaseRetriever, TestEmbeddingIngestor) which bypass
# HybridCaseRetriever entirely. The config value has no effect in test.
```

## CDI Bridge Producer

`HybridSearchProducer` at `io.hortora.garden.inference`:

```java
@ApplicationScoped
public class HybridSearchProducer {

    @Inject
    InferenceModelConfig config;

    @Produces
    @Singleton
    @LookupIfProperty(name = "casehub.inference.models.splade.model-path", stringValue = "", lookupIfMissing = false)
    SparseEmbedder sparseEmbedder(@Inference("splade") InferenceModel spladeModel) {
        return new SparseEmbedder(spladeModel);
    }

    @Produces
    @Singleton
    @LookupIfProperty(name = "casehub.inference.models.reranker.model-path", stringValue = "", lookupIfMissing = false)
    CrossEncoderReranker reranker(@Inference("reranker") InferenceModel rerankerModel) {
        return new CrossEncoderReranker(rerankerModel);
    }
}
```

**CDI conditional registration:** `@LookupIfProperty` (Quarkus ArC) makes the bean genuinely non-resolvable when the config property is absent. `Instance<SparseEmbedder>.isResolvable()` returns `false` when SPLADE is not configured — a clean signal to `RagBeanProducer`. No null beans.

**Note:** `@LookupIfProperty` with `lookupIfMissing = false` means the bean is resolvable only when the property EXISTS (any value). The `stringValue = ""` with `lookupIfMissing = false` effectively means "property must be present" — the empty-string match is the default/fallback behavior. Verify this during TDD; if the semantics don't match, use a boolean enablement property (`hortora.inference.splade.enabled=true`) instead.

**Lifecycle:** Model loading and cleanup are handled by `InferenceModelProducer` in inference-quarkus (ConcurrentHashMap pool + ShutdownEvent). The engine's bridge producer owns no lifecycle.

## Collection Migration

Existing dense-only Qdrant collections lack a sparse vector space. `QdrantEmbeddingIngestor.ensureCollection()` creates new collections with the right schema but doesn't alter existing ones.

Strategy: force a full re-index on first hybrid-enabled startup.

### Detection Mechanism

`CollectionMigration` queries the Qdrant collection schema via `QdrantClient.getCollectionInfoAsync(collectionName)`. The response's `CollectionInfo` → `getConfig()` → `getParams()` → `getSparseVectorsConfig()` → `getMapMap()` tells whether the collection has a sparse vector space. If the collection exists but the sparse vectors map is empty, and `SparseEmbedder` is resolvable in CDI, migration is needed.

This is stateless detection — no marker files, no persistent flags. The collection schema IS the state.

### Migration Steps

`CollectionMigration` — observes `StartupEvent` with `@Priority(10)` (before `CorpusIngestionService.onStart()` which has default priority):

1. Checks `Instance<SparseEmbedder>.isResolvable()` — if false, no-op
2. Checks whether collection exists and lacks sparse vector space (schema introspection)
3. If migration needed: deletes the collection via `EmbeddingIngestor.deleteCorpus(corpusRef)` and resets the ingestion cursor via `CursorStore.save(bindingName, "")` — `FileCursorStore.load()` treats empty content as `Optional.empty()`, triggering `fullScan()` on next poll

The cursor reset via empty-string save is a workaround that couples to `FileCursorStore`'s implementation. `CursorStore.delete()` is the right API — filed as casehubio/neural-text#38. Use the workaround until that ships.

### Migration Matrix

| Collection exists? | Has sparse vectors? | SparseEmbedder resolvable? | Action |
|----|----|----|--------|
| No | — | Yes | No-op — `ensureCollection()` creates with sparse on first ingest |
| Yes | Yes | Yes | No-op — already hybrid |
| Yes | No | Yes | **Migrate** — delete collection, reset cursor |
| — | — | No | No-op — dense-only mode |

One-time migration. After re-index, incremental changes flow through the normal pipeline.

## Testing

**HybridSearchProducer test** — verifies:
- `SparseEmbedder` is resolvable and non-null when SPLADE config is set
- `SparseEmbedder` is NOT resolvable (via `Instance<>.isResolvable()`) when SPLADE config is absent
- Same for `CrossEncoderReranker`

Uses `casehub-inference-inmem` (`InMemoryInferenceModel`) — deterministic stubs, no ONNX JNI, safe in `@QuarkusTest`.

**CollectionMigration test** — verifies the migration matrix:
- Collection without sparse vectors + SparseEmbedder resolvable → deletes collection, resets cursor
- Collection with sparse vectors + SparseEmbedder resolvable → no-op
- Collection doesn't exist + SparseEmbedder resolvable → no-op
- SparseEmbedder not resolvable → no-op regardless of collection state

Uses mock `QdrantClient` / `EmbeddingIngestor` / `CursorStore` to verify the decision logic without a real Qdrant instance.

**Existing tests unchanged** — `SearchResourceTest`, `FederationIntegrationTest`, etc. continue using `TestCaseRetriever` / `TestEmbeddingIngestor` stubs. These test engine search/federation logic, not the retrieval pipeline.

**No integration test with real ONNX models in CI** — models are ~400MB+ and require JNI. CI runs without them. Beans are genuinely non-resolvable without config (no null fallback). Real hybrid search verified manually in dev mode.

## File Inventory

| File | Action |
|------|--------|
| `pom.xml` | Add 2 deps (`casehub-inference-quarkus` compile, `casehub-inference-inmem` test) |
| `src/main/java/io/hortora/garden/inference/HybridSearchProducer.java` | New — CDI bridge producer |
| `src/main/java/io/hortora/garden/inference/CollectionMigration.java` | New — one-time re-index on hybrid enable |
| `src/test/java/io/hortora/garden/inference/HybridSearchProducerTest.java` | New — producer wiring test |
| `src/test/java/io/hortora/garden/inference/CollectionMigrationTest.java` | New — migration decision logic test |
| `src/main/resources/application.properties` | Remove `rerank-enabled=false` override |
| `src/test/resources/application.properties` | Remove `rerank-enabled=false` override |

## Deferred Concerns

| Issue | Description |
|-------|-------------|
| casehubio/neural-text#38 | `CursorStore.delete()` — clean cursor reset without implementation coupling |

## Garden Entries Referenced

- **GE-20260614-b94048** — SPLADE model licensing: use `Splade_PP_en_v1`, not naver (CC NonCommercial)
- **GE-20260609-2abdfd** — Qdrant client ≥1.10.0 required for Query API (already satisfied by casehub-rag)
- **GE-20260609-c1998e** — `QueryFactory.rrf()` vs `fusion()` for configurable k (already used by `HybridCaseRetriever`)
