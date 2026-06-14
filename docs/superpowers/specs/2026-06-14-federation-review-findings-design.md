# Federation Review Findings — Design Spec

Issue: Hortora/engine#6
Branch: `issue-6-federation-review-findings`

## Scope

Two actionable items from the five review findings on the federation chain walk (#5). Three findings (1, 2, and 4) are already resolved or correct in the current code.

## Finding 1: Redundant `@Path("/")` on `RemoteGardenClient` — NO-OP

`RemoteGardenClient` has no class-level `@Path`. Only `@Consumes(MediaType.APPLICATION_JSON)` at class level. Already clean.

## Finding 2: Two executor fields in `ChainWalker` — NO-OP

The two-field pattern (`managedExecutor` + `executor`) is a deliberate type bridge, not redundancy:

- **CDI side:** injects `ManagedExecutor` — unambiguous bean resolution. Injecting by `ExecutorService` would risk CDI ambiguity if Vert.x or other extensions register `ExecutorService` beans.
- **Usage side:** `executor` typed as `ExecutorService` — the widest type supporting `invokeAll()`.
- **Test side:** `setExecutor(ExecutorService)` accepts the test `DirectExecutor`, which implements `ExecutorService` but not `ManagedExecutor`.

The `@PostConstruct` assignment `executor = managedExecutor` is one line bridging these three type requirements. `ManagedExecutor extends ExecutorService` (not the reverse), so typing the field as `ManagedExecutor` would break `setExecutor()` — `ExecutorService` is not assignable to `ManagedExecutor`. The current code is correct and minimal.

## Finding 3: WireMock Dynamic Port in `FederationIntegrationTest`

**Current state:** Port 9999 hardcoded in three places:
- `FederationIntegrationTest.java` — `wireMockConfig().port(9999)` and `WireMock.configureFor("localhost", 9999)`
- `federation-test.yaml` — `url: http://localhost:9999`

**Change:** Introduce a `QuarkusTestResourceLifecycleManager` (`WireMockFederationResource`) that:

1. `start()`: starts WireMock on a dynamic port (`wireMockConfig().dynamicPort()`), writes a temp YAML fixture with `http://localhost:<port>`, returns config overrides:
   - `hortora.garden.schema-path` → temp file path
   - `hortora.garden.id` → `"test-garden"`
   - `hortora.garden.id-prefix` → `"TG"`

2. `inject(TestInjector)`: injects the `WireMockServer` instance into the test class via `testInjector.injectIntoFields(wireMock, new TestInjector.MatchesType(WireMockServer.class))`. The test class declares a plain `WireMockServer wireMock;` field — no annotation needed, no static accessor.

3. `stop()`: stops WireMock and deletes the temp YAML file.

The test annotation must include `restrictToAnnotatedClass = true`:
```java
@QuarkusTestResource(value = WireMockFederationResource.class, restrictToAnnotatedClass = true)
```
Without this, the resource's config overrides bleed into all `@QuarkusTest` classes in the module — `SearchResourceTest` expects default canonical config and would break silently.

`@QuarkusTestResource` is the correct annotation. `@WithTestResource` (introduced in 3.13) has a standing warning in its Javadoc in 3.36.1: "this annotation caused some issues so it was decided to undeprecate `@QuarkusTestResource`... For now, we recommend not using it."

Replace the inner `FederationTestProfile` class with the test resource. Remove manual WireMock start/stop lifecycle (`@BeforeAll`/`@AfterAll`). The `@BeforeEach` reset stays — the lifecycle manager handles server lifecycle, not per-test stub cleanup. `wireMock.resetAll()` prevents stubs from one test leaking into the next.

**`federation-test.yaml` stays** — used by `FederationConfigParserTest` which parses files directly without CDI. That test doesn't use WireMock.

**Files:**
- New: `WireMockFederationResource.java` (test resource lifecycle manager)
- Modified: `FederationIntegrationTest.java` (remove profile + manual WireMock lifecycle, add `@QuarkusTestResource`, declare `WireMockServer` field)
- `federation-test.yaml` — unchanged (parser tests only)

## Finding 4: Dedup key uses string concatenation — NO-OP

Code already uses `record DedupKey(String id, String source) {}` with structural `equals`/`hashCode`. No string concatenation present.

## Finding 5: Assert Provenance Fields in `SearchResourceTest`

**Current state:** `SearchResourceTest` verifies status codes, JSON structure, domain filtering, and limits. No test asserts `source` or `sourcePrefix` on the non-federated path.

**Change:** Add a dedicated test `searchResultsIncludeProvenanceFields` that verifies:
- `source` equals `"garden"` (default `hortora.garden.id`)
- `sourcePrefix` equals `"GE"` (default `hortora.garden.id-prefix`)

**Files:** `SearchResourceTest.java`

## Implementation Order

1. Finding 5 — provenance assertions (add test, no prod code change)
2. Finding 3 — WireMock dynamic port (new test resource class, test refactor)
