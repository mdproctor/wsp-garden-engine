# Embedding Cache Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #83 — Search-time version de-emphasis
**Issue group:** #83, #80

**Goal:** Cache computed `MultiModalEmbedding` results in SQLite so that
reindexing unchanged content skips ONNX inference entirely. Reduces
reindex of 2,590 entries from ~90 min to ~2-3 min.

**Architecture:** CDI `@Decorator` on `MultiModalEmbedder` in a new
`casehub-neocortex-rag-cache` module. Hashes content text, looks up
cached embeddings in SQLite, only delegates to the real embedder on
cache miss. Model version column auto-invalidates on model upgrade.

**Tech Stack:** SQLite (sqlite-jdbc), HikariCP, Flyway, CDI decorators,
SmallRye Config

**Cross-repo:** Tasks 1-3 are in `neocortex`
(`/Users/mdproctor/claude/casehub/neocortex/`). Task 4 is in `engine`
(`/Users/mdproctor/claude/hortora/engine/`).

## Global Constraints

- Java 25, Quarkus 3.36.x
- Package: `io.casehub.neocortex.rag.cache`
- Follow existing patterns from `SqliteRetrievalTracker` (own HikariCP
  pool, Flyway migrations, WAL mode)
- No Quarkus managed datasource — standalone SQLite file
- CDI decorator pattern matches `DedupEmbeddingIngestor`
- `java.util.logging` (not SLF4J) — matches neocortex convention

---

### Task 1: Module scaffolding + EmbeddingSerializer

**Files:**
- Create: `rag-cache/pom.xml`
- Create: `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingSerializer.java`
- Create: `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/EmbeddingSerializerTest.java`
- Modify: `pom.xml` (neocortex parent — add `<module>rag-cache</module>`)

**Interfaces:**
- Consumes: `io.casehub.neocortex.inference.MultiModalEmbedding` (from `inference-api`)
- Produces: `EmbeddingSerializer.serialize(MultiModalEmbedding): byte[]`,
  `EmbeddingSerializer.deserialize(byte[] dense, byte[] sparse, byte[] colbert): MultiModalEmbedding`

