# Hortora engine — Project Handoff

---

## What Just Happened (2026-07-28)

### Branch `issue-50-re-enable-hyde` closed → main

Four HyDE approaches benchmarked, all appeared net-negative. Inverted HyDE infrastructure built (OllamaQueryGenerator, QueryAugmentingExtractor). SessionQueryExpander removed. Neocortex #173 landed. Blog entries and garden entry written.

### Branch `issue-55-score-based-boosting` — open, mid-work

Score boosting implemented (blends entry quality score with CE score). Benchmark showed zero effect at weight=0.1 (CE scores dominate). Then discovered ALL cross-session benchmark comparisons were invalid — garden corpus grew but scoring files were frozen, causing phantom regressions. The -2.2pp attributed to HyDE was from unscored entries, not from HyDE itself. Scored-only precision was stable at ~85% throughout.

`docs/comparison/benchmark-v2.md` documents the methodology fix and invalidates prior measurements.

## Immediate Next Step

Branch `issue-55-score-based-boosting` is open. Run `/work` to continue. The next action: **score ~50-60 new garden entries** in `bge-m3-to-score.json` that appear in benchmark results but aren't scored. Then establish the v2 baseline and run the experiments queue (score boost at 0.5/1.0, HyDE re-evaluation).

To identify unscored entries, run the benchmark on a frozen corpus and check which returned entries have no score. See `docs/comparison/benchmark-v2.md` for the full methodology.

## What's Left

- Score ~50-60 new entries in `bge-m3-to-score.json` · M · Med
- Establish v2 baseline with corrected scoring · S · Low
- Re-benchmark score boosting at weights 0.5, 1.0 · S · Low
- Re-evaluate HyDE against corrected baseline · S · Low
- Implement fixed-corpus snapshot harness (#56) · M · Med

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| **#55** | Score-based boosting in adaptive filter | S | Low | Branch open, code done, needs correct baseline |
| **#56** | Fixed-corpus benchmark snapshots | M | Med | Methodology fix — pair corpus SHA with scoring files |
| **#45** | Subagent-mediated garden retrieval | M | Med | Skill-layer work |

## Key References

| Resource | Location |
|---|---|
| Benchmark v2 methodology | `docs/comparison/benchmark-v2.md` |
| Inverted HyDE design spec | `specs/issue-50-re-enable-hyde/2026-07-24-inverted-hyde-design.md` |
| Blog: The HyDE Wall | `blog/2026-07-25-mdp01-the-hyde-wall.md` |
| Garden entry | `GE-20260725-cae3ad` — expansion harms strong retrievers |
| Benchmark results | `scripts/benchmark/results/` — 10 result files from this session |
