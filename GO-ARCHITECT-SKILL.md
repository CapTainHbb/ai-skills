---
name: golang-clean-architecture
description: Generate and maintain Go services following Clean Architecture conventions.
compatibility: opencode
---

# Go Clean Architecture Skill

## Purpose

Automatically generate new features and enforce architectural consistency across Go backend services following a Clean/Onion Architecture with strict layering, interface-based dependency injection, and event-driven projection patterns.

---

## Architecture Principles

1. **Dependency Inversion** — High-level modules (handlers, usecases) depend on abstractions (interfaces), not concrete implementations.
2. **Strict Inward Dependency** — Dependencies flow: `handler → usecase → repository → domain`. Never outward.
3. **Domain Purity** — The domain layer imports **nothing** outside the Go standard library.
4. **Encapsulation** — Only interfaces are exported; implementations, constructors, and tests stay in the same package.
5. **Interface at Point of Use** — Each layer defines the interface it needs, not what the lower layer provides.
6. **Constructor Injection** — All dependencies are passed explicitly via constructor functions — no global state, no service locators.
7. **Eventual Consistency** — Side effects are published as events on an in-memory broker; projections subscribe and process asynchronously.
8. **Single Responsibility** — One domain entity → one usecase package → one repository package → one handler package.

---

## Layer Responsibilities

### Domain (`internal/domain/`)

**What lives here:** Pure business entities, value objects, constants, enums.

**Rules:**
- Structs with exported fields only. No methods beyond simple constructors.
- No JSON, YAML, GORM, or any serialization tags.
- No imports beyond `"time"`, `"math"`, stdlib.
- Constants for status values, types, and enum-like string literals.
- Config structs (loaded from YAML/env) are defined here.

**Template:**
```go
package domain

type Entity struct {
    ID   uint
    Name string
}
```

### Repository (`internal/repository/<entity>/`)

**What lives here:** Data access contracts (interfaces) and Postgres/GORM implementations.

**Files:**
- `interface.go` — Repository port with `//go:generate mockery` directive
- `entity.go` — GORM model struct + `ToDomain()` / `FromDomainToEntity()`
- `dto.go` — Filter structs for query methods
- `postgres.go` — GORM implementation (unexported struct)
- `postgres_test.go` — Integration tests (testcontainers)
- `mocks/` — Generated mockery mocks

**Rules:**
- Interface matches domain operations (`GetAll`, `GetByID`, `Create`, `Update`, `Delete`).
- Implementation is an unexported struct holding `*gorm.DB`.
- Constructor returns the interface: `func NewXPostgres(db *gorm.DB) XRepository`.
- `GetAll` returns `([]domain.X, int64, error)` — data + total count.
- Entity struct embeds `gorm.Model`.
- `ToDomain()` converts GORM entity → domain model.
- `FromDomainToEntity()` converts domain model → GORM entity.
- Filters are exported structs with slice/pointer fields for optional query params.

**Allowed imports:** `domain`, `context`, `gorm.io/gorm`, `gorm.io/driver/postgres`
**Forbidden imports:** `gin`, `broker`, `events`, other repositories, other usecases

### Use Case (`internal/usecase/<entity>/`)

**What lives here:** Business logic, validation, orchestration, event publishing.

**Files:**
- `crud.go` — Interface + implementation (can be named descriptively e.g. `billing.go`)
- `dto.go` — Usecase-specific filter/request types
- `*_test.go` — Unit tests with mocked repository
- `errors.go` — Sentinel errors (optional)
- `mocks/` — Generated mockery mocks

**Rules:**
- Interface methods mirror business capabilities, not CRUD necessarily.
- Implementation struct holds repository interface(s) + other usecase interfaces.
- Constructor takes dependencies, returns interface: `func NewX(repo XRepository, broker broker.Broker) XUsecase`.
- Business validation happens here before calling repository.
- Events are published after successful mutations via `eventBroker.Publish(...)`.
- Filter DTOs defined here, converted to repo-level filter DTOs inside the method.
- For event consumers: `ProcessEvents(ctx)` subscribes to broker and type-switches.

**Allowed imports:** `domain`, other `usecase/<entity>` (interfaces), `repository/<entity>` (interfaces), `events`, `broker`, `context`
**Forbidden imports:** `gin`, `gorm`, `repository/<entity>` (concrete postgres implementation)

