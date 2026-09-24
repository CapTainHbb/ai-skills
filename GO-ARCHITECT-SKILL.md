---
name: architect
description: Reference for working inside the captain-backend (financial-account) Go codebase. Use when designing, adding, or modifying features — usecases, HTTP handlers, repositories, domain models, events, error handling, translation (en/fa), testing, logging, and resilience/failover patterns. Explains how the project actually does things.
---

# Architect — captain-backend conventions

This skill documents how this repository is actually built, so new code looks like
existing code. Follow these patterns when adding or changing anything.

Module: `github.com/accounting-module/financial-account`. Go 1.26, Gin, GORM + Postgres,
slog logging, Go-channel event broker.

---

## 1. Layering and dependency direction

```
handler (internal/api/http/<entity>)        → depends on usecase interfaces
usecase (internal/usecase/<entity>)         → depends on repository interfaces, other usecase interfaces, events, broker
repository (internal/repository/<entity>)   → depends on domain + GORM
domain (internal/domain)                     → pure structs/constants, stdlib only
```

- Imports flow **inward only**: `handler → usecase → repository → domain`.
- Handlers never import `repository/` or `broker` or `events`.
- Usecases never import `gin`, `gorm`, or a concrete postgres implementation.
- Domain never imports GORM/gin/broker.
- Cross-cutting helpers live in `pkg/` (`broker`, `logger`) — imported by any layer.
- The composition root is `internal/app/app.go` → `Build(deps) *gin.Engine`, shared by
  `cmd/main/main.go` and `internal/integration` tests. Wire there, not inline in main.

### Wire order in `internal/app/app.go`

repos → usecases → handlers → `RegisterRoutes(router)`. Event consumers and periodic
fetchers are launched as goroutines: `go uc.ProcessEvents(ctx)` / `go uc.StartPeriodicFetch(ctx)`.

---

## 2. Domain (`internal/domain/`)

- One file per entity (e.g. `journalentry.go`, `buysellcashdocument.go`).
- Structs with exported fields only; no JSON/GORM/binding tags; no methods beyond trivial helpers.
- Enum-like states are `const` string blocks:

```go
const (
	JournalEntryStatusDraft  = "draft"
	JournalEntryStatusPosted = "posted"
)
```

- Config structs live here too (`config.go`) with `yaml:"..."` tags.
- `AuthClaims` (from JWT) lives here; `domain.AdminRoleName` gates admin shortcuts.

---

## 3. Repository (`internal/repository/<entity>/`)

Files per entity package: `interface.go`, `entity.go`, `dto.go`, `postgres.go`, `postgres_test.go`, `mocks/`.

- **interface.go** — contract + `//go:generate mockery --name=XRepository --output=mocks --outpkg=mocks`.
- **entity.go** — GORM model embedding `gorm.Model`, plus `ToDomain()` and `FromDomainToEntity()` /
  `FromDomain(...)`. Relationships reference other entity GORM structs via `gorm:"foreignKey:..."`.
- **dto.go** — exported filter structs for query methods; optional filters are slices/pointers.
- **postgres.go** — unexported impl struct holding `*gorm.DB`; constructor returns the interface:
  `func NewXPostgres(db *gorm.DB) XRepository`.
- `GetAll` returns `([]domain.X, int64, error)` — data + **total count**. Count uses a cloned
  query before `Limit/Offset`: `query.Session(&gorm.Session{})`.

`GetAll` shapes queries with `r.db.WithContext(ctx).Where(...)`, applies optional filters,
then counts on the clone and pages with `.Limit(limit).Offset(offset)`.

Integration tests (white-box, same package) use `testhelper.SetupDB(t)` + `db.AutoMigrate(...)`
+ `t.Cleanup(func() { testhelper.CleanTables(...) })`.

---

## 4. Usecase (`internal/usecase/<entity>/`)

Files: `crud.go` (or descriptive name), `dto.go`, `error.go`, `*_test.go`, `mocks/`.

- Interface named `XUsecase` / `CRUDUsecase` with the `//go:generate mockery` directive.
- Impl struct holds **interfaces** (repo, other usecases, `broker.Broker`, `idvalidator.IDValidator`).
- Constructor injects dependencies and returns the interface.
- **Business validation happens first** (`validateDocument`), returning sentinel errors.
- Mutations that post to the ledger: `Create` doc → build `domain.JournalEntry` → call
  `journalEntryCrudUsecase.Create` )→ **compensating rollback if it fails** → publish event →
  `eventCRUDUsecase.RecordEvent(...)` for audit. Mirror this shadow flow.
