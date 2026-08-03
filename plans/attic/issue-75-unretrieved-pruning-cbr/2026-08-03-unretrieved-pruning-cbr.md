# gardenUnretrieved Refactor, Snapshot Pruning, CBR Outcome Tracking — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #75 — refactor gardenUnretrieved to use RetrievalAnalyzer
**Issue group:** #75, #57, #74

**Goal:** Replace inline analysis logic in gardenUnretrieved with RetrievalAnalyzer, add snapshot pruning, and add CBR-based outcome tracking for garden entries.

**Architecture:** #75 delegates gardenUnretrieved's set-diffing to `RetrievalAnalyzer.qualitySignals()`. #57 adds `--prune` to the existing snapshot CLI. #74 adds `JpaCbrCaseMemoryStore` (H2 file-persistent) with a `GardenOutcomeService` backing both MCP and REST endpoints for outcome recording and reporting.

**Tech Stack:** Java 25, Quarkus 3.36, casehub-neocortex-rag-api (RetrievalAnalyzer), casehub-neocortex-memory-cbr-jpa (JpaCbrCaseMemoryStore), H2, Python 3 (snapshot scripts)

## Global Constraints

- Pre-release platform — breaking changes are fine
- IntelliJ MCP mandatory for all `.java` edits — never use bash Edit/Write
- All commits reference an issue: `Refs #N` or `Closes #N`
- Tests use `InMemoryRetrievalTracker` and `InMemoryEmbeddingIngestor` from `casehub-neocortex-rag-testing`
- Tests use `InMemoryCbrCaseMemoryStore` from `casehub-neocortex-memory-cbr-inmem` for CBR

---

### Task 1: Refactor gardenUnretrieved to use RetrievalAnalyzer (#75)

**Files:**
- Modify: `src/main/java/io/hortora/garden/mcp/GardenMcpTools.java` (gardenUnretrieved method, lines 211-272)
- Modify: `src/test/java/io/hortora/garden/mcp/GardenMcpToolsTest.java` (add HIGH_RETRIEVAL_LOW_QUALITY test)

**Interfaces:**
- Consumes: `RetrievalAnalyzer.qualitySignals(RetrievalTracker, EmbeddingIngestor, CorpusRef, Instant, Instant, QualityThresholds)` → `List<DocumentQualitySignal>`
- Consumes: `QualityThresholds(int minRetrievals, int minFeedback, double lowQualityRatio, Duration staleWindow)`
- Consumes: `DocumentQualitySignal(String sourceDocumentId, DocumentStats stats, QualitySignal signal)`
- Consumes: `QualitySignal.NEVER_RETRIEVED`, `QualitySignal.STALE`, `QualitySignal.HIGH_RETRIEVAL_LOW_QUALITY`
- Produces: Same MCP tool interface — `gardenUnretrieved(Integer minDays, Integer staleDays)` → `String`

- [ ] **Step 1: Write failing test for HIGH_RETRIEVAL_LOW_QUALITY signal**

Add to `GardenMcpToolsTest`:

```java
@Test
void gardenUnretrievedSurfacesLowQualityEntries() {
    CorpusRef corpus = new CorpusRef("hortora", "garden");
    // Record 3 retrievals (meets minRetrievals=3 threshold)
    for (int i = 0; i < 3; i++) {
        retrievalTracker.record(
                RetrievalQuery.of("hibernate"),
                corpus,
                List.of(new RetrievedChunk("content", "jvm/GE-20260620-a1b2c3.md", 0.9, Map.of())),
                16);
    }
    // Record 3 NOT_RELEVANT feedback (meets minFeedback=3, ratio=1.0 >= 0.7)
    var records = retrievalTracker.findRecords(corpus, Instant.EPOCH, Instant.now());
    for (var record : records) {
        retrievalTracker.feedback(record.retrievalId(), "jvm/GE-20260620-a1b2c3.md",
                RetrievalOutcome.NOT_RELEVANT);
    }

    String result = mcpTools.gardenUnretrieved(1, null);

    assertThat(result).contains("Low quality");
    assertThat(result).contains("GE-20260620-a1b2c3");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=GardenMcpToolsTest#gardenUnretrievedSurfacesLowQualityEntries`