- [ ] **Step 1: Create module pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-neocortex-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-neocortex-rag-cache</artifactId>
    <name>casehub-neocortex :: rag-cache</name>
    <description>Embedding cache for MultiModalEmbedder — SQLite-backed,
        skips ONNX inference on reindex for unchanged content.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-neocortex-inference-api</artifactId>
        </dependency>
        <dependency>
            <groupId>jakarta.enterprise</groupId>
            <artifactId>jakarta.enterprise.cdi-api</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>io.smallrye.config</groupId>
            <artifactId>smallrye-config-core</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>rag-cache</module>` after the `rag-tracking` module entry
in `/Users/mdproctor/claude/casehub/neocortex/pom.xml`.

- [ ] **Step 3: Write the failing serializer test**

Create `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/EmbeddingSerializerTest.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.MultiModalEmbedding;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class EmbeddingSerializerTest {

    @Test
    void roundTripDenseOnly() {
        float[] dense = {1.0f, 2.0f, 3.0f};
        var original = new MultiModalEmbedding(dense, null, null);

        byte[] denseBytes = EmbeddingSerializer.serializeDense(original);
        var restored = EmbeddingSerializer.deserialize(denseBytes, null, null);

        assertArrayEquals(dense, restored.dense(), 1e-6f);
        assertNull(restored.sparse());
        assertNull(restored.colbert());
    }

    @Test
    void roundTripFull() {
        float[] dense = {0.1f, 0.2f, 0.3f, 0.4f};
        Map<Integer, Float> sparse = Map.of(5, 1.5f, 100, 0.8f);
        float[][] colbert = {{0.1f, 0.2f}, {0.3f, 0.4f}, {0.5f, 0.6f}};
        var original = new MultiModalEmbedding(dense, sparse, colbert);

        byte[] denseBytes = EmbeddingSerializer.serializeDense(original);
        byte[] sparseBytes = EmbeddingSerializer.serializeSparse(original);
        byte[] colbertBytes = EmbeddingSerializer.serializeColbert(original);

        var restored = EmbeddingSerializer.deserialize(denseBytes, sparseBytes, colbertBytes);

        assertArrayEquals(dense, restored.dense(), 1e-6f);
        assertEquals(sparse.size(), restored.sparse().size());
        assertEquals(1.5f, restored.sparse().get(5), 1e-6f);
        assertEquals(0.8f, restored.sparse().get(100), 1e-6f);
        assertEquals(3, restored.colbert().length);
        assertArrayEquals(new float[]{0.1f, 0.2f}, restored.colbert()[0], 1e-6f);
    }

    @Test
    void emptyDenseArray() {
        float[] dense = {};
        var original = new MultiModalEmbedding(new float[]{0.0f}, null, null);
        byte[] bytes = EmbeddingSerializer.serializeDense(original);
        assertNotNull(bytes);
        assertTrue(bytes.length > 0);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn -f rag-cache/pom.xml test -pl rag-cache -Dtest=EmbeddingSerializerTest -q`
Expected: compilation failure — `EmbeddingSerializer` does not exist.

- [ ] **Step 5: Implement EmbeddingSerializer**

Create `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingSerializer.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.MultiModalEmbedding;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.LinkedHashMap;
import java.util.Map;

public final class EmbeddingSerializer {

    private EmbeddingSerializer() {}

    public static byte[] serializeDense(MultiModalEmbedding embedding) {
        float[] dense = embedding.dense();
        ByteBuffer buf = ByteBuffer.allocate(4 + dense.length * 4)
                .order(ByteOrder.LITTLE_ENDIAN);
        buf.putInt(dense.length);
        for (float f : dense) buf.putFloat(f);
        return buf.array();
    }

    public static byte[] serializeSparse(MultiModalEmbedding embedding) {
        Map<Integer, Float> sparse = embedding.sparse();
        if (sparse == null) return null;
        ByteBuffer buf = ByteBuffer.allocate(4 + sparse.size() * 8)
                .order(ByteOrder.LITTLE_ENDIAN);
        buf.putInt(sparse.size());
        for (var entry : sparse.entrySet()) {
            buf.putInt(entry.getKey());
            buf.putFloat(entry.getValue());
        }
        return buf.array();
    }

    public static byte[] serializeColbert(MultiModalEmbedding embedding) {
        float[][] colbert = embedding.colbert();
        if (colbert == null) return null;
        int rows = colbert.length;
        int cols = rows > 0 ? colbert[0].length : 0;
        ByteBuffer buf = ByteBuffer.allocate(8 + rows * cols * 4)
                .order(ByteOrder.LITTLE_ENDIAN);
        buf.putInt(rows);
        buf.putInt(cols);
        for (float[] row : colbert) {
            for (float f : row) buf.putFloat(f);
        }
        return buf.array();
    }

    public static MultiModalEmbedding deserialize(byte[] denseBytes,
                                                    byte[] sparseBytes,
                                                    byte[] colbertBytes) {
        float[] dense = deserializeDense(denseBytes);
        Map<Integer, Float> sparse = deserializeSparse(sparseBytes);
        float[][] colbert = deserializeColbert(colbertBytes);
        return new MultiModalEmbedding(dense, sparse, colbert);
    }

    private static float[] deserializeDense(byte[] bytes) {
        ByteBuffer buf = ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN);
        int length = buf.getInt();
        float[] dense = new float[length];
        for (int i = 0; i < length; i++) dense[i] = buf.getFloat();
        return dense;
    }

    private static Map<Integer, Float> deserializeSparse(byte[] bytes) {
        if (bytes == null) return null;
        ByteBuffer buf = ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN);
        int size = buf.getInt();
        Map<Integer, Float> sparse = new LinkedHashMap<>(size);
        for (int i = 0; i < size; i++) {
            sparse.put(buf.getInt(), buf.getFloat());
        }
        return sparse;
    }

    private static float[][] deserializeColbert(byte[] bytes) {
        if (bytes == null) return null;
        ByteBuffer buf = ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN);
        int rows = buf.getInt();
        int cols = buf.getInt();
        float[][] colbert = new float[rows][cols];
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                colbert[r][c] = buf.getFloat();
            }
        }
        return colbert;
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn -f rag-cache/pom.xml test -Dtest=EmbeddingSerializerTest -q`
Expected: 3 tests PASS

- [ ] **Step 7: Commit**

```bash
git add rag-cache/ pom.xml
git commit -m "feat: add rag-cache module with EmbeddingSerializer

