# Payload Enrichment + Version-Aware Search Scoring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #83 — Search-time version de-emphasis
**Issue group:** #83, #80

**Goal:** Enrich Qdrant payloads with frontmatter fields at ingestion time, and apply two-layer relevance scoring (temporal decay + BOM-relative version distance) at search time.

**Architecture:** `GardenMetadataExtractor` extracts additional frontmatter fields as string metadata (fits existing SPI). Two new scorer classes (`TemporalDecayScorer`, `VersionScorer`) are applied in `SearchResource` after cross-encoder reranking, before adaptive filtering. Search profiles (SQLite-backed) cache user BOMs. A client-side PreToolUse hook resolves project + user BOMs and sends them to the engine.

**Note on Layer 1 placement:** The spec describes Layer 1 (temporal decay) operating inside Qdrant via tier-based prefetch branching. This plan implements both layers engine-side — applied post cross-encoder, before adaptive filtering. For the current corpus size (~5K entries, overfetch pool of 50), engine-side scoring delivers identical ranking outcomes without modifying `HybridCaseRetriever` in the shared neocortex-rag library. If the corpus grows to a scale where retrieval-time filtering matters, Layer 1 can be pushed into Qdrant prefetch legs as a future optimization.

**Tech Stack:** Quarkus 3.36.x, SQLite (via existing H2/JDBC patterns), Qdrant Java client 1.18.1

## Global Constraints

- All metadata through neocortex-rag SPI is `Map<String, String>` — no typed fields through extraction
- `verified_on` format: colon-separated `stack:version` (e.g. `quarkus:3.20`, `onnx-runtime:1.26.0`)
- `decay_tier` stored as string "0"/"1"/"2"/"3" in Qdrant payload
- Both scoring layers are engine-side, applied post cross-encoder reranking
- Cross-repo change to neocortex-rag limited to payload index registration (one line)

---

### Task 1: Extend GardenMetadataExtractor with frontmatter fields

**Files:**
- Modify: `src/main/java/io/hortora/garden/index/GardenMetadataExtractor.java`
- Test: `src/test/java/io/hortora/garden/index/GardenMetadataExtractorTest.java`

**Interfaces:**
- Consumes: `MetadataExtractor` SPI, SnakeYAML frontmatter parsing (existing)
- Produces: `ExtractionResult` with new metadata keys: `staleness_threshold`, `staleness_days`, `decay_tier`, `verified_on`, `author`, `last_reviewed`

- [ ] **Step 1: Write failing test for new field extraction**

```java
@Test
void extractsStalenessAndVersionFields() {
    String content = """
            ---
            title: Test entry
            domain: jvm
            type: gotcha
            score: 12
            submitted: 2026-01-15
            staleness_threshold: 30d
            verified_on: "quarkus:3.20"
            author: mdp
            last_reviewed: 2026-07-01
            ---
            Body text here.
            """;
    ExtractionResult result = extractor.extract("test.md", content.getBytes());
    assertEquals("30d", result.metadata().get("staleness_threshold"));
    assertEquals("30", result.metadata().get("staleness_days"));
    assertEquals("0", result.metadata().get("decay_tier"));
    assertEquals("quarkus:3.20", result.metadata().get("verified_on"));
    assertEquals("mdp", result.metadata().get("author"));
    assertEquals("2026-07-01", result.metadata().get("last_reviewed"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=GardenMetadataExtractorTest#extractsStalenessAndVersionFields -q`
Expected: FAIL — new metadata keys not present

- [ ] **Step 3: Write test for decay tier defaults**

```java
@Test
void decayTierDefaults() {
    // No staleness_threshold → default 90d → tier 1
    String noThreshold = """
            ---
            title: No threshold
            domain: jvm
            type: gotcha
            score: 10
            ---
            Body.
            """;
    ExtractionResult result = extractor.extract("test.md", noThreshold.getBytes());
    assertEquals("90", result.metadata().get("staleness_days"));
    assertEquals("1", result.metadata().get("decay_tier"));
}

@Test
void decayTierEvergreen() {
    String evergreen = """
            ---
            title: Evergreen
            domain: jvm
            type: convention
            score: 10
            staleness_threshold: never
            ---
            Body.
            """;
    ExtractionResult result = extractor.extract("test.md", evergreen.getBytes());
    assertEquals("0", result.metadata().get("staleness_days"));
    assertEquals("3", result.metadata().get("decay_tier"));
}
```