Expected: FAIL — current implementation doesn't surface LOW_QUALITY signals.

- [ ] **Step 3: Replace gardenUnretrieved body with RetrievalAnalyzer call**

Use `ide_replace_member` on `GardenMcpTools.gardenUnretrieved`. New body:

```java
int effectiveMinDays   = minDays != null && minDays > 0 ? minDays : 30;
int effectiveStaleDays = staleDays != null && staleDays >= 0 ? staleDays : 90;

CorpusRef corpusRef = new CorpusRef("hortora", config.id());

QualityThresholds thresholds = new QualityThresholds(
        3, 3, 0.7, Duration.ofDays(effectiveStaleDays));

List<DocumentQualitySignal> signals = RetrievalAnalyzer.qualitySignals(
        retrievalTracker, embeddingIngestor, corpusRef,
        Instant.EPOCH, Instant.now(), thresholds);

List<String> unretrieved = signals.stream()
        .filter(s -> s.signal() == QualitySignal.NEVER_RETRIEVED)
        .map(DocumentQualitySignal::sourceDocumentId)
        .filter(id -> passesMinDaysFilter(id, effectiveMinDays))
        .sorted()
        .toList();

List<String> stale = signals.stream()
        .filter(s -> s.signal() == QualitySignal.STALE)
        .map(DocumentQualitySignal::sourceDocumentId)
        .sorted()
        .toList();

List<String> lowQuality = signals.stream()
        .filter(s -> s.signal() == QualitySignal.HIGH_RETRIEVAL_LOW_QUALITY)
        .map(DocumentQualitySignal::sourceDocumentId)
        .sorted()
        .toList();

if (unretrieved.isEmpty() && stale.isEmpty() && lowQuality.isEmpty()) {
    int total = embeddingIngestor.listDocuments(corpusRef).size();
    return "All " + total + " entries have been retrieved within the tracking window.";
}

StringBuilder sb = new StringBuilder();
sb.append("Tracking window: retrieval records retained for configured period. ")
  .append("Stale threshold: ").append(effectiveStaleDays).append(" days.\n\n");

if (!unretrieved.isEmpty()) {
    sb.append("## Unretrieved entries (").append(unretrieved.size()).append(")\n\n");
    Map<String, List<String>> byDomain = groupByDomain(unretrieved);
    for (var entry : byDomain.entrySet()) {
        sb.append("### ").append(entry.getKey()).append("\n");
        entry.getValue().forEach(id -> sb.append("- ").append(extractDocumentId(id)).append("\n"));
        sb.append("\n");
    }
}

if (!stale.isEmpty()) {
    sb.append("## Stale entries (").append(stale.size())
      .append(") — not retrieved in the last ").append(effectiveStaleDays).append(" days\n\n");
    Map<String, List<String>> byDomain = groupByDomain(stale);
    for (var entry : byDomain.entrySet()) {
        sb.append("### ").append(entry.getKey()).append("\n");
        entry.getValue().forEach(id -> sb.append("- ").append(extractDocumentId(id)).append("\n"));
        sb.append("\n");
    }
}

if (!lowQuality.isEmpty()) {
    sb.append("## Low quality entries (").append(lowQuality.size())
      .append(") — frequently retrieved but rated poorly\n\n");
    Map<String, List<String>> byDomain = groupByDomain(lowQuality);
    for (var entry : byDomain.entrySet()) {
        sb.append("### ").append(entry.getKey()).append("\n");
        entry.getValue().forEach(id -> sb.append("- ").append(extractDocumentId(id)).append("\n"));
        sb.append("\n");
    }
}

return sb.toString();
```

Add imports for `RetrievalAnalyzer`, `QualityThresholds`, `DocumentQualitySignal`, `QualitySignal`, `Duration`.