Refs Hortora/engine#83"
```

---

### Task 2: EmbeddingCache (SQLite store)

**Files:**
- Create: `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingCacheConfig.java`
- Create: `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingCache.java`
- Create: `rag-cache/src/main/resources/db/embedding-cache/migration/V1__create_embedding_cache.sql`
- Create: `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/EmbeddingCacheTest.java`
- Create: `rag-cache/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `EmbeddingSerializer` (from Task 1)
- Produces:
  - `EmbeddingCache.get(String contentHash): Optional<MultiModalEmbedding>`
  - `EmbeddingCache.getBatch(List<String> contentHashes): Map<String, MultiModalEmbedding>`
  - `EmbeddingCache.put(String contentHash, MultiModalEmbedding embedding): void`
  - `EmbeddingCache.putBatch(Map<String, MultiModalEmbedding> entries): void`

- [ ] **Step 1: Create Flyway migration**

Create `rag-cache/src/main/resources/db/embedding-cache/migration/V1__create_embedding_cache.sql`:

```sql
CREATE TABLE embedding_cache (
    content_hash  TEXT    NOT NULL,
    model_version TEXT    NOT NULL,
    dense         BLOB    NOT NULL,
    sparse        BLOB,
    colbert       BLOB,
    created_at    INTEGER NOT NULL DEFAULT (unixepoch()),
    PRIMARY KEY (content_hash, model_version)
);
```

- [ ] **Step 2: Create EmbeddingCacheConfig**

Create `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingCacheConfig.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.rag.embedding-cache")
public interface EmbeddingCacheConfig {
    @WithDefault("false")
    boolean enabled();

    @WithDefault("")
    String path();

    @WithDefault("")
    String versionSuffix();
}
```

- [ ] **Step 3: Write the failing cache test**

Create `rag-cache/src/test/resources/application.properties`:

```properties
casehub.rag.embedding-cache.enabled=true
casehub.rag.embedding-cache.path=:memory:
```

