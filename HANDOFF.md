# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-06)

**Native Qdrant v1.19 is production.** Cut over from Podman — native on standard ports 6333/6334, 2623 points, engine plist port override removed. Podman container `qdrant-bench` recreated on 6343/6344 with `--restart=no` as a test bed. ~2.3 GB RAM saved.

**#83 payload fields verified** — `author`, `decay_tier`, `staleness_days`, `verified_on`, `domain`, `score`, `type`, `submitted` all present in Qdrant payloads.

**Embedding cache built and deployed** — `casehub-neocortex-rag-cache` module (5 commits on `feat/rag-cache-embedding` in neocortex). Wired into engine via `HybridSearchProducer`. Cache at `~/.hortora/cache/embeddings.db`. Speeds up developer reindex but doesn't solve end-user first-time setup.

**Garden search fully operational** — MCP → native Qdrant → retrieval tracking (455 records in SQLite). All working.

**#85 filed** — Qdrant snapshot distribution + native installer automation. Ship pre-built snapshots for instant first-time setup; delta ingestion handles new entries since snapshot. Podman container on 6343/6344 available as test bed.

## What's Left

- `work-end` on `issue-83-version-deemphasis-payload-enrichment` · S · Low
- Merge neocortex `feat/rag-cache-embedding` to main · S · Low
- 3 garden entries committed locally but not pushed (pre-push hook blocked) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | Qdrant snapshots + native installer | L | High | End-user setup solution |
| #84 | ClaudeAgentClient crashes when claude CLI missing | XS | Low | New |
| #82 | Investigate alternatives to global OrtSession lock | M | High | Future improvement |
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children |
| #58 | Shadow comparison harness | M | Med | Ongoing |
| #61 | Remove dual search | XS | Low | Blocked on #58 |
