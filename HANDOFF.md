# Hortora engine — Project Handoff

*Last updated: 2026-06-17*

---

## What's In Flight

Nothing — main is clean, no open branches with pending work.

## Immediate Next Step

Pick up Phase 2: SPLADE + cross-encoder reranker. The incremental re-indexing infrastructure is in place — `QdrantClient` with named `"dense"` vectors, `QueryPoints` API, `casehub-corpus` change detection. Phase 2 extends this by adding `inference-splade` (Hortora-eligible from casehub-neural-text), a `"sparse"` vector to the collection schema, and `PrefetchQuery` legs with RRF fusion in `SearchResource`.

## Cross-Module

**No active blockers or dependencies.**

casehubio/parent#255 filed for dependency argumentation graph design — not blocking any work.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 2: SPLADE + cross-encoder reranker | L | High | `inference-splade` is Hortora-eligible; extends existing QdrantClient + QueryPoints |
| — | DirectoryWatcherChangeSource contribution to neural-text | XS | Low | neural-text already migrating to directory-watcher |

## Key References

| Resource | Location |
|---|---|
| Engine spec (incremental re-indexing) | `docs/superpowers/specs/2026-06-16-incremental-reindexing-design.md` |
| Engine plan | `docs/superpowers/plans/2026-06-16-incremental-reindexing.md` |
| Engine design | `docs/DESIGN.md` |
| Qdrant client protocol | `casehub/garden: docs/protocols/universal/qdrant-client-library.md` |
| FS watching protocol | `casehub/garden: docs/protocols/universal/filesystem-watching-library.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
