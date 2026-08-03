# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-03)

### Branch `issue-75-unretrieved-pruning-cbr` — landed on main as `1c112d8`

**Three issues closed: #75, #57, #74.** 194 tests (33 new), 11 files changed, 492 insertions.

**#75 — gardenUnretrieved refactor:** Replaced ~40 lines of inline set-diffing with `RetrievalAnalyzer.qualitySignals()`. Adds HIGH_RETRIEVAL_LOW_QUALITY signal section (dormant until feedback data exists via #74).

**#57 — Snapshot pruning:** `--prune` on `create_snapshot.py` with `--keep N` and `--max-age DAYS`. Both criteria apply. `--dry-run` supported. Orphan directories warned.

**#74 — CBR outcome tracking:** `JpaCbrCaseMemoryStore` + H2 file-persistent at `~/.hortora/stats/cbr`. `GardenOutcomeService` stores one `TextualCbrCase` per GE-ID with store-once/record-many lifecycle. Confidence evolves via `CbrOutcome.adjustConfidence()`. MCP tool `gardenRecordOutcome` + REST `POST /api/garden/outcomes`. `gardenOutcomeReport` MCP tool + REST `GET /api/garden/outcomes/report`.

## Post-Land Action Required

**Run `gardenReindex()` to populate See Also metadata** (from previous branch #58). The `see_also` and `see_also_ids` fields require a reindex since existing entries were indexed without them.

## Open Issues by Epic

### Epic #72 — gardenSearch quality & reliability (open)

| # | Title | Repo | Effort | Notes |
|---|-------|------|--------|-------|
| #58 | Shadow comparison harness | engine | — | Ongoing measurement |

### Soredium skill changes (no epic)

| # | Title | Effort | Notes |
|---|-------|--------|-------|
| #60 | Dual garden search (grep + MCP) for comparison data | S | Skill change |
| #61 | Remove dual search after evaluation | XS | After #58 concludes |

### Independent

| # | Title | Effort | Notes |
|---|-------|--------|-------|
| #45 | Subagent-mediated garden retrieval | M | Architecture — reduce context impact |

## Key References

| Resource | Location |
|---|---|
| Design spec | `docs/specs/issue-75-unretrieved-pruning-cbr/` |
| Blog entry | `blog/2026-08-03-mdp01-closing-the-feedback-loop.md` |
| Shadow session reports | `docs/comparison/shadow-session-reports.md` |
| Epic #72 | `gh issue view 72 --repo Hortora/engine` |
