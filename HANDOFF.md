# Hortora engine — Project Handoff

*Updated: 2026-06-29 — #28 closed.*

---

## What's In Flight

Nothing — main is clean.

## Immediate Next Step

Pick up neural-text work: #47 (Qdrant full-text index) is the highest priority — it's the foundation for BM25 keyword matching, the most direct fix for the keyword catastrophe. Start there.

## What Changed This Session

- **SPLADE hybrid benchmark complete** (#28) — diagnostic benchmark across 14 real-world scenarios with three configs (dense-only, dense+SPLADE, full-hybrid). Key findings:
  - SPLADE (`Splade_PP_en_v1`) has zero Java domain vocabulary — expands `ChatModel` to "hotel, beauty, renovation"
  - Two ONNX integration gaps blocked startup (neural-text #51 input names, #52 output rank) — workarounds applied
  - Full hybrid crashed after one query (cross-encoder +170ms overhead)
  - KW embedding instability: 76% result drift from 1.2% corpus growth
  - Dense+SPLADE latency: +15ms overhead, result quality ambiguous (removed noise but displaced 3 score-2 entries)
- **Six neural-text issues filed** (#47-52) covering: full-text index, BM25 retrieval leg, code-domain model eval, payload indexes, ONNX input naming, ONNX output rank
- **Three garden entries** — KW embedding instability, ONNX naming mismatch, SPLADE zero Java vocab
- **Benchmark harness built** — `scripts/benchmark/` with 31 tests, reusable for future model comparisons

## Cross-Module

**neural-text — six issues filed, all can start now:**

| # | Description | Priority |
|---|-------------|----------|
| #47 | Qdrant full-text index on content | Highest — foundation for BM25 |
| #48 | BM25 as third RRF retrieval leg | Second — depends on #47 |
| #49 | Code-domain embedding model evaluation | Third — research |
| #50 | Keyword payload indexes (sourceDocumentId, tenantId) | Housekeeping |
| #51 | OnnxInferenceModel input name validation | Bug fix |
| #52 | OnnxInferenceModel rank-3 output support | Bug fix |

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #29 | Complementary retrieval capabilities | L | High | Informed by #28 findings — BM25 is the priority path |
| #24 | Retrieval frequency tracking for garden entries | M | Med | Usage-based curation |

## Known Issues

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Key References

| Resource | Location |
|---|---|
| Hybrid benchmark report | `docs/comparison/hybrid-benchmark.md` |
| Benchmark design spec | `docs/superpowers/specs/2026-06-28-splade-hybrid-benchmark-design.md` |
| Real-world benchmark (#27) | `docs/comparison/real-world-benchmark.md` |
| Benchmark harness | `scripts/benchmark/` |
| Blog — SPLADE vocab gap | `blog/2026-06-29-mdp01-splade-hotel-beauty-renovation.md` |
| Blog — embedding vocab gap | `blog/2026-06-27-mdp01-embedding-vocabulary-gap.md` |
| Engine design | `docs/DESIGN.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
