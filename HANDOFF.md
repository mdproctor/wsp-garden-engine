# Hortora engine — Project Handoff

*Updated: 2026-06-28 — #27 closed.*

---

## What's In Flight

Nothing — main is clean.

## Immediate Next Step

SPLADE hybrid benchmark (#28): re-run the #27 benchmark with ONNX models enabled. Same 6 issues, same 2 specs, same queries — measure the delta. Enable ONNX model paths in `application.properties`, verify `CollectionMigration` triggers re-indexing, run benchmark.

## What Changed This Session

- **Real-world benchmark complete** (#27) — gardenSearch vs grep across 14 scenarios (6 issues, 8 spec domains). Result: 6-6-2 split. Neither method dominates. Report at `docs/comparison/real-world-benchmark.md`.
- **Key finding:** `nomic-embed-text` treats Java class names and CDI annotations as generic tokens. gardenSearch with keywords lost 12/14 scenarios. NL queries recovered to competitive precision (62% vs grep's 65%).
- **Migration reframed:** gardenSearch is not a grep replacement — it's a complement. Two-method approach: NL queries for concepts, grep for API names.
- **Garden entry:** GE-20260627-4712de — embedding vocabulary gap gotcha.
- **Follow-up issues created:** #28 (SPLADE hybrid benchmark), #29 (complementary retrieval), casehubio/neural-text#46 (SPLADE/reranker tuning).

## Cross-Module

**casehubio/neural-text#46** — SPLADE model quality and RRF fusion weight tuning. Blocked on #28 results. If SPLADE doesn't fix the keyword catastrophe, neural-text needs model evaluation work.

**neural-text (prior session, still pending):** `@QuarkusMain` fix and batching on local main — need push to blessed repo.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #28 | SPLADE hybrid benchmark — reproduce #27 with sparse+dense | M | Med | Next priority — Track 1 |
| #29 | Complementary retrieval capabilities | L | High | Track 2 — blocked on #28 |
| #24 | Retrieval frequency tracking for garden entries | M | Med | Usage-based curation |

## Known Issues

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Key References

| Resource | Location |
|---|---|
| Real-world benchmark report | `docs/comparison/real-world-benchmark.md` |
| Benchmark design spec | `docs/superpowers/specs/2026-06-26-real-world-benchmark-design.md` |
| Synthetic benchmark report | `docs/comparison/garden-search-vs-grep.md` |
| Blog — vocabulary gap | `blog/2026-06-27-mdp01-embedding-vocabulary-gap.md` |
| Blog — grep firehose | `blog/2026-06-26-mdp01-grep-firehose-vs-ranked-answers.md` |
| Engine design | `docs/DESIGN.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
