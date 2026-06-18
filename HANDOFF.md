# Hortora engine — Project Handoff

*Last updated: 2026-06-18*

---

## What's In Flight

Nothing — main is clean, #9 closed.

## Immediate Next Step

Operator setup: download ONNX models (`Splade_PP_en_v1` for SPLADE, `ms-marco-MiniLM-L-6-v2` for cross-encoder), set `casehub.inference.models.splade.model-path` and `casehub.inference.models.reranker.model-path` in `application.properties`, start with Qdrant running. `CollectionMigration` will detect the dense-only collection and trigger a full re-index automatically.

## Cross-Module

**No active blockers or dependencies.**

casehubio/neural-text#38 filed for `CursorStore.delete()` — clean cursor reset API. Not blocking; workaround in place.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Native image validation — ONNX Runtime JNI in GraalVM native | M | High | Depends on neural-text Epic 2; inference-quarkus ships reachability metadata but e2e native gate not yet confirmed for Hortora |
| — | ONNX model download automation — dev services or build-time fetch | S | Low | Manual operator step today; could be a Quarkus dev service |

## Key References

| Resource | Location |
|---|---|
| Hybrid search spec | `docs/superpowers/specs/2026-06-18-hybrid-search-splade-reranker.md` |
| Engine design | `docs/DESIGN.md` |
| SPLADE licensing gotcha | `GE-20260614-b94048` (garden) |
| CDI @Nonbinding gotcha | `GE-20260618-397bf7` (garden) |
| CursorStore.delete() issue | `casehubio/neural-text#38` |
| Open issues | `gh issue list --repo Hortora/engine` |
