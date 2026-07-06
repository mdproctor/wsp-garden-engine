# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-06)

### Comprehensive retrieval benchmark

Ran 5 config variants against 14 scenarios (KW + NL queries). Built analysis tooling and a definitive grep vs gardenSearch comparison. Key findings:

- **Fusion tuning is exhausted**: CC (+0pp), DBSF (-1pp), ColBERT scalar quant (-1pp), Matryoshka 768 (0pp) — nothing beats RRF (k=60)
- **gardenSearch beats grep 11/14 scenarios** at 87% precision vs 66%
- **Root cause of 87% plateau**: ranking noise pushes relevant entries to rank 9-12, outside the top-8 window. The "DOMAIN_ABSENCE" failure mode was a misdiagnosis — entries exist, they're just buried in noise
- **#33 closed** — RRF stays, no fusion change warranted

### Issues filed

- neocortex#104 (closed) — configurable fusion strategy (RRF/CC/DBSF)
- neocortex#105 (open) — retrieval tracking SPI

### Artifacts produced

- `docs/comparison/fusion-benchmark.md` — CC/DBSF/ColBERT-SQ/Matryoshka vs RRF
- `docs/comparison/grep-vs-gardensearch.md` — definitive grep vs gardenSearch comparison
- `scripts/benchmark/analyze_fusion.py` — multi-variant comparison analysis
- `scripts/benchmark/analyze_grep_comparison.py` — grep head-to-head analysis
- `scripts/benchmark/results/bge-m3-cc.json`, `bge-m3-dbsf.json`, `bge-m3-colbert-sq.json`, `bge-m3-matryoshka-768.json`

## Immediate Next Step

**#39 — increase default limit from 8 to 16.** This is the highest-impact change identified. One-line change in `SearchResource.java`, then re-run benchmark to measure impact. The regression from three-leg (94%) to BGE-M3 (87%) is caused by relevant entries at rank 9-12 being invisible at limit=8.

## Open Issues — Retrieval Quality Roadmap

| # | Title | Scale | Complexity | Blocked by | Priority | Notes |
|---|-------|-------|------------|------------|----------|-------|
| **#39** | Increase default limit 8→16 | XS | Low | — | **P1** | One-line change + benchmark. Addresses the top-8 ranking bottleneck |
| **#42** | Fix DOMAIN_ABSENCE failure mode labels | XS | Low | — | P1 | Correct misdiagnosis in benchmark scenarios |
| **#40** | Wire HyDE query expansion | M | Med | ChatModel provider | P2 | Neocortex has HydeCaseRetriever; needs LLM wired. 3-17% improvement expected on VOCABULARY_GAP |
| **#41** | Two-stage overfetch + rerank | M | Med | — | P2 | Fetch 20-30, rerank to limit. Research shows +7-8% accuracy |
| **#24** | Retrieval frequency tracking | M | Med | neocortex#105 | P3 | Usage-based corpus curation. Blocked on neocortex SPI |

## Open Issues — Other

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#33** | ~~CC fusion test~~ | — | — | — | **CLOSED** — RRF stays |

## Key Insight (record for future sessions)

The retrieval pipeline (embedding model, fusion strategy, quantization) is **not the bottleneck**. All tuning attempts net zero. The bottleneck is:
1. **Result window size** — limit=8 hides relevant entries at rank 9-12
2. **Ranking precision** — noise entries displace relevant ones within the window
3. **Vocabulary gap** — query terms don't overlap with entry vocabulary (fixable via HyDE)

Don't chase new models or fusion strategies. The next gains come from: wider result window (#39), query expansion (#40), and smarter reranking (#41).

## Key References

| Resource | Location |
|---|---|
| Retrieval quality memory | `~/.claude/projects/.../memory/project_retrieval_quality_findings.md` |
| Fusion benchmark report | `docs/comparison/fusion-benchmark.md` |
| grep vs gardenSearch report | `docs/comparison/grep-vs-gardensearch.md` |
| BGE-M3 baseline report | `docs/comparison/bge-m3-benchmark.md` |
| Retrieval research & roadmap | `docs/comparison/retrieval-research.md` |
| neocortex#104 (fusion strategy) | casehubio/neocortex#104 (closed) |
| neocortex#105 (retrieval tracking) | casehubio/neocortex#105 (open) |