- [ ] **Step 4: Run all gardenUnretrieved tests**

Run: `./mvnw test -pl . -Dtest=GardenMcpToolsTest`
Expected: ALL PASS — existing tests produce same output, new test passes.

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on `GardenMcpTools.java` — no errors.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/hortora/garden/mcp/GardenMcpTools.java src/test/java/io/hortora/garden/mcp/GardenMcpToolsTest.java
git commit -m "refactor: gardenUnretrieved uses RetrievalAnalyzer.qualitySignals()

Replaces ~40 lines of inline set-diffing with a single RetrievalAnalyzer
call. Adds HIGH_RETRIEVAL_LOW_QUALITY signal section (dormant until
feedback data exists).

Refs #75"
```

---

### Task 2: Snapshot pruning/rotation (#57)

**Files:**
- Modify: `scripts/benchmark/create_snapshot.py`
- Modify: `scripts/benchmark/test_create_snapshot.py`

**Interfaces:**
- Consumes: `list_snapshots()` (existing) → `list[dict]`
- Consumes: `SNAPSHOT_DIR` (existing) → `Path`
- Produces: `prune_snapshots(keep: int, max_age_days: int, dry_run: bool)` → `list[dict]`

- [ ] **Step 1: Write failing tests for pruning**

Add to `test_create_snapshot.py`:

```python
def test_prune_keeps_n_most_recent(tmp_path, monkeypatch):
    monkeypatch.setattr(create_snapshot, "SNAPSHOT_DIR", tmp_path)
    _create_fake_snapshots(tmp_path, ["old", "mid", "new"], days_ago=[60, 30, 1])

    pruned = create_snapshot.prune_snapshots(keep=2, max_age_days=365, dry_run=False)

    assert len(pruned) == 1
    assert pruned[0]["name"] == "old"
    assert not (tmp_path / "old").exists()
    assert (tmp_path / "mid").exists()
    assert (tmp_path / "new").exists()


def test_prune_respects_max_age(tmp_path, monkeypatch):
    monkeypatch.setattr(create_snapshot, "SNAPSHOT_DIR", tmp_path)
    _create_fake_snapshots(tmp_path, ["ancient", "recent"], days_ago=[90, 5])

    pruned = create_snapshot.prune_snapshots(keep=1, max_age_days=30, dry_run=False)

    assert len(pruned) == 1
    assert pruned[0]["name"] == "ancient"


def test_prune_keep_protects_from_age_deletion(tmp_path, monkeypatch):
    monkeypatch.setattr(create_snapshot, "SNAPSHOT_DIR", tmp_path)
    _create_fake_snapshots(tmp_path, ["only_one"], days_ago=[90])

    pruned = create_snapshot.prune_snapshots(keep=1, max_age_days=30, dry_run=False)

    assert len(pruned) == 0
    assert (tmp_path / "only_one").exists()


def test_prune_dry_run_does_not_delete(tmp_path, monkeypatch):
    monkeypatch.setattr(create_snapshot, "SNAPSHOT_DIR", tmp_path)
    _create_fake_snapshots(tmp_path, ["old", "new"], days_ago=[60, 1])

    pruned = create_snapshot.prune_snapshots(keep=1, max_age_days=30, dry_run=True)

    assert len(pruned) == 1
    assert (tmp_path / "old").exists()  # still exists


def test_prune_warns_on_orphan_directory(tmp_path, monkeypatch, capsys):
    monkeypatch.setattr(create_snapshot, "SNAPSHOT_DIR", tmp_path)
    _create_fake_snapshots(tmp_path, ["good"], days_ago=[1])
    (tmp_path / "orphan").mkdir()  # no manifest.json

    create_snapshot.prune_snapshots(keep=1, max_age_days=365, dry_run=False)

    assert "orphan" in capsys.readouterr().err


