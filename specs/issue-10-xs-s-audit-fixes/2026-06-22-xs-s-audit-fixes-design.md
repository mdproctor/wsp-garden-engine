# XS/S Audit Fixes — Design Spec

Covers issues #10–#18: code quality, test gaps, config, API contracts, docs, build.

## Blocking prerequisite — RetrievalQuery migration

`casehub-rag-api` 0.2-SNAPSHOT changed `CaseRetriever.retrieve()` from `String` to `RetrievalQuery`. The engine doesn't compile:

- `SearchResource.searchLocal()` line 80: passes raw `String` — must wrap with `RetrievalQuery.of(query)`
- `TestCaseRetriever` implements `retrieve(String, ...)` — no longer matches the interface

This is folded into #11 (SearchResource rework) since the fix is in `searchLocal()`. The `TestCaseRetriever` breakage resolves automatically when #16 deletes it — `InMemoryCaseRetriever` from `casehub-rag-testing` already takes `RetrievalQuery`.

---

## Implementation order

Issues are not independent. Required sequence:

1. **#15** — remove unused testcontainers (no dependencies, cleans pom.xml)
2. **#14** — minor code quality fixes (no dependencies on other issues)
3. **#10** — frontmatter CRLF fix + edge-case tests (no dependencies)
4. **#17** — DESIGN.md update (no dependencies)
5. **#11** — SearchResource rework including `RetrievalQuery` migration (**must land before #16** — project won't compile without this, and `InMemoryCaseRetriever` expects `RetrievalQuery`)
6. **#12** — ChainWalker configurable timeouts (depends on #14.3 — `FederationConfig` compact constructor should be in place before adding the new field)
7. **#16** — test doubles reconciliation (depends on #11 — SearchResource must use `RetrievalQuery` before `TestCaseRetriever` is deleted)
8. **#13** — GardenMcpTools tests (depends on #16 — test behaviour depends on which stubs are active)
9. **#18** — ONNX download script (independent, last because it's a new feature not a fix)

---

## #10 — Frontmatter parsing: CRLF + edge-case tests (XS)

**Problem:** `FederationConfigParser.parse()` splits on `"---\n"` and `GardenMetadataExtractor.extract()` uses `text.indexOf("\n---", 3)` — both fail on `\r\n` line endings.

**Fix:** Normalize line endings at parse boundary: `text = text.replace("\r\n", "\n")` before splitting. One line in each class.

**Tests to add (GardenMetadataExtractor):**
- CRLF frontmatter — file with `---\r\n` delimiters parses correctly (verifies the fix)
- Malformed YAML between `---` delimiters — expect empty `Optional` (not unhandled `YamlException`)
- Missing `title` field in frontmatter — verify content still extracted without title prefix
- Unclosed frontmatter (starts with `---`, no closing `---`) — expect empty `Optional`

**Tests to add (FederationConfigParser):**
- CRLF SCHEMA.md fixture — `---\r\n` delimited frontmatter parses to valid `FederationConfig` (verifies the fix)

---

## #11 — SearchResource: typed responses, scope, validation, RetrievalQuery migration, tests (S)

**Design decisions (verified from first principles):**

1. **Return type:** Change `search()` from `Response` to `List<SearchResult>`. No caller parses the error body. `RemoteGardenClient` already declares `List<SearchResult>`. `searchFor()` already returns `List<SearchResult>`. The `Response` wrapper is an inconsistency.

2. **Error handling:** Throw `WebApplicationException(message, BAD_REQUEST)` for missing/blank query. No caller deserializes the 400 body.

3. **Scope:** Add `@ApplicationScoped` — no per-request state. The `visited` set is created fresh per call inside `doSearch()`, not stored as a field.

4. **Default limit:** Extract `private static final int DEFAULT_LIMIT = 8` and `private static final int MAX_LIMIT = 50`. Apply cap as `Math.min(limit, MAX_LIMIT)` in both `search()` and `searchFor()` — both entry points must enforce the cap, not just `doSearch()`.

5. **parseVisited trim:** Add `.map(String::trim)` after split to handle `"a, b, c"` header values.

6. **RetrievalQuery migration:** In `searchLocal()`, wrap the query string: `caseRetriever.retrieve(RetrievalQuery.of(query), corpusRef, maxResults, filter)`. This is the blocking compilation fix.

