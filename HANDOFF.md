# Hortora engine — Project Handoff

*Last updated: 2026-06-18*

---

## What's In Flight

Nothing — main is clean, #8 closed.

## Immediate Next Step

Pick up Phase 2: SPLADE sparse embeddings. The engine now delegates to neural-text's `casehub-rag` — adding SPLADE is just providing a `SparseEmbedder` CDI bean (neural-text already supports optional sparse via `Instance<>`). No engine code changes needed beyond adding the `inference-splade` dependency and a bean producer.

## Cross-Module

**No active blockers or dependencies.**

casehubio/parent#255 filed for dependency argumentation graph design — not blocking any work.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 2: SPLADE + cross-encoder reranker | M | Med | Simpler now — just add `SparseEmbedder` bean; neural-text `casehub-rag` handles hybrid search |

## Key References

| Resource | Location |
|---|---|
| neural-text enablement issue | `casehubio/neural-text#35` |
| Engine design | `docs/DESIGN.md` |
| Qdrant client protocol | `casehub/garden: docs/protocols/universal/qdrant-client-library.md` |
| FS watching protocol | `casehub/garden: docs/protocols/universal/filesystem-watching-library.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