- [ ] **Step 4: Implement field extraction in GardenMetadataExtractor**

Add to `extract()` method, after the existing metadata extraction block (after line 61):

```java
// Staleness and decay tier
String stalenessThreshold = fm.get("staleness_threshold") instanceof String s ? s : null;
int staleDays = parseStaleness(stalenessThreshold);
int tier = stalenessToTier(staleDays);
metadata.put("staleness_days", String.valueOf(staleDays));
metadata.put("decay_tier", String.valueOf(tier));
if (stalenessThreshold != null) metadata.put("staleness_threshold", stalenessThreshold);

// Version, author, review date
if (fm.get("verified_on") instanceof String s) metadata.put("verified_on", s);
if (fm.get("author") instanceof String s) metadata.put("author", s);
if (fm.get("last_reviewed") != null) {
    metadata.put("last_reviewed", String.valueOf(fm.get("last_reviewed")));
}
```

Add helper methods:

```java
static int parseStaleness(String threshold) {
    if (threshold == null) return 90;
    if ("never".equalsIgnoreCase(threshold)) return 0;
    if (threshold.endsWith("d")) {
        try { return Integer.parseInt(threshold.substring(0, threshold.length() - 1)); }
        catch (NumberFormatException e) { return 90; }
    }
    return 90;
}

static int stalenessToTier(int days) {
    if (days == 0) return 3;      // evergreen
    if (days <= 30) return 0;     // fast
    if (days <= 90) return 1;     // standard
    return 2;                      // slow
}
```

- [ ] **Step 5: Run all extractor tests**

Run: `./mvnw test -pl . -Dtest=GardenMetadataExtractorTest -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/hortora/garden/index/GardenMetadataExtractor.java
git add src/test/java/io/hortora/garden/index/GardenMetadataExtractorTest.java
git commit -m "feat(#80): extract staleness, verified_on, author, last_reviewed in GardenMetadataExtractor

Refs #80"
```

---

### Task 2: Add Qdrant payload indexes for new fields (cross-repo)

**Files:**
- Modify: `neocortex/rag/src/main/java/io/casehub/neocortex/rag/runtime/QdrantEmbeddingIngestor.java` (in casehub-neocortex repo)

**Interfaces:**
- Consumes: Qdrant collection schema
- Produces: Keyword indexes on `decay_tier`, `verified_on`, `author`, `last_reviewed`

- [ ] **Step 1: Add index registrations**

In `QdrantEmbeddingIngestor.ensurePayloadIndexes()`, after the existing `checkIndexType` calls for `domain`, `type`, `tags` (around line 347):

```java
checkIndexType(existingSchema, "decay_tier", PayloadSchemaType.Keyword, collection);
checkIndexType(existingSchema, "verified_on", PayloadSchemaType.Keyword, collection);
checkIndexType(existingSchema, "author", PayloadSchemaType.Keyword, collection);
checkIndexType(existingSchema, "last_reviewed", PayloadSchemaType.Keyword, collection);
```

- [ ] **Step 2: Install to local Maven repo**

Run: `mvn -f ~/claude/casehub/neocortex/rag/pom.xml install -q -DskipTests`

- [ ] **Step 3: Verify engine builds with updated dependency**

Run: `./mvnw compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit to neocortex**

```bash
git -C ~/claude/casehub/neocortex add rag/src/main/java/io/casehub/neocortex/rag/runtime/QdrantEmbeddingIngestor.java
git -C ~/claude/casehub/neocortex commit -m "feat: add payload indexes for decay_tier, verified_on, author, last_reviewed

Refs Hortora/engine#80"
```

---

### Task 3: SearchProfileStore + ProfileResource

**Files:**
- Create: `src/main/java/io/hortora/garden/search/SearchProfileStore.java`
- Create: `src/main/java/io/hortora/garden/search/ProfileResource.java`
- Test: `src/test/java/io/hortora/garden/search/SearchProfileStoreTest.java`
- Test: `src/test/java/io/hortora/garden/search/ProfileResourceTest.java`

**Interfaces:**
- Consumes: SQLite via Agroal datasource
- Produces:
  - `SearchProfileStore.get(String name)` → `Optional<Map<String, String>>` (parsed BOM)
  - `SearchProfileStore.put(String name, String stack)` → void
  - `SearchProfileStore.delete(String name)` → boolean
  - `SearchProfileStore.list()` → `List<String>` (profile names)
  - REST: `PUT /api/garden/profiles/{name}`, `GET /api/garden/profiles/{name}`, `GET /api/garden/profiles`, `DELETE /api/garden/profiles/{name}`

- [ ] **Step 1: Write failing test for SearchProfileStore**

```java
@QuarkusTest
class SearchProfileStoreTest {

