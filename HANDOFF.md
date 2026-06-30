# Hortora engine — Project Handoff

*Updated: 2026-06-30 — #29 closed, #30 filed.*

---

## What's In Flight

Nothing — main is clean.

## Immediate Next Step

Benchmark validation: re-run the #27 methodology with three-leg retrieval (dense + sparse + BM25) to measure whether BM25 closes the keyword gap. Run `gardenReindex()` first to rebuild the Qdrant collection with BM25 Document vectors and list-valued tags.

## What Changed This Session

- **#29 complementary retrieval complete** — BM25 keyword matching as a third RRF retrieval leg alongside dense and sparse. Architecture evolved from planned Java-side RRF to Qdrant-native BM25 via Document vectors (v1.18+). Three legs fuse inside Qdrant in a single gRPC call.
- **Cross-repo neural-text work** — 12 commits on `casehubio/neural-text` branch `issue-53-payload-hardening-bm25`: `BM25Index`, `BM25IndexRegistry`, `CamelCaseExpander`, `CodeDomainTokenizer`, `ExtractionResult`/`ChunkInput` listMetadata, metadata payload indexes, `HybridCaseRetriever` BM25 prefetch leg
- **Engine MCP enrichment** — `gardenSearch` gains `type` and `tags` filter parameters; `GardenMetadataExtractor` emits tags as list metadata
- **Three garden entries** — Qdrant BM25 RRF composition (revision), CamelCase compound preservation technique, BM25 test isolation gotcha
- **#30 filed** — federation type/tags propagation (type/tags filters not forwarded to upstream/peer nodes)
- **Qdrant v1.18+ required** — Document vector inference for BM25

## Cross-Module

**neural-text — 12 commits on `issue-53-payload-hardening-bm25`, not yet merged to main:**

| What | Status |
|------|--------|
| BM25Index + CodeDomainTokenizer | Complete, tested |
| ExtractionResult/ChunkInput listMetadata SPI | Complete, backward-compatible |
| HybridCaseRetriever BM25 prefetch leg | Complete, 207 tests pass |
| Metadata payload indexes (domain, type, tags) | Complete |
| Branch merge to neural-text main | Pending — needs neural-text session |

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Benchmark validation (re-run #27 with BM25) | M | Low | Reuse existing harness; key metric: does BM25 close keyword gap? |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Known Issues

- Neural-text branch `issue-53-payload-hardening-bm25` needs merging to main via neural-text session
- Re-index required after deployment (`gardenReindex()`) for BM25 + tags-as-list migration
- In-process `BM25Index` exists but is not used on the retrieval path (dead code — Qdrant-native BM25 is primary)

## Key References

| Resource | Location |
|---|---|
| Complementary retrieval spec | `docs/superpowers/specs/2026-06-29-complementary-retrieval-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-29-complementary-retrieval.md` |
| Blog — keyword fix lands | `blog/2026-06-30-mdp01-keyword-fix-lands.md` |
| Hybrid benchmark report | `docs/comparison/hybrid-benchmark.md` |
| Real-world benchmark (#27) | `docs/comparison/real-world-benchmark.md` |
| Engine design | `docs/DESIGN.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
