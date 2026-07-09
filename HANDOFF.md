# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-09)

### Branch `issue-44-adaptive-ce-filtering` closed → main

**Closes #44.** Adaptive CE score filtering replaces the old `adaptiveExtend()`. Two-layer filtering: score floor (CE < 0 excluded) + gap detection (first CE score drop ≥ 2.0 trims noise tail). `minResults=3` prevents single-result trimming. Dense-only mode preserves existing extension behavior. Benchmark validated: 205 noise entries removed across 28 scenarios with zero loss of relevant (CE > 3.0) entries. Also filed #45 (subagent-mediated retrieval — reduce context impact on main LLM).

Cross-repo: neocortex branch `fix-ce-score-promotion` has CE score promotion fix (uncommitted to main — user reverted). Engine reads `crossEncoderScore` directly, independent of platform fix.

## Immediate Next Step

**#45 — Subagent-mediated garden retrieval.** Now that the retrieval pipeline filters noise, the next layer is a cheaper model (Haiku/Sonnet) to distill results before they reach the main LLM's context. Implementation lives in the skill layer, not the engine.

## Neocortex Dependencies

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#24** | Retrieval frequency tracking | M | Med | — | Unblocked (neocortex #105 closed) |
| **#45** | Subagent-mediated garden retrieval | M | Med | — | Skill-layer work, not engine |

## Key References

| Resource | Location |
|---|---|
| Adaptive filtering spec | `docs/specs/2026-07-09-adaptive-ce-filtering-design.md` |
| Adaptive filtering plan | `docs/plans/2026-07-09-adaptive-ce-filtering.md` |
| Benchmark validation script | `scripts/benchmark/validate_filtering.py` |
| CE pool-50 benchmark data | `scripts/benchmark/results/crossencoder-pool50-scored.json` |