- `Update` = `Revert` (reversing entry) then `Create`; roll back the reversing entry if create fails.
- Filter DTOs are usecase-owned and converted to repo DTOs inside the method (`GetAll`).
- Log via `logger.FromContext(ctx)` (never the global), so `request_id` attaches.

### Event consumers

Consumers (`runningbalance`, `generalledger`, `billing`, `documentgroupcleanup`) implement
`ProcessEvents(ctx)`:

```go
ch := u.broker.Subscribe()
defer u.broker.Unsubscribe(ch)
for {
	select {
	case event := <-ch:
		eventCtx := ctx
		if cid := event.GetCorrelationID(); cid != "" {
			eventCtx = logger.ContextWithRequestID(ctx, cid)
		}
		switch event.EventType() {
		case events.EventJournalEntryPosted:
			je, ok := event.GetEventData().(domain.JournalEntry)
			if !ok {
				logger.FromContext(eventCtx).Error("failed to cast event ...")
				break
			}
			u.handleJournalEntryPosted(eventCtx, &je)
		...
		}
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

Type-switch on `EventType()`, cast data with the `ok, ok :=` guard, log on cast failure,
and **log-and-continue on handler errors** (never panic/stop the consumer loop).

---

## 5. Events (`internal/events/`)

- One file per event. Every event implements the `events.Event` interface:
  `EventType() string`, `GetEventData() interface{}`, `GetCorrelationID() string`.
- Constructor returns `Event` and takes `correlationID string` (from `logger.RequestIDFromContext(ctx)`)
  so async consumers can thread the request id into their logs:

```go
const EventJournalEntryPosted = "journal_entry_posted"

type JournalEntryPosted struct {
	CorrelationID string
	JournalEntry  domain.JournalEntry
}

func NewJournalEntryPosted(correlationID string, j domain.JournalEntry) Event {
	return JournalEntryPosted{CorrelationID: correlationID, JournalEntry: j}
}
```

- Event constants are lowercase snake_case (`journal_entry_posted`, `buy_sell_cash_document_deleted`).
- Broker (`pkg/broker`): in-memory pub/sub. `Subscribe() chan events.Event`,
  `Unsubscribe(ch)`, `Publish(msg events.Event)`. Events are fire-and-forget.
- Audit trail uses `eventusecase.CRUDUsecase.RecordEvent(ctx, eventType, entityType, entityID,
  oldData, newData, metadata)` which JSON-serializes payloads into the `events` table. These
  are **also** recorded on mutations, alongside published events.

---

## 6. Error handling

Three-layer pattern:

1. **Usecase** defines sentinel errors in `error.go`:

```go
var (
	ErrInvalidTradeType   = errors.New("trade type must be 'buy' or 'sell'")
	ErrUnbalancedJournalEntry = errors.New("unbalanced journal entry")
	...
)
```

2. **Handler** maps sentinels → (HTTP status, message code) with `translateErrorToCode`:

```go
func translateErrorToCode(err error) (int, string) {
	switch {
	case errors.Is(err, buysellcashusecase.ErrInvalidTradeType):
		return http.StatusBadRequest, INVALID_TRADE_TYPE
	...
	default:
		return http.StatusInternalServerError, INTERNAL_SERVER_ERROR
	}
}
```

Always use `errors.Is`, never typed assertions; default is `500 + INTERNAL_SERVER_ERROR`.

3. **Handler** responds via `writeError(c, status, code)` (see Translation below).

Other conventions:

- Rollback/compensation failures combine errors with `errors.Join(err, delErr)`.
- Lookup fallbacks log a `Warn` and return `nil` instead of failing (e.g. missing
  "Foreign Exchange Gain/Loss" accounts skip FX lines).
- Repository errors are returned as-is (no wrapping) and bubble to the usecase.
- Non-error parser failures (bad JSON body) log the raw error and return a localized code:

```go
if err := c.ShouldBindJSON(&request); err != nil {
	logger.FromContext(c.Request.Context()).Error("invalid json body", "error", err)
	writeError(c, http.StatusBadRequest, INVALID_REQUEST)
	return
}
```

Status codes in use: 400 (bad input/id bind), 401 (auth), 403 (RBAC), 404 (not found when
`doc.ID == 0` after a successful fetch), 409 avoided, 429 (rate limit), 500.

---

## 7. Translation (en/fa)

- **Locale resolution**: `internal/middleware/locale.go`. Priority:
  1. `Language` header
  2. `Accept-Language`
  3. default `"en"` (`fa*` prefix → fa, otherwise en).
  The resolved locale is set on the Gin context (`c.Set("locale", locale)`) and echoed back
  in the `Language` response header.
- **Per-handler-package message map**, keyed by snake_case code constants:

```go
const INVALID_LIMIT = "invalid_limit"