Create `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/EmbeddingCacheTest.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.MultiModalEmbedding;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class EmbeddingCacheTest {

    private EmbeddingCache cache;

    @BeforeEach
    void setUp() {
        cache = new EmbeddingCache(":memory:", "test-model:1024:768:");
        cache.init();
    }

    @AfterEach
    void tearDown() {
        cache.shutdown();
    }

    @Test
    void putAndGet() {
        var embedding = new MultiModalEmbedding(
                new float[]{1.0f, 2.0f}, null, null);
        cache.put("abc123", embedding);

        Optional<MultiModalEmbedding> result = cache.get("abc123");
        assertTrue(result.isPresent());
        assertArrayEquals(new float[]{1.0f, 2.0f}, result.get().dense(), 1e-6f);
    }

    @Test
    void getMissReturnsEmpty() {
        Optional<MultiModalEmbedding> result = cache.get("nonexistent");
        assertTrue(result.isEmpty());
    }

    @Test
    void modelVersionIsolation() {
        var embedding = new MultiModalEmbedding(
                new float[]{1.0f}, null, null);
        cache.put("hash1", embedding);

        var otherCache = new EmbeddingCache(":memory:", "other-model:512:256:");
        otherCache.init();
        assertTrue(otherCache.get("hash1").isEmpty());
        otherCache.shutdown();
    }

    @Test
    void getBatchReturnsOnlyHits() {
        var e1 = new MultiModalEmbedding(new float[]{1.0f}, null, null);
        var e2 = new MultiModalEmbedding(new float[]{2.0f}, null, null);
        cache.put("h1", e1);
        cache.put("h2", e2);

        Map<String, MultiModalEmbedding> result =
                cache.getBatch(List.of("h1", "h2", "h3"));

        assertEquals(2, result.size());
        assertTrue(result.containsKey("h1"));
        assertTrue(result.containsKey("h2"));
        assertFalse(result.containsKey("h3"));
    }

    @Test
    void putBatch() {
        var e1 = new MultiModalEmbedding(new float[]{1.0f}, null, null);
        var e2 = new MultiModalEmbedding(new float[]{2.0f}, null, null);
        cache.putBatch(Map.of("h1", e1, "h2", e2));

        assertTrue(cache.get("h1").isPresent());
        assertTrue(cache.get("h2").isPresent());
    }

    @Test
    void fullRoundTripWithSparseAndColbert() {
        var embedding = new MultiModalEmbedding(
                new float[]{0.1f, 0.2f},
                Map.of(5, 1.5f, 100, 0.8f),
                new float[][]{{0.3f, 0.4f}, {0.5f, 0.6f}});
        cache.put("full", embedding);

        var result = cache.get("full").orElseThrow();
        assertArrayEquals(new float[]{0.1f, 0.2f}, result.dense(), 1e-6f);
        assertEquals(1.5f, result.sparse().get(5), 1e-6f);
        assertArrayEquals(new float[]{0.3f, 0.4f}, result.colbert()[0], 1e-6f);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn -f rag-cache/pom.xml test -Dtest=EmbeddingCacheTest -q`
Expected: compilation failure — `EmbeddingCache` does not exist.

- [ ] **Step 5: Implement EmbeddingCache**

Create `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/EmbeddingCache.java`:

