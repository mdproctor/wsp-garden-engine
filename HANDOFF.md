# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-11)

### Branch `issue-24-retrieval-frequency-tracking` closed → main

**Closes #24.** Added retrieval frequency tracking using the existing `RetrievalTracker` SPI from `casehub-neocortex-rag-api` and `casehub-neocortex-rag-tracking` module. `TrackingCaseRetriever` CDI decorator transparently records every `CaseRetriever.retrieve()` call. `SqliteRetrievalTracker` persists to SQLite (WAL mode, 180-day retention). New `gardenUnretrieved` MCP tool surfaces zero-retrieval and stale entries for harvest-driven curation.

Design review discovered the existing `RetrievalTracker` infrastructure — the original spec proposed a new `RetrievalStatsStore` SPI from scratch. The review rewrote the spec to use what already existed.

Neocortex gap: `SqliteRetrievalTracker.init()` doesn't call `Files.createDirectories()` for the SQLite parent directory — first-run on fresh deployments will fail if `stats/` doesn't exist. One-line fix needed in neocortex.

## Immediate Next Step

Pick up #45 (subagent-mediated retrieval — skill-layer work) or #24's neocortex companion fix (directory creation in `SqliteRetrievalTracker.init()`).

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#45** | Subagent-mediated garden retrieval | M | Med | — | Skill-layer work, not engine |
| **#50** | Re-enable HyDE after per-leg separation | M | Med | neocortex #117 | Urgency reduced |
| **#51** | Expansion drift metrics integration | S | Med | neocortex #118, #120 | Urgency reduced |

## Key References

| Resource | Location |
|---|---|
| Design spec | `docs/specs/2026-07-10-retrieval-frequency-tracking-design.md` |
| Implementation plan | `docs/plans/2026-07-11-retrieval-frequency-tracking.md` |
| Design review workspace | `~/adr/hortora-engine/retrieval-frequency-tracking-20260710-211329/` |
| Neocortex directory creation gap | `SqliteRetrievalTracker.init()` in `casehub-neocortex-rag-tracking` |
