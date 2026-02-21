---
name: go-pro
description: Go development principles and decision-making. Idiomatic patterns, concurrency, clean architecture, project structure, testing, and performance. Teaches thinking, not copying.
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Go Pro

> Go development principles and decision-making for 2025.
> **Learn to THINK like a Go developer, not memorize syntax.**

---

## ⚠️ How to Use This Skill

This skill teaches **decision-making principles**, not fixed code to copy.

- ASK user for framework/library preference when unclear
- Choose pattern based on PROJECT CONTEXT (scale, team, lifetime)
- Standard Library first—add dependencies only when justified

---

## 1. Go Philosophy (Non-Negotiable)

### Core Tenets

```
"Simplicity is complicated." — Rob Pike

├── Clear is better than clever
├── A little copying is better than a little dependency
├── Don't panic (literally—avoid panic in library code)
├── Accept interfaces, return structs
├── Make the zero value useful
└── Errors are values, not exceptions
```

### Standard Library First

```
Before adding a dependency, ask:
├── Can `net/http` handle this? (Go 1.22+ has pattern routing)
├── Can `encoding/json` handle this?
├── Can `database/sql` handle this?
├── Can `log/slog` handle this? (Go 1.21+ structured logging)
└── Will this dependency survive 5 years?

Add dependency ONLY when:
├── Significant complexity reduction (e.g., sqlc, pgx)
├── Non-trivial to implement (e.g., JWT validation)
└── Well-maintained with clear ownership
```

---

## 2. Project Structure

### Decision Tree

```
How big is the project?
│
├── Single binary / Script / CLI tool
│   └── Flat layout
│       ├── main.go
│       ├── handler.go
│       ├── store.go
│       └── go.mod
│
├── Medium service (1 team, 1 domain)
│   └── Standard layout
│       ├── cmd/server/main.go
│       ├── internal/
│       │   ├── handler/
│       │   ├── service/
│       │   ├── repository/
│       │   └── model/
│       ├── pkg/            # Only if reuse is real
│       └── go.mod
│
└── Large system (multi-domain, multi-team)
    └── Domain-driven layout
        ├── cmd/
        │   ├── api/main.go
        │   └── worker/main.go
        ├── internal/
        │   ├── user/        # Domain package
        │   │   ├── handler.go
        │   │   ├── service.go
        │   │   ├── repository.go
        │   │   └── model.go
        │   ├── order/       # Domain package
        │   └── platform/    # Shared infra
        │       ├── database/
        │       ├── logger/
        │       └── middleware/
        └── go.mod
```

### Key Rules

```
├── cmd/        → Entry points ONLY (wire up, start server)
├── internal/   → Private application code (compiler-enforced)
├── pkg/        → Truly reusable libraries (use sparingly!)
│
├── NEVER put business logic in cmd/
├── NEVER put HTTP concerns in service layer
├── NEVER import internal/ from another module
└── Keep main.go thin (< 50 lines ideally)
```

---

## 3. Concurrency Patterns

### When to Use What

| Pattern | Use When |
|---------|----------|
| `goroutine + channel` | Fan-out/fan-in, pipelines |
| `sync.WaitGroup` | Wait for N goroutines to complete |
| `errgroup.Group` | Wait + collect first error + cancel others |
| `sync.Mutex/RWMutex` | Protect shared state |
| `sync.Once` | One-time initialization |
| `context.Context` | Cancellation, timeout, request-scoped values |

### The Golden Rules

```
Concurrency rules:
├── Always pass context.Context as first parameter
├── Never start a goroutine without knowing how it will stop
├── Use errgroup for concurrent operations that can fail
├── Prefer channels for communication, mutexes for state
├── Don't communicate by sharing memory—share memory by communicating
├── Always handle context cancellation
└── Use sync.Pool for frequently allocated objects (measure first!)

Context propagation:
├── Accept ctx in every function that does I/O
├── Pass ctx to downstream calls
├── Check ctx.Err() in long loops
├── Use context.WithTimeout for external calls
└── NEVER store context in a struct
```

### Goroutine Lifecycle

```
ALWAYS ensure goroutine cleanup:
├── Use context for cancellation
├── Use done channels for signaling
├── Defer cleanup in goroutines
├── Prevent goroutine leaks
└── Log goroutine lifecycle in debug mode
```

---

## 4. Error Handling

### Idiomatic Error Handling

