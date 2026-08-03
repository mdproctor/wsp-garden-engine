# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-03)

**Four issues closed, one filed cross-repo.**

- **#77 + #76** — Startup reconcile deferred by 60s (`delayed="60s"` on `@Scheduled`). Prevents ONNX SIGSEGV from concurrent model loading + reconcile inference. Landed as `0beb919`.
- **#60** — Dual garden search (grep + MCP) added to 6 soredium skills for shadow comparison data collection. Soredium commit `1a96485`.
- **#45** — Subagent-mediated garden retrieval via `garden-retriever` agent type (`~/.claude/agents/garden-retriever.md`, Haiku). 7 skill files updated. Soredium commit `b888f9d`.
- **#78** → filed as `casehubio/neocortex#201` (batch ONNX inference). Already closed in neural-text.

**Garden reindex complete** — 2482 points with See Also metadata populated. Manual reindex via Qdrant REST API + cursor reset (MCP client couldn't reconnect after restart).

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children, no active work |
| #58 | Shadow comparison harness | — | — | Ongoing measurement, 8 reports |
| #61 | Remove dual search | XS | Low | Blocked on #58 |

## Key References

| Resource | Location |
|---|---|
| Epic #72 body | `gh issue view 72 --repo Hortora/engine` |
| Garden retriever agent | `~/.claude/agents/garden-retriever.md` |
| Blog entry | `blog/2026-08-03-mdp02-ops-day.md` |
| Forage entries | `GE-20260803-e363e6` (ONNX crash), `GE-20260804-2e5ca2` (manual reindex), `GE-20260804-d7ed92` (@Scheduled fires immediately) |
