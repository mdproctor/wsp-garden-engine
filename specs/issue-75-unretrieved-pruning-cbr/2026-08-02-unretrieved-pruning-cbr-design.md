# gardenUnretrieved Refactor, Snapshot Pruning, CBR Outcome Tracking — Design Spec

**Issues:** #75, #57, #74
**Date:** 2026-08-02

## Implementation Order

1. **#75** — refactor gardenUnretrieved to RetrievalAnalyzer (pure refactor, no new deps)
2. **#57** — snapshot pruning script (independent Python work)
3. **#74** — CBR outcome tracking (new deps: JPA + H2 + memory-cbr-jpa)

## #75 — Refactor gardenUnretrieved to use RetrievalAnalyzer

### Change

Replace ~40 lines of inline set-diffing and stale-detection in `GardenMcpTools.gardenUnretrieved()` with a single call to `RetrievalAnalyzer.qualitySignals(tracker, ingestor, corpus, since, until, thresholds)`.

### What stays in GardenMcpTools

- `passesMinDaysFilter()` — post-filters NEVER_RETRIEVED signals by GE-ID date (garden-specific)
- `groupByDomain()` — organizes output by domain path prefix
- Markdown formatting — structured text output for MCP consumers

### Signal mapping

| QualitySignal | Current behavior | After refactor |
|---|---|---|
| NEVER_RETRIEVED | Surfaced as "Unretrieved entries" | Same |
| STALE | Surfaced as "Stale entries" | Same |
| HIGH_RETRIEVAL_LOW_QUALITY | Not surfaced | New section: "Low quality entries" — dormant until feedback data exists |

### QualityThresholds mapping

- `staleDays` MCP param → `QualityThresholds.staleWindow(Duration.ofDays(staleDays))`
- Feedback thresholds use `QualityThresholds.defaults()` initially (minRetrievals=3, minFeedback=3, lowQualityRatio=0.7)

### Testing

Existing `GardenMcpToolsTest` gardenUnretrieved tests remain — same inputs, same outputs. Add:
- Test for HIGH_RETRIEVAL_LOW_QUALITY signal rendering (with seeded feedback data)
- Verify `passesMinDaysFilter()` still applies only to NEVER_RETRIEVED signals

## #57 — Snapshot Pruning/Rotation

### Change

Add `--prune` subcommand to `scripts/benchmark/create_snapshot.py`.

### CLI interface

```
python3 scripts/benchmark/create_snapshot.py --prune [--keep N] [--max-age DAYS]
python3 scripts/benchmark/create_snapshot.py <name> --prune  # create then prune
```

- `--keep N` (default 3): always retain the N most recent snapshots
- `--max-age DAYS` (default 30): delete snapshots older than this
- Both criteria apply: a snapshot is pruned only if it exceeds max-age AND removing it still leaves at least `keep` snapshots
- `--dry-run`: show what would be pruned without deleting

### Implementation

- Read `manifest.json` from each snapshot directory for `created` timestamp
- Sort by creation date
- Apply retention policy: protect newest N, then prune those older than max-age
- Delete snapshot directories (`shutil.rmtree`)
- Report what was pruned

### Testing

- `test_create_snapshot.py`: prune with keep=2, prune with max-age, prune with both, dry-run, prune with fewer than keep snapshots (no-op)

## #74 — Garden Entry Outcome Tracking via CBR

### Architecture

Use existing `JpaCbrCaseMemoryStore` from `casehub-neocortex-memory-cbr-jpa` with H2 in file-persistent mode. No external database server.

### New dependencies

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
```

### Configuration

```properties
# H2 file-persistent datasource for CBR
quarkus.datasource.cbr.db-kind=h2
quarkus.datasource.cbr.jdbc.url=jdbc:h2:file:${hortora.data.path:${user.home}/.hortora}/stats/cbr;AUTO_SERVER=TRUE
quarkus.datasource.cbr.username=sa
quarkus.datasource.cbr.password=

