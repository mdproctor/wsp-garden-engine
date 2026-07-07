# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-07)

### Branch `issue-40-wire-hyde-query-expansion` closed → main

**Closes #40.** HyDE query expansion wired via `SessionQueryExpander` using platform `AgentProvider.openSession()` for persistent Claude sessions. Benchmark: +15 relevant entries (204→219, +7%), 99% precision, ~10s latency per query.

Root causes resolved during wiring:
1. Arc silently skips `@Decorator` on `@Produces` method beans (neocortex #114 fixed)
2. `AgentProviderChatModel` uses `langchain4j.close-timeout` not `agent.claude.default-timeout` — two configs for same chain
3. Platform modules need `quarkus.index-dependency` for CDI discovery

Two scenario regressions (spec1-d1-cdi-priority-tiers -3 relevant, issue-4-rest-messaging -1). Filed neocortex #115-#118 epic for regression-free expansion.

## Immediate Next Step

**Engine-side HyDE prompt tuning** — three tweaks to try on a new branch, each benchmarkable independently:

1. **Domain-aware prompting** — if query mentions CDI, Hibernate, testing etc., inject domain vocabulary anchors into the HyDE prompt template. Change `application.properties` `casehub.rag.expansion.prompt-template`. · XS · Low

2. **Shorter hypotheticals** — current prompt asks for 3-5 sentences. Try 1-2 sentences to reduce embedding drift. Config-only change. · XS · Low

3. **Confidence gating** — in `SessionQueryExpander.expand()`, check if the hypothetical is too generic (too short, no domain keywords). If so, skip expansion and use original query. ~10 lines of Java. · S · Low

Each can be benchmarked against `scripts/benchmark/results/hyde-session.json` (the current baseline with HyDE). Qdrant needs re-indexing first (Podman machine now has 12GB — `podman machine set --memory 12288` was applied this session).

## Neocortex Dependencies

| Neocortex # | Status | What engine needs |
|---|---|---|
| #105 | **Open** | Retrieval tracking SPI — blocks engine #24 |
| #115 | **Open** | Epic: regression-free query expansion |
| #116 | **Open** | Always include original query in expanded set — eliminates regressions |
| #117 | **Open** | Per-leg embedding separation (supersedes #113) |
| #118 | **Open** | Expansion drift metrics + auto-fallback |

After neocortex #116 lands, re-benchmark to confirm zero regressions.

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#41** | Two-stage overfetch + rerank | M | Med | — | P2, unblocked |
| **#24** | Retrieval frequency tracking | M | Med | neocortex #105 | P3 |

## Key References

| Resource | Location |
|---|---|
| HyDE design spec | `docs/superpowers/specs/2026-07-06-hyde-query-expansion-design.md` |
| HyDE benchmark results | `scripts/benchmark/results/hyde-session.json` |
| Design review workspace | `~/adr/hortora-engine/hyde-query-expansion-20260706-104333/` |
| Garden entries | GE-20260707-23d0ab (timeout gotcha), GE-20260707-4ea952 (session technique) |
