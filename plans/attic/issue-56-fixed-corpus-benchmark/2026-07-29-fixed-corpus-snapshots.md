# Fixed-Corpus Benchmark Snapshots — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #56 — bench: fixed-corpus benchmark snapshots for reproducible regression testing
**Issue group:** #56

**Goal:** Add Qdrant snapshot-based corpus freezing to the benchmark harness so retrieval precision measurements are reproducible across sessions.

**Architecture:** New `create_snapshot.py` script creates and downloads Qdrant collection snapshots to `~/.hortora/snapshots/<name>/`. Modified `run_queries.py` gains `--corpus-snapshot <name>` to restore a snapshot before running queries, plus an unscored-entry warning that catches scoring gaps. Shared Qdrant utilities extracted to `benchmark/qdrant_utils.py`.

**Tech Stack:** Python 3.11+ (stdlib only — `urllib.request`, `hashlib`, `json`, `pathlib`, `argparse`, `subprocess`). Pytest for tests.

## Global Constraints

- Python stdlib only — no new dependencies. The existing `requirements.txt` (numpy, onnxruntime, tokenizers) is for other scripts; benchmark harness uses stdlib.
- Snapshot storage at `~/.hortora/snapshots/` — not git-tracked.
- Garden path from `HORTORA_GARDEN_PATH` env var, defaulting to `~/.hortora/garden`.
- Qdrant collection name: `hortora_garden` (constant in `qdrant_utils.py`).
- All scripts run from repo root as `python3 scripts/benchmark/<script>.py`.
- Tests run as `python3 -m pytest scripts/benchmark/test_*.py -v`.

---

### Task 1: Extract `benchmark/qdrant_utils.py`

**Files:**
- Create: `scripts/benchmark/qdrant_utils.py`
- Create: `scripts/benchmark/test_qdrant_utils.py`
- Modify: `scripts/benchmark/run_queries.py` (remove extracted code, add imports)
- Modify: `scripts/benchmark/test_run_queries.py` (update import paths for monkeypatched defaults)

**Interfaces:**
- Produces:
  - `ENGINE_URL: str` = `"http://localhost:8080"`
  - `QDRANT_URL: str` = `"http://localhost:6333"`
  - `COLLECTION_NAME: str` = `"hortora_garden"`
  - `MIN_INDEXED_POINTS: int` = `1900`
  - `READINESS_POLL_S: int` = `5`
  - `check_qdrant_ready(qdrant_url: str = QDRANT_URL) -> int` — returns point count
  - `wait_for_readiness(engine_url: str = ENGINE_URL, qdrant_url: str = QDRANT_URL, min_points: int = MIN_INDEXED_POINTS) -> int` — returns point count

- [ ] **Step 1: Write tests for `check_qdrant_ready` and `wait_for_readiness`**

```python
# scripts/benchmark/test_qdrant_utils.py
import json
from unittest.mock import patch, MagicMock
from benchmark.qdrant_utils import (
    check_qdrant_ready, wait_for_readiness,
    ENGINE_URL, QDRANT_URL, COLLECTION_NAME, MIN_INDEXED_POINTS,
)


def _mock_urlopen_response(data: dict, status: int = 200):
    mock_resp = MagicMock()
    mock_resp.read.return_value = json.dumps(data).encode()
    mock_resp.__enter__ = lambda s: s
    mock_resp.__exit__ = MagicMock(return_value=False)
    mock_resp.status = status
    return mock_resp


class TestCheckQdrantReady:
    @patch("benchmark.qdrant_utils.urllib.request.urlopen")
    def test_returns_point_count(self, mock_urlopen):
        mock_urlopen.return_value = _mock_urlopen_response(
            {"result": {"points_count": 2400}}
        )
        assert check_qdrant_ready() == 2400

    @patch("benchmark.qdrant_utils.urllib.request.urlopen")
    def test_uses_custom_qdrant_url(self, mock_urlopen):
        mock_urlopen.return_value = _mock_urlopen_response(
            {"result": {"points_count": 100}}
        )
        check_qdrant_ready("http://custom:6333")
        call_args = mock_urlopen.call_args
        req = call_args[0][0]
        assert "custom:6333" in req.full_url

    @patch("benchmark.qdrant_utils.urllib.request.urlopen")
    def test_raises_on_connection_error(self, mock_urlopen):
        mock_urlopen.side_effect = ConnectionError("refused")
        try:
            check_qdrant_ready()
            assert False, "Should have raised"
        except ConnectionError:
            pass


class TestWaitForReadiness:
    @patch("benchmark.qdrant_utils.urllib.request.urlopen")
    def test_returns_when_engine_and_qdrant_ready(self, mock_urlopen):
        engine_resp = _mock_urlopen_response(
            [{"id": "test.md", "title": "t"}]
        )
        qdrant_resp = _mock_urlopen_response(
            {"result": {"points_count": 2000}}
        )
        mock_urlopen.side_effect = [engine_resp, qdrant_resp, qdrant_resp, qdrant_resp]
        count = wait_for_readiness(min_points=1900)
        assert count == 2000

    @patch("benchmark.qdrant_utils.urllib.request.urlopen")
    def test_raises_if_engine_never_responds(self, mock_urlopen):
        mock_urlopen.side_effect = ConnectionError("refused")
        try:
            wait_for_readiness()
            assert False, "Should have raised"
        except RuntimeError as e:
            assert "not responding" in str(e)


def test_constants():
    assert ENGINE_URL == "http://localhost:8080"
    assert QDRANT_URL == "http://localhost:6333"
    assert COLLECTION_NAME == "hortora_garden"
    assert MIN_INDEXED_POINTS == 1900
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest scripts/benchmark/test_qdrant_utils.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'benchmark.qdrant_utils'`

