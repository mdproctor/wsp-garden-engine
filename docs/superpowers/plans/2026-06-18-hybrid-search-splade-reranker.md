# Hybrid Search — SPLADE + Cross-Encoder Reranker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable hybrid dense+sparse retrieval with cross-encoder reranking in the Hortora garden search engine by providing `SparseEmbedder` and `CrossEncoderReranker` CDI beans via the `casehub-inference-quarkus` extension.

**Architecture:** The engine adds `casehub-inference-quarkus` for ONNX model lifecycle and native image metadata. A bridge producer (`HybridSearchProducer`) wraps `@Inference`-qualified `InferenceModel` beans into `SparseEmbedder` / `CrossEncoderReranker`, conditionally registered via `@LookupIfProperty`. A `CollectionMigration` startup observer detects dense-only Qdrant collections and triggers a full re-index when SPLADE is newly enabled.

**Tech Stack:** Quarkus 3.36.x, casehub-inference-quarkus (ONNX Runtime JVM), casehub-rag (Qdrant hybrid search), CDI `@LookupIfProperty` with `StringValueMatch.REGEX`

**Spec:** `docs/superpowers/specs/2026-06-18-hybrid-search-splade-reranker.md`
**Issue:** Hortora/engine#9

---

## File Structure

| File | Responsibility |
|------|---------------|
| `pom.xml` | Add `casehub-inference-quarkus` (compile) and `casehub-inference-inmem` (test) |
| `src/main/java/io/hortora/garden/inference/HybridSearchProducer.java` | CDI bridge: `@Inference InferenceModel` → `SparseEmbedder` / `CrossEncoderReranker` |
| `src/main/java/io/hortora/garden/inference/CollectionMigration.java` | Startup observer: detect dense-only collection, delete + reset cursor |
| `src/test/java/io/hortora/garden/inference/HybridSearchProducerTest.java` | Verify conditional bean registration |
| `src/test/java/io/hortora/garden/inference/CollectionMigrationTest.java` | Verify migration decision matrix |
| `src/main/resources/application.properties` | Remove `rerank-enabled=false` override |
| `src/test/resources/application.properties` | Remove `rerank-enabled=false` override |

---

### Task 1: Add Dependencies

**Files:**
- Modify: `pom.xml`

- [ ] **Step 1: Add casehub-inference-quarkus compile dependency**

In `pom.xml`, add after the `casehub-rag` dependency (line ~93):

```xml
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-inference-quarkus</artifactId>
            <version>0.2-SNAPSHOT</version>
        </dependency>
```

- [ ] **Step 2: Add casehub-inference-inmem test dependency**

In `pom.xml`, add after the `casehub-rag-testing` dependency (line ~144):

```xml
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-inference-inmem</artifactId>
            <version>0.2-SNAPSHOT</version>
            <scope>test</scope>
        </dependency>
```

- [ ] **Step 3: Remove rerank-enabled overrides**

In `src/main/resources/application.properties`, remove the line:
```
casehub.rag.retrieval.rerank-enabled=false
```

In `src/test/resources/application.properties`, remove the line:
```
casehub.rag.retrieval.rerank-enabled=false
```

`RagConfig.RetrievalConfig.rerankEnabled()` defaults to `true`. The override was suppressing it. Reranking is inert without a `CrossEncoderReranker` bean, so removing the override is safe.

- [ ] **Step 4: Verify existing tests still pass**

Run: `./mvnw verify -pl .`
Expected: All existing tests pass. Adding inference-quarkus to the classpath doesn't break anything because `InferenceModelConfig.models()` resolves to an empty map when no properties are set, and `@LookupIfProperty` conditions (on beans we haven't created yet) prevent any inference beans from being resolvable.

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/resources/application.properties src/test/resources/application.properties
git commit -m "chore: add inference-quarkus dep, remove rerank-enabled override  Refs #9"
```

---

### Task 2: HybridSearchProducer — TDD

**Files:**
- Create: `src/test/java/io/hortora/garden/inference/HybridSearchProducerTest.java`
- Create: `src/main/java/io/hortora/garden/inference/HybridSearchProducer.java`

- [ ] **Step 1: Write failing test — SparseEmbedder resolvable when config present**

Create `src/test/java/io/hortora/garden/inference/HybridSearchProducerTest.java`:

```java
package io.hortora.garden.inference;

