# XS/S Audit Fixes — Design Spec

Covers issues #10–#18: code quality, test gaps, config, API contracts, docs, build.

---

## #10 — Frontmatter parsing: CRLF + edge-case tests (XS)

**Problem:** `FederationConfigParser.parse()` splits on `"---\n"` and `GardenMetadataExtractor.extract()` uses `text.indexOf("\n---", 3)` — both fail on `\r\n` line endings.

**Fix:** Normalize line endings at parse boundary: `text = text.replace("\r\n", "\n")` before splitting. One line in each class.

**Tests to add (GardenMetadataExtractor):**
- Malformed YAML between `---` delimiters → expect empty `Optional` (not unhandled `YamlException`)
- Missing `title` field in frontmatter → verify content still extracted without title prefix
- Unclosed frontmatter (starts with `---`, no closing `---`) → expect empty `Optional`

---

## #11 — SearchResource: typed responses, scope, validation, tests (S)

**Design decisions (verified from first principles):**

1. **Return type:** Change `search()` from `Response` to `List<SearchResult>`. No caller parses the error body. `RemoteGardenClient` already declares `List<SearchResult>`. `searchFor()` already returns `List<SearchResult>`. The `Response` wrapper is an inconsistency.

2. **Error handling:** Throw `WebApplicationException(message, BAD_REQUEST)` for missing/blank query. No caller deserializes the 400 body.

3. **Scope:** Add `@ApplicationScoped` — no per-request state, all injected beans are singletons.

4. **Default limit:** Extract `private static final int DEFAULT_LIMIT = 8` used by both `search()` and `searchFor()`. Add `private static final int MAX_LIMIT = 50` cap.

5. **parseVisited trim:** Add `.map(String::trim)` after split to handle `"a, b, c"` header values.

**Tests to add:**
- `searchFor()` direct call — verify returns results (exercises MCP code path)
- Domain filtering via HTTP — `?domain=jvm` returns only jvm entries

---

## #12 — ChainWalker: configurable timeouts (S)

**Fix:** Add `peerTimeoutSeconds` to `FederationConfig` record. Wire through `FederationConfigParser` (parse from SCHEMA.md YAML, default 5). Use in `ChainWalker.walk()` (`invokeAll` timeout) and `buildClient()` (`readTimeout`).

---

## #13 — GardenMcpTools: test coverage + exception logging (S)

**Fix:**
- Log exception in `gardenStatus()` catch block: `Log.warn("Failed to count indexed entries", e)`
- Add `GardenMcpToolsTest` with tests for `gardenSearch()` formatting (provenance labels, result joining, empty-results message) and `gardenStatus()` error path.
- The `limit` default coupling with SearchResource is addressed by #11 (DEFAULT_LIMIT constant).

---

## #14 — Minor code quality fixes (XS)

Six one-liner fixes:

1. `CollectionMigration`: replace `java.util.logging.Logger` with `io.quarkus.logging.Log`
2. `RemoteGardenClient`: add `@Produces(MediaType.APPLICATION_JSON)` — explicitly set Accept header
3. `FederationConfig`: compact constructor normalizing `null` → `List.of()` for `upstream` and `peers`, remove null checks from `hasUpstream()`/`hasPeers()`
4. Remove `hortora.garden.path` from `application.properties` — `@WithDefault` in `GardenConfig` is the single source
5. Rename `ChainWalkerTest.depthExceededReturnsOwnResultsOnly` → `visitedSetExceedingMaxDepthReturnsOwnResults` (accurate description)
6. Add `FederationConfigParserTest.invalidSearchOrderThrows()` test

---

## #15 — Remove unused testcontainers dependency (XS)

**Verified:** No `@Testcontainers`, `@Container`, or any testcontainers API usage in any `.java` file. Remove both `testcontainers` and `junit-jupiter` testcontainers stanzas from `pom.xml`.

---

## #16 — Reconcile test doubles with casehub-rag-testing (S)

**Design decision (verified from first principles):**

Adopt `casehub-rag-testing` stubs. Delete engine's hand-written `TestCaseRetriever` and `TestEmbeddingIngestor`.

**Rationale:**
- PLATFORM.md cross-repo dependency map explicitly documents `casehub-rag-testing` as providing stubs for Hortora/engine
- `InMemoryCaseRetriever` and `InMemoryEmbeddingIngestor` are `@Alternative @Priority(1)` — activating by classpath presence (already on test classpath)
- `PayloadFilter.matches()` logic is duplicated identically between `TestCaseRetriever` and `InMemoryCaseRetriever`
- Ingest-then-retrieve is a more realistic test pattern than hardcoded constructor fixtures

**Keep `TestEmbeddingModel`:** Needed to prevent `quarkus-langchain4j-ollama` Dev Services from starting an Ollama container in tests. `GardenBindingProducer` does NOT inject `EmbeddingModel` (only `GardenConfig` and `GardenMetadataExtractor`), but the production `EmbeddingIngestor` from `casehub-rag` does, and CDI validates its injection point even when the alternative is selected.

**Test migration:**
- `SearchResourceTest.searchReturnsJsonArray()` — needs `@BeforeEach` to seed fixture data via `InMemoryEmbeddingIngestor.ingest()`
- `FederationIntegrationTest` — same: seed local fixture data before federation tests
- Score values change from 0.92/0.85 to 1.0 — no test currently asserts exact scores

---

## #17 — DESIGN.md: Phase 2 complete (XS)

Update Phase 2 from "pending" to complete. Add brief summary: `HybridSearchProducer`, `CollectionMigration`, SPLADE sparse embeddings, cross-encoder reranking, RRF fusion, `@LookupIfProperty` fallback to dense-only.

---

## #18 — ONNX model download automation (S)

**Approach:** Shell script in project root (`scripts/download-models.sh`) that fetches both ONNX models to `~/.hortora/models/` if not already present. Document in CLAUDE.md and `application.properties` comments. Quarkus Dev Service is overkill for two static model files — a script is simpler and works in all environments (dev, CI, production).

Models:
- `Splade_PP_en_v1` — from Hugging Face
- `ms-marco-MiniLM-L-6-v2` — from Hugging Face

Convention: `~/.hortora/models/{model-name}/` with `model.onnx` and `tokenizer.json`.
