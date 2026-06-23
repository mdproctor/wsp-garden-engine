# Hortora engine — Project Handoff

*Updated: #19, #20 closed — removed from backlog. #21 filed.*

---

## What's In Flight

#21 — garden MCP skill integration. Spec approved, implementation plan next.

## Immediate Next Step

Invoke writing-plans for #21 to create the implementation plan, then `/work` to start.

## Cross-Module

**neural-text commit:** `188a633` on `casehubio/neural-text` main — added `@Inject` to `InMemoryCaseRetriever` constructor for Quarkus CDI compatibility (needed when class has multiple constructors). This was required for engine's test double reconciliation (#16).

**casehubio/neural-text#38** filed for `CursorStore.delete()` — clean cursor reset API. Not blocking; workaround in place.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 21 | Garden MCP skill integration — gardenSearch as primary retrieval for skills | M | Med | Spec approved. Engine output enrichment + skill diffs. |
| — | Native image for future CLI client | M | High | Engine is JVM-by-design (#20). Native image only relevant for future Hortora CLI client (fast startup, single binary). inference-quarkus reachability metadata available. |

## Key References

| Resource | Location |
|---|---|
| Garden MCP integration spec | `docs/superpowers/specs/2026-06-23-garden-mcp-skill-integration-design.md` |
| Hybrid search spec | `docs/superpowers/specs/2026-06-18-hybrid-search-splade-reranker.md` |
| Engine design | `docs/DESIGN.md` |
| CRLF test fixture gotcha | `GE-20260623-aeda6f` (garden) |
| SPLADE licensing gotcha | `GE-20260614-b94048` (garden) |
| CDI @Nonbinding gotcha | `GE-20260618-397bf7` (garden) |
| CursorStore.delete() issue | `casehubio/neural-text#38` |
| Open issues | `gh issue list --repo Hortora/engine` |
