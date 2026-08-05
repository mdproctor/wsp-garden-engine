# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-05)

**#81 closed — ONNX ReentrantLock fix** (neocortex `86a77b0`). Serializes `OrtSession.run()` to prevent SIGSEGV.

**Branch `issue-83-version-deemphasis-payload-enrichment` — code complete, 234 tests pass, NOT merged.**
- 6 commits: metadata extraction, search profiles, temporal decay scorer, version scorer, pipeline wiring, BOM resolution
- 2 cross-repo commits to neocortex: ONNX lock (#81) + payload indexes (#80)
- Spec + plan in workspace `specs/` and `plans/`

## ⚠️ Blocker — Qdrant Data Corrupted

**The engine cannot start.** During a native Qdrant migration attempt, the Podman volume's collection entered a persistent `yellow` state. The engine hangs during startup on this yellow collection — even on the `main` branch (before any feature changes). This is NOT caused by #83/#80 code.

**What happened:** Created a Qdrant snapshot from Podman, restored it into native Qdrant v1.19. The restore wrote back through the shared Podman volume. Now both Podman and native Qdrant show `yellow` status with 2590 points, and the engine startup hangs at `CollectionMigration`.

**Recovery options (pick one):**
1. Delete the Qdrant collection and let the engine reindex from garden files (~60 min): `curl -X DELETE http://localhost:6333/collections/hortora_garden`
2. Wait — the yellow collection may finish optimizing eventually and become queryable

**After Qdrant is healthy:** switch project back to feature branch (`git checkout issue-83-version-deemphasis-payload-enrichment`), rebuild, and run work-end.

## What's Left

- Qdrant recovery — get to green status · S · Med
- Native Qdrant migration — binary installed at `~/.hortora/qdrant/qdrant` (v1.19), launchd plist at `io.hortora.qdrant.plist` (needs `WorkingDirectory` fix already applied). Needs clean data · M · Med
- Update `hortora-setup.sh` version to match installed Qdrant · XS · Low
- End-to-end verification of #83/#80 after reindex (new payload fields populated) · S · Low
- `work-end` on `issue-83-version-deemphasis-payload-enrichment` · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #82 | Investigate alternatives to global OrtSession lock | M | High | Future improvement |
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children |
| #58 | Shadow comparison harness | M | Med | Ongoing |
| #61 | Remove dual search | XS | Low | Blocked on #58 |