### HTTP Handler (`internal/api/http/<entity>/`)

**What lives here:** HTTP transport — parse requests, call usecases, serialize responses.

**Files:**
- `handler.go` — Handler struct + `RegisterRoutes()` + HTTP method handlers
- `dto.go` — Request/Response structs with JSON tags + `FromDomainToResponse()`

**Rules:**
- Handler struct holds usecase interface(s).
- `NewHandler(usecase XUsecase) *Handler` constructor.
- `RegisterRoutes(router gin.IRouter)` registers a group with route methods.
- Swagger annotations on each handler method.
- Request validation via `c.ShouldBindJSON(&req)` with Gin binding tags.
- Pagination via `limit`/`offset` query parameters.
- Response structs have `json:"..."` tags.
- `ErrorResponse` struct with `Error string \`json:"error"\``.
- `FromDomainToResponse([]domain.X) []XResponse` helper function.
- ID parsing: `strconv.ParseUint(c.Param("id"), 10, 64)`.

**Allowed imports:** `domain`, `usecase/<entity>`, `gin`, `strconv`, `net/http`, `log`
**Forbidden imports:** `repository`, `gorm`, `broker`, `events`

### Events (`internal/events/`)

**What lives here:** Event type definitions for the async event bus.

**Conventions:**
- One file per event: `accountgroupcreated.go`, `journalentryposted.go`, etc.
- Each event struct carries the relevant domain entity.
- `New*()` constructor returns the `Event` interface.
- Event type constant: `EventX = "x_created"`.

**Template:**
```go
package events

import "github.com/<module>/internal/domain"

const EventEntityCreated = "entity_created"

type EntityCreated struct {
    Entity domain.Entity
}

func NewEntityCreated(entity domain.Entity) Event {
    return EntityCreated{Entity: entity}
}

func (e EntityCreated) EventType() string             { return EventEntityCreated }
func (e EntityCreated) GetEventData() interface{}     { return e.Entity }
```

### Event Broker (`pkg/broker/`)

**What lives here:** In-memory Go channel pub/sub broker.

**Interface:**
```go
type Broker interface {
    Subscribe() chan events.Event
    Unsubscribe(ch chan events.Event)
    Publish(msg events.Event)
}
```

---

## Directory Conventions

### Standard Project Layout

```
cmd/
    main/
        main.go              # Entrypoint, DI wiring, startup
    utils/
        main.go              # CLI tools (cobra-based migration, seeding)
internal/
    api/
        http/
            router.go        # NewRouter(cfg) returns *gin.Engine
            <entity>/
                handler.go   # RegisterRoutes, HTTP methods
                dto.go       # Request/Response structs
    config/
        loader.go            # Load(path) reads YAML + env overrides
    domain/
        entity.go            # Pure domain structs + constants
        config.go            # Config struct
    events/
        interface.go         # Event interface
        entitycreated.go     # One event type per file
    middleware/
        authrequired.go      # JWT validation middleware
        requirepermission.go # RBAC middleware
    repository/
        <entity>/
            interface.go     # Repository contract
            entity.go        # GORM model + mapping
            dto.go           # Filter structs
            postgres.go      # GORM implementation
            postgres_test.go # Integration tests
            mocks/           # Generated mocks
    usecase/
        <entity>/
            crud.go          # Interface + implementation
            dto.go           # Usecase filter structs
            errors.go        # Sentinel errors (optional)
            *_test.go        # Unit tests
            mocks/           # Generated mocks
    testhelper/
        testdb.go            # testcontainers singleton DB
pkg/
    broker/
        interface.go         # Broker contract
        gochannel.go         # In-memory implementation
migrations/
    <timestamp>_<name>.up.sql   # One pair per feature
    <timestamp>_<name>.down.sql
docs/
    swagger.json             # Generated swagger (checked in)
config.yaml                  # Default configuration
Makefile                     # Build/test/seed commands
Dockerfile                   # Container build
go.mod
```

### Package Naming