- [ ] **Step 3: Create `qdrant_utils.py` with extracted code**

```python
# scripts/benchmark/qdrant_utils.py
"""Shared Qdrant utilities for benchmark scripts."""

import json
import time
import urllib.request

ENGINE_URL = "http://localhost:8080"
QDRANT_URL = "http://localhost:6333"
COLLECTION_NAME = "hortora_garden"
MIN_INDEXED_POINTS = 1900
READINESS_POLL_S = 5


def check_qdrant_ready(qdrant_url: str = QDRANT_URL) -> int:
    url = f"{qdrant_url}/collections/{COLLECTION_NAME}"
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=10) as resp:
        data = json.loads(resp.read())
    return data["result"]["points_count"]


def wait_for_readiness(engine_url: str = ENGINE_URL, qdrant_url: str = QDRANT_URL,
                       min_points: int = MIN_INDEXED_POINTS) -> int:
    print("Waiting for engine readiness...")
    for attempt in range(60):
        try:
            req = urllib.request.Request(
                f"{engine_url}/search?q={urllib.parse.quote('test query')}"
            )
            with urllib.request.urlopen(req, timeout=10) as resp:
                resp.read()
            break
        except Exception:
            time.sleep(READINESS_POLL_S)
    else:
        raise RuntimeError("Engine not responding after 5 minutes")

    print("Waiting for indexing to complete...")
    prev_count = -1
    stable_checks = 0
    for attempt in range(120):
        try:
            count = check_qdrant_ready(qdrant_url)
            print(f"  Indexed points: {count}")
            if count >= min_points and count == prev_count:
                stable_checks += 1
                if stable_checks >= 2:
                    print(f"Indexing complete: {count} points")
                    return count
            else:
                stable_checks = 0
            prev_count = count
        except Exception as e:
            print(f"  Qdrant check failed: {e}")
        time.sleep(READINESS_POLL_S)
    raise RuntimeError("Indexing did not stabilise")
```

Note: add `import urllib.parse` at the top since `wait_for_readiness` now uses it directly instead of going through `search()`.

- [ ] **Step 4: Run `test_qdrant_utils.py` to verify tests pass**

Run: `python3 -m pytest scripts/benchmark/test_qdrant_utils.py -v`
Expected: PASS

- [ ] **Step 5: Update `run_queries.py` — remove extracted code, add imports**

Remove from `run_queries.py`:
- Constants: `ENGINE_URL`, `QDRANT_URL`, `COLLECTION_NAME`, `READINESS_POLL_S`, `MIN_INDEXED_POINTS`
- Functions: `check_qdrant_ready`, `wait_for_readiness`

Add import at top:
```python
from benchmark.qdrant_utils import (
    ENGINE_URL, QDRANT_URL, COLLECTION_NAME,
    MIN_INDEXED_POINTS, check_qdrant_ready, wait_for_readiness,
)
```

Keep in `run_queries.py`: `RESULTS_DIR`, `NUM_PASSES`, `QUERY_PAUSE_S`, `search()`, `parse_search_response()`, `compute_median()`, `run_all_queries()`, `main()`.

- [ ] **Step 6: Update `test_run_queries.py` — fix default value references**

The existing test `test_main_accepts_min_points_argument` references `rq.ENGINE_URL`, `rq.QDRANT_URL`, `rq.MIN_INDEXED_POINTS` in the `fake_wait` defaults. After extraction these are re-exported via the import, so `rq.ENGINE_URL` still resolves. Verify no changes needed — if the import is `from benchmark.qdrant_utils import ENGINE_URL`, then `rq.ENGINE_URL` is the imported name.

Run: `python3 -m pytest scripts/benchmark/test_run_queries.py -v`
Expected: PASS (existing tests unchanged)

- [ ] **Step 7: Run full benchmark test suite**

Run: `python3 -m pytest scripts/benchmark/test_*.py -v`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add scripts/benchmark/qdrant_utils.py scripts/benchmark/test_qdrant_utils.py scripts/benchmark/run_queries.py scripts/benchmark/test_run_queries.py
git commit -m "refactor: extract qdrant_utils.py from run_queries.py

Refs #56"
```

---

### Task 2: Add unscored entry warning to `run_queries.py`

**Files:**
- Modify: `scripts/benchmark/run_queries.py` (add `compute_unscored_pct()`, integrate into `main()`)
- Modify: `scripts/benchmark/test_run_queries.py` (add unscored tests)

**Interfaces:**
- Consumes: `baseline_scores.json` (loaded from `BASELINE_PATH`)
- Produces:
  - `compute_unscored_pct(results: list[dict], baseline: dict) -> float` — returns fraction 0.0-1.0
  - `unscored_pct` field in output JSON (always present)

- [ ] **Step 1: Write tests for `compute_unscored_pct`**

Add to `scripts/benchmark/test_run_queries.py`:

```python
from benchmark.run_queries import compute_unscored_pct

SAMPLE_BASELINE = {
    "GE-20260428-fd7a65": {
        "issue-1-reactive-async": {"benchmark_score": 2, "methods": ["gardenSearch-KW"]}
    },
    "GE-20260604-ed1b02": {
        "issue-1-reactive-async": {"benchmark_score": 1, "methods": ["gardenSearch-NL"]}
    },
}


def test_compute_unscored_pct_all_scored():
    results = [
        {"scenario_id": "issue-1-reactive-async", "entries": [
            {"id": "jvm/GE-20260428-fd7a65.md"},
            {"id": "jvm/GE-20260604-ed1b02.md"},
        ]},
    ]
    assert compute_unscored_pct(results, SAMPLE_BASELINE) == 0.0