var errorMessages = map[string]map[string]string{
	INVALID_LIMIT: {"en": "Invalid limit", "fa": "محدوده نامعتبر است"},
}

func writeError(c *gin.Context, status int, code string) {
	locale, _ := c.Get("locale")
	lang, ok := locale.(string)
	if !ok || lang == "" {
		lang = "en"
	}
	msg := errorMessages[code][lang]
	if msg == "" {
		msg = errorMessages[code]["en"] // always fall back to English
	}
	c.JSON(status, ErrorResponse{Code: code, Message: msg})
}
```

- The response shape is always `{"code": "...", "message": "..."}` via the handler's own
  `ErrorResponse` struct (defined per handler `dto.go`).
- Message codes live in `internal/api/http/<entity>/error.go`. Translation lives with the
  handler, not in the usecase layer. Do not translate inside usecases.

---

## 8. HTTP handlers (`internal/api/http/<entity>/`)

`handler.go`, `dto.go`, `error.go` (+ shared router).

- Handler struct holds **usecase interfaces + policyEnforcer + claimUsecase**:

```go
type Handler struct {
	crudUsecase    buysellcashusecase.CRUDUsecase
	policyEnforcer policyusecase.PolicyEnforcer
	claimUsecase   authusecase.ClaimUsecase
}
```

- `RegisterRoutes(router gin.IRouter)`:
  - `group := router.Group("/<plural>")`
  - `group.Use(middleware.AuthRequired(h.claimUsecase))`
  - Each route wrapped in `middleware.Authorize(h.policyEnforcer, policyusecase.ObjectTypeX,
    policyusecase.ActionY, resourceIDFunc)` — pass `nil` for collection endpoints; pass a func
    returning the path id for per-resource endpoints.
  - **Order matters in gin**: static routes before `/:id` — e.g. `GET /latest-id` before `GET /:id`.
- Swagger annotations (`@Summary`, `@Security Bearer`, `@Tags`, `@Param`, `@Success`,
  `@Failure {object} ErrorResponse`, `@Router`) on every method. Regenerate docs with `make swagger`.
- **Pagination**: parse `limit`/`offset` manually with `strconv.Atoi`, defaults `100`/`0`,
  invalid values → `400 INVALID_LIMIT` / `INVALID_OFFSET`.
- ID parsing: `strconv.Atoi(c.Param("id"))` → `400 INVALID_ID`; response to userID from
  `c.Get("userID")` (a string set by `AuthRequired`); convert with `strconv.Atoi`.
- Request binding: `c.ShouldBindJSON(&req)` → `400 INVALID_REQUEST`; mutations then call the
  usecase and map errors through `translateErrorToCode`.
- Responses: `c.JSON(http.StatusOK, buildXToResponse(doc))`; slice conversion helper
  `func xToResponse(d domain.X) XResponse` (and loop at call site); mutations return 201 Created,
  deletes 204 No Content.
- Middleware chain (global, in `internal/api/http/router.go`): `Recovery` → `RequestID` →
  `Logging` → `Locale` → CORS. `RequestID` reads/assigns `X-Request-ID` and puts it in ctx;
  `Logging` logs every request with status/method/path/latency/request_id.

---

## 9. Auth, RBAC, and utilities

- `middleware.AuthRequired(claimUsecase)`: validates `Authorization: Bearer <token>`, sets
  `userID`, `username`, `authClaims` on the Gin context.
- `middleware.Authorize(...)`: `admin` role bypasses; otherwise policy enforcer checks
  role/object/action. Returns 401/400/403 on failure.
- `middleware.AdminRequired()`: role must be `domain.AdminRoleName`.
- `middleware.LoginRateLimiter`: in-memory per-IP sliding window for `/auth/login`; returns 429.
- JWT: `authusecase.NewClaimUsecase(cfg.JWTSecret, 24*time.Hour)`; claims carry
  `Subject` (user id), `Username`, `RoleName`.
- `idvalidator.IDValidator` (`internal/usecase/idvalidator/`): a shared service used by
  usecases to verify foreign keys (account group, currency, financial account) belong to
  `ServiceTypeX` constants; returns descriptive wrap errors.
- Policy domain: objects/actions/`Enforce(role, user, objType, objID, act)`; default policies
  and roles are seeded at startup (`SeedDefaults`).
- `cmd/utils/` (cobra): `add-admin`, `migrate`, `seed-*`, `replay-events`. Known limitation:
  utils bypass usecases, so seeding does not publish events — the GL startup reconciliation
  covers that (see Failover).

---

## 10. Logging (`pkg/logger`)

- slog wrapper. `logger.Init(level, format)` where format ∈ `text` (default) | `json` | `color`.
- `logger.FromContext(ctx)` returns the request logger, always attaching `request_id` when
  present. **Prefer `logger.FromContext(ctx)` over `logger.Info`/`logger.Error`** inside
  request-scoped code (handlers, usecases, consumers).
- Attrs are key/value pairs: `logger.FromContext(ctx).Error("msg", "error", err, "id", id)`.
- `logger.Fatal / FatalErr` for startup failures (config/db/planner), `logger.Warn` for
  fallbacks and config defaults.
- `logger.ContextWithRequestID` / `logger.RequestIDFromContext` are used to propagate the id
  onto event consumer contexts.

---

## 11. Testing

### Regenerating mocks

After adding/changing any interface with the mockery directive:
`go generate ./...` (which runs the `//go:generate mockery` comments).

