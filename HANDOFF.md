# Hortora engine — Project Handoff

---

## What Just Shipped (2026-08-07)

**#80 closed — Payload enrichment.** `GardenMetadataExtractor` writes `staleness_threshold`, `tags`, `last_reviewed`, `author`, `verified_on` as Qdrant payload fields at ingestion. `SearchResult` exposes them via derived `@JsonProperty` methods in the REST API response. Null-guarded for federation-deserialized instances where metadata is null. Backfill on reindex. Grove can now read these directly without parsing entry content.

**#83 closed — Version de-emphasis.** `TemporalDecayScorer` applies half-life decay by staleness tier (fast/standard/slow/evergreen). `VersionScorer` applies BOM-relative version distance with topic weighting. Both wired into `SearchResource.applyScoring()`, configurable via `SearchScoringConfig`. `SearchProfileStore` (SQLite) + `ProfileResource` (REST CRUD) for named BOM snapshots. `gardenSearch` MCP tool accepts `profile` and `stack` params. Client-side BOM resolution script and hook.

**Embedding cache enabled** for `HybridSearchProducer` via `CachingMultiModalEmbedder`.

**Garden entry:** `GE-20260807-9a4872` — Jackson `@JsonIgnore` on record component passes null during deserialization, causing NPE in derived `@JsonProperty` methods.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | Qdrant snapshots + native installer | L | High | End-user setup solution |
| #84 | ClaudeAgentClient crashes when claude CLI missing | XS | Low | — |
| #82 | Investigate alternatives to global OrtSession lock | M | High | Future improvement |
| #72 | Epic: gardenSearch quality & reliability | — | — | 2 open children |
| #58 | Shadow comparison harness | M | Med | Ongoing |
| #61 | Remove dual search | XS | Low | Blocked on #58 |

## Key References

| Resource | Location |
|---|---|
| Design specs | `docs/specs/issue-83-version-deemphasis-payload-enrichment/` |
| Diary entry | `blog/2026-08-07-mdp01-invisible-payload.md` |
| Garden entry | `GE-20260807-9a4872` (Jackson @JsonIgnore null on records) |
