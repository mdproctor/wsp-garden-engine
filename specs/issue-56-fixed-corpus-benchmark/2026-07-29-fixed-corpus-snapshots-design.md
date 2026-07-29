# Fixed-Corpus Benchmark Snapshots

*2026-07-29 · Refs #56*

## Problem

The benchmark harness (`run_queries.py`) indexes the live garden corpus before each run. Corpus growth between runs makes cross-session precision comparisons invalid — #55 proved a -2.2pp "regression" was entirely phantom, caused by 18 unscored entries displacing scored ones in the top-16.

## Solution

Qdrant collection snapshots downloaded to local storage. Create a snapshot after indexing a known corpus state, restore it before benchmark runs. Both corpus and embeddings are constant across runs — only retrieval/ranking code changes affect results.

## Storage

```
~/.hortora/snapshots/
  <name>/
    collection.snapshot    # Qdrant snapshot binary
    manifest.json          # provenance metadata
```

Snapshot names are user-chosen (e.g. `v2-baseline`, `2026-07-28-pre-hyde`). Not git-tracked — files are large (ColBERT multi-vectors dominate). Survives Qdrant container recreation.

### manifest.json

```json
{
  "name": "v2-baseline",
  "point_count": 2400,
  "created": "2026-07-29T10:30:00Z",
  "engine_commit": "7abba11",
  "garden_sha": "973b326a",
  "qdrant_snapshot_name": "hortora_garden-2026-07-29-10-30-00.snapshot"
}
```

## New script: `create_snapshot.py`

Location: `scripts/benchmark/create_snapshot.py`

### Usage

```bash
python3 scripts/benchmark/create_snapshot.py <name> [--engine-url URL] [--qdrant-url URL]
```

### Workflow

1. Verify engine readiness — reuse `wait_for_readiness()` from `run_queries.py` (polls engine `/search` endpoint + Qdrant point count stability).
2. Create snapshot: `POST /collections/hortora_garden/snapshots` — Qdrant returns the snapshot name.
3. Download snapshot: `GET /collections/hortora_garden/snapshots/{snapshot_name}` — write to `~/.hortora/snapshots/<name>/collection.snapshot`.
4. Write `manifest.json` with: name, point count (from Qdrant collection info), creation timestamp, engine git SHA (`git rev-parse --short HEAD`), garden git SHA (`git -C ~/.hortora/garden rev-parse --short HEAD`).
5. Print summary: snapshot path, point count, size on disk.

### Error handling

- Engine not ready after timeout → exit with error, no snapshot created.
- Qdrant snapshot creation fails → exit with error.
- Snapshot download fails → clean up partial directory, exit with error.
- Snapshot name already exists → exit with error (no silent overwrite).

## Modified `run_queries.py`

### New flag

`--snapshot <name>` — restore a named snapshot before running queries.

### Behaviour when `--snapshot` is provided

1. Verify snapshot exists at `~/.hortora/snapshots/<name>/collection.snapshot`.
2. Read `manifest.json` for provenance — print snapshot metadata.
3. Delete existing collection: `DELETE /collections/hortora_garden`.
4. Recover from snapshot: upload the snapshot file via `POST /collections/hortora_garden/snapshots/upload` (multipart form). This works regardless of whether Qdrant runs in Docker or natively — the file is uploaded over HTTP, not accessed via filesystem path.
5. Wait for collection ready: poll `GET /collections/hortora_garden` until `status == "green"` and point count matches manifest.
6. Run queries as normal.

### Behaviour when `--snapshot` is not provided

Unchanged — waits for live engine indexing via `wait_for_readiness()`.

### Unscored entry warning

After query execution, before writing results:

1. Load `baseline_scores.json` and `results/bge-m3-to-score.json`.
2. For each result entry, check if its GE-ID has a score for that scenario.
3. Compute unscored percentage across all results.
4. If >5%: print warning and include `unscored_pct` in output JSON.

```
⚠️  12.3% of returned entries are unscored (28/228). Score new entries before accepting results.
```

### Result metadata changes

When `--snapshot` is used, the output JSON includes:

```json
{
  "config": "v2-baseline",
  "snapshot": "v2-baseline",
  "snapshot_manifest": {
    "point_count": 2400,
    "engine_commit": "7abba11",
    "garden_sha": "973b326a"
  },
  "timestamp": "...",
  "point_count": 2400,
  "unscored_pct": 0.0,
  "results": [...]
}
```

When `--snapshot` is not used, `snapshot` and `snapshot_manifest` are absent. `unscored_pct` is always present.

## Scoring files

No structural change. `baseline_scores.json` and `results/bge-m3-to-score.json` remain in git, manually maintained. Extra scores for entries not in a snapshot's results are harmless. The unscored-entry warning catches the gap case.

## Qdrant API reference

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Create snapshot | POST | `/collections/hortora_garden/snapshots` |
| List snapshots | GET | `/collections/hortora_garden/snapshots` |
| Download snapshot | GET | `/collections/hortora_garden/snapshots/{name}` |
| Delete collection | DELETE | `/collections/hortora_garden` |
| Upload snapshot to recover | POST | `/collections/hortora_garden/snapshots/upload` |
| Collection info | GET | `/collections/hortora_garden` |

Upload recovery: multipart form with snapshot file. Works with Docker-hosted Qdrant (file uploaded over HTTP, not filesystem access).

## Out of scope

- **Mode A (frozen garden dir + re-index)** — deferred until embedding model changes are on the roadmap.
- **Snapshot pruning/rotation** — manual (`rm -rf ~/.hortora/snapshots/<old>`).
- **Scoring the 18 new entries** — manual scoring task, not harness code.
- **Automated snapshot creation in CI** — not needed for dev-local benchmarking.

## Test plan

- `test_create_snapshot.py` — unit tests for manifest generation, directory creation, error cases (existing name, missing engine).
- `test_run_queries.py` — extend existing tests for `--snapshot` flag parsing, unscored percentage calculation, result metadata inclusion.
- Manual verification: create a snapshot, run benchmark with `--snapshot`, confirm results match a live-index run on the same corpus.