| Layer | Package | Example |
|---|---|---|
| Domain | `internal/domain` | `package domain` |
| Repo | `internal/repository/<entity>` | `package accountgroup` |
| Usecase | `internal/usecase/<entity>` | `package billing` |
| Handler | `internal/api/http/<entity>` | `package accountgroup` |
| Events | `internal/events` | `package events` |
| Middleware | `internal/middleware` | `package middleware` |
| Broker | `pkg/broker` | `package broker` |

### Import Aliasing

Always alias imports when the package name doesn't match the last path segment, or when naming conflicts arise:
```go
accountgrouprepo "github.com/module/internal/repository/accountgroup"
accountgroupusecase "github.com/module/internal/usecase/accountgroup"
```

---

## Feature Creation Workflow

When asked to add a new entity `X` to the system, follow this exact workflow:

### Step 1: Domain

Create `internal/domain/<entity>.go`:

```go
package domain

type X struct {
    ID   uint
    Name string
    // business fields only — no JSON/GORM tags
}
```

Add any constants the entity needs.

### Step 2: Repository Interface + Implementation

Create 4 files under `internal/repository/<entity>/`:

**interface.go:**
```go
package x

import (
    "context"
    "github.com/module/internal/domain"
)

//go:generate mockery --dir . --name=XRepository --output=mocks --outpkg=mocks
type XRepository interface {
    GetAll(ctx context.Context, limit, offset int, filters XFilters) ([]domain.X, int64, error)
    GetByID(ctx context.Context, id uint) (domain.X, error)
    Create(ctx context.Context, x domain.X) (domain.X, error)
    Update(ctx context.Context, x domain.X) (domain.X, error)
    Delete(ctx context.Context, id uint) error
}
```

**entity.go:**
```go
package x

import (
    "github.com/module/internal/domain"
    "gorm.io/gorm"
)

type X struct {
    gorm.Model
    Name string `gorm:"not null"`
}

func (e *X) ToDomain() domain.X {
    return domain.X{ID: e.ID, Name: e.Name}
}

func FromDomainToEntity(d *domain.X) X {
    return X{Model: gorm.Model{ID: d.ID}, Name: d.Name}
}
```

**dto.go:** Filter struct with optional fields as slices/pointers.

**postgres.go:** Unexported struct + constructor implementing interface.

### Step 3: Usecase

Create files under `internal/usecase/<entity>/`:

```go
package x

//go:generate mockery --name=XUsecase --output=mocks --outpkg=mocks
type XUsecase interface {
    GetAll(ctx context.Context, limit, offset int, filters GetXFilters) ([]domain.X, int64, error)
    GetByID(ctx context.Context, id uint) (domain.X, error)
    Create(ctx context.Context, x domain.X) (domain.X, error)
    Update(ctx context.Context, x domain.X) (domain.X, error)
    Delete(ctx context.Context, id uint) error
}

type xUsecase struct {
    repo        xRepo.XRepository
    eventBroker broker.Broker
}

func NewXUsecase(repo xRepo.XRepository, eventBroker broker.Broker) XUsecase {
    return &xUsecase{repo: repo, eventBroker: eventBroker}
}
```

If the mutation should trigger side effects, publish an event:
```go
func (u *xUsecase) Create(ctx context.Context, x domain.X) (domain.X, error) {
    created, err := u.repo.Create(ctx, x)
    if err != nil {
        return domain.X{}, err
    }
    u.eventBroker.Publish(events.NewXCreated(created))
    return created, nil
}
```

### Step 4: Migration

Create `migrations/<timestamp>_add_x.up.sql` and `<timestamp>_add_x.down.sql` with the new table, foreign keys, and indexes. Follow existing migration naming.

### Step 5: Event (if needed)

Create `internal/events/xcreated.go`.

### Step 6: HTTP Handler

Create files under `internal/api/http/<entity>/`:

```go
package x

type Handler struct {
    usecase xUsecase.XUsecase
}

func NewHandler(usecase xUsecase.XUsecase) *Handler {
    return &Handler{usecase: usecase}
}

func (h *Handler) RegisterRoutes(router gin.IRouter) {
    g := router.Group("/xs")
    g.GET("/", h.GetAll)
    g.GET("/:id", h.GetByID)
    g.POST("/", h.Create)
    g.PUT("/:id", h.Update)
    g.DELETE("/:id", h.Delete)
}
```

Include swagger annotations (`@Summary`, `@Param`, `@Success`, `@Router`) on each method.

