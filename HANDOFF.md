# Hortora engine — Project Handoff

---

## What Just Shipped (2026-07-25)

### Branch `issue-50-re-enable-hyde` closed → main

**Closes #50.** Comprehensive HyDE investigation: four approaches benchmarked (double-retrieval, single-retrieval, inverted HyDE, no-HyDE baseline), all -2.2 to -2.5pp precision. Root cause confirmed by Weller et al. (EACL 2024): expansion harms strong multi-signal retrievers.

Inverted HyDE infrastructure built and tested: `OllamaQueryGenerator` (gemma3:4b via Ollama), `QueryAugmentingExtractor` (CDI decorator on MetadataExtractor), sidecar `.queries` cache. Disabled in config. `SessionQueryExpander` removed (query-time HyDE definitively closed).

Cross-repo: neocortex #173 (single-retrieval HyDE — remove original-query prepend) committed to neocortex main. Neocortex #118, #120, #173 closed. Engine #53, #54 closed.

## Immediate Next Step

Pick up #55 (score-based boosting in adaptive filter) — quick experiment testing whether garden entry quality scores improve ranking precision.

## Open Issues

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| **#55** | Score-based boosting in adaptive filter | S | Low | — | Next quality experiment |
| **#45** | Subagent-mediated garden retrieval | M | Med | — | Skill-layer work, not engine |
| **#51** | Expansion drift metrics integration | S | Med | neocortex #118/#120 (closed) | May need new issue if revisited |

## Neocortex Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| **#178** | Configurable per-leg RRF/fusion weights | S | Med |
| **#179** | RetrievalAnalyzer — analytics over retrieval tracking data | M | Med |
| **#180** | Payload-based score boosting in Qdrant prefetch | S | Med |

## Key References

| Resource | Location |
|---|---|
| Inverted HyDE design spec | `specs/issue-50-re-enable-hyde/2026-07-24-inverted-hyde-design.md` |
| Query-time HyDE design spec | `specs/issue-50-re-enable-hyde/2026-07-23-re-enable-hyde-design.md` |
| Benchmark results | `scripts/benchmark/results/hyde-*.json`, `inverted-hyde.json` |
| Blog: The HyDE Wall | `blog/2026-07-25-mdp01-the-hyde-wall.md` |
| Garden entry | `GE-20260725-cae3ad` — expansion harms strong retrievers |
| Design review workspace | `~/adr/hortora-engine/inverted-hyde-20260724-131810/` |
| Weller et al. (EACL 2024) | https://aclanthology.org/2024.findings-eacl.134/ |