**Tests to add:**
- `searchFor()` direct call — verify returns results (exercises MCP code path)
- Domain filtering via HTTP — `?domain=jvm` returns only jvm entries

---

## #12 — ChainWalker: configurable federation timeout (S)

**Problem:** Two hardcoded timeouts with different semantics:
- `executor.invokeAll(peerCalls, 5, TimeUnit.SECONDS)` (line 104) — total wall-clock deadline for all peer calls combined
- `readTimeout(5, TimeUnit.SECONDS)` in `buildClient()` (line 178) — per-request HTTP read timeout, applied to both upstream AND peer clients

**Fix:** Single config property `federationTimeoutSeconds` (default 5) applied uniformly to both. Rationale: the per-request read timeout and the fanout deadline serve the same purpose — bounding how long the engine waits for federation partners. A per-request timeout of N seconds and a fanout deadline of N seconds are consistent: no individual request exceeds N, and the total parallel fanout can't exceed N either (since peers run concurrently, the fanout time equals the slowest peer's response time). Splitting into two config properties adds complexity without a real use case — if you want peers to respond faster, lower one value and it naturally applies to both.

Add `federationTimeoutSeconds` to `FederationConfig` record. Wire through `FederationConfigParser` (parse from SCHEMA.md `federation:` block, key `timeout-seconds`, default 5). Use in `ChainWalker`:
- `buildClient()`: `readTimeout(config.federationTimeoutSeconds(), TimeUnit.SECONDS)`
- `walk()`: `executor.invokeAll(peerCalls, config.federationTimeoutSeconds(), TimeUnit.SECONDS)`

---

## #13 — GardenMcpTools: test coverage + exception logging (S)

**Fix:**
- Log exception in `gardenStatus()` catch block: `Log.warn("Failed to count indexed entries", e)`
- Add `GardenMcpToolsTest` with tests for `gardenSearch()` formatting (provenance labels, result joining, empty-results message) and `gardenStatus()` error path.
- The `limit` default coupling with SearchResource is addressed by #11 (`DEFAULT_LIMIT` constant).

---

## #14 — Minor code quality fixes (XS)

