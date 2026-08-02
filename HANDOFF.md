*Updated: neocortex#190, neocortex#195 closed — removed from backlog.*

# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-02)

### Branch `issue-58-rag-shadow-harness` — landed on main

**Epic #72: gardenSearch retrieval quality and reliability.** 8 commits, 13 new tests (148→161), 10 issues closed (#62-#71), 8 real-world shadow comparison reports.

**Retrieval quality fixes:**
- BM25 keyword separation (#64) — keywords go to BM25 via `RetrievalQuery.text()`, NL+keywords to dense/sparse via `expandedText`. Previously-missing entries now surface at position #2.
- Adaptive filter bypass (#67/#68) — score-floor and gap-trimming disabled when keywords present. Fixed 2→16 results for broad searches.
- See Also expansion (#70) — `GardenMetadataExtractor` parses "See also" cross-references (922 entries, 3500 refs). Query-time expansion fetches adjacent entries not in initial results. **Requires `gardenReindex()` after deploy to populate see_also metadata in Qdrant.**
- Tool description (#66) — forces Claude to provide keywords. 100% adoption in post-deploy sessions.

**Reliability fixes:**
- Qdrant graceful degradation (#65) — returns diagnostic message instead of MCP -32603.
- Startup readiness probe (#69) — `CollectionMigration.waitForQdrant()` retries 5x with 2s delay. Fixes gRPC TRANSIENT_FAILURE when Qdrant starts after engine.
- Shadow hook debouncer (#71) — fixed PYTHONPATH symlink resolution, added stderr logging.
- Cursor persistence — moved from tmpdir to `~/.hortora/cursors/` (survives reboots).
- Periodic reconcile — `ReconcileScheduler` every 6h catches orphaned Qdrant entries.

## Post-Land Action Required

**Run `gardenReindex()` after deploying to populate See Also metadata.** The `see_also` and `see_also_ids` fields are extracted at index time but require a reindex since existing entries were indexed without them. Without this, the #70 See Also expansion won't find adjacent entries.

## Open Issues by Epic

### Epic #72 — gardenSearch quality & reliability (open)

| # | Title | Repo | Effort | Notes |
|---|-------|------|--------|-------|
| #58 | Shadow comparison harness | engine | — | Ongoing measurement — 8 reports in `docs/comparison/shadow-session-reports.md` |

### Soredium skill changes (no epic)

| # | Title | Effort | Notes |
|---|-------|--------|-------|
| #60 | Dual garden search (grep + MCP) for comparison data | S | Skill change — run both search paths for comparison |
| #61 | Remove dual search after evaluation | XS | After #58 concludes — depends on grep retirement decision |

### Independent

| # | Title | Effort | Notes |
|---|-------|--------|-------|
| #57 | Snapshot pruning/rotation for ~/.hortora/snapshots | S | Housekeeping script |
| #45 | Subagent-mediated garden retrieval | M | Architecture — reduce context impact on main LLM |
| #74 | Garden entry outcome tracking via CBR | M | Close the usage feedback loop |
| #75 | Refactor gardenUnretrieved to use RetrievalAnalyzer | S | neocortex alignment |

## Shadow Comparison Results Summary

8 reports across real working sessions show consistent MCP advantage:

| Scenario | MCP precision | Grep precision | Verdict |
|----------|--------------|----------------|---------|
| Broad domain (tmux) | 90% in top 16 | ~30% in 22 | MCP wins after #67/#68 fix |
| Cross-project terms | 16 results | 1 result | MCP strict superset |
| Polysemous terms (MCP, principal) | 90% in 50 | 5% in 350 | MCP categorically better |
| Adjacent knowledge | Missed 3 critical | Found them | Grep still wins here — #70 addresses this |
| Qdrant down | 0 | Fallback works | Grep needed as fallback (#65 adds diagnostic) |

**Grep is not ready to retire.** Adjacent knowledge gap (Report 5) and Qdrant-down fallback are still real. Keep both paths.

## Key References

| Resource | Location |
|---|---|
| Shadow session reports | `docs/comparison/shadow-session-reports.md` |
| Epic #72 (full tracker) | `gh issue view 72 --repo Hortora/engine` |
| Dedup scan results | `scripts/dedup-scan-results.json` |
| Dedup scan script | `scripts/dedup_scan.py` |
| Benchmark v2 methodology | `docs/comparison/benchmark-v2.md` |
