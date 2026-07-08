# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-08)

### Branch `issue-43-hyde-prompt-tuning` closed → main

**Closes #43.** Cross-encoder reranking wired via `casehub-neocortex-rag-crossencoder` (neocortex #121). Pool-50 sweet spot: +8 highly-relevant entries at 1.15s median latency. HyDE proved net-zero at limit=16 — all three retrieval legs use `searchText()` which replaces the original query; blocked on neocortex #117 (per-leg embedding separation). Tuned prompt + confidence gating preserved for when #117 lands.

## Immediate Next Step

**Adaptive cross-encoder score filtering** — scores are now surfaced in the REST response (`crossEncoderScore` field). 40% of returned entries have negative CE scores (cross-encoder thinks they're irrelevant). Adaptive gap detection on CE scores could trim noise entries. Needs a design decision on variable vs fixed result count — changes the API contract for MCP consumers.

## Neocortex Dependencies

| Neocortex # | Status | What engine needs |
|---|---|---|
| #105 | **Closed** | Retrieval tracking SPI — blocks engine #24 |
| #115 | **Open** | Epic: regression-free query expansion |
| #116 | **Open** | Always include original query in expanded set |
| #117 | **Open** | Per-leg embedding separation — **unlocks HyDE** |
| #118 | **Open** | Expansion drift metrics + auto-fallback |
| #121 | **Closed** | RerankingCaseRetriever — **wired in this session** |

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#24** | Retrieval frequency tracking | M | Med | neocortex #105 (closed — check if unblocked) | P3 |

## Key References

| Resource | Location |
|---|---|
| Cross-encoder pool-50 benchmark | `scripts/benchmark/results/crossencoder-pool50-scored.json` |
| No-HyDE baseline | `scripts/benchmark/results/no-hyde-baseline.json` |
| CE pool sweep (30/50/100) | `scripts/benchmark/results/crossencoder-*.json` |
| HyDE A/B benchmarks | `scripts/benchmark/results/hyde-*.json` |
| Garden entry: @IfBuildProperty + @ConfigMapping | `GE-20260708-055d01` |