def test_compute_unscored_pct_some_unscored():
    results = [
        {"scenario_id": "issue-1-reactive-async", "entries": [
            {"id": "jvm/GE-20260428-fd7a65.md"},
            {"id": "jvm/GE-20260604-ed1b02.md"},
            {"id": "jvm/GE-20260999-unknown.md"},
            {"id": "jvm/GE-20260999-other00.md"},
        ]},
    ]
    pct = compute_unscored_pct(results, SAMPLE_BASELINE)
    assert pct == 0.5  # 2 of 4 unscored


def test_compute_unscored_pct_wrong_scenario():
    results = [
        {"scenario_id": "issue-2-cdi-wiring", "entries": [
            {"id": "jvm/GE-20260428-fd7a65.md"},
        ]},
    ]
    pct = compute_unscored_pct(results, SAMPLE_BASELINE)
    assert pct == 1.0  # scored for issue-1, not issue-2


def test_compute_unscored_pct_empty_results():
    assert compute_unscored_pct([], SAMPLE_BASELINE) == 0.0


def test_compute_unscored_pct_no_entries():
    results = [{"scenario_id": "issue-1-reactive-async", "entries": []}]
    assert compute_unscored_pct(results, SAMPLE_BASELINE) == 0.0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest scripts/benchmark/test_run_queries.py::test_compute_unscored_pct_all_scored -v`
Expected: FAIL — `ImportError: cannot import name 'compute_unscored_pct'`

- [ ] **Step 3: Implement `compute_unscored_pct` in `run_queries.py`**

Add to `run_queries.py` after `compute_median`:

```python
BASELINE_PATH = Path(__file__).parent / "baseline_scores.json"


def compute_unscored_pct(results: list[dict], baseline: dict) -> float:
    total = 0
    unscored = 0
    for r in results:
        scenario_id = r["scenario_id"]
        for entry in r.get("entries", []):
            total += 1
            ge_id = Path(entry["id"]).stem
            entry_scores = baseline.get(ge_id, {})
            if scenario_id not in entry_scores:
                unscored += 1
    if total == 0:
        return 0.0
    return unscored / total
```

- [ ] **Step 4: Run unscored tests to verify they pass**

Run: `python3 -m pytest scripts/benchmark/test_run_queries.py -k "unscored" -v`
Expected: All 5 tests PASS

- [ ] **Step 5: Integrate into `main()` — load baseline, compute, warn, add to output**

In `run_queries.py` `main()`, after `results = run_all_queries(...)` and before writing the output file:

```python
    baseline = {}
    if BASELINE_PATH.exists():
        baseline = json.loads(BASELINE_PATH.read_text())

    unscored_pct = compute_unscored_pct(results, baseline)
    if unscored_pct > 0.05:
        total_entries = sum(len(r.get("entries", [])) for r in results)
        unscored_count = int(unscored_pct * total_entries)
        print(f"\n⚠️  {unscored_pct:.1%} of returned entries are unscored "
              f"({unscored_count}/{total_entries}). "
              f"Score new entries before accepting results.")
```

Add `"unscored_pct": round(unscored_pct, 4)` to the `output` dict.

- [ ] **Step 6: Write test for main() unscored integration**

Add to `test_run_queries.py`:

```python
def test_main_includes_unscored_pct(monkeypatch, tmp_path):
    import benchmark.run_queries as rq

    monkeypatch.setattr(rq, "wait_for_readiness", lambda **kw: 2000)
    monkeypatch.setattr(rq, "run_all_queries", lambda eu, limit=None: [
        {"scenario_id": "issue-1-reactive-async", "query_type": "KW",
         "query_text": "test", "entries": [{"id": "jvm/GE-unknown.md"}],
         "latency_ms": [10.0], "latency_median_ms": 10.0},
    ])
    monkeypatch.setattr(rq, "RESULTS_DIR", tmp_path)
    monkeypatch.setattr(rq, "BASELINE_PATH", tmp_path / "empty_baseline.json")
    (tmp_path / "empty_baseline.json").write_text("{}")
    monkeypatch.setattr("sys.argv", ["run_queries.py", "test-config", "--min-points", "1"])

    rq.main()

    result = json.loads((tmp_path / "test-config.json").read_text())
    assert "unscored_pct" in result
    assert result["unscored_pct"] == 1.0
```

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest scripts/benchmark/test_*.py -v`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add scripts/benchmark/run_queries.py scripts/benchmark/test_run_queries.py
git commit -m "feat: add unscored entry warning to benchmark harness

Warns when >5% of returned entries lack scores in baseline_scores.json.
Adds unscored_pct to output JSON for all runs.

Refs #56"
```

---

### Task 3: Create `create_snapshot.py`

**Files:**
- Create: `scripts/benchmark/create_snapshot.py`
- Create: `scripts/benchmark/test_create_snapshot.py`

**Interfaces:**
- Consumes:
  - `wait_for_readiness(engine_url, qdrant_url, min_points)` from `benchmark.qdrant_utils`
  - `QDRANT_URL`, `COLLECTION_NAME`, `ENGINE_URL`, `MIN_INDEXED_POINTS` from `benchmark.qdrant_utils`
- Produces:
  - `SNAPSHOT_DIR: Path` = `Path.home() / ".hortora" / "snapshots"`
  - `create_snapshot(name: str, engine_url: str, qdrant_url: str) -> dict` — returns manifest dict
  - `list_snapshots() -> list[dict]` — returns list of manifest dicts
  - `main()` — CLI entry point

- [ ] **Step 1: Write tests for manifest generation and `list_snapshots`**

```python
# scripts/benchmark/test_create_snapshot.py
import json
from pathlib import Path
from unittest.mock import patch, MagicMock
from benchmark.create_snapshot import (
    SNAPSHOT_DIR, create_manifest, list_snapshots,
)