import io.casehub.inference.splade.SparseEmbedder;
import io.casehub.inference.tasks.CrossEncoderReranker;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(HybridSearchProducerTest.WithModelsProfile.class)
class HybridSearchProducerTest {

    @Inject
    Instance<SparseEmbedder> sparseEmbedderInstance;

    @Inject
    Instance<CrossEncoderReranker> rerankerInstance;

    @Test
    void sparseEmbedderIsResolvableWhenConfigured() {
        assertThat(sparseEmbedderInstance.isResolvable()).isTrue();
        assertThat(sparseEmbedderInstance.get()).isNotNull();
    }

    @Test
    void rerankerIsResolvableWhenConfigured() {
        assertThat(rerankerInstance.isResolvable()).isTrue();
        assertThat(rerankerInstance.get()).isNotNull();
    }

    public static class WithModelsProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                "casehub.inference.models.splade.model-path", "stub",
                "casehub.inference.models.splade.tokenizer-path", "stub",
                "casehub.inference.models.reranker.model-path", "stub",
                "casehub.inference.models.reranker.tokenizer-path", "stub"
            );
        }
    }
}
```

Note: `"stub"` values will be intercepted by inference-quarkus's `InferenceModelProducer`. Since `InMemoryInferenceModel` stubs from `casehub-inference-inmem` are `@Alternative` beans, they displace the ONNX producer in test mode. If inference-quarkus doesn't support `@Alternative` override of its producer, we'll need a `@Mock` `InferenceModel` bean instead — TDD will reveal this immediately.

- [ ] **Step 2: Run the test to verify it fails**

Run: `./mvnw test -pl . -Dtest=HybridSearchProducerTest`
Expected: FAIL — `HybridSearchProducer` class does not exist yet.

- [ ] **Step 3: Write the HybridSearchProducer implementation**

Create `src/main/java/io/hortora/garden/inference/HybridSearchProducer.java`:

```java
package io.hortora.garden.inference;

import io.casehub.inference.InferenceModel;
import io.casehub.inference.quarkus.Inference;
import io.casehub.inference.splade.SparseEmbedder;
import io.casehub.inference.tasks.CrossEncoderReranker;
import io.quarkus.arc.lookup.LookupIfProperty;
import io.quarkus.arc.properties.StringValueMatch;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Singleton;

@ApplicationScoped
public class HybridSearchProducer {

    @Produces
    @Singleton
    @LookupIfProperty(name = "casehub.inference.models.splade.model-path",
                       stringValue = ".+", match = StringValueMatch.REGEX)
    SparseEmbedder sparseEmbedder(@Inference("splade") InferenceModel spladeModel) {
        return new SparseEmbedder(spladeModel);
    }

