# Hortora engine — Project Handoff

*Updated: 2026-06-26 — #23 closed, #25 closed, #26 closed.*

---

## What's In Flight

Nothing — main is clean.

## Immediate Next Step

Real-world benchmark (#27): pick actual GitHub issues across casehub repos, compare gardenSearch vs grep on each. The synthetic benchmark proved the value proposition; #27 tests it against the actual workflow.

## What Changed This Session

- **E2E verification complete** — engine indexes 1,949 points from the garden corpus via Ollama `nomic-embed-text`. Benchmark report at `docs/comparison/garden-search-vs-grep.md`.
- **Skill migration done** — `code-review/{java,python,typescript}.md` and `{java-dev,python-dev,ts-dev}/SKILL.md` updated in soredium with gardenSearch + git grep fallback.
- **@QuarkusMain fix** — `NativeImageGateCommand` in `casehub-inference-quarkus` hijacked consuming apps. Fixed in neural-text (`cf7f381`).
- **Chunking config** — recursive document splitting (6000 chars, 500 overlap) for large approach files exceeding `nomic-embed-text`'s 2048-token context.
- **3 garden entries** — @QuarkusMain hijack (GE-20260626-dd667c), Ollama context-length 400 (GE-20260626-773613), cursor checkpoint overwrite (GE-20260626-15a2e1).

## Cross-Module

**neural-text:** `@QuarkusMain` removed from `NativeImageGateCommand` (`cf7f381`). Batching added to `QdrantEmbeddingIngestor`. Both on main, need push to blessed repo.

**soredium:** `9103132` — gardenSearch consultation blocks in code-review and *-dev skills.

**casehubio/neural-text#38** — `CursorStore.delete()` clean cursor reset API. Still open.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Real-world benchmark — gardenSearch vs grep on actual issues | M | Med | Next priority |
| #24 | Retrieval frequency tracking for garden entries | M | Med | Usage-based curation |

## Known Issues

- **Startup thread embedding** — `doProcessBinding` on `@Observes StartupEvent` calls `embedAll` via the Vert.x REST client on the main thread. Works but may be fragile. Watcher callback path (different thread) works reliably.
- **Cursor checkpoint overwrite** — `checkpointCursors` timer saves watcher state even when initial scan failed. Workaround: delete cursor file and restart.
- **DESIGN.md stale** — still says `garden_search` + `garden_status`; actual tool names are `gardenSearch` + `gardenStatus` + `gardenReindex`. Pre-existing from #21.

## Key References

| Resource | Location |
|---|---|
| E2E verification spec | `docs/superpowers/specs/2026-06-24-garden-e2e-verification-and-skill-migration-design.md` |
| Implementation plan | `docs/superpowers/plans/2026-06-25-garden-e2e-and-skill-migration.md` |
| Benchmark report | `docs/comparison/garden-search-vs-grep.md` |
| Blog entry | `blog/2026-06-26-mdp01-grep-firehose-vs-ranked-answers.md` |
| Engine design | `docs/DESIGN.md` |
| Open issues | `gh issue list --repo Hortora/engine` |