def test_create_manifest_fields(tmp_path):
    manifest = create_manifest(
        name="test-snap",
        point_count=2400,
        qdrant_snapshot_name="hortora_garden-2026-07-29.snapshot",
        snapshot_sha256="abc123def456",
        snapshot_size_bytes=524288000,
        qdrant_version="1.12.6",
        engine_commit="7abba11",
        garden_sha="973b326a",
        scoring_sha="e8f1a2b3",
    )
    assert manifest["name"] == "test-snap"
    assert manifest["point_count"] == 2400
    assert manifest["qdrant_snapshot_name"] == "hortora_garden-2026-07-29.snapshot"
    assert manifest["snapshot_sha256"] == "abc123def456"
    assert manifest["snapshot_size_bytes"] == 524288000
    assert manifest["qdrant_version"] == "1.12.6"
    assert manifest["engine_commit"] == "7abba11"
    assert manifest["garden_sha"] == "973b326a"
    assert manifest["scoring_sha"] == "e8f1a2b3"
    assert "created" in manifest


def test_list_snapshots_empty(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.create_snapshot.SNAPSHOT_DIR", tmp_path)
    assert list_snapshots() == []


def test_list_snapshots_reads_manifests(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.create_snapshot.SNAPSHOT_DIR", tmp_path)
    snap_dir = tmp_path / "v2-baseline"
    snap_dir.mkdir()
    manifest = {
        "name": "v2-baseline",
        "point_count": 2400,
        "created": "2026-07-29T10:30:00Z",
        "engine_commit": "7abba11",
        "snapshot_size_bytes": 524288000,
    }
    (snap_dir / "manifest.json").write_text(json.dumps(manifest))
    result = list_snapshots()
    assert len(result) == 1
    assert result[0]["name"] == "v2-baseline"
    assert result[0]["point_count"] == 2400


def test_list_snapshots_skips_dirs_without_manifest(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.create_snapshot.SNAPSHOT_DIR", tmp_path)
    (tmp_path / "broken-snap").mkdir()
    assert list_snapshots() == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest scripts/benchmark/test_create_snapshot.py -v`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 3: Write tests for `create_snapshot` error cases**

Add to `test_create_snapshot.py`:

```python
import pytest
from benchmark.create_snapshot import create_snapshot


def test_create_snapshot_rejects_existing_name(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.create_snapshot.SNAPSHOT_DIR", tmp_path)
    existing = tmp_path / "existing"
    existing.mkdir()
    (existing / "manifest.json").write_text("{}")
    with pytest.raises(SystemExit):
        create_snapshot("existing", "http://localhost:8080", "http://localhost:6333")
```

- [ ] **Step 4: Implement `create_snapshot.py`**

```python
# scripts/benchmark/create_snapshot.py
"""Create and download Qdrant collection snapshots for reproducible benchmarking."""

import hashlib
import json
import os
import subprocess
import sys
import urllib.parse
import urllib.request
from datetime import datetime, timezone
from pathlib import Path

from benchmark.qdrant_utils import (
    ENGINE_URL, QDRANT_URL, COLLECTION_NAME, MIN_INDEXED_POINTS,
    wait_for_readiness, check_qdrant_ready,
)

SNAPSHOT_DIR = Path.home() / ".hortora" / "snapshots"
BASELINE_SCORES_PATH = Path(__file__).parent / "baseline_scores.json"
GARDEN_PATH_DEFAULT = Path.home() / ".hortora" / "garden"


def create_manifest(*, name: str, point_count: int, qdrant_snapshot_name: str,
                    snapshot_sha256: str, snapshot_size_bytes: int,
                    qdrant_version: str, engine_commit: str,
                    garden_sha: str, scoring_sha: str) -> dict:
    return {
        "name": name,
        "point_count": point_count,
        "created": datetime.now(timezone.utc).isoformat(),
        "engine_commit": engine_commit,
        "garden_sha": garden_sha,
        "qdrant_version": qdrant_version,
        "qdrant_snapshot_name": qdrant_snapshot_name,
        "snapshot_sha256": snapshot_sha256,
        "snapshot_size_bytes": snapshot_size_bytes,
        "scoring_sha": scoring_sha,
    }


def list_snapshots() -> list[dict]:
    if not SNAPSHOT_DIR.exists():
        return []
    result = []
    for d in sorted(SNAPSHOT_DIR.iterdir()):
        manifest_path = d / "manifest.json"
        if d.is_dir() and manifest_path.exists():
            try:
                result.append(json.loads(manifest_path.read_text()))
            except (json.JSONDecodeError, OSError):
                pass
    return result


def _get_qdrant_version(qdrant_url: str) -> str:
    req = urllib.request.Request(f"{qdrant_url}/")
    with urllib.request.urlopen(req, timeout=10) as resp:
        data = json.loads(resp.read())
    return data.get("version", "unknown")


def _get_git_describe(repo_path: str = ".") -> str:
    try:
        result = subprocess.run(
            ["git", "-C", repo_path, "describe", "--dirty", "--always"],
            capture_output=True, text=True, timeout=10,
        )
        return result.stdout.strip() if result.returncode == 0 else "unknown"
    except Exception:
        return "unknown"


def _get_git_short_sha(repo_path: str) -> str:
    try:
        result = subprocess.run(
            ["git", "-C", repo_path, "rev-parse", "--short", "HEAD"],
            capture_output=True, text=True, timeout=10,
        )
        return result.stdout.strip() if result.returncode == 0 else "unknown"
    except Exception:
        return "unknown"


def _get_scoring_sha() -> str:
    try:
        result = subprocess.run(
            ["git", "hash-object", str(BASELINE_SCORES_PATH)],
            capture_output=True, text=True, timeout=10,
        )
        return result.stdout.strip()[:8] if result.returncode == 0 else "unknown"
    except Exception:
        return "unknown"


def _compute_sha256(file_path: Path) -> str:
    h = hashlib.sha256()
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()


def create_snapshot(name: str, engine_url: str = ENGINE_URL,
                    qdrant_url: str = QDRANT_URL) -> dict:
    snap_dir = SNAPSHOT_DIR / name
    if snap_dir.exists():
        print(f"Error: snapshot '{name}' already exists at {snap_dir}", file=sys.stderr)
        sys.exit(1)

    point_count = wait_for_readiness(engine_url, qdrant_url)

    print(f"Creating Qdrant snapshot...")
    create_url = f"{qdrant_url}/collections/{COLLECTION_NAME}/snapshots"
    req = urllib.request.Request(create_url, method="POST",
                                headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=300) as resp:
        create_result = json.loads(resp.read())
    qdrant_snapshot_name = create_result["result"]["name"]
    print(f"  Snapshot created: {qdrant_snapshot_name}")

    snap_dir.mkdir(parents=True, exist_ok=True)
    snapshot_path = snap_dir / "collection.snapshot"
    try:
        print(f"Downloading snapshot...")
        download_url = f"{qdrant_url}/collections/{COLLECTION_NAME}/snapshots/{qdrant_snapshot_name}"
        req = urllib.request.Request(download_url)
        with urllib.request.urlopen(req, timeout=600) as resp:
            with open(snapshot_path, "wb") as f:
                while True:
                    chunk = resp.read(8192)
                    if not chunk:
                        break
                    f.write(chunk)
    except Exception as e:
        print(f"Error downloading snapshot: {e}", file=sys.stderr)
        if snap_dir.exists():
            import shutil
            shutil.rmtree(snap_dir)
        raise

    snapshot_sha256 = _compute_sha256(snapshot_path)
    snapshot_size = snapshot_path.stat().st_size
    print(f"  Downloaded: {snapshot_size / 1024 / 1024:.1f} MB, SHA-256: {snapshot_sha256[:16]}...")

    print(f"Cleaning up server-side snapshot...")
    try:
        delete_url = f"{qdrant_url}/collections/{COLLECTION_NAME}/snapshots/{qdrant_snapshot_name}"
        req = urllib.request.Request(delete_url, method="DELETE")
        with urllib.request.urlopen(req, timeout=30) as resp:
            resp.read()
    except Exception:
        print(f"  Warning: failed to delete server-side snapshot (non-fatal)")

    garden_path = os.environ.get("HORTORA_GARDEN_PATH", str(GARDEN_PATH_DEFAULT))
    manifest = create_manifest(
        name=name,
        point_count=point_count,
        qdrant_snapshot_name=qdrant_snapshot_name,
        snapshot_sha256=snapshot_sha256,
        snapshot_size_bytes=snapshot_size,
        qdrant_version=_get_qdrant_version(qdrant_url),
        engine_commit=_get_git_describe(),
        garden_sha=_get_git_short_sha(garden_path),
        scoring_sha=_get_scoring_sha(),
    )
    (snap_dir / "manifest.json").write_text(json.dumps(manifest, indent=2))
    print(f"\nSnapshot '{name}' saved to {snap_dir}")
    print(f"  Points: {point_count}")
    print(f"  Size: {snapshot_size / 1024 / 1024:.1f} MB")
    print(f"  SHA-256: {snapshot_sha256[:16]}...")
    return manifest


def main():
    import argparse
    parser = argparse.ArgumentParser(
        description="Create and manage Qdrant collection snapshots for benchmarking"
    )
    parser.add_argument("name", nargs="?", help="Snapshot name")
    parser.add_argument("--engine-url", default=ENGINE_URL, help="Engine base URL")
    parser.add_argument("--qdrant-url", default=QDRANT_URL, help="Qdrant REST API URL")
    parser.add_argument("--list", action="store_true", help="List available snapshots")
    args = parser.parse_args()

    if args.list:
        snapshots = list_snapshots()
        if not snapshots:
            print("No snapshots found.")
            return
        print(f"{'NAME':<20} {'POINTS':<8} {'CREATED':<25} {'SIZE':<10} {'ENGINE'}")
        for s in snapshots:
            size_mb = s.get("snapshot_size_bytes", 0) / 1024 / 1024
            print(f"{s['name']:<20} {s.get('point_count', '?'):<8} "
                  f"{s.get('created', '?'):<25} {size_mb:<10.0f} MB "
                  f"{s.get('engine_commit', '?')}")
        return

    if not args.name:
        parser.error("snapshot name is required (or use --list)")

    create_snapshot(args.name, args.engine_url, args.qdrant_url)


if __name__ == "__main__":
    main()
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest scripts/benchmark/test_create_snapshot.py -v`
Expected: All PASS

- [ ] **Step 6: Write test for `_compute_sha256`**

Add to `test_create_snapshot.py`:

```python
from benchmark.create_snapshot import _compute_sha256


def test_compute_sha256(tmp_path):
    test_file = tmp_path / "test.bin"
    test_file.write_bytes(b"hello world")
    sha = _compute_sha256(test_file)
    assert sha == "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"
```

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest scripts/benchmark/test_*.py -v`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add scripts/benchmark/create_snapshot.py scripts/benchmark/test_create_snapshot.py
git commit -m "feat: add create_snapshot.py for Qdrant collection snapshots

Creates, downloads, and manages named snapshots with SHA-256 integrity
verification, Qdrant version tracking, and scoring drift detection.

Refs #56"
```

---

### Task 4: Add `--corpus-snapshot` restore to `run_queries.py`

**Files:**
- Modify: `scripts/benchmark/run_queries.py` (add `restore_snapshot()`, new CLI flags, integration)
- Modify: `scripts/benchmark/test_run_queries.py` (add restore tests)

**Interfaces:**
- Consumes:
  - `SNAPSHOT_DIR` from `benchmark.create_snapshot`
  - `_compute_sha256` from `benchmark.create_snapshot`
  - `QDRANT_URL`, `COLLECTION_NAME` from `benchmark.qdrant_utils`
- Produces:
  - `restore_snapshot(name: str, qdrant_url: str) -> dict` — returns manifest, restores collection
  - `--corpus-snapshot <name>` CLI flag
  - `--qdrant-url <url>` CLI flag
  - `corpus_snapshot` and `snapshot_manifest` fields in output JSON

- [ ] **Step 1: Write tests for `restore_snapshot`**

Add to `test_run_queries.py`:

```python
import pytest
from unittest.mock import patch, MagicMock
from benchmark.run_queries import restore_snapshot


def _make_snapshot(tmp_path, name="test-snap", point_count=2400,
                   sha256="abc123", qdrant_version="1.12.6"):
    snap_dir = tmp_path / name
    snap_dir.mkdir(parents=True)
    snapshot_file = snap_dir / "collection.snapshot"
    snapshot_file.write_bytes(b"fake snapshot data")
    real_sha = __import__("hashlib").sha256(b"fake snapshot data").hexdigest()
    manifest = {
        "name": name,
        "point_count": point_count,
        "created": "2026-07-29T10:00:00Z",
        "engine_commit": "7abba11",
        "garden_sha": "973b326a",
        "qdrant_version": qdrant_version,
        "qdrant_snapshot_name": "hortora_garden.snapshot",
        "snapshot_sha256": real_sha,
        "snapshot_size_bytes": len(b"fake snapshot data"),
        "scoring_sha": "e8f1a2b3",
    }
    (snap_dir / "manifest.json").write_text(json.dumps(manifest))
    return manifest


def test_restore_snapshot_not_found(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.run_queries.SNAPSHOT_DIR", tmp_path)
    with pytest.raises(SystemExit):
        restore_snapshot("nonexistent", "http://localhost:6333")


def test_restore_snapshot_integrity_failure(tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.run_queries.SNAPSHOT_DIR", tmp_path)
    snap_dir = tmp_path / "bad-snap"
    snap_dir.mkdir()
    (snap_dir / "collection.snapshot").write_bytes(b"data")
    manifest = {"snapshot_sha256": "wrong_hash", "snapshot_size_bytes": 4,
                "point_count": 100, "qdrant_version": "1.12.6",
                "scoring_sha": "abc", "name": "bad-snap"}
    (snap_dir / "manifest.json").write_text(json.dumps(manifest))
    with pytest.raises(SystemExit):
        restore_snapshot("bad-snap", "http://localhost:6333")


def _mock_urlopen_response(data, status=200):
    mock_resp = MagicMock()
    if isinstance(data, bytes):
        mock_resp.read.return_value = data
    else:
        mock_resp.read.return_value = json.dumps(data).encode()
    mock_resp.__enter__ = lambda s: s
    mock_resp.__exit__ = MagicMock(return_value=False)
    mock_resp.status = status
    return mock_resp


@patch("benchmark.run_queries.urllib.request.urlopen")
def test_restore_snapshot_success(mock_urlopen, tmp_path, monkeypatch):
    monkeypatch.setattr("benchmark.run_queries.SNAPSHOT_DIR", tmp_path)
    monkeypatch.setattr("benchmark.run_queries.BASELINE_PATH",
                        tmp_path / "baseline.json")
    (tmp_path / "baseline.json").write_text("{}")
    manifest = _make_snapshot(tmp_path, "good-snap")

    qdrant_version_resp = _mock_urlopen_response({"version": "1.12.6"})
    delete_resp = _mock_urlopen_response({"result": True})
    upload_resp = _mock_urlopen_response({"result": True})
    collection_resp = _mock_urlopen_response(
        {"result": {"status": "green", "points_count": 2400}}
    )
    mock_urlopen.side_effect = [
        qdrant_version_resp, delete_resp, upload_resp, collection_resp,
    ]

    result = restore_snapshot("good-snap", "http://localhost:6333")
    assert result["name"] == "good-snap"
    assert result["point_count"] == 2400
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest scripts/benchmark/test_run_queries.py::test_restore_snapshot_not_found -v`
Expected: FAIL — `ImportError: cannot import name 'restore_snapshot'`

- [ ] **Step 3: Implement `restore_snapshot` in `run_queries.py`**

Add imports at top of `run_queries.py`:

```python
import hashlib
from benchmark.create_snapshot import SNAPSHOT_DIR, _compute_sha256
```

Add function after `compute_unscored_pct`:

```python
def restore_snapshot(name: str, qdrant_url: str = QDRANT_URL) -> dict:
    snap_dir = SNAPSHOT_DIR / name
    snapshot_path = snap_dir / "collection.snapshot"
    manifest_path = snap_dir / "manifest.json"

    if not snapshot_path.exists() or not manifest_path.exists():
        print(f"Error: snapshot '{name}' not found at {snap_dir}", file=sys.stderr)
        sys.exit(1)

    manifest = json.loads(manifest_path.read_text())
    print(f"Restoring snapshot '{name}':")
    print(f"  Points: {manifest.get('point_count')}")
    print(f"  Created: {manifest.get('created')}")
    print(f"  Engine: {manifest.get('engine_commit')}")
    print(f"  Garden: {manifest.get('garden_sha')}")

    # Integrity check
    actual_sha = _compute_sha256(snapshot_path)
    expected_sha = manifest.get("snapshot_sha256", "")
    if actual_sha != expected_sha:
        print(f"Error: snapshot integrity check failed.\n"
              f"  Expected SHA-256: {expected_sha[:16]}...\n"
              f"  Actual SHA-256:   {actual_sha[:16]}...",
              file=sys.stderr)
        sys.exit(1)

    actual_size = snapshot_path.stat().st_size
    expected_size = manifest.get("snapshot_size_bytes", 0)
    if actual_size != expected_size:
        print(f"Error: snapshot size mismatch.\n"
              f"  Expected: {expected_size} bytes\n"
              f"  Actual:   {actual_size} bytes",
              file=sys.stderr)
        sys.exit(1)

    # Scoring drift detection
    if BASELINE_PATH.exists():
        import subprocess
        try:
            result = subprocess.run(
                ["git", "hash-object", str(BASELINE_PATH)],
                capture_output=True, text=True, timeout=10,
            )
            current_scoring_sha = result.stdout.strip()[:8] if result.returncode == 0 else ""
        except Exception:
            current_scoring_sha = ""
        manifest_scoring_sha = manifest.get("scoring_sha", "")
        if current_scoring_sha and manifest_scoring_sha and current_scoring_sha != manifest_scoring_sha:
            print(f"  Note: scoring data has changed since snapshot was created "
                  f"(was {manifest_scoring_sha}, now {current_scoring_sha})")

    # Qdrant version check
    try:
        req = urllib.request.Request(f"{qdrant_url}/")
        with urllib.request.urlopen(req, timeout=10) as resp:
            qdrant_info = json.loads(resp.read())
        running_version = qdrant_info.get("version", "unknown")
        manifest_version = manifest.get("qdrant_version", "unknown")
        if running_version.split(".")[0] != manifest_version.split(".")[0]:
            print(f"Error: Qdrant major version mismatch.\n"
                  f"  Snapshot created with: {manifest_version}\n"
                  f"  Running Qdrant: {running_version}",
                  file=sys.stderr)
            sys.exit(1)
        if running_version != manifest_version:
            print(f"  Note: Qdrant version differs "
                  f"(snapshot: {manifest_version}, running: {running_version})")
    except Exception as e:
        print(f"  Warning: could not check Qdrant version: {e}")

    # Delete existing collection (404 is fine)
    print(f"  Deleting existing collection...")
    try:
        delete_url = f"{qdrant_url}/collections/{COLLECTION_NAME}"
        req = urllib.request.Request(delete_url, method="DELETE")
        with urllib.request.urlopen(req, timeout=30) as resp:
            resp.read()
    except urllib.error.HTTPError as e:
        if e.code != 404:
            raise
    except Exception:
        pass

    # Upload snapshot
    print(f"  Uploading snapshot ({actual_size / 1024 / 1024:.1f} MB)...")
    upload_url = f"{qdrant_url}/collections/{COLLECTION_NAME}/snapshots/upload"
    boundary = "----SnapshotUploadBoundary"
    with open(snapshot_path, "rb") as f:
        snapshot_data = f.read()

    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="snapshot"; filename="collection.snapshot"\r\n'
        f"Content-Type: application/octet-stream\r\n\r\n"
    ).encode() + snapshot_data + f"\r\n--{boundary}--\r\n".encode()

    req = urllib.request.Request(
        upload_url, data=body, method="POST",
        headers={"Content-Type": f"multipart/form-data; boundary={boundary}"},
    )
    try:
        with urllib.request.urlopen(req, timeout=600) as resp:
            resp.read()
    except Exception as e:
        print(f"Error: snapshot upload failed. Collection was deleted.\n"
              f"  Re-run with --corpus-snapshot to retry — "
              f"the snapshot file on disk is intact.\n"
              f"  Details: {e}", file=sys.stderr)
        sys.exit(1)

    # Wait for collection ready
    print(f"  Waiting for collection to be ready...")
    expected_points = manifest.get("point_count", 0)
    for attempt in range(24):  # 24 * 5s = 120s
        try:
            info_url = f"{qdrant_url}/collections/{COLLECTION_NAME}"
            req = urllib.request.Request(info_url)
            with urllib.request.urlopen(req, timeout=10) as resp:
                info = json.loads(resp.read())
            status = info["result"].get("status", "")
            points = info["result"].get("points_count", 0)
            if status == "green" and points >= expected_points:
                print(f"  Collection ready: {points} points")
                return manifest
        except Exception:
            pass
        time.sleep(5)

    print(f"Error: collection did not become ready after restore.\n"
          f"  Check Qdrant logs for index corruption or resource issues.",
          file=sys.stderr)
    sys.exit(1)
```

- [ ] **Step 4: Run restore tests to verify they pass**

Run: `python3 -m pytest scripts/benchmark/test_run_queries.py -k "restore" -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Add `--corpus-snapshot` and `--qdrant-url` to `main()`**

Modify `main()` in `run_queries.py`:

Add argparse arguments:
```python
    parser.add_argument("--corpus-snapshot", default=None,
                        help="Restore named snapshot before running (skips live indexing)")
    parser.add_argument("--qdrant-url", default=QDRANT_URL,
                        help=f"Qdrant REST API URL (default: {QDRANT_URL})")
```

Replace the readiness/query/output block in `main()`:

```python
    if args.corpus_snapshot:
        manifest = restore_snapshot(args.corpus_snapshot, args.qdrant_url)
        point_count = manifest["point_count"]
    else:
        point_count = wait_for_readiness(args.engine_url, qdrant_url=args.qdrant_url,
                                         min_points=args.min_points)

    print(f"\nRunning benchmark for config: {args.config_name}")
    results = run_all_queries(args.engine_url, limit=args.limit)

    baseline = {}
    if BASELINE_PATH.exists():
        baseline = json.loads(BASELINE_PATH.read_text())

    unscored_pct = compute_unscored_pct(results, baseline)
    if unscored_pct > 0.05:
        total_entries = sum(len(r.get("entries", [])) for r in results)
        unscored_count = int(unscored_pct * total_entries)
        print(f"\n⚠️  {unscored_pct:.1%} of returned entries are unscored "
              f"({unscored_count}/{total_entries}). "
              f"Score new entries before accepting results.")

    output = {
        "config": args.config_name,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "point_count": point_count,
        "num_passes": NUM_PASSES,
        "unscored_pct": round(unscored_pct, 4),
        "results": results,
    }

    if args.corpus_snapshot:
        output["corpus_snapshot"] = args.corpus_snapshot
        output["snapshot_manifest"] = {
            "point_count": manifest["point_count"],
            "engine_commit": manifest.get("engine_commit"),
            "garden_sha": manifest.get("garden_sha"),
            "qdrant_version": manifest.get("qdrant_version"),
        }

    output_path = RESULTS_DIR / f"{args.config_name}.json"
    output_path.write_text(json.dumps(output, indent=2))
    print(f"\nResults written to {output_path}")
```

Also update `engine_url` positional arg to use `args.engine_url`:
```python
    parser.add_argument("engine_url", nargs="?", default=ENGINE_URL,
                        help="Engine base URL")
```

- [ ] **Step 6: Write test for `main()` with `--corpus-snapshot`**

Add to `test_run_queries.py`:

```python
@patch("benchmark.run_queries.restore_snapshot")
def test_main_with_corpus_snapshot(mock_restore, monkeypatch, tmp_path):
    import benchmark.run_queries as rq

    mock_restore.return_value = {
        "name": "test-snap", "point_count": 2400,
        "engine_commit": "7abba11", "garden_sha": "973b326a",
        "qdrant_version": "1.12.6",
    }
    monkeypatch.setattr(rq, "run_all_queries", lambda eu, limit=None: [])
    monkeypatch.setattr(rq, "RESULTS_DIR", tmp_path)
    monkeypatch.setattr(rq, "BASELINE_PATH", tmp_path / "b.json")
    (tmp_path / "b.json").write_text("{}")
    monkeypatch.setattr("sys.argv", [
        "run_queries.py", "snap-test", "--corpus-snapshot", "test-snap",
    ])

    rq.main()

    result = json.loads((tmp_path / "snap-test.json").read_text())
    assert result["corpus_snapshot"] == "test-snap"
    assert result["snapshot_manifest"]["point_count"] == 2400
    assert result["snapshot_manifest"]["engine_commit"] == "7abba11"
    mock_restore.assert_called_once_with("test-snap", rq.QDRANT_URL)


def test_main_without_snapshot_has_no_snapshot_field(monkeypatch, tmp_path):
    import benchmark.run_queries as rq

    monkeypatch.setattr(rq, "wait_for_readiness", lambda **kw: 2000)
    monkeypatch.setattr(rq, "run_all_queries", lambda eu, limit=None: [])
    monkeypatch.setattr(rq, "RESULTS_DIR", tmp_path)
    monkeypatch.setattr(rq, "BASELINE_PATH", tmp_path / "b.json")
    (tmp_path / "b.json").write_text("{}")
    monkeypatch.setattr("sys.argv", ["run_queries.py", "no-snap", "--min-points", "1"])

    rq.main()

    result = json.loads((tmp_path / "no-snap.json").read_text())
    assert "corpus_snapshot" not in result
    assert "snapshot_manifest" not in result
    assert "unscored_pct" in result


def test_main_snapshot_ignores_min_points(monkeypatch, tmp_path):
    """When --corpus-snapshot is used, --min-points is ignored."""
    import benchmark.run_queries as rq

    wait_called = {"called": False}
    original_wait = rq.wait_for_readiness
    def spy_wait(**kw):
        wait_called["called"] = True
        return original_wait(**kw)

    monkeypatch.setattr(rq, "wait_for_readiness", spy_wait)

    with patch("benchmark.run_queries.restore_snapshot") as mock_restore:
        mock_restore.return_value = {"name": "s", "point_count": 100,
                                     "engine_commit": "x", "garden_sha": "y",
                                     "qdrant_version": "1.0.0"}
        monkeypatch.setattr(rq, "run_all_queries", lambda eu, limit=None: [])
        monkeypatch.setattr(rq, "RESULTS_DIR", tmp_path)
        monkeypatch.setattr(rq, "BASELINE_PATH", tmp_path / "b.json")
        (tmp_path / "b.json").write_text("{}")
        monkeypatch.setattr("sys.argv", [
            "run_queries.py", "test", "--corpus-snapshot", "s", "--min-points", "9999",
        ])
        rq.main()

    assert not wait_called["called"]
```

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest scripts/benchmark/test_*.py -v`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add scripts/benchmark/run_queries.py scripts/benchmark/test_run_queries.py
git commit -m "feat: add --corpus-snapshot restore to run_queries.py

Restores a named Qdrant snapshot before running benchmark queries.
Includes integrity verification, Qdrant version check, scoring drift
detection, and restore error handling.

Refs #56"
```
