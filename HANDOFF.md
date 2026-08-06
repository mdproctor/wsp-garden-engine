# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-04)

**#79 closed — REST reindex endpoint.**

- `POST /api/garden/reindex` wraps `GardenReindexService.reindex()`, shared with `gardenReindex` MCP tool. Returns `{"status":"ok|error","message":"..."}`. Grove Phase 2's "Trigger Reindex" button calls this.
- Pre-existing test infrastructure fixed — SmallRye Config group validation required full model stubs in test properties. 196 tests pass (were broken before). Garden entry `GE-20260804-6076a3`.

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children, no active work |
| #58 | Shadow comparison harness | — | — | Ongoing measurement |
| #61 | Remove dual search | XS | Low | Blocked on #58 |

## Key References

| Resource | Location |
|---|---|
| Epic #72 body | `gh issue view 72 --repo Hortora/engine` |
| Blog entry | `blog/2026-08-04-mdp01-config-group-trap.md` |
| Forage entry | `GE-20260804-6076a3` (SmallRye Config group validation trap) |