```
Core patterns:
├── if err != nil { return fmt.Errorf("doing X: %w", err) }
├── Wrap errors with context using %w verb
├── Use errors.Is() for sentinel error comparison
├── Use errors.As() for type assertion on errors
├── Return errors, don't panic (except truly unrecoverable)
└── Handle errors at the right level (don't swallow!)

Error wrapping chain:
├── Repository:  fmt.Errorf("query user %d: %w", id, err)
├── Service:     fmt.Errorf("get user profile: %w", err)
├── Handler:     Log full error, return sanitized HTTP error
└── Client sees: {"error": "user not found"} (no internals!)
```

### Custom Error Types

```
When to create custom errors:
├── Need to carry structured data (error code, field name)
├── Need to distinguish error categories (NotFound, Conflict)
├── Need to map to HTTP status codes
└── Shared across multiple packages

When sentinel errors are enough:
├── Simple "not found" / "already exists" cases
├── Single package usage
└── No extra data needed
```

### Error Design Principles

```
├── Errors should be opaque to callers (use Is/As, not string matching)
├── Package-level sentinel: var ErrNotFound = errors.New("not found")
├── Don't log AND return the same error (pick one per layer)
├── Handler layer: log + return HTTP response
├── Service layer: wrap + return
└── Repository layer: wrap + return
```

---

## 5. Interface Design

### Core Principles

```
"The bigger the interface, the weaker the abstraction." — Rob Pike

├── Accept interfaces, return concrete types
├── Define interfaces where they are USED, not implemented
├── Keep interfaces small (1-3 methods ideal)
├── Use implicit satisfaction (no "implements" keyword needed)
└── io.Reader / io.Writer are the gold standard

Interface location:
├── Consumer package defines the interface
├── Producer package returns concrete struct
├── This allows testing without mocks of the whole world
└── Example: service/ defines Repository interface,
    repository/ returns *PostgresRepo struct
```

### Common Patterns

```
Repository interface (defined in service package):
├── type UserRepository interface {
│       GetByID(ctx context.Context, id int64) (*User, error)
│       Create(ctx context.Context, user *User) error
│   }
│
Service depends on interface:
├── type UserService struct {
│       repo UserRepository  // interface, not concrete
│   }
│
Testing becomes trivial:
└── Pass a mock/stub that satisfies the interface
```

---

## 6. Framework Selection (2025)

### Decision Tree

```
What are you building?
│
├── API with simple routing (Go 1.22+)
│   └── net/http (ServeMux now supports patterns!)
│
├── REST API with middleware needs
│   └── Chi (lightweight, net/http compatible)
│
├── High-performance API / Microservice
│   └── Gin or Echo (battle-tested, fast)
│
├── Maximum performance (benchmarks matter)
│   └── Fiber (fasthttp-based, Express-like API)
│
├── gRPC service
│   └── google.golang.org/grpc + protobuf
│
└── CLI tool
    └── cobra + viper
```

### Comparison Principles

| Factor | net/http | Chi | Gin | Fiber |
|--------|----------|-----|-----|-------|
| **Best for** | Simple APIs, Go 1.22+ | Composable middleware | Production REST APIs | Max throughput |
| **Dependencies** | Zero | Minimal | Moderate | fasthttp (non-std) |
| **net/http compatible** | ✅ | ✅ | ❌ (own context) | ❌ (fasthttp) |
| **Middleware ecosystem** | Manual | Rich, composable | Rich, built-in | Growing |
| **Learning curve** | Low | Low | Low | Low |

### Selection Questions to Ask:
1. Does the project need to stay net/http compatible?
2. Is raw throughput critical (>100K req/s)?
3. Does the team already know a specific framework?
4. How important is the middleware ecosystem?

> **Default recommendation for new projects**: `net/http` (Go 1.22+) or `Chi` for composability. Only reach for Gin/Fiber when justified.

---

## 7. Database Patterns

### ORM/Driver Selection

| Tool | Best For |
|------|----------|
| `database/sql` + raw SQL | Full control, simple queries |
| `sqlc` | Type-safe SQL → Go code generation |
| `pgx` | PostgreSQL-specific features, performance |
| `GORM` | Rapid prototyping, complex relations |
| `Bun` | Modern ORM, good performance |
| `ent` | Graph-based schema, code generation |

### Database Principles

```
├── Use connection pooling (sql.DB handles this)
├── Always use context-aware queries (QueryContext, ExecContext)
├── Use prepared statements for repeated queries
├── Close rows immediately: defer rows.Close()
├── Scan into structs, not individual variables
├── Use transactions for multi-step operations
├── Set reasonable timeouts via context
└── Monitor connection pool metrics

sqlc recommendation (for most projects):
├── Write SQL → Generate type-safe Go code
├── No runtime reflection
├── Catches SQL errors at compile time
├── Works with PostgreSQL, MySQL, SQLite
└── Pairs perfectly with pgx driver
```

