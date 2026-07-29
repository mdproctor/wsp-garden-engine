*Updated: #56 closed — removed from "What Just Shipped" section (already landed on main).*

# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-29)

### Branch `issue-56-fixed-corpus-benchmark` closed → main

**Closes #56.** Qdrant snapshot-based corpus freezing for the benchmark harness. `create_snapshot.py` creates and downloads named snapshots with SHA-256 integrity, Qdrant version tracking, and scoring drift detection. `run_queries.py` gains `--corpus-snapshot <name>` to restore a snapshot before running, plus an unscored-entry warning (>5% threshold) that catches the exact scoring gap that produced phantom regressions in #50/#55. Shared utilities extracted to `qdrant_utils.py`.

Design spec adversarially reviewed (5 rounds, 20 issues → 17 verified, 3 accepted).

## Immediate Next Step

Score the 18 new garden entries listed in #56's "immediate action" section, then create the first snapshot (`python3 scripts/benchmark/create_snapshot.py v2-baseline`) to establish the reference precision. Alternatively, pick up #45 (subagent-mediated retrieval) — skill-layer work, not engine.

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| **#45** | Subagent-mediated garden retrieval | M | Med | Skill-layer work, not engine |
| **#57** | Snapshot pruning/rotation | S | Low | Manual `rm -rf` until then |

## Key References

| Resource | Location |
|---|---|
| Benchmark v2 methodology | `docs/comparison/benchmark-v2.md` |
| Snapshot design spec | `specs/issue-56-fixed-corpus-benchmark/2026-07-29-fixed-corpus-snapshots-design.md` |
| Design review tracker | `~/adr/hortora-engine/fixed-corpus-snapshots-20260729-015549/tracker.md` |
| HyDE design specs | `specs/issue-50-re-enable-hyde/` |
| Blog: The HyDE Wall | `blog/2026-07-25-mdp01-the-hyde-wall.md` |
| Garden entry | `GE-20260725-cae3ad` — expansion harms strong retrievers |
