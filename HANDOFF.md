# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-10)

### Branch `issue-46-retrieval-quality-improvements` closed → main

**Closes #48, #47. Epic #46 closed.** Scored all 138 unscored benchmark entries — the reported 3pp precision gap (90% → 87%) was a scoring artifact. Three-leg drops to 86% with complete scoring; BGE-M3 stays at 87% (+1pp ahead). Root-caused all 8 regressions as intrinsic to embedding space differences between model stacks — no engine-side fix available. Also closed #49 (DOMAIN_ABSENCE reclassified as POLYSEMY, diminishing returns) and the parent epic #46.

Garden entry GE-20260709-19a59a submitted: excluding unscored entries from retrieval precision silently inflates precision for noisier methods.

## Immediate Next Step

Pick up #45 (subagent-mediated retrieval — skill-layer work) or #24 (retrieval frequency tracking). Retrieval quality is confirmed good at 87%.

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#24** | Retrieval frequency tracking | M | Med | — | Unblocked |
| **#45** | Subagent-mediated garden retrieval | M | Med | — | Skill-layer work, not engine |
| **#50** | Re-enable HyDE after per-leg separation | M | Med | neocortex #117 | Urgency reduced — no precision gap to close |
| **#51** | Expansion drift metrics integration | S | Med | neocortex #118, #120 | Urgency reduced |

## Key References

| Resource | Location |
|---|---|
| Regression analysis | `docs/comparison/regression-analysis.md` |
| Updated BGE-M3 benchmark | `docs/comparison/bge-m3-benchmark.md` |
| Complete baseline scores | `scripts/benchmark/baseline_scores.json` |
| Garden entry (scoring bias gotcha) | GE-20260709-19a59a |