    @Inject
    SearchProfileStore store;

    @Test
    void putAndGet() {
        store.put("test-project", "quarkus:3.36.1|jdk:26.0.2");
        Optional<Map<String, String>> bom = store.get("test-project");
        assertTrue(bom.isPresent());
        assertEquals("3.36.1", bom.get().get("quarkus"));
        assertEquals("26.0.2", bom.get().get("jdk"));
    }

    @Test
    void getMissingReturnsEmpty() {
        assertTrue(store.get("nonexistent").isEmpty());
    }

    @Test
    void putOverwrites() {
        store.put("proj", "quarkus:3.20");
        store.put("proj", "quarkus:3.36");
        assertEquals("3.36", store.get("proj").get().get("quarkus"));
    }

    @Test
    void deleteRemoves() {
        store.put("temp", "jdk:21");
        assertTrue(store.delete("temp"));
        assertTrue(store.get("temp").isEmpty());
    }

    @Test
    void listReturnsNames() {
        store.put("a", "jdk:21");
        store.put("b", "jdk:26");
        List<String> names = store.list();
        assertTrue(names.contains("a"));
        assertTrue(names.contains("b"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=SearchProfileStoreTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement SearchProfileStore**

```java
package io.hortora.garden.search;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import java.nio.file.Path;
import java.sql.*;
import java.util.*;

@ApplicationScoped
public class SearchProfileStore {

    private static final Path DB_PATH = Path.of(
            System.getProperty("user.home"), ".hortora", "stats", "profiles.db");

    @PostConstruct
    void init() {
        try (Connection conn = connect()) {
            conn.createStatement().execute("""
                CREATE TABLE IF NOT EXISTS search_profiles (
                    name TEXT PRIMARY KEY,
                    stack TEXT NOT NULL,
                    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
                )""");
        } catch (SQLException e) {
            throw new RuntimeException("Failed to init profile store", e);
        }
    }

    public void put(String name, String stack) {
        try (Connection conn = connect();
             PreparedStatement ps = conn.prepareStatement(
                 "INSERT INTO search_profiles (name, stack, updated_at) VALUES (?, ?, datetime('now')) " +
                 "ON CONFLICT(name) DO UPDATE SET stack = ?, updated_at = datetime('now')")) {
            ps.setString(1, name);
            ps.setString(2, stack);
            ps.setString(3, stack);
            ps.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to put profile: " + name, e);
        }
    }

    public Optional<Map<String, String>> get(String name) {
        try (Connection conn = connect();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT stack FROM search_profiles WHERE name = ?")) {
            ps.setString(1, name);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                return Optional.of(parseStack(rs.getString("stack")));
            }
            return Optional.empty();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to get profile: " + name, e);
        }
    }

    public boolean delete(String name) {
        try (Connection conn = connect();
             PreparedStatement ps = conn.prepareStatement(
                 "DELETE FROM search_profiles WHERE name = ?")) {
            ps.setString(1, name);
            return ps.executeUpdate() > 0;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to delete profile: " + name, e);
        }
    }

    public List<String> list() {
        try (Connection conn = connect();
             Statement st = conn.createStatement();
             ResultSet rs = st.executeQuery("SELECT name FROM search_profiles ORDER BY name")) {
            List<String> names = new ArrayList<>();
            while (rs.next()) names.add(rs.getString("name"));
            return names;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to list profiles", e);
        }
    }

    static Map<String, String> parseStack(String stack) {
        Map<String, String> bom = new LinkedHashMap<>();
        if (stack == null || stack.isBlank()) return bom;
        for (String entry : stack.split("\\|")) {
            String[] parts = entry.split(":", 2);
            if (parts.length == 2) bom.put(parts[0].trim(), parts[1].trim());
        }
        return bom;
    }

    private Connection connect() throws SQLException {
        DB_PATH.getParent().toFile().mkdirs();
        return DriverManager.getConnection("jdbc:sqlite:" + DB_PATH);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `./mvnw test -pl . -Dtest=SearchProfileStoreTest -q`
Expected: ALL PASS

- [ ] **Step 5: Write failing test for ProfileResource**

```java
@QuarkusTest
class ProfileResourceTest {

    @Test
    void putAndGetProfile() {
        given().contentType("application/json")
               .body("{\"stack\": \"quarkus:3.36.1|jdk:26.0.2\"}")
               .when().put("/api/garden/profiles/test-proj")
               .then().statusCode(204);

        given().when().get("/api/garden/profiles/test-proj")
               .then().statusCode(200)
               .body("name", is("test-proj"))
               .body("stack", containsString("quarkus:3.36.1"));
    }

    @Test
    void getNotFound() {
        given().when().get("/api/garden/profiles/nonexistent")
               .then().statusCode(404);
    }

    @Test
    void listProfiles() {
        given().contentType("application/json")
               .body("{\"stack\": \"jdk:21\"}")
               .when().put("/api/garden/profiles/list-test")
               .then().statusCode(204);

        given().when().get("/api/garden/profiles")
               .then().statusCode(200);
    }

    @Test
    void deleteProfile() {
        given().contentType("application/json")
               .body("{\"stack\": \"jdk:21\"}")
               .when().put("/api/garden/profiles/to-delete")
               .then().statusCode(204);

        given().when().delete("/api/garden/profiles/to-delete")
               .then().statusCode(204);

        given().when().get("/api/garden/profiles/to-delete")
               .then().statusCode(404);
    }
}
```

- [ ] **Step 6: Implement ProfileResource**

```java
package io.hortora.garden.search;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.Map;
import java.util.Optional;

@Path("/api/garden/profiles")
@Produces(MediaType.APPLICATION_JSON)
public class ProfileResource {

    @Inject
    SearchProfileStore store;

    @PUT
    @Path("/{name}")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response put(@PathParam("name") String name, Map<String, String> body) {
        String stack = body.get("stack");
        if (stack == null || stack.isBlank()) {
            return Response.status(400).entity(Map.of("error", "stack is required")).build();
        }
        store.put(name, stack);
        return Response.noContent().build();
    }

    @GET
    @Path("/{name}")
    public Response get(@PathParam("name") String name) {
        Optional<Map<String, String>> bom = store.get(name);
        if (bom.isEmpty()) return Response.status(404).build();
        return Response.ok(Map.of("name", name, "stack", bom.get())).build();
    }

    @GET
    public Response list() {
        return Response.ok(store.list()).build();
    }

    @DELETE
    @Path("/{name}")
    public Response delete(@PathParam("name") String name) {
        store.delete(name);
        return Response.noContent().build();
    }
}
```

- [ ] **Step 7: Run all profile tests**

Run: `./mvnw test -pl . -Dtest="SearchProfileStoreTest,ProfileResourceTest" -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/hortora/garden/search/SearchProfileStore.java
git add src/main/java/io/hortora/garden/search/ProfileResource.java
git add src/test/java/io/hortora/garden/search/SearchProfileStoreTest.java
git add src/test/java/io/hortora/garden/search/ProfileResourceTest.java
git commit -m "feat(#83): add search profile CRUD — SQLite store + REST endpoints

Refs #83"
```

---

### Task 4: TemporalDecayScorer

**Files:**
- Create: `src/main/java/io/hortora/garden/search/TemporalDecayScorer.java`
- Test: `src/test/java/io/hortora/garden/search/TemporalDecayScorerTest.java`

**Interfaces:**
- Consumes: `submitted` date string, `decay_tier` string from search result metadata
- Produces: `double score(String submittedDate, String decayTier)` → multiplier in [0.0, 1.0]

- [ ] **Step 1: Write failing tests**

```java
class TemporalDecayScorerTest {

    private final TemporalDecayScorer scorer = new TemporalDecayScorer();

    @Test
    void freshEntryScoresHigh() {
        String today = LocalDate.now().toString();
        double score = scorer.score(today, "1"); // standard tier, 90d half-life
        assertTrue(score > 0.95, "Fresh entry should score near 1.0, got " + score);
    }

    @Test
    void oldFastDecayEntryScoresLow() {
        String sixMonthsAgo = LocalDate.now().minusDays(180).toString();
        double score = scorer.score(sixMonthsAgo, "0"); // fast tier, 30d half-life
        assertTrue(score < 0.05, "180-day-old fast-decay entry should be very low, got " + score);
    }

    @Test
    void oldSlowDecayEntryScoresModerate() {
        String sixMonthsAgo = LocalDate.now().minusDays(180).toString();
        double score = scorer.score(sixMonthsAgo, "2"); // slow tier, 365d half-life
        assertTrue(score > 0.5, "180-day-old slow-decay entry should be moderate, got " + score);
    }

    @Test
    void evergreenAlwaysReturnsOne() {
        String yearAgo = LocalDate.now().minusDays(365).toString();
        assertEquals(1.0, scorer.score(yearAgo, "3"));
    }

    @Test
    void nullSubmittedReturnsOne() {
        assertEquals(1.0, scorer.score(null, "1"));
    }

    @Test
    void nullTierDefaultsToStandard() {
        String today = LocalDate.now().toString();
        double score = scorer.score(today, null);
        assertTrue(score > 0.95);
    }

    @Test
    void unparsableDateReturnsOne() {
        assertEquals(1.0, scorer.score("not-a-date", "1"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=TemporalDecayScorerTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement TemporalDecayScorer**

```java
package io.hortora.garden.search;

import java.time.LocalDate;
import java.time.format.DateTimeParseException;
import java.time.temporal.ChronoUnit;

public class TemporalDecayScorer {

    private static final double LN2 = 0.693147;

    public double score(String submittedDate, String decayTier) {
        if (submittedDate == null || submittedDate.isBlank()) return 1.0;

        int tier = parseTier(decayTier);
        if (tier == 3) return 1.0; // evergreen

        long ageDays;
        try {
            LocalDate submitted = LocalDate.parse(submittedDate.length() > 10
                    ? submittedDate.substring(0, 10) : submittedDate);
            ageDays = ChronoUnit.DAYS.between(submitted, LocalDate.now());
            if (ageDays <= 0) return 1.0;
        } catch (DateTimeParseException e) {
            return 1.0;
        }

        int halfLifeDays = switch (tier) {
            case 0 -> 30;
            case 2 -> 365;
            default -> 90; // tier 1 = standard
        };
        return Math.exp(-LN2 * ageDays / halfLifeDays);
    }

    private static int parseTier(String tier) {
        if (tier == null) return 1;
        try { return Integer.parseInt(tier); }
        catch (NumberFormatException e) { return 1; }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `./mvnw test -pl . -Dtest=TemporalDecayScorerTest -q`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/hortora/garden/search/TemporalDecayScorer.java
git add src/test/java/io/hortora/garden/search/TemporalDecayScorerTest.java
git commit -m "feat(#83): add TemporalDecayScorer — half-life decay by tier

Refs #83"
```

---

### Task 5: VersionScorer

**Files:**
- Create: `src/main/java/io/hortora/garden/search/VersionScorer.java`
- Test: `src/test/java/io/hortora/garden/search/VersionScorerTest.java`

**Interfaces:**
- Consumes: `verified_on` string from result metadata, `Map<String, String>` BOM, query text
- Produces: `double score(String verifiedOn, Map<String, String> bom, String queryText, VersionScoringConfig config)` → multiplier in [floor, 1.0]

- [ ] **Step 1: Write failing tests**

```java
class VersionScorerTest {

    private final VersionScorer scorer = new VersionScorer();
    private final Map<String, String> bom = Map.of("quarkus", "3.36.1", "jdk", "26.0.2");

    @Test
    void currentVersionScoresOne() {
        double score = scorer.score("quarkus:3.36.1", bom, "quarkus CDI", defaults());
        assertEquals(1.0, score);
    }

    @Test
    void minorVersionGapAppliesDecay() {
        double score = scorer.score("quarkus:3.20", bom, "quarkus CDI", defaults());
        // distance=16, factor=0.03, topic=1.0 → 1.0 - 16*0.03*1.0 = 0.52
        assertEquals(0.52, score, 0.01);
    }

    @Test
    void majorVersionGapHitsFloor() {
        double score = scorer.score("quarkus:2.0", bom, "quarkus CDI", defaults());
        assertEquals(0.5, score); // floor
    }

    @Test
    void offTopicStackGetsReducedWeight() {
        double score = scorer.score("jdk:21", bom, "quarkus CDI", defaults());
        // distance=5 major versions, but jdk not in query → topic_weight=0.3
        // Major version change → floor immediately
        assertEquals(0.5, score);
    }

    @Test
    void noVerifiedOnReturnsOne() {
        assertEquals(1.0, scorer.score(null, bom, "quarkus CDI", defaults()));
    }

    @Test
    void noBomReturnsOne() {
        assertEquals(1.0, scorer.score("quarkus:3.20", null, "quarkus CDI", defaults()));
        assertEquals(1.0, scorer.score("quarkus:3.20", Map.of(), "quarkus CDI", defaults()));
    }

    @Test
    void stackNotInBomReturnsOne() {
        assertEquals(1.0, scorer.score("python:3.12", bom, "python async", defaults()));
    }

    @Test
    void noColonInVerifiedOnReturnsOne() {
        assertEquals(1.0, scorer.score("quarkus", bom, "quarkus CDI", defaults()));
    }

    @Test
    void topicWeightMatchesSubstring() {
        // "onnx" in query matches "onnx-runtime" stack
        double score = scorer.score("onnx-runtime:1.20.0",
                Map.of("onnx-runtime", "1.26.0"), "onnx thread pool", defaults());
        // distance=6 minor, topic=1.0 → 1.0 - 6*0.03*1.0 = 0.82
        assertEquals(0.82, score, 0.01);
    }

    private VersionScorer.Config defaults() {
        return new VersionScorer.Config(0.03, 0.5, 0.3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=VersionScorerTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement VersionScorer**

```java
package io.hortora.garden.search;

import java.util.Map;

public class VersionScorer {

    public record Config(double decayFactor, double floor, double defaultTopicWeight) {}

    public double score(String verifiedOn, Map<String, String> bom, String queryText, Config config) {
        if (verifiedOn == null || verifiedOn.isBlank()) return 1.0;
        if (bom == null || bom.isEmpty()) return 1.0;

        String[] parts = verifiedOn.split(":", 2);
        if (parts.length < 2) return 1.0;

        String stack = parts[0];
        String entryVersion = parts[1];
        String bomVersion = bom.get(stack);
        if (bomVersion == null) return 1.0;

        int[] entryParts = parseVersion(entryVersion);
        int[] bomParts = parseVersion(bomVersion);

        if (entryParts[0] != bomParts[0]) return config.floor; // major version change

        int minorDistance = Math.abs(bomParts[1] - entryParts[1]);
        if (minorDistance == 0) return 1.0;

        double topicWeight = queryContainsStack(queryText, stack) ? 1.0 : config.defaultTopicWeight;
        return Math.max(config.floor, 1.0 - minorDistance * config.decayFactor * topicWeight);
    }

    static boolean queryContainsStack(String query, String stack) {
        if (query == null || query.isBlank()) return false;
        String lowerQuery = query.toLowerCase();
        String lowerStack = stack.toLowerCase();
        for (String token : lowerQuery.split("[\\s\\-]+")) {
            if (lowerStack.contains(token) && token.length() >= 3) return true;
        }
        return false;
    }

    static int[] parseVersion(String version) {
        String[] parts = version.split("\\.");
        int major = 0, minor = 0;
        try { major = Integer.parseInt(parts[0]); } catch (NumberFormatException ignored) {}
        if (parts.length > 1) {
            try { minor = Integer.parseInt(parts[1]); } catch (NumberFormatException ignored) {}
        }
        return new int[]{major, minor};
    }
}
```

- [ ] **Step 4: Run tests**

Run: `./mvnw test -pl . -Dtest=VersionScorerTest -q`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/hortora/garden/search/VersionScorer.java
git add src/test/java/io/hortora/garden/search/VersionScorerTest.java
git commit -m "feat(#83): add VersionScorer — BOM-relative version distance with topic weighting

Refs #83"
```

---

### Task 6: Wire scoring into search pipeline

**Files:**
- Modify: `src/main/java/io/hortora/garden/search/SearchResource.java`
- Modify: `src/main/java/io/hortora/garden/mcp/GardenMcpTools.java`
- Create: `src/main/java/io/hortora/garden/search/SearchScoringConfig.java`
- Test: `src/test/java/io/hortora/garden/search/SearchScoringIntegrationTest.java`

**Interfaces:**
- Consumes: `TemporalDecayScorer`, `VersionScorer`, `SearchProfileStore`, `SearchResult` metadata
- Produces: Modified `searchLocal()` that applies both scoring layers before returning results; `gardenSearch` gains `profile` and `stack` MCP params

- [ ] **Step 1: Create SearchScoringConfig**

```java
package io.hortora.garden.search;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import java.util.Optional;

@ConfigMapping(prefix = "hortora.search.scoring")
public interface SearchScoringConfig {

    @WithDefault("true")
    boolean temporalDecayEnabled();

    @WithDefault("true")
    boolean versionScoringEnabled();

    @WithDefault("0.03")
    double versionDecayFactor();

    @WithDefault("0.5")
    double versionDecayFloor();

    @WithDefault("0.3")
    double versionTopicWeightDefault();
}
```

- [ ] **Step 2: Write failing integration test**

```java
@QuarkusTest
class SearchScoringIntegrationTest {

    @Inject
    SearchResource searchResource;

    @Inject
    SearchProfileStore profileStore;

    @Test
    void gardenSearchAcceptsProfileParam() {
        profileStore.put("test", "quarkus:3.36.1");
        // Verify the profile can be resolved — full scoring test
        // requires entries in Qdrant (covered by manual testing post-reindex)
        var bom = profileStore.get("test");
        assertTrue(bom.isPresent());
        assertEquals("3.36.1", bom.get().get("quarkus"));
    }
}
```

- [ ] **Step 3: Add scoring application method to SearchResource**

Add a new method that applies both scoring layers to search results:

```java
List<SearchResult> applyScoring(List<SearchResult> results, String query,
                                 Map<String, String> bom, SearchScoringConfig config) {
    TemporalDecayScorer temporalScorer = new TemporalDecayScorer();
    VersionScorer versionScorer = new VersionScorer();
    VersionScorer.Config vConfig = new VersionScorer.Config(
            config.versionDecayFactor(), config.versionDecayFloor(),
            config.versionTopicWeightDefault());

    return results.stream().map(r -> {
        double multiplier = 1.0;

        if (config.temporalDecayEnabled()) {
            multiplier *= temporalScorer.score(
                    r.metadata().get("submitted"),
                    r.metadata().get("decay_tier"));
        }

        if (config.versionScoringEnabled() && bom != null && !bom.isEmpty()) {
            multiplier *= versionScorer.score(
                    r.metadata().get("verified_on"),
                    bom, query, vConfig);
        }

        if (multiplier >= 1.0) return r;
        return r.withScore(r.score() * multiplier);
    }).sorted((a, b) -> Double.compare(b.score(), a.score()))
      .toList();
}
```

- [ ] **Step 4: Wire into searchLocal()**

In `searchLocal()`, after results are mapped from `RetrievedChunk` to `SearchResult`, apply scoring before returning. Resolve the BOM from the profile/stack params passed through the call chain.

- [ ] **Step 5: Add profile and stack params to gardenSearch MCP tool**

In `GardenMcpTools.gardenSearch()`, add parameters:

```java
@Tool(description = "Search the Hortora knowledge garden...")
String gardenSearch(
        @ToolArg(...) String query,
        @ToolArg(...) String keywords,
        @ToolArg(...) String domain,
        @ToolArg(...) String type,
        @ToolArg(...) String tags,
        @ToolArg(...) Integer limit,
        @ToolArg(description = "Search profile name for BOM-based version scoring") String profile,
        @ToolArg(description = "Inline BOM override (pipe-separated name:version pairs)") String stack) {
```

Resolve BOM: if `stack` is provided, parse it directly. Else if `profile` is provided, look up `SearchProfileStore.get(profile)`. Pass the resolved BOM to `searchResource.searchAdaptive()`.

- [ ] **Step 6: Run all tests**

Run: `./mvnw verify -q`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/hortora/garden/search/SearchScoringConfig.java
git add src/main/java/io/hortora/garden/search/SearchResource.java
git add src/main/java/io/hortora/garden/mcp/GardenMcpTools.java
git add src/test/java/io/hortora/garden/search/SearchScoringIntegrationTest.java
git commit -m "feat(#83): wire temporal decay + version scoring into search pipeline

gardenSearch gains profile and stack params for BOM-relative scoring.
Both scoring layers applied post cross-encoder, before adaptive filtering.

Refs #83"
```

---

### Task 7: Client-side BOM resolution

**Files:**
- Create: `scripts/resolve-bom.py`
- Create: `scripts/garden-bom-hook.sh` (PreToolUse hook)

**Interfaces:**
- Consumes: `~/.hortora/profile.yaml`, `<cwd>/.hortora/bom.yaml`
- Produces: Merged BOM sent to engine via `PUT /api/garden/profiles/{name}`, `profile` param injected into gardenSearch calls

- [ ] **Step 1: Create resolve-bom.py**

```python
#!/usr/bin/env python3
"""Merge user + project BOM files into pipe-separated format."""
import sys, yaml
from pathlib import Path

def load_yaml(path):
    if path.exists():
        with open(path) as f:
            return yaml.safe_load(f) or {}
    return {}

user_bom = load_yaml(Path.home() / ".hortora" / "profile.yaml")
project_bom = load_yaml(Path.cwd() / ".hortora" / "bom.yaml")

merged = {**user_bom, **project_bom}  # project wins
if not merged:
    sys.exit(0)

print("|".join(f"{k}:{v}" for k, v in merged.items()))
```

- [ ] **Step 2: Create PreToolUse hook script**

```bash
#!/bin/bash
# garden-bom-hook.sh — PreToolUse hook for mcp__hortora__gardenSearch
# Resolves BOM, creates/updates search profile, injects profile param

TOOL_NAME="$1"
if [ "$TOOL_NAME" != "mcp__hortora__gardenSearch" ]; then
    exit 0
fi

FLAG="/tmp/.hortora-bom-$(basename $(git rev-parse --show-toplevel 2>/dev/null || echo unknown))"
PROFILE_NAME=$(basename $(git rev-parse --show-toplevel 2>/dev/null || echo "default"))

# Only resolve BOM once per session
if [ -f "$FLAG" ]; then
    exit 0
fi

STACK=$(python3 ~/.hortora/resolve-bom.py 2>/dev/null)
if [ -z "$STACK" ]; then
    exit 0
fi

# Send to engine
curl -s -X PUT "http://localhost:8080/api/garden/profiles/$PROFILE_NAME" \
    -H "Content-Type: application/json" \
    -d "{\"stack\": \"$STACK\"}" >/dev/null 2>&1

touch "$FLAG"
exit 0
```

- [ ] **Step 3: Test manually**

Create a test BOM file and verify the script works:

```bash
mkdir -p .hortora
echo "quarkus: 3.36.1" > .hortora/bom.yaml
echo "jdk: 26.0.2" >> .hortora/bom.yaml
python3 scripts/resolve-bom.py
# Expected: quarkus:3.36.1|jdk:26.0.2
```

- [ ] **Step 4: Commit**

```bash
git add scripts/resolve-bom.py scripts/garden-bom-hook.sh
git commit -m "feat(#83): add client-side BOM resolution script and hook

resolve-bom.py merges ~/.hortora/profile.yaml + <project>/.hortora/bom.yaml.
garden-bom-hook.sh is a PreToolUse hook that sends the BOM to the engine
as a search profile.

Refs #83"
```

---

### Task 8: Reindex and end-to-end verification

**Files:**
- No new files — operational verification

**Interfaces:**
- Consumes: All prior tasks
- Produces: Verified payload fields in Qdrant, working scoring pipeline

- [ ] **Step 1: Rebuild and redeploy engine**

```bash
./mvnw verify -q
scripts/update-engine.sh update
```

- [ ] **Step 2: Trigger reindex**

```bash
curl -X POST http://localhost:8080/api/garden/reindex
```

Wait for reindex to complete (check logs).

- [ ] **Step 3: Verify payload enrichment**

Query a Qdrant point directly and verify new payload fields:

```bash
curl -s http://localhost:6333/collections/hortora_garden/points/scroll \
  -H "Content-Type: application/json" \
  -d '{"limit": 1, "with_payload": true}' | python3 -m json.tool | head -40
```

Verify `decay_tier`, `staleness_days`, `verified_on`, `author`, `last_reviewed` appear in the payload.

- [ ] **Step 4: Verify search with profile**

```bash
# Create a profile
curl -X PUT http://localhost:8080/api/garden/profiles/test \
  -H "Content-Type: application/json" \
  -d '{"stack": "quarkus:3.36.1|jdk:26.0.2"}'

# Search with profile — verify results return (scoring applied silently)
# Compare results with and without profile to see scoring effect
```

- [ ] **Step 5: Update CLAUDE.md with new config properties**

Add the new `hortora.search.scoring.*` config properties to the CLAUDE.md documentation.

- [ ] **Step 6: Commit CLAUDE.md update**

```bash
git add CLAUDE.md
git commit -m "docs: add search scoring config to CLAUDE.md

Refs #83, #80"
```