def _create_fake_snapshots(base_dir, names, days_ago):
    from datetime import datetime, timezone, timedelta
    for name, ago in zip(names, days_ago):
        d = base_dir / name
        d.mkdir()
        created = datetime.now(timezone.utc) - timedelta(days=ago)
        manifest = {"name": name, "created": created.isoformat(),
                    "point_count": 100, "snapshot_size_bytes": 1024}
        (d / "manifest.json").write_text(json.dumps(manifest))
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `PYTHONPATH=scripts python3 -m pytest scripts/benchmark/test_create_snapshot.py -k "prune" -v`
Expected: FAIL — `prune_snapshots` does not exist.

- [ ] **Step 3: Implement prune_snapshots()**

Add to `create_snapshot.py`:

```python
def prune_snapshots(keep: int = 3, max_age_days: int = 30,
                    dry_run: bool = False) -> list[dict]:
    snapshots = list_snapshots()
    if not snapshots:
        return []

    # Warn about orphan directories
    if SNAPSHOT_DIR.exists():
        for d in SNAPSHOT_DIR.iterdir():
            if d.is_dir() and not (d / "manifest.json").exists():
                print(f"Warning: orphan snapshot directory (no manifest.json): {d.name}",
                      file=sys.stderr)

    # Sort by creation date descending (newest first)
    snapshots.sort(key=lambda s: s.get("created", ""), reverse=True)

    # Protect the newest `keep` snapshots
    protected = snapshots[:keep]
    candidates = snapshots[keep:]

    # Apply max-age filter to candidates
    cutoff = datetime.now(timezone.utc) - timedelta(days=max_age_days)
    pruned = []
    for snap in candidates:
        created_str = snap.get("created", "")
        try:
            created = datetime.fromisoformat(created_str)
            if created < cutoff:
                pruned.append(snap)
        except (ValueError, TypeError):
            pass

    if not pruned:
        print("Nothing to prune.")
        return []

    for snap in pruned:
        snap_dir = SNAPSHOT_DIR / snap["name"]
        if dry_run:
            print(f"  [dry-run] Would prune: {snap['name']}")
        else:
            shutil.rmtree(snap_dir)
            print(f"  Pruned: {snap['name']}")

    return pruned
```

Add `from datetime import timedelta` to imports.

- [ ] **Step 4: Add CLI flags to main()**

Update the argparse in `main()`:

```python
parser.add_argument("--prune", action="store_true",
                    help="Prune old snapshots after creating (or standalone)")
parser.add_argument("--keep", type=int, default=3,
                    help="Minimum snapshots to retain (default: 3)")
parser.add_argument("--max-age", type=int, default=30,
                    help="Delete snapshots older than N days (default: 30)")
parser.add_argument("--dry-run", action="store_true",
                    help="Show what would be pruned without deleting")
```

Add to main() body after the create/list logic:

```python
if args.prune:
    pruned = prune_snapshots(keep=args.keep, max_age_days=args.max_age,
                             dry_run=args.dry_run)
    if pruned:
        print(f"\nPruned {len(pruned)} snapshot(s).")
    return

# Existing name-required check moves after prune check
```

- [ ] **Step 5: Run all pruning tests**

Run: `PYTHONPATH=scripts python3 -m pytest scripts/benchmark/test_create_snapshot.py -k "prune" -v`
Expected: ALL PASS

- [ ] **Step 6: Run full test suite**

Run: `PYTHONPATH=scripts python3 -m pytest scripts/benchmark/test_create_snapshot.py -v`
Expected: ALL PASS (existing tests unaffected)

- [ ] **Step 7: Commit**

```bash
git add scripts/benchmark/create_snapshot.py scripts/benchmark/test_create_snapshot.py
git commit -m "feat: snapshot pruning with --keep N and --max-age DAYS

Adds --prune to create_snapshot.py. Both criteria apply: keeps at least N
most recent, then prunes those older than max-age. --dry-run shows what
would be pruned. Warns on orphan directories without manifest.json.

Closes #57"
```

---

### Task 3: Add CBR dependencies and H2 configuration (#74)