### Migration Strategy

```
├── golang-migrate/migrate  → SQL-based, simple
├── goose                   → SQL or Go-based
├── atlas                   → Declarative, modern
│
├── ALWAYS use versioned migrations
├── NEVER modify existing migration files
├── Test migrations up AND down
└── Run migrations separately from app startup
```

---

## 8. Testing

### Table-Driven Tests (Idiomatic Go)

```
Pattern:
├── Define test cases as slice of structs
├── Loop through cases with t.Run(name, func)
├── Each case: input → expected output
├── Name each case descriptively
├── Add edge cases and error cases
└── Use t.Parallel() when tests are independent

Benefits:
├── Easy to add new cases
├── Clear what's being tested
├── Consistent structure across codebase
└── Great for debugging (run single case)
```

### Testing Strategy

| Type | Purpose | Tools |
|------|---------|-------|
| **Unit** | Business logic, pure functions | `testing` (stdlib) |
| **Integration** | Database, external services | `testcontainers-go` |
| **Benchmark** | Performance measurement | `testing.B` |
| **Fuzz** | Input discovery (Go 1.18+) | `testing.F` |
| **E2E** | Full API workflows | `net/http/httptest` |

### Testing Principles

```
├── Use stdlib testing package first
├── testify is OK for assertions (require/assert)
├── Use httptest.NewServer for HTTP tests
├── Use t.TempDir() for file-based tests
├── Use testcontainers for real database tests
├── Mock at interface boundaries only
├── Benchmark before optimizing: go test -bench=.
├── Fuzz test parsers and validators: go test -fuzz=.
└── Use t.Helper() in test helper functions
```

---

## 9. Performance

### Profiling First (Measure, Don't Guess)

```
Tools:
├── go tool pprof      → CPU, memory, goroutine profiling
├── go test -bench     → Micro-benchmarks
├── go test -benchmem  → Memory allocation counts
├── runtime/trace      → Execution tracer
├── expvar / metrics   → Runtime metrics endpoint
└── go build -gcflags="-m" → Escape analysis

Common optimizations (ONLY after profiling):
├── Reduce allocations (sync.Pool, pre-allocate slices)
├── Use strings.Builder for string concatenation
├── Avoid interface{}/any in hot paths
├── Use struct embedding to reduce pointer chasing
├── Pre-size maps and slices: make([]T, 0, expectedCap)
├── Use io.Reader/Writer for streaming (avoid loading all into memory)
└── Consider msgpack/protobuf over JSON for internal services
```

### Memory Layout

```
Struct field ordering matters:
├── Group same-size fields together
├── Larger fields first, smaller fields last
├── Reduces padding, saves memory
├── Use fieldalignment linter to detect issues
└── ONLY matters at scale (millions of structs)
```

---

## 10. Clean Architecture in Go

### Layer Separation

```
Request Flow:
│
├── Handler (Transport Layer)
│   ├── HTTP/gRPC specifics
│   ├── Parse request, validate input
│   ├── Call service method
│   └── Format response
│
├── Service (Business Logic Layer)
│   ├── Pure business rules
│   ├── Depends on repository INTERFACES
│   ├── No HTTP, no SQL
│   └── Testable in isolation
│
├── Repository (Data Access Layer)
│   ├── Database queries
│   ├── Implements repository interface
│   ├── No business logic
│   └── Returns domain models
│
└── Model (Domain Layer)
    ├── Structs representing business entities
    ├── No dependencies on other layers
    └── Validation methods optional
```

### Dependency Injection

```
DI in Go (no magic, no framework needed):
├── Constructor injection via New* functions
│   func NewUserService(repo UserRepository) *UserService
│
├── Wire up in cmd/main.go:
│   db := database.Connect(cfg)
│   userRepo := postgres.NewUserRepository(db)
│   userService := service.NewUserService(userRepo)
│   userHandler := handler.NewUserHandler(userService)
│
├── For large projects, consider google/wire
│   (compile-time DI code generation)
│
└── AVOID runtime DI containers (not idiomatic Go)
```

### When Clean Architecture is Overkill

```
Skip full layering when:
├── Simple CRUD with < 5 entities
├── CLI tools or scripts
├── Prototypes / POCs
└── Single-person, short-lived projects

Use full layering when:
├── Multiple developers
├── Complex business rules
├── Long-lived project (> 1 year)
├── Need to swap infrastructure
└── Extensive testing required
```

