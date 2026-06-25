# TODO

> **Current Version:** V1

## V1 (Current)

> Status: Partial — middleware and routes wired, net/http ServeMux in use. All handlers return empty results. Typesense not initialized.

### OpenAPI

- **[L]** Complete `openapi.yaml`

### Open Items

- **[H]** Initialize Typesense client after secrets load (`TODO(typesense)` in main.go)
- **[H]** Implement `GET /search` — build query params from request, call Typesense, return results
- **[H]** Implement `POST /v1/index/sync` — full re-index from shop-items-api S3 data into Typesense (previously referred to as `/internal/index/sync` — path corrected to match ROUTES.md)
- **[M]** Wire events-api subscriber to listen for `shop_item.created/updated/deleted` → incremental index updates
- **[M]** Implement `DELETE /v1/index` — drop and recreate Typesense collection for schema migrations (previously referred to as `/internal/index` — path corrected to match ROUTES.md)
- **[L]** Add integration tests for search query building and index sync
- **[L]** Wire `GET /health/ready` in `cmd/public/main.go` once Typesense is initialized — checkers: `HTTPChecker("typesense", "http://"+os.Getenv("TYPESENSE_HOST")+":"+os.Getenv("TYPESENSE_PORT")+"/health")`; Typesense exposes a native `/health` endpoint; blocked on forge SDK `api/handlers/health` release and Typesense init (tracked above)

## Testing

- **[M]** **Implement CI test stack** — add `github.com/stretchr/testify` and `go.uber.org/mock` to `go.mod`; generate mocks from the Typesense client interface via `mockgen -source`; convert stub `*_test.go` files to real unit tests (table-driven, `t.Run` subtests) for query building with `net/http/httptest` for handler layer; add `testutil.Component(t)` / `testutil.Integration(t)` tier decorators from the SDK (`github.com/rdevitto86/komodo-forge-sdk-go/testing/testutil`, `TEST_TIER`-gated; default tier is `unit`); add `testcontainers-go` for integration tests against a Typesense container; apply section banners. Reference auth-api as the canonical pattern once its retrofit is complete.

## Audit findings — gaps from audit

### Escape/allowlist Typesense filter values when implementing Search (filter injection)
**Problem:** The `Search` stub's reference implementation builds `FilterBy` by concatenating user input — `FilterBy = "type:=" + params.ItemType` and `" && category:=" + params.Category` (`internal/repository/typesense.go:67-72`). `ItemType`/`Category` originate from the request, so as written this is a filter-injection vector: a crafted value can inject additional filter clauses or break out of the expression.
**Action:** When wiring Typesense, validate `ItemType` against the known enum and `Category` against an allowlist (or use the client's parameterized filter API / escape reserved chars). Never concatenate raw request strings into `FilterBy`. Capture this in the search query-building unit tests.