quarkus.hibernate-orm.cbr.datasource=cbr
quarkus.hibernate-orm.cbr.packages=io.casehub.neocortex.memory.cbr.jpa

quarkus.flyway.cbr.migrate-at-start=true
quarkus.flyway.cbr.locations=classpath:db/cbr/migration
```

### CBR case mapping

| CBR field | Garden mapping |
|---|---|
| cbrType | `"textual"` (TextualCbrCase) |
| problem | Work context — "Designing retrieval tracking for hortora engine" |
| solution | Garden entry GE-ID + title |
| outcome | SUCCESS / PARTIAL / FAILURE text |
| confidence | Starts at retrieval score, adjusted via `CbrOutcome.adjustConfidence()` |
| entityId | GE-ID |
| caseType | `"garden-outcome"` |
| tenantId | `config.id()` (garden ID) |
| domain | `MemoryDomain.KNOWLEDGE` |

### Service layer — GardenOutcomeService

New `@ApplicationScoped` service backing both MCP and REST:

```java
public class GardenOutcomeService {
    @Inject CbrCaseMemoryStore cbrStore;
    @Inject RetrievalTracker retrievalTracker;

    // Record outcome for a garden entry consulted during work
    public String recordOutcome(String geId, String issueRepo, int issueNumber,
                                 String workContext, double successRate, String detail);

    // Report entries with declining confidence
    public List<OutcomeReport> outcomeReport(int minOutcomes);
}
```

`recordOutcome()`:
1. Creates `TextualCbrCase` with work context as problem, GE-ID + title as solution
2. Stores via `cbrStore.store()`
3. Records outcome via `cbrStore.recordOutcome()` with `CbrOutcome.of(successRate, detail, Instant.now())`
4. Also records via `retrievalTracker.feedback()` to feed RetrievalAnalyzer quality signals

### MCP tool — gardenRecordOutcome

```java
@Tool(description = "Record whether a garden entry was helpful for a task.")
String gardenRecordOutcome(
    @ToolArg(description = "Garden entry ID (e.g. GE-20260620-a1b2c3)") String geId,
    @ToolArg(description = "GitHub repo (e.g. Hortora/engine)") String issueRepo,
    @ToolArg(description = "Issue number") int issueNumber,
    @ToolArg(description = "Brief description of the work context") String workContext,
    @ToolArg(description = "Success rate 0.0-1.0 (1.0=fully helpful, 0.0=not helpful)") double successRate,
    @ToolArg(description = "Optional detail about why", required = false) String detail)
```

### REST endpoint

```
POST /api/garden/outcomes
{
    "geId": "GE-20260620-a1b2c3",
    "issueRepo": "Hortora/engine",
    "issueNumber": 75,
    "workContext": "Refactoring gardenUnretrieved",
    "successRate": 0.8,
    "detail": "Entry was relevant but slightly outdated"
}
```

### MCP tool — gardenOutcomeReport

```java
@Tool(description = "Report garden entries with outcome tracking data — declining confidence, high success, or low success.")
String gardenOutcomeReport(
    @ToolArg(description = "Minimum outcomes recorded to include (default 2)", required = false) Integer minOutcomes)
```

### Testing

- `GardenOutcomeServiceTest`: record outcome creates CBR case + records feedback, outcome report surfaces declining confidence entries, multiple outcomes adjust confidence correctly
- `GardenMcpToolsTest`: gardenRecordOutcome MCP tool delegates to service, gardenOutcomeReport renders formatted output
- Test infrastructure: `InMemoryCbrCaseMemoryStore` as `@Alternative` in test scope

## Out of Scope

- Automatic outcome recording at work-end (skill-layer change, not engine)
- Trellis UI integration (separate repo/issue)
- Supersession tracking when garden entries are updated (future — API exists)
- Migration of SqliteRetrievalTracker to H2 (separate concern, future)