    @Produces
    @Singleton
    @LookupIfProperty(name = "casehub.inference.models.reranker.model-path",
                       stringValue = ".+", match = StringValueMatch.REGEX)
    CrossEncoderReranker reranker(@Inference("reranker") InferenceModel rerankerModel) {
        return new CrossEncoderReranker(rerankerModel);
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./mvnw test -pl . -Dtest=HybridSearchProducerTest`
Expected: PASS — both beans resolvable with config present.

If the test fails because inference-quarkus's `InferenceModelProducer` tries to load a real ONNX model from path `"stub"`, we need to provide a `@Mock` `InferenceModel` that displaces the producer in test:

```java
// Add to HybridSearchProducerTest.java as inner class if needed:
@io.quarkus.test.Mock
@ApplicationScoped
public static class StubInferenceModelProducer {
    @Produces
    @Inference("splade")
    InferenceModel spladeModel() {
        return InMemoryInferenceModel.returning(0.1f, 0.2f, 0.3f);
    }

    @Produces
    @Inference("reranker")
    InferenceModel rerankerModel() {
        return InMemoryInferenceModel.returning(0.5f);
    }
}
```

Adjust based on what TDD reveals about inference-quarkus's test-time behavior.

- [ ] **Step 5: Write test — beans NOT resolvable when config absent**

Add a second test class that runs WITHOUT model config. Create `src/test/java/io/hortora/garden/inference/HybridSearchProducerAbsentTest.java`:

```java
package io.hortora.garden.inference;

import io.casehub.inference.splade.SparseEmbedder;
import io.casehub.inference.tasks.CrossEncoderReranker;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class HybridSearchProducerAbsentTest {

    @Inject
    Instance<SparseEmbedder> sparseEmbedderInstance;

    @Inject
    Instance<CrossEncoderReranker> rerankerInstance;

    @Test
    void sparseEmbedderNotResolvableWithoutConfig() {
        assertThat(sparseEmbedderInstance.isResolvable()).isFalse();
    }

    @Test
    void rerankerNotResolvableWithoutConfig() {
        assertThat(rerankerInstance.isResolvable()).isFalse();
    }
}
```

- [ ] **Step 6: Run to verify the absent-config test passes**

Run: `./mvnw test -pl . -Dtest=HybridSearchProducerAbsentTest`
Expected: PASS — beans are not resolvable when no model path properties are set.

- [ ] **Step 7: Run all tests**

Run: `./mvnw verify -pl .`
Expected: All tests pass (existing + new).

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/hortora/garden/inference/HybridSearchProducer.java \
        src/test/java/io/hortora/garden/inference/HybridSearchProducerTest.java \
        src/test/java/io/hortora/garden/inference/HybridSearchProducerAbsentTest.java
git commit -m "feat: HybridSearchProducer — bridge @Inference models to SparseEmbedder/CrossEncoderReranker  Refs #9"
```

---

### Task 3: CollectionMigration — TDD

**Files:**
- Create: `src/test/java/io/hortora/garden/inference/CollectionMigrationTest.java`
- Create: `src/main/java/io/hortora/garden/inference/CollectionMigration.java`

- [ ] **Step 1: Write failing test — migration triggers when collection lacks sparse vectors**

Create `src/test/java/io/hortora/garden/inference/CollectionMigrationTest.java`:

```java
package io.hortora.garden.inference;

import io.casehub.inference.splade.SparseEmbedder;
import io.casehub.rag.CorpusRef;
import io.casehub.rag.CursorStore;
import io.casehub.rag.EmbeddingIngestor;
import io.casehub.rag.runtime.RagConfig;
import io.casehub.rag.runtime.TenancyStrategy;
import io.hortora.garden.config.GardenConfig;
import io.qdrant.client.QdrantClient;
import io.qdrant.client.grpc.Collections.CollectionInfo;
import io.qdrant.client.grpc.Collections.CollectionConfig;
import io.qdrant.client.grpc.Collections.CollectionParams;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.util.concurrent.CompletableFuture;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class CollectionMigrationTest {

    private QdrantClient qdrantClient;
    private EmbeddingIngestor embeddingIngestor;
    private CursorStore cursorStore;
    private GardenConfig gardenConfig;
    private RagConfig ragConfig;
    private Instance<SparseEmbedder> sparseEmbedderInstance;
    private CollectionMigration migration;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        qdrantClient = mock(QdrantClient.class);
        embeddingIngestor = mock(EmbeddingIngestor.class);
        cursorStore = mock(CursorStore.class);
        gardenConfig = mock(GardenConfig.class);
        ragConfig = mock(RagConfig.class);
        sparseEmbedderInstance = mock(Instance.class);

        when(gardenConfig.id()).thenReturn("garden");
        when(ragConfig.tenancyStrategy()).thenReturn(TenancyStrategy.SEPARATE_COLLECTIONS);

        migration = new CollectionMigration(
            sparseEmbedderInstance, qdrantClient, embeddingIngestor,
            cursorStore, gardenConfig, ragConfig);
    }

    @Test
    void migratesWhenCollectionExistsWithoutSparseVectors() throws Exception {
        when(sparseEmbedderInstance.isResolvable()).thenReturn(true);
        when(qdrantClient.collectionExistsAsync("hortora_garden"))
            .thenReturn(CompletableFuture.completedFuture(true));

        CollectionParams params = CollectionParams.newBuilder().build();
        CollectionConfig config = CollectionConfig.newBuilder().setParams(params).build();
        CollectionInfo info = CollectionInfo.newBuilder().setConfig(config).build();
        when(qdrantClient.getCollectionInfoAsync("hortora_garden"))
            .thenReturn(CompletableFuture.completedFuture(info));

        migration.onStartup(null);

        verify(embeddingIngestor).deleteCorpus(eq(new CorpusRef("hortora", "garden")));
        verify(cursorStore).save("garden", "");
    }

    @Test
    void noOpWhenCollectionAlreadyHasSparseVectors() throws Exception {
        when(sparseEmbedderInstance.isResolvable()).thenReturn(true);
        when(qdrantClient.collectionExistsAsync("hortora_garden"))
            .thenReturn(CompletableFuture.completedFuture(true));

        CollectionParams params = CollectionParams.newBuilder()
            .setSparseVectorsConfig(
                io.qdrant.client.grpc.Collections.SparseVectorConfig.newBuilder()
                    .putMap("sparse", io.qdrant.client.grpc.Collections.SparseVectorParams.getDefaultInstance())
                    .build())
            .build();
        CollectionConfig config = CollectionConfig.newBuilder().setParams(params).build();
        CollectionInfo info = CollectionInfo.newBuilder().setConfig(config).build();
        when(qdrantClient.getCollectionInfoAsync("hortora_garden"))
            .thenReturn(CompletableFuture.completedFuture(info));

        migration.onStartup(null);

        verify(embeddingIngestor, never()).deleteCorpus(any());
        verify(cursorStore, never()).save(any(), any());
    }

    @Test
    void noOpWhenCollectionDoesNotExist() throws Exception {
        when(sparseEmbedderInstance.isResolvable()).thenReturn(true);
        when(qdrantClient.collectionExistsAsync("hortora_garden"))
            .thenReturn(CompletableFuture.completedFuture(false));

        migration.onStartup(null);

        verify(qdrantClient, never()).getCollectionInfoAsync(any());
        verify(embeddingIngestor, never()).deleteCorpus(any());
    }

    @Test
    void noOpWhenSparseEmbedderNotResolvable() {
        when(sparseEmbedderInstance.isResolvable()).thenReturn(false);

        migration.onStartup(null);

        verifyNoInteractions(qdrantClient, embeddingIngestor, cursorStore);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./mvnw test -pl . -Dtest=CollectionMigrationTest`
Expected: FAIL — `CollectionMigration` class does not exist yet.

- [ ] **Step 3: Write the CollectionMigration implementation**

Create `src/main/java/io/hortora/garden/inference/CollectionMigration.java`:

```java
package io.hortora.garden.inference;

import io.casehub.inference.splade.SparseEmbedder;
import io.casehub.rag.CorpusRef;
import io.casehub.rag.CursorStore;
import io.casehub.rag.EmbeddingIngestor;
import io.casehub.rag.runtime.RagConfig;
import io.hortora.garden.config.GardenConfig;
import io.qdrant.client.QdrantClient;
import io.qdrant.client.grpc.Collections.CollectionInfo;
import io.qdrant.client.grpc.Collections.CollectionParams;
import io.quarkus.runtime.StartupEvent;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.concurrent.ExecutionException;
import java.util.logging.Level;
import java.util.logging.Logger;

@ApplicationScoped
public class CollectionMigration {

    private static final Logger LOG = Logger.getLogger(CollectionMigration.class.getName());

    private final Instance<SparseEmbedder> sparseEmbedderInstance;
    private final QdrantClient qdrantClient;
    private final EmbeddingIngestor embeddingIngestor;
    private final CursorStore cursorStore;
    private final GardenConfig gardenConfig;
    private final RagConfig ragConfig;

    @Inject
    public CollectionMigration(
            Instance<SparseEmbedder> sparseEmbedderInstance,
            QdrantClient qdrantClient,
            EmbeddingIngestor embeddingIngestor,
            CursorStore cursorStore,
            GardenConfig gardenConfig,
            RagConfig ragConfig) {
        this.sparseEmbedderInstance = sparseEmbedderInstance;
        this.qdrantClient = qdrantClient;
        this.embeddingIngestor = embeddingIngestor;
        this.cursorStore = cursorStore;
        this.gardenConfig = gardenConfig;
        this.ragConfig = ragConfig;
    }

    void onStartup(@Observes @Priority(10) StartupEvent event) {
        if (!sparseEmbedderInstance.isResolvable()) {
            return;
        }

        CorpusRef corpusRef = new CorpusRef("hortora", gardenConfig.id());
        String collectionName = ragConfig.tenancyStrategy().collectionName(corpusRef);

        try {
            if (!qdrantClient.collectionExistsAsync(collectionName).get()) {
                return;
            }

            CollectionInfo info = qdrantClient.getCollectionInfoAsync(collectionName).get();
            CollectionParams params = info.getConfig().getParams();

            if (params.hasSparseVectorsConfig()) {
                LOG.info(() -> "Collection '" + collectionName + "' already has sparse vectors — no migration needed");
                return;
            }

            LOG.info(() -> "Collection '" + collectionName + "' lacks sparse vectors — migrating to hybrid");
            embeddingIngestor.deleteCorpus(corpusRef);
            cursorStore.save(gardenConfig.id(), "");
            LOG.info(() -> "Migration complete — collection deleted and cursor reset. Full re-index will run on next ingestion cycle.");

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            LOG.log(Level.WARNING, "Interrupted during collection migration check", e);
        } catch (ExecutionException e) {
            LOG.log(Level.WARNING, "Failed to check collection for migration", e.getCause());
        }
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./mvnw test -pl . -Dtest=CollectionMigrationTest`
Expected: PASS — all four migration matrix cases verified.

- [ ] **Step 5: Run all tests**

Run: `./mvnw verify -pl .`
Expected: All tests pass (existing + new from Task 2 + Task 3).

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/hortora/garden/inference/CollectionMigration.java \
        src/test/java/io/hortora/garden/inference/CollectionMigrationTest.java
git commit -m "feat: CollectionMigration — detect dense-only collections and trigger re-index  Refs #9"
```

---

### Task 4: Final Verification and Close

**Files:**
- No new files

- [ ] **Step 1: Run the full build**

Run: `./mvnw verify -pl .`
Expected: All tests pass. Build succeeds.

- [ ] **Step 2: Verify existing tests haven't regressed**

Run: `./mvnw test -pl . -Dtest="SearchResourceTest,FederationIntegrationTest,ChainWalkerTest,FederationConfigParserTest,GardenMetadataExtractorTest"`
Expected: All pass. These tests use `@Mock` stubs that bypass the entire rag/inference layer.

- [ ] **Step 3: Code review**

Invoke `superpowers:requesting-code-review` before committing the final state.

- [ ] **Step 4: Implementation doc sync**

Invoke `implementation-doc-sync` to update any docs affected by the changes.