---

## 11. API Design Principles

### HTTP Handler Patterns

```
Go 1.22+ ServeMux patterns:
├── mux.HandleFunc("GET /users/{id}", handler.GetUser)
├── mux.HandleFunc("POST /users", handler.CreateUser)
├── Method + path in one string
└── Path parameters via r.PathValue("id")

Middleware chain:
├── func Middleware(next http.Handler) http.Handler
├── Compose: Logger(Auth(RateLimit(handler)))
├── Use for: logging, auth, CORS, recovery, metrics
└── Keep middleware focused (single responsibility)
```

### JSON Handling

```
Encoding/Decoding:
├── Use json.NewDecoder(r.Body) for requests (streaming)
├── Use json.NewEncoder(w) for responses
├── Define struct tags: `json:"field_name,omitempty"`
├── Use json.RawMessage for deferred parsing
├── Limit request body size: http.MaxBytesReader
└── Set Content-Type header: "application/json"

Validation:
├── Validate after decoding, not during
├── Use struct tags + validator package for complex rules
├── Return 422 for validation errors with field-level details
└── Sanitize output (never expose internal struct fields)
```

---

## 12. Logging & Observability

### Structured Logging (Go 1.21+)

```
Use log/slog (stdlib):
├── slog.Info("user created", "user_id", id, "email", email)
├── Structured key-value pairs
├── JSON output for production
├── Text output for development
├── Create child loggers with With()
└── No need for zerolog/zap unless specific features needed

Logging levels:
├── Debug → Development diagnostics
├── Info  → Normal operations
├── Warn  → Recoverable issues
├── Error → Failures requiring attention
└── NEVER log sensitive data (passwords, tokens, PII)
```

---

## 13. Go Modules & Dependencies

### Module Management

```
├── go mod init → Start new module
├── go mod tidy → Clean up go.mod/go.sum
├── go mod vendor → Vendor dependencies (optional)
├── go mod verify → Verify integrity
│
├── Pin major versions in go.mod
├── Review go.sum changes in code review
├── Use govulncheck for vulnerability scanning
├── Prefer fewer, well-maintained dependencies
└── Check license compatibility
```

### Build & Release

```
├── go build -ldflags "-s -w" → Strip debug for smaller binary
├── CGO_ENABLED=0 → Static binary (no C dependencies)
├── Use multi-stage Docker builds
├── Cross-compile: GOOS=linux GOARCH=amd64 go build
└── Use goreleaser for automated releases
```

---

## 14. Anti-Patterns to Avoid

### ❌ DON'T:
- Use `init()` for complex initialization (hard to test, surprising)
- Use `panic` in library code (return errors instead)
- Ignore errors with `_ = someFunc()` without explicit comment
- Use global mutable state (use dependency injection)
- Use `interface{}` / `any` when generics (Go 1.18+) work better
- Import from another project's `internal/` package
- Use Gin/Fiber for simple APIs where `net/http` suffices
- Store `context.Context` in struct fields
- Start goroutines without lifecycle management
- Use `sync.Mutex` when a channel is more appropriate

### ✅ DO:
- Choose framework based on actual project needs
- Write table-driven tests for all business logic
- Use `context.Context` for cancellation and timeouts
- Profile before optimizing
- Keep interfaces small (1-3 methods)
- Use `errgroup` for concurrent operations
- Run `go vet`, `staticcheck`, `golangci-lint` in CI
- Use `sqlc` or `pgx` for database interactions
- Document exported functions and types
- Use `go generate` for code generation workflows

---

## 15. Decision Checklist

Before implementing:

- [ ] **Asked user about framework preference?**
- [ ] **Chosen net/http vs framework for THIS context?**
- [ ] **Decided project structure scale?** (flat / standard / domain-driven)
- [ ] **Planned error handling strategy?** (sentinel / custom types / wrapping)
- [ ] **Identified concurrency needs?** (goroutines, channels, errgroup)
- [ ] **Chosen database approach?** (raw SQL, sqlc, GORM)
- [ ] **Planned testing strategy?** (table-driven, integration, benchmarks)
- [ ] **Considered clean architecture necessity?** (skip if simple CRUD)
- [ ] **Set up linting?** (golangci-lint with project config)

---

> **Remember**: Go's power is in its simplicity. Don't fight the language—embrace `if err != nil`, small interfaces, explicit code, and the standard library. The best Go code looks boring—and that's the point.
