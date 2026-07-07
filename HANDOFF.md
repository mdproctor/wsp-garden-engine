# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-07)

### Branch `issue-41-two-stage-overfetch-rerank` closed → main

**Closes #41.** Widened ColBERT MAX_SIM reranking pool from 10 to 30 candidates (`casehub.rag.retrieval.rerank-top-n=30`). Benchmark: +74% relevant entries (153→266), 27/28 scenarios improved, 1 trivial regression (-1 entry, within HyDE noise).

Filed neocortex#121 for `RerankingCaseRetriever` cross-encoder decorator — client-side reranking using `max(limit, rerankPoolSize)` overfetch pattern. ONNX model already at `~/.hortora/models/reranker/`. Blocked on neocortex implementation.

## Immediate Next Step

**Wire cross-encoder reranking when neocortex#121 lands** — add `@Inference("reranker")` config, enable the decorator, benchmark A+B vs A-only (`rerank-topn-30.json` is the A baseline). Until then, engine-side HyDE prompt tuning (#40 follow-on) is the next unblocked work.

## Neocortex Dependencies

| Neocortex # | Status | What engine needs |
|---|---|---|
| #105 | **Open** | Retrieval tracking SPI — blocks engine #24 |
| #115 | **Open** | Epic: regression-free query expansion |
| #116 | **Open** | Always include original query in expanded set — eliminates HyDE regressions |
| #117 | **Open** | Per-leg embedding separation (supersedes #113) |
| #118 | **Open** | Expansion drift metrics + auto-fallback |
| #121 | **Open** | RerankingCaseRetriever cross-encoder decorator — blocks engine A+B benchmark |

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#24** | Retrieval frequency tracking | M | Med | neocortex #105 | P3 |

## Key References

| Resource | Location |
|---|---|
| rerankTopN=30 benchmark | `scripts/benchmark/results/rerank-topn-30.json` |
| HyDE baseline benchmark | `scripts/benchmark/results/hyde-session.json` |
| Four-signal baseline | `scripts/benchmark/results/bge-m3-four-signal.json` |
| Blog: rerankTopN bottleneck | `blog/2026-07-07-mdp01-reranktopn-bottleneck.md` |
