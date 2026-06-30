# Hortora engine — Project Handoff

---

## What's In Flight

Nothing — main is clean. One unpushed commit (`2e6e682` — docs update).

## Immediate Next Step

Score the 87 unscored entries from the three-leg benchmark. Template at `scripts/benchmark/results/three-leg-to-score.json`. This establishes the true precision baseline and determines whether there's a remaining quality gap worth pursuing with BGE-M3 or other track 3 work.

## What Changed This Session

- **Three-leg benchmark validated** — 45% → 94% precision across 14 real-world scenarios. BM25 is the dominant contributor to closing the keyword gap. SPLADE alone didn't fix it.
- **Research document written** — `docs/comparison/retrieval-research.md` captures literature survey: BGE-M3 multi-mode, ColBERT as reranker (not retriever), HyDE doesn't help, CRAG doesn't beat hybrid fusion, nomic-embed-code exists. All paper links included.
- **Four stale docs updated** — DESIGN.md (Phase 3), hybrid-benchmark.md (three-leg results + closed issues), complementary-retrieval-design.md (outcome section: planned vs actual architecture).
- **neural-text cross-check** — 5/9 requests done (#47/#48/#50/#53/#54). Four outstanding (#46/#51/#52/#49) but none will improve retrieval quality — BM25 solved the problem they targeted.
- **Two garden entries** — cursor reset gotcha (GE-20260630-676593), FileCursorStore path undocumented (GE-20260630-a4fc8b)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Score 87 unscored entries | M | Low | Manual relevance scoring against #27 rubric; determines true precision |
| — | BGE-M3 adoption (neural-text #30) | L | Med | Track 3: single model replaces nomic-embed-text + SPLADE + cross-encoder; ColBERT as reranker |
| — | Convex Combination fusion test | S | Low | CC (α=0.5) vs RRF — config change, no code |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Known Issues

- 1 unpushed project commit (`2e6e682`) — docs only
- In-process `BM25Index` in neural-text is dead code on retrieval path (Qdrant-native BM25 is primary)
- neural-text #51/#52 (ONNX model validation) workaround'd in download-models.sh, not fixed upstream

## Key References

| Resource | Location |
|---|---|
| Three-leg benchmark results | `scripts/benchmark/results/three-leg.json` |
| Retrieval research + roadmap | `docs/comparison/retrieval-research.md` |
| Hybrid benchmark report | `docs/comparison/hybrid-benchmark.md` |
| Complementary retrieval spec | `docs/superpowers/specs/2026-06-29-complementary-retrieval-design.md` |
| Blog — BM25 closes the gap | `blog/2026-06-30-mdp02-bm25-closes-the-gap.md` |
| Unscored entries template | `scripts/benchmark/results/three-leg-to-score.json` |
| Engine design | `docs/DESIGN.md` |