### Step 7: Wire in `cmd/main/main.go`

```go
xRepo := xRepo.NewXPostgres(db)
xUsecase := xUsecase.NewXUsecase(xRepo, eventBroker)
xHandler := x.NewHandler(xUsecase)
xHandler.RegisterRoutes(router)
```

### Step 8: Tests

- `internal/usecase/<entity>/crud_test.go` — Unit tests with mockery mocks
- `internal/repository/<entity>/postgres_test.go` — Integration tests with testcontainers

---

## Rules for Adding Packages

### Adding a new entity (e.g., "Invoice")

1. Create exactly: `domain/invoice.go`, `repository/invoice/{interface,entity,dto,postgres}.go`, `usecase/invoice/crud.go`, `api/http/invoice/{handler,dto}.go`
2. Create migration files `migrations/<timestamp>_add_invoice.{up,down}.sql`
3. Wire in `main.go` following existing patterns
4. Each new package must be in its own directory (no putting multiple entities in one package)

### Adding a new layer to an existing entity

- Never add a file that breaks the dependency direction. E.g., don't import `gin` in a usecase.
- If you need to expose a new capability, add a method to the relevant interface.

### Adding a cross-cutting concern

- **Middleware**: `internal/middleware/`. Accept usecase interfaces via closure.
- **Config fields**: Add to `domain.Config` struct + `config.yaml` + `config.Load()`
- **Event type**: Create `internal/events/<name>.go` + wire event constant into consumer `ProcessEvents()`
- **Periodic fetcher**: Create `internal/usecase/<entity>/sheetfetcher.go` with `StartPeriodicFetch(ctx)` method, launch in `main.go`

---

## Rules for Dependency Injection

1. **Manual wiring in main.go** — No DI frameworks (wire, dig, etc.) unless the project explicitly adds one.
2. **Constructor injection only** — All dependencies are constructor parameters. Never use `init()`, global variables, or `sync.Once` for production dependencies.
3. **Interface parameters** — Constructors accept interface types, not concrete implementations.
4. **One instance per dependency** — Share instances by passing them to multiple constructors (e.g., same `eventBroker` to all usecases).
5. **Order matters** — Wire repos first, then usecases, then handlers, then route registration.
6. **Feature gating** — Conditionally wire components based on config flags (e.g., `cfg.AgenticModeEnabled`).

```go
// Correct wiring pattern
userRepo := userrepo.NewUserPostgres(db)
userUsecase := userusecase.NewCRUDUsecase(userRepo)
userHandler := userhandler.NewHandler(userUsecase)
userHandler.RegisterRoutes(router)
```

---

## Testing Conventions

### Unit Tests (usecases)

```go
package x_test // external test package

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

**Patterns:**
- `mock.Anything` for context
- `mock.MatchedBy(func(T) bool)` for complex matchers
- `require.NoError` / `require.Error` for error checking
- `assert.Equal` / `assert.Len` for value assertions
- `mock.AssertExpectations(t)` to verify all expected calls happened
- Use `newUsecase()` helper to reduce boilerplate

### Integration Tests (repositories)

```go
package x // same package as implementation (white-box)

func setupRepo(t *testing.T) XRepository {
    t.Helper()
    db := testhelper.SetupDB(t)
    require.NoError(t, db.AutoMigrate(&X{}))
    t.Cleanup(func() { testhelper.CleanTables(t, db, "xs") })
    return NewXPostgres(db)
}

func TestCreateAndGetByID(t *testing.T) {
    repo := setupRepo(t)
    created, err := repo.Create(context.Background(), domain.X{Name: "test"})
    require.NoError(t, err)
    require.NotZero(t, created.ID)

    got, err := repo.GetByID(context.Background(), created.ID)
    require.NoError(t, err)
    require.Equal(t, created.ID, got.ID)
}
```

### Test Infrastructure

- `testhelper.SetupDB(t)` returns a shared Postgres container via `sync.Once`
- `testhelper.CleanTables(t, db, "table1", "table2")` truncates tables in order
- Use `go test ./...` for all tests, `go test ./internal/usecase/...` for unit only

---

## Refactoring Rules

### When to extract a usecase

If a `handler.go`, `crud.go`, or `postgres.go` exceeds ~300 lines, or handles more than one clear responsibility, extract.

**Signs of needed extraction:**
- A usecase imports more than 5 different repository/usecase interfaces
- A repository has >10 methods
- A handler has >6 route handlers
- Comment blocks like `// -- Billing --` sectioning within a single file