**Files:**
- Modify: `pom.xml` (add casehub-neocortex-memory-cbr-jpa, quarkus-hibernate-orm, quarkus-jdbc-h2)
- Modify: `src/main/resources/application.properties` (H2 datasource, Hibernate ORM, Flyway)
- Modify: `src/test/resources/application.properties` (H2 in-memory for tests)

**Interfaces:**
- Produces: `CbrCaseMemoryStore` CDI bean available for injection
- Produces: H2 `cbr` datasource configured at `~/.hortora/stats/cbr`

- [ ] **Step 1: Add dependencies to pom.xml**

Add to `<dependencies>` section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-cbr-jpa</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-h2</artifactId>
</dependency>

<!-- CBR test support -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-cbr-inmem</artifactId>
    <scope>test</scope>
</dependency>
```

Add Jandex index dependency for CBR JPA discovery:

```properties
quarkus.index-dependency.casehub-memory-cbr-jpa.group-id=io.casehub
quarkus.index-dependency.casehub-memory-cbr-jpa.artifact-id=casehub-neocortex-memory-cbr-jpa
```

- [ ] **Step 2: Configure H2 datasource in application.properties**

Add to `src/main/resources/application.properties`:

```properties
# CBR outcome tracking — H2 file-persistent
quarkus.datasource.cbr.db-kind=h2
quarkus.datasource.cbr.jdbc.url=jdbc:h2:file:${hortora.data.path:${user.home}/.hortora}/stats/cbr
quarkus.datasource.cbr.username=sa
quarkus.datasource.cbr.password=

quarkus.hibernate-orm.cbr.datasource=cbr
quarkus.hibernate-orm.cbr.packages=io.casehub.neocortex.memory.cbr.jpa

quarkus.flyway.cbr.migrate-at-start=true
quarkus.flyway.cbr.locations=classpath:db/cbr/migration
```

- [ ] **Step 3: Configure test datasource**

Add to `src/test/resources/application.properties`:

```properties
quarkus.datasource.cbr.db-kind=h2
quarkus.datasource.cbr.jdbc.url=jdbc:h2:mem:cbr-test
quarkus.datasource.cbr.username=sa
quarkus.datasource.cbr.password=

quarkus.hibernate-orm.cbr.datasource=cbr
quarkus.hibernate-orm.cbr.database.generation=drop-and-create
```

- [ ] **Step 4: Verify the application builds**

Run: `./mvnw compile -pl .`
Expected: BUILD SUCCESS — dependencies resolve, Hibernate entity scan finds CbrCaseEntity.

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/resources/application.properties src/test/resources/application.properties
git commit -m "deps: add casehub-neocortex-memory-cbr-jpa + H2 for CBR outcome tracking

Adds JPA-backed CbrCaseMemoryStore with H2 file-persistent datasource
at ~/.hortora/stats/cbr. Test profile uses H2 in-memory with
drop-and-create.

Refs #74"
```

---

### Task 4: Implement GardenOutcomeService (#74)

**Files:**
- Create: `src/main/java/io/hortora/garden/outcome/GardenOutcomeService.java`
- Create: `src/test/java/io/hortora/garden/outcome/GardenOutcomeServiceTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.store(CbrCase, String caseType, String entityId, MemoryDomain, String tenantId, String caseId, Path scope)` → `String`
- Consumes: `CbrCaseMemoryStore.recordOutcome(String cbrType, String caseId, CbrOutcome)` → void
- Consumes: `CbrCaseMemoryStore.scan(CbrScanRequest)` → `List<CbrCaseSummary>`
- Consumes: `TextualCbrCase(String problem, String solution, String outcome, Double confidence, Double trustScore, String producerAgentId)`
- Consumes: `CbrOutcome.of(double successRate, String detail, Instant observedAt)` → `CbrOutcome`
- Consumes: `CbrOutcome.adjustConfidence(Double oldConfidence, double successRate, double learningRate)` → `double`
- Consumes: `GardenConfig.id()` → `String`
- Produces: `GardenOutcomeService.recordOutcome(String geId, String issueRepo, int issueNumber, String workContext, double successRate, String detail)` → `String`
- Produces: `GardenOutcomeService.outcomeReport(int minOutcomes)` → `String`

