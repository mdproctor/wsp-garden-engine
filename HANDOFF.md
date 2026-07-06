# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-06)

### Branch `issue-39-increase-default-limit` closed → `5f023e1` on main

**Closes #39, #42, #33.** Three issues landed in one branch:

1. **#39 — Default limit 8→16.** `SearchResource.DEFAULT_LIMIT` changed from 8 to 16. Benchmark result: +24 relevant entries found (+12%), precision 78% (still +12pp above grep's 66%). Adaptive extension returns ~10 results per query (not 16) due to relevance gap trimming.

2. **#33 — Fusion benchmark complete.** Tested CC, DBSF, ColBERT scalar quantization, and Matryoshka 768-dim — none beats RRF (k=60). RRF stays as the fusion strategy.

3. **#42 — DOMAIN_ABSENCE labels corrected.** The failure mode was misdiagnosed in #27 — entries exist in corpus (grep finds them). Actual failures are POLYSEMY and VOCABULARY_GAP. Garden entry GE-20260706-146e14 captures this insight.

### Artifacts produced

- `docs/comparison/fusion-benchmark.md` — 5-config comparison
- `docs/comparison/grep-vs-gardensearch.md` — definitive grep vs gardenSearch
- `scripts/benchmark/analyze_fusion.py`, `analyze_grep_comparison.py` — analysis tooling
- 5 new benchmark result JSON files in `scripts/benchmark/results/`

## Immediate Next Step

**#40 — Wire HyDE query expansion.** VOCABULARY_GAP is the #1 remaining failure mode. Neocortex already has `HydeCaseRetriever` and `QueryExpandingCaseRetriever` in `rag-expansion`. Engine needs to wire a ChatModel provider and enable the HyDE decorator. Research shows 3-17% BM25 improvement from query expansion.

## Open Issues — Retrieval Quality Roadmap

| # | Title | Scale | Complexity | Blocked by | Priority | Notes |
|---|-------|-------|------------|------------|----------|-------|
| **#40** | Wire HyDE query expansion | M | Med | ChatModel provider | **P1** | Neocortex has HydeCaseRetriever; engine needs LLM wired |
| **#41** | Two-stage overfetch + rerank | M | Med | — | P2 | Fetch 20-30, rerank to limit. +7-8% accuracy expected |
| **#24** | Retrieval frequency tracking | M | Med | neocortex#105 | P3 | Usage-based corpus curation |

## Neocortex Dependencies

| Neocortex # | Status | What engine needs |
|---|---|---|
| #104 | **Closed** | Configurable fusion strategy — used in #33 benchmark |
| #105 | **Open** | Retrieval tracking SPI — blocks engine #24 |

Detailed request for neocortex session written in this session — covers priorities, what doesn't help, and the three things that do (retrieval tracking, HyDE wiring confirmation, overfetch+rerank pattern).

## Key Insight (record for future sessions)

The retrieval pipeline (embedding model, fusion strategy, quantization) is **not the bottleneck**. All tuning attempts net zero. The bottleneck is:
1. **Result window size** — limit=8 hid relevant entries at rank 9-12 (fixed by #39)
2. **Ranking precision** — noise entries displace relevant ones within the window
3. **Vocabulary gap** — query terms don't overlap with entry vocabulary (fix: HyDE #40)

Don't chase new models or fusion strategies. The next gains come from: query expansion (#40) and smarter reranking (#41).

## Key References

| Resource | Location |
|---|---|
| Retrieval quality memory | `~/.claude/projects/.../memory/project_retrieval_quality_findings.md` |
| Fusion benchmark report | `docs/comparison/fusion-benchmark.md` |
| grep vs gardenSearch report | `docs/comparison/grep-vs-gardensearch.md` |
| BGE-M3 baseline report | `docs/comparison/bge-m3-benchmark.md` |
| Retrieval research & roadmap | `docs/comparison/retrieval-research.md` |
| Garden entry on misdiagnosis | GE-20260706-146e14 |