```java
package io.casehub.neocortex.rag.cache;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import io.casehub.neocortex.inference.MultiModalEmbedding;
import org.flywaydb.core.Flyway;
import org.sqlite.SQLiteConfig;
import org.sqlite.SQLiteConfig.JournalMode;
import org.sqlite.SQLiteConfig.SynchronousMode;
import org.sqlite.SQLiteDataSource;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.*;
import java.util.logging.Level;
import java.util.logging.Logger;

public class EmbeddingCache {

    private static final Logger LOG =
            Logger.getLogger(EmbeddingCache.class.getName());

    private final String path;
    private final String modelVersion;
    private HikariDataSource dataSource;

    public EmbeddingCache(String path, String modelVersion) {
        this.path = path;
        this.modelVersion = modelVersion;
    }

    public void init() {
        boolean isMemory = ":memory:".equals(path) || path.isBlank();
        SQLiteConfig sqLiteConfig = new SQLiteConfig();
        if (!isMemory) {
            sqLiteConfig.setJournalMode(JournalMode.WAL);
        }
        sqLiteConfig.setSynchronous(SynchronousMode.NORMAL);
        sqLiteConfig.setBusyTimeout(5000);

        SQLiteDataSource sqLiteDataSource = new SQLiteDataSource(sqLiteConfig);
        sqLiteDataSource.setUrl("jdbc:sqlite:" + path);

        HikariConfig hikari = new HikariConfig();
        hikari.setDataSource(sqLiteDataSource);
        hikari.setMaximumPoolSize(isMemory ? 1 : 3);
        hikari.setMinimumIdle(1);
        dataSource = new HikariDataSource(hikari);

        Flyway.configure()
                .dataSource(dataSource)
                .locations("classpath:db/embedding-cache/migration")
                .load()
                .migrate();
    }

    public void shutdown() {
        if (dataSource != null) {
            dataSource.close();
        }
    }

    public Optional<MultiModalEmbedding> get(String contentHash) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT dense, sparse, colbert FROM embedding_cache "
                     + "WHERE content_hash = ? AND model_version = ?")) {
            ps.setString(1, contentHash);
            ps.setString(2, modelVersion);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(EmbeddingSerializer.deserialize(
                            rs.getBytes("dense"),
                            rs.getBytes("sparse"),
                            rs.getBytes("colbert")));
                }
            }
        } catch (SQLException e) {
            LOG.log(Level.WARNING, "Cache read failed for " + contentHash, e);
        }
        return Optional.empty();
    }

    public Map<String, MultiModalEmbedding> getBatch(List<String> contentHashes) {
        if (contentHashes.isEmpty()) return Map.of();
        Map<String, MultiModalEmbedding> result = new LinkedHashMap<>();
        String placeholders = String.join(",",
                Collections.nCopies(contentHashes.size(), "?"));
        String sql = "SELECT content_hash, dense, sparse, colbert "
                + "FROM embedding_cache WHERE content_hash IN ("
                + placeholders + ") AND model_version = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            int idx = 1;
            for (String hash : contentHashes) {
                ps.setString(idx++, hash);
            }
            ps.setString(idx, modelVersion);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    String hash = rs.getString("content_hash");
                    result.put(hash, EmbeddingSerializer.deserialize(
                            rs.getBytes("dense"),
                            rs.getBytes("sparse"),
                            rs.getBytes("colbert")));
                }
            }
        } catch (SQLException e) {
            LOG.log(Level.WARNING, "Batch cache read failed", e);
        }
        return result;
    }

    public void put(String contentHash, MultiModalEmbedding embedding) {
        String sql = "INSERT OR REPLACE INTO embedding_cache "
                + "(content_hash, model_version, dense, sparse, colbert) "
                + "VALUES (?, ?, ?, ?, ?)";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, contentHash);
            ps.setString(2, modelVersion);
            ps.setBytes(3, EmbeddingSerializer.serializeDense(embedding));
            ps.setBytes(4, EmbeddingSerializer.serializeSparse(embedding));
            ps.setBytes(5, EmbeddingSerializer.serializeColbert(embedding));
            ps.executeUpdate();
        } catch (SQLException e) {
            LOG.log(Level.WARNING, "Cache write failed for " + contentHash, e);
        }
    }

    public void putBatch(Map<String, MultiModalEmbedding> entries) {
        if (entries.isEmpty()) return;
        String sql = "INSERT OR REPLACE INTO embedding_cache "
                + "(content_hash, model_version, dense, sparse, colbert) "
                + "VALUES (?, ?, ?, ?, ?)";
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try (PreparedStatement ps = conn.prepareStatement(sql)) {
                for (var entry : entries.entrySet()) {
                    ps.setString(1, entry.getKey());
                    ps.setString(2, modelVersion);
                    ps.setBytes(3, EmbeddingSerializer.serializeDense(entry.getValue()));
                    ps.setBytes(4, EmbeddingSerializer.serializeSparse(entry.getValue()));
                    ps.setBytes(5, EmbeddingSerializer.serializeColbert(entry.getValue()));
                    ps.addBatch();
                }
                ps.executeBatch();
                conn.commit();
            } catch (Exception e) {
                conn.rollback();
                LOG.log(Level.WARNING, "Batch cache write failed", e);
            } finally {
                conn.setAutoCommit(true);
            }
        } catch (SQLException e) {
            LOG.log(Level.WARNING, "Batch cache write failed (connection)", e);
        }
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn -f rag-cache/pom.xml test -Dtest=EmbeddingCacheTest -q`
Expected: 6 tests PASS

- [ ] **Step 7: Commit**

```bash
git add rag-cache/src/
git commit -m "feat: add EmbeddingCache — SQLite store for cached embeddings

Refs Hortora/engine#83"
```

---

### Task 3: CachingMultiModalEmbedder (CDI decorator)

**Files:**
- Create: `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/CachingMultiModalEmbedder.java`
- Create: `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/CachingEmbedderDecorator.java`
- Create: `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/CachingMultiModalEmbedderTest.java`