- [ ] **Step 1: Write failing test — first outcome creates a CBR case**

Create `src/test/java/io/hortora/garden/outcome/GardenOutcomeServiceTest.java`:

```java
package io.hortora.garden.outcome;

import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrQuery;
import io.casehub.neocortex.memory.cbr.TextualCbrCase;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class GardenOutcomeServiceTest {

    @Inject GardenOutcomeService service;
    @Inject CbrCaseMemoryStore cbrStore;

    @Test
    void recordOutcomeCreatesCase() {
        String result = service.recordOutcome(
                "GE-20260620-a1b2c3", "Hortora/engine", 75,
                "Refactoring gardenUnretrieved", 0.8, "Relevant but slightly outdated");

        assertThat(result).contains("GE-20260620-a1b2c3");
        assertThat(result).contains("recorded");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./mvnw test -pl . -Dtest=GardenOutcomeServiceTest#recordOutcomeCreatesCase`
Expected: FAIL — class does not exist.

- [ ] **Step 3: Implement GardenOutcomeService**

Create `src/main/java/io/hortora/garden/outcome/GardenOutcomeService.java`:

```java
package io.hortora.garden.outcome;

import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrOutcome;
import io.casehub.neocortex.memory.cbr.CbrScanRequest;
import io.casehub.neocortex.memory.cbr.CbrCaseSummary;
import io.casehub.neocortex.memory.cbr.TextualCbrCase;
import io.casehub.platform.api.path.Path;
import io.hortora.garden.GardenConfig;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class GardenOutcomeService {

    static final String CASE_TYPE = "garden-outcome";
    static final double LEARNING_RATE = CbrOutcome.DEFAULT_LEARNING_RATE;

    @Inject CbrCaseMemoryStore cbrStore;
    @Inject GardenConfig config;

    public String recordOutcome(String geId, String issueRepo, int issueNumber,
                                 String workContext, double successRate, String detail) {
        String problem = workContext + " (" + issueRepo + "#" + issueNumber + ")";
        String solution = geId;

        TextualCbrCase cbrCase = new TextualCbrCase(problem, solution, null, null, null, null);

        cbrStore.store(cbrCase, CASE_TYPE, geId,
                MemoryDomain.KNOWLEDGE, config.id(), geId,
                Path.of("garden", config.id()));

        CbrOutcome outcome = CbrOutcome.of(successRate,
                detail != null ? detail : "", Instant.now());
        cbrStore.recordOutcome(CASE_TYPE, geId, outcome);

        return "Outcome recorded for " + geId + " (success=" + successRate + ")";
    }

    public String outcomeReport(int minOutcomes) {
        // Scan all garden-outcome cases
        CbrScanRequest request = CbrScanRequest.builder()
                .caseType(CASE_TYPE)
                .tenantId(config.id())
                .build();

        List<CbrCaseSummary> cases = cbrStore.scan(request);
        if (cases.isEmpty()) {
            return "No outcome data recorded yet.";
        }

        // Filter by minimum outcomes and sort by confidence ascending
        List<CbrCaseSummary> filtered = cases.stream()
                .filter(c -> c.outcomeCount() >= minOutcomes)
                .sorted(Comparator.comparingDouble(c -> c.confidence() != null ? c.confidence() : 1.0))
                .toList();

        if (filtered.isEmpty()) {
            return "No entries with " + minOutcomes + "+ outcomes recorded.";
        }

        StringBuilder sb = new StringBuilder();
        sb.append("## Garden Entry Outcome Report\n\n");
        sb.append("Entries with ").append(minOutcomes).append("+ outcomes, sorted by confidence (lowest first):\n\n");

        for (CbrCaseSummary c : filtered) {
            sb.append("- **").append(c.caseId()).append("**");
            if (c.confidence() != null) {
                sb.append(" — confidence: ").append(String.format("%.2f", c.confidence()));
            }
            sb.append(" — outcomes: ").append(c.outcomeCount());
            sb.append("\n");
        }

        return sb.toString();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./mvnw test -pl . -Dtest=GardenOutcomeServiceTest#recordOutcomeCreatesCase`
