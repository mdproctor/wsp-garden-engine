# Inverted HyDE: Index-Time Query Generation

**Issue:** Hortora/engine#50
**Date:** 2026-07-24
**Status:** Approved

## Context

HyDE (query-time hypothetical document generation) was benchmarked twice and is definitively net-negative for this corpus (-2.2pp precision in both double-retrieval and single-retrieval modes). The root cause, confirmed by Weller et al. (EACL 2024), is that generative query/document expansion harms strong retrievers — expansion improves recall but adds noise that degrades ranking precision among top candidates.

Inverted HyDE flips the approach: instead of generating hypothetical *documents* from queries at query time, generate hypothetical *queries* from documents at *index time*. This grounds the expansion in actual corpus content and eliminates the query-time latency penalty. The generated queries bridge the vocabulary gap between how developers search and how garden entries are written.

### Prior benchmark results (this branch)

| Config | File | Precision | Delta vs baseline |
|--------|------|-----------|-------------------|
| No HyDE (baseline) | `crossencoder-pool50-scored.json` | 61.6% | — |
| Double-retrieval HyDE | `hyde-perleg-separation.json` | 59.4% | -2.2pp |
| Single-retrieval HyDE | `hyde-single-retrieval.json` | 59.4% | -2.2pp |

## Approach

**Content augmentation with separator — engine-only, no neocortex changes.**

### Query generation

`GardenMetadataExtractor` gains an optional Ollama dependency. After extracting content and metadata from a garden entry, it generates 3 hypothetical queries using a local model (gemma3:4b) with a domain-tuned prompt:

```
Given this knowledge garden entry about JVM development, generate exactly 3 short questions
that a developer would type into a search box to find this entry.
Use the same technical vocabulary the entry uses — class names, annotations, error messages.
One question per line, no numbering, no explanations.
```

Input: entry title + first ~500 chars of body. Output: 3 lines of query text.

### Content augmentation

Generated queries are appended to the `ExtractionResult.content` with a separator:

```
[original content — title + body]

---HYPOTHETICAL-QUERIES---
How does @DefaultBean interact with @Produces in Quarkus?
Why does CDI ignore @DefaultBean on class-level annotations?
What pattern enables consumer-replaceable CDI defaults?
```

BGE-M3 embeds the full combined text. Dense, sparse, BM25, and ColBERT all pick up the query vocabulary. The hypothesis: sparse/BM25 benefit most from vocabulary bridging, dense benefits less (consistent with Weller et al.'s finding that expansion helps recall more than precision).

### Sidecar caching

Generated queries are cached in `.queries` files alongside each garden entry:

```
~/.hortora/garden/jvm/GE-20260515-fd3156.md       ← entry
~/.hortora/garden/jvm/GE-20260515-fd3156.queries   ← cached queries
```

One query per line, plain text. Regeneration logic:
- `.queries` exists and entry `.md` is not newer → use cached (no LLM call)
- `.queries` missing or `.md` is newer → generate via Ollama, write cache

Files are local artifacts, not checked into the garden repo (`*.queries` in `.gitignore`).

### Display stripping

`SearchResource` strips the `---HYPOTHETICAL-QUERIES---` section from `chunk.content()` before returning results. Search results show only original entry content.

### Graceful degradation

If Ollama is unavailable (not running, model not pulled):
- `GardenMetadataExtractor` logs a warning and returns content without augmentation
- Entries are indexed normally — same as current baseline
- Tests: `InMemoryEmbeddingIngestor` doesn't embed, so no impact

### Configuration

```properties
# Inverted HyDE — local query generation via Ollama
hortora.inverted-hyde.enabled=true
hortora.inverted-hyde.ollama.host=http://localhost:11434
hortora.inverted-hyde.ollama.model=gemma3:4b
hortora.inverted-hyde.query-count=3
```

## Changes

| File | Change |
|------|--------|
| `GardenMetadataExtractor` | Add Ollama query generation + sidecar cache read/write |
| `SearchResource` | Strip `---HYPOTHETICAL-QUERIES---` section from content |
| `application.properties` | Inverted HyDE config (Ollama host, model, enabled, query count) |
| `pom.xml` | HTTP client dependency for Ollama API (if not already present) |

## Verification

Re-run the existing benchmark harness (`scripts/benchmark/`) against all 14 real-world scenarios.

**Comparison:**
- **Baseline:** `crossencoder-pool50-scored.json` (no HyDE, current pipeline)
- **Treatment:** inverted HyDE (augmented content, same pipeline)

**Acceptance criteria:**
- Zero precision regressions on any individual scenario
- At least 1 scenario shows improvement
- Latency: no query-time overhead expected (augmentation is index-time only)

**Rollback:** Qdrant snapshot `hortora_garden-2913056546253732-2026-07-24-03-04-39.snapshot` restores the baseline index. No re-indexing needed.

### Testing

Unit tests for:
- `GardenMetadataExtractor` — content augmentation with/without queries, separator format
- `SearchResource` — stripping logic for the separator
- Sidecar cache — read/write/staleness detection

No integration tests against Ollama — graceful degradation means tests run without it.

## Out of Scope

- Separate named vectors for query embeddings (approach C — graduate to this if A works)
- Expansion drift detection (#118/#120 — complementary but independent)
- LLM quality evaluation framework (manual sampling for now)
- Query-time HyDE re-enablement (definitively closed by benchmarks)

## References

- [Weller et al., "When do Generative Query and Document Expansions Fail?" EACL 2024](https://aclanthology.org/2024.findings-eacl.134/)
- [Inverted HyDE: Solving Real-World Dense Retrieval Challenges](https://behitek.com/blog/inverted-hyde/)
- Prior benchmarks: `scripts/benchmark/results/hyde-perleg-separation.json`, `hyde-single-retrieval.json`