Five fixes (reduced from six — see #14.5 rationale):

1. `CollectionMigration`: replace `java.util.logging.Logger` with `io.quarkus.logging.Log`
2. `RemoteGardenClient`: **replace** `@Consumes(MediaType.APPLICATION_JSON)` with `@Produces(MediaType.APPLICATION_JSON)`. `@Consumes` is wrong for a GET-only interface (no request body to consume). `@Produces` sets the `Accept` header, which is what's needed.
3. `FederationConfig`: compact constructor normalizing `null` → `List.of()` for `upstream` and `peers`, remove null checks from `hasUpstream()`/`hasPeers()`
4. Remove `hortora.garden.path` from `application.properties` — `@WithDefault` in `GardenConfig` is the single source. Note: `GardenConfig.schemaPath()` uses `@WithDefault("${hortora.garden.path}/SCHEMA.md")` — this property expression resolves from the `@WithDefault` on `path()` via SmallRye Config's property expression resolution, which treats `@WithDefault` as a config source. Add a test verifying `schemaPath()` resolves correctly when no explicit `hortora.garden.path` property is set.
5. Add `FederationConfigParserTest.invalidSearchOrderThrows()` test

**Deleted item — former #14.5:** `ChainWalkerTest.depthExceededReturnsOwnResultsOnly`. The original spec proposed renaming this test, but the test itself is vacuous. It calls `walker.walk()` with a large visited set and asserts `results.isNotEmpty()` — but `ChainWalker.walk()` doesn't check depth at all (depth is enforced in `SearchResource.doSearch()`). The test comments acknowledge this explicitly. The assertion (`isNotEmpty()`) is already covered by other ChainWalker tests. Delete the test entirely. Depth enforcement is already properly tested at the correct level by `FederationIntegrationTest.depthExceededReturnsOwnResultsWithoutWalking()`, which verifies via HTTP that upstream is NOT called when the visited set exceeds max depth.

---

## #15 — Remove unused testcontainers dependency (XS)

**Verified:** No `@Testcontainers`, `@Container`, or any testcontainers API usage in any `.java` file. Remove both `testcontainers` and `junit-jupiter` testcontainers stanzas from `pom.xml`.

---

## #16 — Reconcile test doubles with casehub-rag-testing (S)

**Prerequisite:** #11 must land first. `SearchResource.searchLocal()` must use `RetrievalQuery.of(query)` before `TestCaseRetriever` (which implements the old `String` signature) is deleted. `InMemoryCaseRetriever` already takes `RetrievalQuery` — deleting `TestCaseRetriever` before fixing SearchResource would leave the project uncompilable.

**Design decision (verified from first principles):**

Adopt `casehub-rag-testing` stubs. Delete engine's hand-written `TestCaseRetriever` and `TestEmbeddingIngestor`.

**Rationale:**
- PLATFORM.md cross-repo dependency map explicitly documents `casehub-rag-testing` as providing stubs for Hortora/engine
- `InMemoryCaseRetriever` and `InMemoryEmbeddingIngestor` are `@Alternative @Priority(1)` — activate by classpath presence (already on test classpath)
- `PayloadFilter.matches()` logic is duplicated identically between `TestCaseRetriever` and `InMemoryCaseRetriever`
- Ingest-then-retrieve is a more realistic test pattern than hardcoded constructor fixtures

**Keep `TestEmbeddingModel`:** Needed to prevent `quarkus-langchain4j-ollama` Dev Services from starting an Ollama container in tests. The production `EmbeddingIngestor` from `casehub-rag` injects `EmbeddingModel`, and CDI validates its injection point at build time even when the alternative is selected.

**Test migration:**
- `SearchResourceTest.searchReturnsJsonArray()` — needs `@BeforeEach` to seed fixture data via `InMemoryEmbeddingIngestor.ingest()` with `ChunkInput` objects matching the test fixtures
- `FederationIntegrationTest` — same: seed local fixture data before federation tests. Tests assert `hasItem("test-garden")` for local provenance — the seeded data must use the same corpus ref.
- Score values change from 0.92/0.85 to 1.0 — no test currently asserts exact scores. Federation integration tests rely on count-vs-limit insufficiency (not score thresholds) for triggering upstream queries.

---

## #17 — DESIGN.md: Phase 2 complete (XS)

Update Phase 2 from "pending" to complete. Add brief summary: `HybridSearchProducer`, `CollectionMigration`, SPLADE sparse embeddings, cross-encoder reranking, RRF fusion, `@LookupIfProperty` fallback to dense-only.

Also update `SPI Contracts` section: `CaseRetriever.retrieve()` now takes `RetrievalQuery` instead of `String`.

---

## #18 — ONNX model download automation (S)

**Approach:** Shell script at `scripts/download-models.sh`.

**Models:**
- `Splade_PP_en_v1` — SPLADE sparse embeddings (from Hugging Face)
- `ms-marco-MiniLM-L-6-v2` — cross-encoder reranker (from Hugging Face)

**Convention:** `~/.hortora/models/{model-name}/` containing `model.onnx` and `tokenizer.json`.

**Operational requirements:**
- **Checksum verification:** SHA-256 checksum for each `model.onnx` and `tokenizer.json`. Script verifies after download; fails with clear error on mismatch. Checksums hardcoded in the script (pinned to specific model versions).
- **Partial download handling:** Download to `{target}.tmp` then `mv` to final path. On Ctrl-C or network failure, the `.tmp` file is left (harmless) and re-running the script re-downloads cleanly. `curl -fSL --retry 3` for resilience.
- **Idempotent:** Skip download if file exists and checksum matches. Re-download if checksum mismatches (model version upgrade).

**Config wiring:** The engine discovers models via `casehub-inference-quarkus` config properties:
- `casehub.inference.models.splade.model-path=~/.hortora/models/Splade_PP_en_v1/model.onnx`
- `casehub.inference.models.splade.tokenizer-path=~/.hortora/models/Splade_PP_en_v1/tokenizer.json`
- `casehub.inference.models.reranker.model-path=~/.hortora/models/ms-marco-MiniLM-L-6-v2/model.onnx`
- `casehub.inference.models.reranker.tokenizer-path=~/.hortora/models/ms-marco-MiniLM-L-6-v2/tokenizer.json`

These go in `application.properties` under a `%dev` profile (dev-mode only — production deployments configure paths via env vars). `HybridSearchProducer` uses `@LookupIfProperty` on `model-path` — beans activate only when paths are configured.

Script prints a summary of what to add to `application.properties` after successful download.
