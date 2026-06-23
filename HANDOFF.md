# Hortora engine — Project Handoff

*Last updated: 2026-06-23*

---

## What's In Flight

Nothing — main is clean, #10–#18 closed.

## Immediate Next Step

Hybrid search dev setup: run `scripts/download-models.sh` to download ONNX models, uncomment the `%dev` model paths in `application.properties`, start Qdrant (`docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant`), then `./mvnw quarkus:dev`. `CollectionMigration` detects the dense-only collection and triggers a full re-index automatically when SPLADE is newly enabled.

## Cross-Module

**neural-text commit:** `188a633` on `casehubio/neural-text` main — added `@Inject` to `InMemoryCaseRetriever` constructor for Quarkus CDI compatibility (needed when class has multiple constructors). This was required for engine's test double reconciliation (#16).

**casehubio/neural-text#38** filed for `CursorStore.delete()` — clean cursor reset API. Not blocking; workaround in place.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 19 | Add SHA-256 checksum verification to download-models.sh | XS | Low | Filed from code review on #18; checksums captured after first download |
| — | Native image validation — ONNX Runtime JNI in GraalVM native | M | High | Depends on neural-text Epic 2; inference-quarkus ships reachability metadata but e2e native gate not yet confirmed |

## Key References

| Resource | Location |
|---|---|
| Audit fixes spec | `specs/issue-10-xs-s-audit-fixes/2026-06-22-xs-s-audit-fixes-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-23-xs-s-audit-fixes.md` |
| Hybrid search spec | `docs/superpowers/specs/2026-06-18-hybrid-search-splade-reranker.md` |
| Engine design | `docs/DESIGN.md` |
| CRLF test fixture gotcha | `GE-20260623-aeda6f` (garden) |
| SPLADE licensing gotcha | `GE-20260614-b94048` (garden) |
| CDI @Nonbinding gotcha | `GE-20260618-397bf7` (garden) |
| CursorStore.delete() issue | `casehubio/neural-text#38` |
| Checksum follow-up | `Hortora/engine#19` |
| Open issues | `gh issue list --repo Hortora/engine` |