### Unit tests (usecases) — `internal/usecase/<entity>/..._test.go`

Pattern:

```go
func newUsecase(repo *repoMocks.XRepository, broker broker.Broker) x.XUsecase {
	return x.NewXUsecase(repo, broker)
}

func TestCreate_Success(t *testing.T) {
	mockRepo := new(repoMocks.XRepository)
	uc := newUsecase(mockRepo, broker.NewGoChannelBroker())
	expected := domain.X{ID: 1, Name: "test"}
	mockRepo.On("Create", mock.Anything, mock.Anything).Return(expected, nil)
	result, err := uc.Create(context.Background(), domain.X{Name: "test"})
	require.NoError(t, err)
	assert.Equal(t, "test", result.Name)
	mockRepo.AssertExpectations(t)
}
```

Rules: external test package `X_test`; `mock.Anything` for ctx; `mock.MatchedBy` for complex
matchers; `require./assert.` from testify; verify expectations. Broker can be a real
`broker.NewGoChannelBroker()`.

### Repository integration — `internal/repository/<entity>/postgres_test.go`

White-box, testcontainers:
`db := testhelper.SetupDB(t)`, `db.AutoMigrate(&X{})`, `t.Cleanup(func() { testhelper.CleanTables(t, db, "xs") })`.
Docker required: tests panic if the container is unavailable.

### Full-stack integration — `internal/integration/`

`newTestEnv(t)` wires the entire real stack via `app.Build` (real Postgres + migrations +
router + event consumers). Exercise HTTP with the returned `*gin.Engine`, assert on the DB,
and `t.Cleanup(env.cleanup)` which cancels consumers + truncates tables in FK order.

### Commands

| Scope | Command |
|---|---|
| Unit only (no docker) | `go test ./internal/usecase/...` |
| One repo | `go test ./internal/repository/accountgroup/...` |
| All | `go test ./...` (Docker required) |
| Static | `go vet ./...` and `staticcheck ./...` |

CI order: `gofmt` → `go test ./...` → `go vet ./...` → `staticcheck ./...`.

---

## 12. Failover / resilience patterns

There is no generic retry/circuit-breaker layer — resilience is baked into the domain flow:

- **Startup reconciliation safety net**: GL usecase runs `ReconcileInitialEntries` before
  `ProcessEvents`; it scans existing account groups/financial accounts and creates missing
  initial GL entries (`date_timestamp=0`) — covers entities seeded by `cmd/utils` that never
  emitted events.
