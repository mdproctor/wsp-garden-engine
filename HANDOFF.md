# Hortora engine — Project Handoff

*Updated: 2026-06-24 — #19, #20, #21 closed.*

---

## What's In Flight

Nothing — main is clean. All open issues closed this session.

## Immediate Next Step

E2e verification of garden MCP integration: start the engine against the real garden, configure Claude Code MCP (`hortora-garden` SSE server at `localhost:8080/mcp/sse`), run a skill session and verify `gardenSearch` returns semantically relevant results with enriched metadata (entry IDs, domain, type, relevance). This was Task 5 of #21 — deferred because it requires running infrastructure.

## Cross-Module

**casehubio/neural-text#38** filed for `CursorStore.delete()` — clean cursor reset API. Not blocking; workaround in place.

**soredium commit:** `ca2d2fe` — work-start Step 3b and forage SEARCH updated to call `gardenSearch` MCP tool with git grep fallback.

**garden repo commit:** `9e097da` — YAML frontmatter added to 6 approach docs for engine indexing.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | E2e verification of garden MCP | S | Low | Start engine + Qdrant + Ollama, configure Claude Code MCP, verify gardenSearch in a live session |
| — | Update superpowers plugin skills | S | Low | brainstorming, systematic-debugging, code-review, java-dev/python-dev/ts-dev need garden consultation blocks (spec has exact diffs) |
| — | Native image for future CLI client | M | High | Engine is JVM-by-design (#20). inference-quarkus reachability metadata available. |

## Key References

| Resource | Location |
|---|---|
| Garden MCP integration spec | `docs/superpowers/specs/2026-06-23-garden-mcp-skill-integration-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-23-garden-mcp-skill-integration.md` |
| Engine design | `docs/DESIGN.md` |
| CursorStore.delete() issue | `casehubio/neural-text#38` |
| Open issues | `gh issue list --repo Hortora/engine` |
