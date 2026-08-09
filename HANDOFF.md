# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-09)

**#85 closed — Qdrant snapshots + native installer.** New `hortora-setup.sh` with modular subcommands (`install-qdrant`, `install-models`, `install-snapshot`, `install-engine`, `status`, `uninstall`). Downloads pre-built ONNX models and Qdrant snapshot from GitHub Releases — first-time setup drops from ~90 min to download time + 2 min build. Cursor portability via `__GARDEN_PATH__` placeholder substitution. GitHub Actions `snapshot.yml` workflow builds and publishes split archives. E2E test workflow validates full cycle with 8-entry test corpus. `update-engine.sh` slimmed to `update|status|logs`. Plist templates replace hardcoded plists.

**Garden entries:** `GE-20260809-056ccb` (cursor path portability gotcha), `GE-20260809-903561` (GitHub Release tag overwrite race).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Run `snapshot.yml` workflow_dispatch with `corpus=test` | — | — | First CI validation |
| #84 | ClaudeAgentClient crashes when claude CLI missing | XS | Low | — |
| #82 | Investigate alternatives to global OrtSession lock | M | High | Future improvement |
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children |
| #58 | Shadow comparison harness | M | Med | Ongoing |
| #61 | Remove dual search | XS | Low | Blocked on #58 |

## Key References

| Resource | Location |
|---|---|
| Design spec | `docs/specs/issue-85-qdrant-snapshots-installer/` |
| Diary entry | `blog/2026-08-09-mdp01-instant-setup.md` |
| Garden entries | `GE-20260809-056ccb`, `GE-20260809-903561` |