**Interfaces:**
- Consumes: `EmbeddingCache` (Task 2), `MultiModalEmbedder` (inference-api),
  `EmbeddingCacheConfig` (Task 2)
- Produces: CDI decorator — transparent to all callers of `MultiModalEmbedder`

- [ ] **Step 1: Write the failing test**

Create `rag-cache/src/test/java/io/casehub/neocortex/rag/cache/CachingMultiModalEmbedderTest.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.EmbeddingMode;
import io.casehub.neocortex.inference.MultiModalEmbedder;
import io.casehub.neocortex.inference.MultiModalEmbedding;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.security.MessageDigest;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class CachingMultiModalEmbedderTest {

    private EmbeddingCache cache;
    private CountingEmbedder delegate;
    private CachingMultiModalEmbedder caching;

    @BeforeEach
    void setUp() {
        cache = new EmbeddingCache(":memory:", "test:3:512:");
        cache.init();
        delegate = new CountingEmbedder();
        caching = new CachingMultiModalEmbedder(delegate, cache, true);
    }

    @AfterEach
    void tearDown() {
        cache.shutdown();
    }

    @Test
    void cacheMissCallsDelegate() {
        MultiModalEmbedding result = caching.embed("hello");
        assertNotNull(result);
        assertEquals(1, delegate.callCount());
    }

    @Test
    void cacheHitSkipsDelegate() {
        caching.embed("hello");
        assertEquals(1, delegate.callCount());

        caching.embed("hello");
        assertEquals(1, delegate.callCount());
    }

    @Test
    void embedBatchSplitsHitsAndMisses() {
        caching.embed("a");
        assertEquals(1, delegate.callCount());

        List<MultiModalEmbedding> results =
                caching.embedBatch(List.of("a", "b", "c"));

        assertEquals(3, results.size());
        assertEquals(2, delegate.callCount());
    }

    @Test
    void disabledPassesThrough() {
        var disabled = new CachingMultiModalEmbedder(delegate, cache, false);
        disabled.embed("x");
        disabled.embed("x");
        assertEquals(2, delegate.callCount());
    }

    @Test
    void passthroughMethods() {
        assertEquals(3, caching.denseDimension());
        assertEquals(512, caching.maxSequenceLength());
        assertTrue(caching.supportedModes().contains(EmbeddingMode.DENSE));
    }

    static class CountingEmbedder implements MultiModalEmbedder {
        private final AtomicInteger calls = new AtomicInteger();

        @Override
        public MultiModalEmbedding embed(String text) {
            calls.incrementAndGet();
            return new MultiModalEmbedding(new float[]{1.0f, 2.0f, 3.0f}, null, null);
        }

        @Override
        public List<MultiModalEmbedding> embedBatch(List<String> texts) {
            calls.incrementAndGet();
            return texts.stream()
                    .map(t -> new MultiModalEmbedding(new float[]{1.0f, 2.0f, 3.0f}, null, null))
                    .toList();
        }

        @Override
        public Set<EmbeddingMode> supportedModes() {
            return Set.of(EmbeddingMode.DENSE);
        }

        @Override
        public int denseDimension() { return 3; }

        @Override
        public OptionalInt colbertDimension() { return OptionalInt.empty(); }

        @Override
        public int maxSequenceLength() { return 512; }

        int callCount() { return calls.get(); }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f rag-cache/pom.xml test -Dtest=CachingMultiModalEmbedderTest -q`
Expected: compilation failure — `CachingMultiModalEmbedder` does not exist.

- [ ] **Step 3: Implement CachingMultiModalEmbedder**