- **Compensating rollback**: document `Create` deletes the created document if journal-entry
  creation fails; `Revert` restores line type to `posted` and deletes the reversing entry if
  persistence fails. Failing side-effects are rolled back best-effort and reported with
  `errors.Join`. Always mirror this ordering when adding new mutations.
- **Async consumers are isolated**: `ProcessEvents` handlers log errors and **continue** the
  loop; a failed projection does not take down the process. Consumers stop only on `ctx.Done()`.
- **Optional-dependency fallback**: missing lookup data degrades gracefully with a `Warn`
  (e.g. FX gain/loss account absent → cost/interest lines skipped).
- **Config fallback chain**: env (`DATABASE_URL`, `JWT_SECRET`, `AI_API_KEY`, `LOG_LEVEL`,
  `LOG_FORMAT`) → `config.yaml` → hardcoded defaults. Required secrets (`JWT_SECRET`,
  `AI_API_KEY` when agentic mode on) panic at startup rather than run insecure.
- **Translation fallback**: unknown/absent message codes fall back to the English entry.
- **Locale fallback**: no header → `"en"`.
- **Event correlation**: request ids ride along on events and are re-attached to consumer
  contexts, so a failed async projection is traceable back to the originating request.
- **Periodic fetchers**: started only when config provides a sheet id; fetch errors are logged
  and the ticker continues.

---

## 13. Things I noticed while reading the code

- **Document ↔ JournalEntry shadowing**: documents are write-append-only; journal entries are
  the source of truth for balances. Never edit a posted journal entry in place; post a
  reversing entry (`JournalEntryLineTypeReverting`) and re-create. A "delete" is `Revert` +
  `Delete`, always returning the reversal entry.
- **Balanced-entry invariant** in `journalentry` usecase: ≥2 lines, non-negative debits/credits,
  `totalDebit == totalCredit`, non-zero totals, currency + financial account validated through
  `idValidator`.
- **Ledger entry uniqueness**: `general_ledger_entries` has a unique on
  `(account_id, type, date_timestamp)`; `id` doubles as the account id set in
  `CreateNewLedgerEntry`.
- **Entity naming/aliasing**: repositories/usecases are imported with aliases like
  `buysellcashrepo`, `buysellcashusecase`, `journalentrycrudusecase`. Follow this.
- **Package naming**: each entity owns `domain/<entity>.go`, `repository/<entity>/`,
  `usecase/<entity>/`, `api/http/<entity>/`, `events/<event>.go`. One entity per directory.
- **Feature gating**: agentic AI wiring (`internal/usecase/agentplanner|agentorchestrator|agentexecuter`)
  is constructed only when `cfg.AgenticModeEnabled`; planner chosen by `cfg.AIModel`
  (`deepseek`/`groq`/`avalai`, default OpenAI `gpt-4o-mini`); executor audits every action to
  `agent_audits`.
- **Config**: do not commit a real `AI_API_KEY`. Use env overrides.
- **`config.yaml`/model notes**: keep fmt with `gofmt`; no `init()` side effects for production
  dependencies (the logger global is the one exception); constructor injection only.

---

## 14. Adding a new entity — checklist

1. `internal/domain/<entity>.go` — pure struct + constants.
2. `internal/repository/<entity>/` — `interface.go` (with mockery directive), `entity.go`
   (GORM + `ToDomain`/`FromDomain...`), `dto.go` (filters), `postgres.go`, `postgres_test.go`.
3. `internal/usecase/<entity>/` — `crud.go` (interface + impl), `dto.go`, `error.go`
   (sentinel errors), `*_test.go`.
4. Publish events for mutations and attach an audit trail via `eventCRUDUsecase.RecordEvent`
   when the entity is business data.
5. `internal/events/<event>.go` if consumers need to react; wire into consumer `ProcessEvents`.
6. `internal/api/http/<entity>/` — `handler.go` (RegisterRoutes + authz + swagger), `dto.go`,
   `error.go` (codes + en/fa map + `writeError` + `translateErrorToCode`).
7. Wire in `internal/app/app.go`, launch goroutines for consumers/fetchers.
8. `go generate ./...` after every interface change; run `go test ./internal/usecase/...`,
   then `go vet ./...` and `staticcheck ./...`.
