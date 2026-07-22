# Re-enable HyDE Query Expansion

**Issue:** Hortora/engine#50
**Date:** 2026-07-23
**Status:** Approved

## Context

HyDE (Hypothetical Document Embeddings) query expansion was implemented in `SessionQueryExpander` but disabled after benchmarking showed it was net-zero at limit=16 with four-signal retrieval. The root cause: all retrieval legs used `searchText()`, so the hypothetical replaced the original query everywhere — dense got better semantic coverage but sparse/BM25 lost their lexical signal.

Since then, neocortex #113 landed per-leg embedding separation in `HybridCaseRetriever` and `ReactiveHybridCaseRetriever`:

- **Dense** → `searchText()` (hypothetical — semantic match)
- **Sparse/ColBERT** → `text()` (original — lexical signal preserved)
- **BM25** → `query.text()` (original — keyword match preserved)

The separation is tested (`embedBatch([searchText, text])` with expansion, `embed(searchText)` without). HyDE should now be purely additive.

## Approach

**Config-only change — no code modifications.**

### application.properties

Uncomment three lines:

```properties
casehub.rag.expansion.enabled=true
casehub.rag.expansion.mode=session
casehub.rag.expansion.prompt-template=Question: %s
```

Remove the "DISABLED" comment explaining the old rationale.

### Activation path

1. `@IfBuildProperty(name = "casehub.rag.expansion.mode", stringValue = "session")` activates `SessionQueryExpander`
2. `QueryExpandingCaseRetriever` (CDI decorator from `casehub-neocortex-rag-expansion`) calls `SessionQueryExpander.expand()`
3. `SessionQueryExpander` opens an `AgentSession` via `AgentProvider` (Claude on Vertex AI), generates a hypothetical garden entry
4. `RetrievalQuery.withExpansion(hypothetical)` sets the expansion text
5. `HybridCaseRetriever` routes: dense uses `searchText()` (hypothetical), sparse/BM25 use `text()` (original)

### Dependencies

- Claude agent via Vertex AI (`ANTHROPIC_VERTEX_PROJECT_ID`, `CLOUD_ML_REGION`, `CLAUDE_CODE_USE_VERTEX` env vars)
- `casehub-neocortex-rag-expansion` module (already a dependency)
- `casehub-platform-agent-claude` module (already a dependency)

## Verification

Re-run the existing benchmark harness (`scripts/benchmark/`) against all 14 real-world scenarios.

**Comparison:**
- **Baseline:** current config (HyDE off, five-signal + cross-encoder + adaptive filtering)
- **Treatment:** HyDE on (same pipeline, expansion enabled)

**Acceptance criteria:**
- Zero precision regressions on any individual scenario
- At least 1 scenario shows improvement (HyDE is net-positive, not net-zero)
- Latency measured and documented (expected ~2.4s overhead per query from LLM round-trip)

### Testing

Existing `SessionQueryExpanderTest` covers confidence gating and skip logic. No new unit tests needed — the change is config, not code. The benchmark is the integration test.

## Out of Scope

- Latency optimisation (caching, async prefetch) — future work if needed
- Expansion drift metrics (#51) — independent, blocked on neocortex #118/#120
- API-level embedding contract (neocortex #117) — design improvement to prevent regression, not a correctness gate
- Issue #50 acceptance criteria update — remove stale #117 blocker language
