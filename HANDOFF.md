# Hortora engine — Project Handoff

---

## What's In Flight

Branch `issue-36-bge-m3-benchmark` open for #36. Benchmark code (Tasks 1-3) complete, reviewed, committed. Task 4 (run benchmark + generate report) blocked on casehubio/neocortex#67. BGE-M3 ONNX model exported and verified at `~/.hortora/models/bge-m3/` (3MB graph + 2.1GB weights + 16MB tokenizer).

## Immediate Next Step

Fix casehubio/neocortex#67 — `CorpusIngestionService.doProcessBinding()` runs `fullScan`, saves cursor (674KB), but produces zero chunks. No errors logged. Once fixed: delete cursor (`rm -rf $TMPDIR/casehub-ingestion-cursors/`), restart engine, run `python3 scripts/benchmark/run_queries.py bge-m3-four-signal --min-points 6300`, then `python3 scripts/benchmark/analyze_bge_m3.py`.

## Cross-Module

**Blocked by:**
- `neocortex` — ingestion pipeline regression (casehubio/neocortex#67) gates #36 · S · Med

## What Changed This Session

- **ONNX export fixed** — `external_data=True` (torch 2.12 renamed the parameter), atomic `os.replace()` moves, three-file checksums. Model validated: 2.1GB weights, 6 test sentences passed.
- **`--min-points` CLI** — `run_queries.py` converted to argparse, corpus grew from 1900→7050 entries.
- **`analyze_bge_m3.py`** — imports from `analyze.py`, compares vs three-leg baseline, 8-section report, 6 tests.
- **Design review:** 3 rounds, 15 issues raised, 13 verified, 2 accepted, $9.25.
- **3 garden entries:** GE-20260703-e0af92 (torch external_data rename), GE-20260703-eca34b (stale cursor), GE-20260703-05f666 (duplicate binding).
- **Key finding:** `casehub.corpus.corpora.garden.source` must NOT be added — engine's `GardenBindingProducer` is the intended binding; adding the config creates a duplicate.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #36 | Run BGE-M3 benchmark (Task 4) | S | Low | Blocked on neocortex#67 |
| — | SeparateModelEmbedder for non-BGE-M3 deployments | S | Low | File neocortex issue |
| #33 | Convex Combination fusion test | S | Low | Needs #36 baseline |
| #34 | Matryoshka truncation + ColBERT quantization | M | Med | Needs #36 baseline |
| #30 | Federation type/tags propagation | S | Low | Mechanical parameter threading |
| #24 | Retrieval frequency tracking | M | Med | Usage-based curation |

## Key References

| Resource | Location |
|---|---|
| Benchmark design spec | `docs/superpowers/specs/2026-07-02-bge-m3-benchmark-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-07-02-bge-m3-benchmark.md` |
| SDD progress ledger | `.superpowers/sdd/progress.md` |
| Neocortex blocker | casehubio/neocortex#67 |
