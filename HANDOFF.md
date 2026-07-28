# Hortora engine — Project Handoff

*Updated: neocortex#178, #179, #180 closed — removed from backlog.*

---

## What Just Shipped (2026-07-28)

### Branch `issue-55-score-based-boosting` closed → main

**Closes #55.** Score-based boosting implemented and benchmarked — zero precision effect (CE scores dominate at 0.1 weight). Investigation then discovered ALL prior cross-session benchmark comparisons were invalid: garden corpus grew (~230 entries) but scoring files were frozen, causing phantom regressions. The -2.9pp attributed to neocortex changes was entirely from unscored entries. Scored-only precision identical at 85.1-85.2%.

Score-boost-weight disabled (set to 0.0). `docs/comparison/benchmark-v2.md` documents the methodology fix. Regression findings posted to casehubio/neocortex#181. Unrecovered blog and HyDE specs cherry-picked from closed branches.

## Immediate Next Step

Pick up #56 (fixed-corpus benchmark snapshots) or #45 (subagent-mediated retrieval). #56 is the natural follow-on — formalises the methodology fix discovered during #55.

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| **#56** | Fixed-corpus benchmark snapshots | M | Med | Methodology fix — pair corpus SHA with scoring files |
| **#45** | Subagent-mediated garden retrieval | M | Med | Skill-layer work, not engine |

## Neocortex Issues Filed

*All closed — neocortex#178, #179, #180 landed.*

## Key References

| Resource | Location |
|---|---|
| Benchmark v2 methodology | `docs/comparison/benchmark-v2.md` |
| HyDE design specs | `specs/issue-50-re-enable-hyde/` |
| Blog: The HyDE Wall | `blog/2026-07-25-mdp01-the-hyde-wall.md` |
| Garden entry | `GE-20260725-cae3ad` — expansion harms strong retrievers |