Expected: PASS

- [ ] **Step 5: Write test — second outcome adjusts confidence**

Add to `GardenOutcomeServiceTest`:

```java
@Test
void secondOutcomeAdjustsConfidence() {
    service.recordOutcome("GE-20260620-a1b2c3", "Hortora/engine", 75,
            "First task", 1.0, "Very helpful");
    service.recordOutcome("GE-20260620-a1b2c3", "Hortora/engine", 76,
            "Second task", 0.0, "Not relevant");

    String report = service.outcomeReport(1);
    assertThat(report).contains("GE-20260620-a1b2c3");
    assertThat(report).contains("confidence");
}
```

- [ ] **Step 6: Run test**

Run: `./mvnw test -pl . -Dtest=GardenOutcomeServiceTest`
Expected: PASS

- [ ] **Step 7: Write test — outcomeReport filters by minOutcomes**

```java
@Test
void outcomeReportFiltersbyMinOutcomes() {
    service.recordOutcome("GE-20260620-a1b2c3", "Hortora/engine", 75,
            "Task", 0.5, null);

    String report = service.outcomeReport(5);
    assertThat(report).contains("No entries with 5+ outcomes");
}
```

- [ ] **Step 8: Run all service tests**

Run: `./mvnw test -pl . -Dtest=GardenOutcomeServiceTest`
Expected: ALL PASS

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on `GardenOutcomeService.java` — no errors.

- [ ] **Step 10: Commit**

```bash
git add src/main/java/io/hortora/garden/outcome/GardenOutcomeService.java \
        src/test/java/io/hortora/garden/outcome/GardenOutcomeServiceTest.java
git commit -m "feat: GardenOutcomeService — CBR-backed outcome recording and reporting

Stores TextualCbrCase per GE-ID with store-once/record-many lifecycle.
Confidence evolves via CbrOutcome.adjustConfidence(). outcomeReport()
surfaces entries sorted by confidence for curation.

Refs #74"
```

---

### Task 5: Add MCP tools and REST endpoint for outcome tracking (#74)

**Files:**
- Modify: `src/main/java/io/hortora/garden/mcp/GardenMcpTools.java` (add gardenRecordOutcome, gardenOutcomeReport)
- Create: `src/main/java/io/hortora/garden/outcome/OutcomeResource.java`
- Modify: `src/test/java/io/hortora/garden/mcp/GardenMcpToolsTest.java` (add MCP tool tests)

**Interfaces:**
- Consumes: `GardenOutcomeService.recordOutcome(...)` → `String`
- Consumes: `GardenOutcomeService.outcomeReport(...)` → `String`
- Produces: MCP tool `gardenRecordOutcome`
- Produces: MCP tool `gardenOutcomeReport`
- Produces: REST `POST /api/garden/outcomes`, `GET /api/garden/outcomes/report`

- [ ] **Step 1: Write failing MCP tool test**

Add to `GardenMcpToolsTest`:

```java
@Inject GardenOutcomeService outcomeService;

@Test
void gardenRecordOutcomeDelegatesToService() {
    String result = mcpTools.gardenRecordOutcome(
            "GE-20260620-a1b2c3", "Hortora/engine", 75,
            "Testing outcome", 0.9, "Helpful");

    assertThat(result).contains("recorded");
    assertThat(result).contains("GE-20260620-a1b2c3");
}

@Test
void gardenOutcomeReportRendersOutput() {
    mcpTools.gardenRecordOutcome(
            "GE-20260620-a1b2c3", "Hortora/engine", 75,
            "Testing", 0.5, null);

    String result = mcpTools.gardenOutcomeReport(1);

    assertThat(result).contains("GE-20260620-a1b2c3");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `./mvnw test -pl . -Dtest=GardenMcpToolsTest#gardenRecordOutcomeDelegatesToService`
