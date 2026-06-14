# Federation Review Findings — Design Spec

Issue: Hortora/engine#6
Branch: `issue-6-federation-review-findings`

## Scope

Three actionable items from the five review findings on the federation chain walk (#5). Two findings (1 and 4) are already resolved in the current code.

## Finding 1: Redundant `@Path("/")` on `RemoteGardenClient` — NO-OP

`RemoteGardenClient` has no class-level `@Path`. Only `@Consumes(MediaType.APPLICATION_JSON)` at class level. Already clean.

## Finding 2: Collapse two executor fields in `ChainWalker`

**Current state:** `managedExecutor` (CDI-injected `ManagedExecutor`) and `executor` (`ExecutorService`) coexist. `@PostConstruct` assigns `executor = managedExecutor`. The split exists so `setExecutor()` can override in tests.

**Change:** Remove `managedExecutor`. Inject `ManagedExecutor` directly into the `executor` field (typed as `ExecutorService`). `setExecutor()` stays for test override.

```java
// Before
@Inject
org.eclipse.microprofile.context.ManagedExecutor managedExecutor;
private ExecutorService executor;

@PostConstruct
void init() {
    executor = managedExecutor;
    // ...
}

// After
@Inject
org.eclipse.microprofile.context.ManagedExecutor executor;

@PostConstruct
void init() {
    // executor already injected — no assignment needed
    // ...
}
```

The field type stays `ManagedExecutor` (CDI resolves the bean by declared type). `setExecutor()` takes `ExecutorService` — the test `DirectExecutor` is not a `ManagedExecutor`, which is correct: the setter exists precisely to swap the implementation in tests.

**Files:** `ChainWalker.java`
**Tests:** Existing `ChainWalkerTest` — no changes needed, `setExecutor()` API unchanged.

## Finding 3: WireMock dynamic port in `FederationIntegrationTest`

**Current state:** Port 9999 hardcoded in three places:
- `FederationIntegrationTest.java` — `wireMockConfig().port(9999)` and `WireMock.configureFor("localhost", 9999)`
- `federation-test.yaml` — `url: http://localhost:9999`

**Change:** Introduce a `QuarkusTestResourceLifecycleManager` that:
1. Starts WireMock on a dynamic port
2. Writes a temp YAML fixture with `http://localhost:<port>`
3. Returns config overrides: `hortora.garden.schema-path`, `hortora.garden.id`, `hortora.garden.id-prefix`

Replace `@TestProfile(FederationTestProfile.class)` with `@QuarkusTestResource(WireMockFederationResource.class)`.

The test accesses the `WireMockServer` instance via `@InjectWireMock` or a static accessor on the resource class.

**federation-test.yaml stays** — used by `FederationConfigParserTest` which parses files directly without CDI. That test doesn't use WireMock.

**Files:**
- New: `WireMockFederationResource.java` (test resource lifecycle manager)
- Modified: `FederationIntegrationTest.java` (remove profile, remove manual WireMock lifecycle, use test resource)
- `federation-test.yaml` — unchanged (still used by parser tests)

## Finding 4: Dedup key uses string concatenation — NO-OP

Code already uses `record DedupKey(String id, String source) {}` with structural `equals`/`hashCode`. No string concatenation present.

## Finding 5: Assert provenance fields in `SearchResourceTest`

**Current state:** `SearchResourceTest` verifies status codes, JSON structure, domain filtering, and limits. No test asserts `source` or `sourcePrefix` on the non-federated path.

**Change:** Add a dedicated test `searchResultsIncludeProvenanceFields` that verifies:
- `source` equals `"garden"` (default `hortora.garden.id`)
- `sourcePrefix` equals `"GE"` (default `hortora.garden.id-prefix`)

**Files:** `SearchResourceTest.java`

## Implementation Order

1. Finding 2 — `ChainWalker` executor collapse (smallest, no test changes)
2. Finding 5 — provenance assertions (add test, no prod code change)
3. Finding 3 — WireMock dynamic port (new test resource class, test refactor)