Create `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/CachingMultiModalEmbedder.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.EmbeddingMode;
import io.casehub.neocortex.inference.MultiModalEmbedder;
import io.casehub.neocortex.inference.MultiModalEmbedding;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;
import java.util.logging.Logger;

public class CachingMultiModalEmbedder implements MultiModalEmbedder {

    private static final Logger LOG =
            Logger.getLogger(CachingMultiModalEmbedder.class.getName());

    private final MultiModalEmbedder delegate;
    private final EmbeddingCache cache;
    private final boolean enabled;

    public CachingMultiModalEmbedder(MultiModalEmbedder delegate,
                                      EmbeddingCache cache,
                                      boolean enabled) {
        this.delegate = delegate;
        this.cache = cache;
        this.enabled = enabled;
    }

    @Override
    public MultiModalEmbedding embed(String text) {
        return embedBatch(List.of(text)).getFirst();
    }

    @Override
    public List<MultiModalEmbedding> embedBatch(List<String> texts) {
        if (!enabled) return delegate.embedBatch(texts);

        String[] hashes = new String[texts.size()];
        List<String> hashList = new ArrayList<>(texts.size());
        for (int i = 0; i < texts.size(); i++) {
            hashes[i] = sha256(texts.get(i));
            hashList.add(hashes[i]);
        }

        Map<String, MultiModalEmbedding> cached = cache.getBatch(hashList);

        List<Integer> missIndices = new ArrayList<>();
        List<String> missTexts = new ArrayList<>();
        for (int i = 0; i < texts.size(); i++) {
            if (!cached.containsKey(hashes[i])) {
                missIndices.add(i);
                missTexts.add(texts.get(i));
            }
        }

        List<MultiModalEmbedding> computed = List.of();
        if (!missTexts.isEmpty()) {
            computed = delegate.embedBatch(missTexts);
            Map<String, MultiModalEmbedding> toStore = new LinkedHashMap<>();
            for (int j = 0; j < missIndices.size(); j++) {
                toStore.put(hashes[missIndices.get(j)], computed.get(j));
            }
            cache.putBatch(toStore);
        }

        MultiModalEmbedding[] results = new MultiModalEmbedding[texts.size()];
        int missIdx = 0;
        for (int i = 0; i < texts.size(); i++) {
            MultiModalEmbedding hit = cached.get(hashes[i]);
            if (hit != null) {
                results[i] = hit;
            } else {
                results[i] = computed.get(missIdx++);
            }
        }

        int hits = cached.size();
        if (hits > 0) {
            LOG.fine(() -> "Embedding cache: " + hits + " hits, "
                    + missTexts.size() + " misses out of " + texts.size());
        }

        return List.of(results);
    }

    @Override
    public Set<EmbeddingMode> supportedModes() {
        return delegate.supportedModes();
    }

    @Override
    public int denseDimension() {
        return delegate.denseDimension();
    }

    @Override
    public OptionalInt colbertDimension() {
        return delegate.colbertDimension();
    }

    @Override
    public int maxSequenceLength() {
        return delegate.maxSequenceLength();
    }

    static String sha256(String text) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(
                    text.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new AssertionError("SHA-256 not available", e);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f rag-cache/pom.xml test -Dtest=CachingMultiModalEmbedderTest -q`
Expected: 5 tests PASS

- [ ] **Step 5: Create CDI producer**

Create `rag-cache/src/main/java/io/casehub/neocortex/rag/cache/CachingEmbedderDecorator.java`:

```java
package io.casehub.neocortex.rag.cache;

import io.casehub.neocortex.inference.MultiModalEmbedder;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import jakarta.decorator.Decorator;
import jakarta.decorator.Delegate;
import jakarta.enterprise.inject.Any;
import jakarta.inject.Inject;
import jakarta.annotation.Priority;

import java.io.File;
import java.util.logging.Level;
import java.util.logging.Logger;

@Decorator
@Priority(100)
public class CachingEmbedderDecorator implements MultiModalEmbedder {

    private static final Logger LOG =
            Logger.getLogger(CachingEmbedderDecorator.class.getName());

    @Inject @Delegate @Any
    MultiModalEmbedder delegate;

    @Inject
    EmbeddingCacheConfig config;

    private CachingMultiModalEmbedder caching;
    private EmbeddingCache cache;

    @PostConstruct
    void init() {
        if (!config.enabled() || config.path().isBlank()) {
            LOG.info("Embedding cache disabled");
            return;
        }

        try {
            File dbFile = new File(config.path());
            File parent = dbFile.getParentFile();
            if (parent != null && !parent.exists()) {
                parent.mkdirs();
            }

            String modelVersion = delegate.denseDimension()
                    + ":" + delegate.maxSequenceLength()
                    + ":" + config.versionSuffix();
            cache = new EmbeddingCache(config.path(), modelVersion);
            cache.init();
            caching = new CachingMultiModalEmbedder(delegate, cache, true);
            LOG.info(() -> "Embedding cache enabled at " + config.path()
                    + " (model version: " + modelVersion + ")");
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Embedding cache init failed — proceeding without cache", e);
            caching = null;
        }
    }

    @PreDestroy
    void shutdown() {
        if (cache != null) cache.shutdown();
    }

    @Override
    public io.casehub.neocortex.inference.MultiModalEmbedding embed(String text) {
        return active() ? caching.embed(text) : delegate.embed(text);
    }

    @Override
    public java.util.List<io.casehub.neocortex.inference.MultiModalEmbedding>
            embedBatch(java.util.List<String> texts) {
        return active() ? caching.embedBatch(texts) : delegate.embedBatch(texts);
    }

    @Override
    public io.casehub.neocortex.inference.MultiModalEmbedding
            embedSeparate(String denseText, String nonDenseText) {
        return active()
                ? caching.embedSeparate(denseText, nonDenseText)
                : delegate.embedSeparate(denseText, nonDenseText);
    }

    @Override
    public java.util.Set<io.casehub.neocortex.inference.EmbeddingMode> supportedModes() {
        return delegate.supportedModes();
    }

    @Override
    public int denseDimension() {
        return delegate.denseDimension();
    }

    @Override
    public java.util.OptionalInt colbertDimension() {
        return delegate.colbertDimension();
    }

    @Override
    public int maxSequenceLength() {
        return delegate.maxSequenceLength();
    }

    private boolean active() {
        return caching != null;
    }
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn -f rag-cache/pom.xml test -q`
Expected: all tests PASS (serializer + cache + caching embedder)

- [ ] **Step 7: Commit**

```bash
git add rag-cache/src/
git commit -m "feat: add CachingMultiModalEmbedder — CDI decorator with SQLite cache

Refs Hortora/engine#83"
```

---

### Task 4: Wire into engine

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/engine/pom.xml` — add dependency
- Modify: `/Users/mdproctor/claude/hortora/engine/src/main/resources/application.properties` — add cache config
- Modify: `/Users/mdproctor/claude/hortora/engine/src/test/resources/application.properties` — disable cache in tests

**Interfaces:**
- Consumes: `casehub-neocortex-rag-cache` module (Tasks 1-3)
- Produces: embedded cache active in engine dev/prod mode

- [ ] **Step 1: Install neocortex SNAPSHOT**

Run from neocortex root: `mvn install -pl rag-cache -am -q`

- [ ] **Step 2: Add dependency to engine pom.xml**

Add after the existing `casehub-neocortex-rag-tracking` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag-cache</artifactId>
</dependency>
```

- [ ] **Step 3: Add config to engine application.properties**

Add to `src/main/resources/application.properties`:

```properties
casehub.rag.embedding-cache.enabled=true
casehub.rag.embedding-cache.path=${user.home}/.hortora/cache/embeddings.db
```

- [ ] **Step 4: Disable cache in test properties**

Add to `src/test/resources/application.properties`:

```properties
casehub.rag.embedding-cache.enabled=false
```

- [ ] **Step 5: Build and run engine tests**

Run: `mvn verify -q`
Expected: 234 tests PASS, BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add pom.xml src/main/resources/application.properties src/test/resources/application.properties
git commit -m "feat: enable embedding cache in engine

Refs Hortora/engine#83"
```