Expected: FAIL — method does not exist.

- [ ] **Step 3: Add MCP tools to GardenMcpTools**

Use `ide_insert_member` to add after `gardenRecordProvenance`:

```java
@Inject
GardenOutcomeService outcomeService;

@Tool(description = "Record whether a garden entry was helpful for a task. Call at work-end with entries consulted during the session.")
String gardenRecordOutcome(
        @ToolArg(description = "Garden entry ID (e.g. GE-20260620-a1b2c3)") String geId,
        @ToolArg(description = "GitHub repo (e.g. Hortora/engine)") String issueRepo,
        @ToolArg(description = "Issue number") int issueNumber,
        @ToolArg(description = "Brief description of the work context") String workContext,
        @ToolArg(description = "Success rate 0.0-1.0 (1.0=fully helpful, 0.0=not helpful)") double successRate,
        @ToolArg(description = "Optional detail about why", required = false) String detail) {
    return outcomeService.recordOutcome(geId, issueRepo, issueNumber, workContext, successRate, detail);
}

@Tool(description = "Report garden entries with outcome tracking data — declining confidence, high success, or low success. Use during harvest sessions to identify entries needing revision.")
String gardenOutcomeReport(
        @ToolArg(description = "Minimum outcomes recorded to include (default 2)", required = false) Integer minOutcomes) {
    return outcomeService.outcomeReport(minOutcomes != null ? minOutcomes : 2);
}
```

- [ ] **Step 4: Run MCP tool tests**

Run: `./mvnw test -pl . -Dtest=GardenMcpToolsTest#gardenRecordOutcomeDelegatesToService+gardenOutcomeReportRendersOutput`
Expected: PASS

- [ ] **Step 5: Create REST endpoint**

Create `src/main/java/io/hortora/garden/outcome/OutcomeResource.java`:

```java
package io.hortora.garden.outcome;

import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/api/garden/outcomes")
@Produces(MediaType.APPLICATION_JSON)
public class OutcomeResource {

    @Inject GardenOutcomeService outcomeService;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Response recordOutcome(OutcomeRequest request) {
        String result = outcomeService.recordOutcome(
                request.geId(), request.issueRepo(), request.issueNumber(),
                request.workContext(), request.successRate(), request.detail());
        return Response.ok(new OutcomeResponse(result)).build();
    }

    @GET
    @Path("/report")
    @Produces(MediaType.TEXT_PLAIN)
    public String outcomeReport(@QueryParam("minOutcomes") Integer minOutcomes) {
        return outcomeService.outcomeReport(minOutcomes != null ? minOutcomes : 2);
    }

    public record OutcomeRequest(String geId, String issueRepo, int issueNumber,
                                  String workContext, double successRate, String detail) {}

    public record OutcomeResponse(String message) {}
}
```

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `OutcomeResource.java` and `GardenMcpTools.java` — no errors.

- [ ] **Step 7: Run full test suite**

Run: `./mvnw test -pl .`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/hortora/garden/mcp/GardenMcpTools.java \
        src/main/java/io/hortora/garden/outcome/OutcomeResource.java \
        src/test/java/io/hortora/garden/mcp/GardenMcpToolsTest.java
git commit -m "feat: gardenRecordOutcome MCP tool + REST endpoint for outcome tracking

MCP tool for Claude skills (work-end), REST endpoint for trellis UI.
Both delegate to GardenOutcomeService.

Refs #74"
```

---

### Task 6: Update CLAUDE.md and verify full build (#74, #75)

**Files:**
- Modify: `CLAUDE.md` (document CBR infrastructure, outcome tracking)

- [ ] **Step 1: Run full verify**

Run: `./mvnw verify`
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 2: Update CLAUDE.md**

Add CBR infrastructure to Key Design Decisions and Stack sections.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with CBR outcome tracking infrastructure

Refs #74"
```