### How to rename an entity

1. Rename domain struct + file
2. Rename repository directory + all files within (interface, entity, dto, postgres)
3. Rename usecase directory + all files within
4. Rename handler directory + all files within
5. Update all imports across the codebase
6. Regenerate mocks: `go generate ./...`
7. Update tests
8. Update wire-up in `cmd/main/main.go`

### When to add an event

Add a new event type when:
- A usecase mutation needs to trigger updates in 2+ other usecases
- A new projection/view needs to react to existing mutations

---

## Validation Checklist

Use this checklist when reviewing any new code generated within this architecture.

### All Layers
- [ ] No circular imports
- [ ] No unused imports
- [ ] `go vet ./...` passes
- [ ] `staticcheck ./...` passes
- [ ] Mockery directives present on all interfaces
- [ ] Tests compile and pass

### Domain
- [ ] No imports outside stdlib
- [ ] No JSON/GORM tags
- [ ] No logic methods (pure data structs)
- [ ] All fields exported

### Repository
- [ ] Interface defined with `//go:generate mockery`
- [ ] Implementation struct unexported
- [ ] Constructor returns interface
- [ ] `ToDomain()` / `FromDomainToEntity()` methods exist
- [ ] `GetAll` returns data + total count
- [ ] Integration test covers CRUD operations
- [ ] No `events` or `broker` imports

### Usecase
- [ ] Interface defined with `//go:generate mockery`
- [ ] Implementation struct unexported
- [ ] Dependencies received via constructor
- [ ] Business validation before persistence
- [ ] Events published after successful mutations
- [ ] Filter DTOs converted to repo filter DTOs internally
- [ ] Unit tests cover success + error cases
- [ ] No `gin` or HTTP-related imports
- [ ] No direct GORM usage

### Handler
- [ ] Swagger annotations present on all endpoints
- [ ] `RegisterRoutes` method exists
- [ ] Request validation via `ShouldBindJSON` or query parsing
- [ ] Pagination: `limit`/`offset` with defaults
- [ ] Standard error response format
- [ ] `FromDomainToResponse` helper for slice conversions
- [ ] No repository or broker imports

### Migrations
- [ ] Up migration creates table with all columns, foreign keys, and indexes
- [ ] Down migration drops table (safe reverse order if dependencies exist)
- [ ] Migration timestamp matches feature order (newer > older)

### Main Wiring
- [ ] All dependencies explicitly constructed in order
- [ ] Event consumers launched as goroutines: `go usecase.ProcessEvents(ctx)`
- [ ] Periodic fetchers launched as goroutines: `go usecase.StartPeriodicFetch(ctx)`
- [ ] Initial reconciliation called before starting server (if applicable)
- [ ] Fetchers gated behind config checks

---

## Anti-Pattern Detection

| Anti-Pattern | Detection | Fix |
|---|---|---|
| Handler imports repository | `grep "repository/" internal/api/http/` | Move logic to usecase |
| Usecase imports `gin` | `grep "gin" internal/usecase/` | Remove, handlers handle HTTP concerns |
| Domain has GORM tags | `grep "gorm:" internal/domain/` | Move GORM model to repository/entity.go |
| Circular imports | `go test ./...` fails with import cycle | Extract shared interface to usecase |
| `context.Background()` in usecase/repo | `grep "Background()" internal/` | Accept context from caller |
| Service locator / global registry | `grep "sync.Once" internal/` (non-test) | Use constructor injection |
| Repository calling another repository | `grep "repository/" internal/repository/` | Compose at usecase level |
| Usecase calls `db *gorm.DB` directly | `grep "gorm" internal/usecase/` | Delegate to repository interface |
| No mockery directive | `grep -L "go:generate mockery" internal/*/interface.go` | Add `//go:generate mockery` line |
| `init()` functions | `grep "func init()" internal/` | Move to explicit wiring in main.go |
| DTO duplicated across layers | Compare `dto.go` files across repo vs usecase vs handler | Each layer owns its DTOs |
