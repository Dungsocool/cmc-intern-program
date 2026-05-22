# 🏗️ PROJECT STRUCTURE & CLEAN ARCHITECTURE

## Purpose

This document explains the Mini ASM project structure and the applied Clean Architecture principles.

---

## 1. PROJECT STRUCTURE

```
mini-asm/
├── cmd/                     # Application entry points
│   └── server/
│       └── main.go          # Main entry point - initialize and run the server
│
├── internal/                # Private application code
│   ├── handler/             # HTTP handlers (Presentation Layer)
│   │   ├── asset_handler.go
│   │   ├── health_handler.go
│   │   └── handler.go       # Handler registry
│   │
│   ├── service/             # Business logic (Use Case Layer)
│   │   ├── asset_service.go
│   │   └── service.go       # Service interface
│   │
│   ├── storage/             # Data access (Infrastructure Layer)
│   │   ├── storage.go       # Storage interface
│   │   ├── memory/          # In-memory implementation
│   │   │   └── memory.go
│   │   └── postgres/        # Database implementation
│   │       └── postgres.go
│   │
│   └── model/               # Domain models (Entity Layer)
│       ├── asset.go
│       └── errors.go
│
├── test/                    # Integration tests
│   ├── handler_test.go
│   └── integration_test.go
│
├── web/                     # Frontend static files
│   ├── index.html
│   └── app.js
│
├── migrations/              # Database migrations
│   ├── 001_create_assets.up.sql
│   └── 001_create_assets.down.sql
│
├── .env.example             # Environment variables template
├── .gitignore
├── go.mod
├── go.sum
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## 2. CLEAN ARCHITECTURE LAYERS

Clean Architecture divides the system into independent layers, with the dependency flow from the outside in.

### 2.1 Dependency Rule

**The most important principle:**

> Code in the inner layers MUST NOT know anything about the outer layers.

```
┌─────────────────────────────────────┐
│   Presentation Layer (handler/)     │  ← HTTP, JSON, UI concerns
├─────────────────────────────────────┤
│   Use Case Layer (service/)         │  ← Business logic
├─────────────────────────────────────┤
│   Infrastructure Layer (storage/)   │  ← Database, external services
└─────────────────────────────────────┘
           ↓ depends on
┌─────────────────────────────────────┐
│   Entity Layer (model/)              │  ← Core domain models
└─────────────────────────────────────┘
```

### 2.2 Layer Details

#### **Entity Layer (model/)**

Core domain models - do not depend on any other layers.

```go
// model/asset.go
package model

import "time"

type Asset struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Type      string    `json:"type"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// Asset types
const (
    TypeDomain  = "domain"
    TypeIP      = "ip"
    TypeService = "service"
)

// Asset status
const (
    StatusActive   = "active"
    StatusInactive = "inactive"
)
```

#### **Infrastructure Layer (storage/)**

Handles data persistence - implements storage interface.

```go
// storage/storage.go
package storage

import "mini-asm/internal/model"

// Storage defines the interface for data access
type Storage interface {
    Create(asset *model.Asset) error
    GetAll() ([]*model.Asset, error)
    GetByID(id string) (*model.Asset, error)
    Update(id string, asset *model.Asset) error
    Delete(id string) error
    Filter(assetType, status string) ([]*model.Asset, error)
    Search(query string) ([]*model.Asset, error)
}
```

**Advantages:**

- Easily swap between in-memory and database
- Testable without a real database
- Isolate business logic from implementation details

#### **Use Case Layer (service/)**

Business logic - orchestrate operations, validations, and business rules.

```go
// service/asset_service.go
package service

import (
    "errors"
    "mini-asm/internal/model"
    "mini-asm/internal/storage"
    "time"

    "github.com/google/uuid"
)

type AssetService struct {
    storage storage.Storage
}

func NewAssetService(storage storage.Storage) *AssetService {
    return &AssetService{storage: storage}
}

func (s *AssetService) CreateAsset(name, assetType string) (*model.Asset, error) {
    // Validation
    if name == "" {
        return nil, errors.New("name is required")
    }

    if !isValidType(assetType) {
        return nil, errors.New("invalid asset type")
    }

    // Business logic
    asset := &model.Asset{
        ID:        uuid.New().String(),
        Name:      name,
        Type:      assetType,
        Status:    model.StatusActive,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    // Delegate to storage
    if err := s.storage.Create(asset); err != nil {
        return nil, err
    }

    return asset, nil
}

func isValidType(t string) bool {
    return t == model.TypeDomain || t == model.TypeIP || t == model.TypeService
}
```

**Responsibilities:**

- Input validation
- Business rules enforcement
- Orchestrate operations
- Error handling

#### **Presentation Layer (handler/)**

HTTP handlers - handle HTTP requests/responses and JSON marshalling.

```go
// handler/asset_handler.go
package handler

import (
    "encoding/json"
    "net/http"

    "mini-asm/internal/service"
)

type AssetHandler struct {
    service *service.AssetService
}

func NewAssetHandler(service *service.AssetService) *AssetHandler {
    return &AssetHandler{service: service}
}

func (h *AssetHandler) CreateAsset(w http.ResponseWriter, r *http.Request) {
    // Parse request
    var req struct {
        Name string `json:"name"`
        Type string `json:"type"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        RespondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    // Call service
    asset, err := h.service.CreateAsset(req.Name, req.Type)
    if err != nil {
        RespondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Return response
    RespondJSON(w, http.StatusCreated, asset)
}

// Helper functions
func RespondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func RespondError(w http.ResponseWriter, status int, message string) {
    RespondJSON(w, status, map[string]string{"error": message})
}
```

**Responsibilities:**

- Parse HTTP requests
- Validate HTTP-specific concerns
- Call the appropriate service method
- Format the HTTP response
- Set appropriate status codes

---

## 3. WIRE UP - Main Entry Point

The `cmd/server/main.go` file is where all dependencies are wired up.

```go
// cmd/server/main.go
package main

import (
    "log"
    "net/http"

    "mini-asm/internal/handler"
    "mini-asm/internal/service"
    "mini-asm/internal/storage/memory"
)

func main() {
    // Initialize storage (Session 2: memory, Session 3: postgres)
    store := memory.NewMemoryStorage()

    // Initialize service
    assetService := service.NewAssetService(store)

    // Initialize handlers
    assetHandler := handler.NewAssetHandler(assetService)
    healthHandler := handler.NewHealthHandler()

    // Setup routes
    mux := http.NewServeMux()

    mux.HandleFunc("GET /health", healthHandler.Check)
    mux.HandleFunc("POST /assets", assetHandler.CreateAsset)
    mux.HandleFunc("GET /assets", assetHandler.ListAssets)
    mux.HandleFunc("GET /assets/{id}", assetHandler.GetAsset)
    mux.HandleFunc("PUT /assets/{id}", assetHandler.UpdateAsset)
    mux.HandleFunc("DELETE /assets/{id}", assetHandler.DeleteAsset)

    // Start server
    log.Println("Server starting on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err)
    }
}
```

---

## 4. BENEFITS OF CLEAN ARCHITECTURE

### 4.1 Testability

Each layer can be tested independently:

```go
// Test service without real storage
func TestAssetService_CreateAsset(t *testing.T) {
    mockStorage := &MockStorage{}
    service := service.NewAssetService(mockStorage)

    asset, err := service.CreateAsset("example.com", "domain")

    assert.NoError(t, err)
    assert.NotEmpty(t, asset.ID)
}
```

### 4.2 Flexibility

Easily change the implementation:

```go
// Session 2: In-memory
store := memory.NewMemoryStorage()

// Session 3: Switch to database - other code remains unchanged
store := postgres.NewPostgresStorage(db)
```

### 4.3 Maintainability

- Clearly structured code
- Easy to debug (know exactly which layer has the issue)
- Easy to onboard new developers

### 4.4 Separation of Concerns

Each layer has its own responsibility:

- Handler: HTTP concerns
- Service: Business logic
- Storage: Data access
- Model: Domain entities

---

## 5. COMPARING CLEAN ARCHITECTURE WITH MVC

### 5.1 MVC Architecture Overview

MVC (Model-View-Controller) is a popular pattern for web applications:

```
┌──────────┐      ┌────────────┐      ┌───────┐
│   View   │ ←──→ │ Controller │ ←──→ │ Model │
└──────────┘      └────────────┘      └───────┘
```

**MVC in Go can look like this:**

```
project/
├── main.go
├── controllers/
│   └── asset_controller.go
├── models/
│   └── asset.go
└── views/
    └── templates/
```

### 5.2 Detailed Comparison

| Aspect | MVC | Clean Architecture |
| ------------------------ | ----------------------------------------- | ---------------------------------------------------------- |
| **Number of layers** | 3 layers (Model, View, Controller) | 4+ layers (Entity, Use Case, Interface Adapter, Framework) |
| **Business Logic** | Usually in Model or Controller | Isolated in Use Case/Service layer |
| **Dependency Direction** | Controller → Model, View → Model | Always directed inward (Dependency Rule) |
| **Testability** | Hard to test Controller (framework-dependent) | Easy to test (each layer is independent) |
| **Database Coupling** | Model is usually coupled with ORM/DB | Storage layer isolated, easy to swap implementation |
| **Flexibility** | Hard to change framework/database | Easily change external dependencies |
| **Complexity** | Simple, easy to learn | More complex, more abstraction |
| **Best For** | Small to medium projects, CRUD apps | Large projects, complex business logic |

### 5.3 Code Comparison

#### MVC Style (Simple but Coupled)

```go
// controllers/asset_controller.go
type AssetController struct {
    db *sql.DB
}

func (c *AssetController) CreateAsset(w http.ResponseWriter, r *http.Request) {
    var req Request
    json.NewDecoder(r.Body).Decode(&req)

    // Validation + Business logic + SQL - all in controller
    if req.Name == "" {
        http.Error(w, "name required", 400)
        return
    }

    // Direct SQL query
    _, err := c.db.Exec(
        "INSERT INTO assets (name, type) VALUES (?, ?)",
        req.Name, req.Type,
    )

    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }

    json.NewEncoder(w).Encode(map[string]string{"status": "created"})
}
```

**Issues:**

- ❌ Controller knows about database implementation
- ❌ Business logic mixed with HTTP handling
- ❌ Hard to test (requires a real database)
- ❌ Hard to scale when logic gets complex

#### Clean Architecture Style (More Layers, More Flexible)

```go
// handler/asset_handler.go (Presentation Layer)
func (h *AssetHandler) CreateAsset(w http.ResponseWriter, r *http.Request) {
    var req Request
    json.NewDecoder(r.Body).Decode(&req)

    // Only handle HTTP concerns
    asset, err := h.service.CreateAsset(req.Name, req.Type)
    if err != nil {
        RespondError(w, http.StatusBadRequest, err.Error())
        return
    }

    RespondJSON(w, http.StatusCreated, asset)
}

// service/asset_service.go (Use Case Layer)
func (s *AssetService) CreateAsset(name, assetType string) (*model.Asset, error) {
    // Business logic and validation
    if name == "" {
        return nil, errors.New("name is required")
    }

    asset := &model.Asset{
        ID:     uuid.New().String(),
        Name:   name,
        Type:   assetType,
        Status: model.StatusActive,
    }

    // Delegate to storage interface
    return asset, s.storage.Create(asset)
}

// storage/postgres/postgres.go (Infrastructure Layer)
func (s *PostgresStorage) Create(asset *model.Asset) error {
    // Pure data access logic
    _, err := s.db.Exec(
        "INSERT INTO assets (id, name, type, status) VALUES ($1, $2, $3, $4)",
        asset.ID, asset.Name, asset.Type, asset.Status,
    )
    return err
}
```

**Advantages:**

- ✅ Each layer has clear responsibility
- ✅ Service layer is testable without HTTP or DB
- ✅ Easy to swap storage (memory → postgres → mongodb)
- ✅ Business logic is separated and reusable

### 5.4 When to Use MVC vs Clean Architecture?

#### Use MVC when:

✅ Small projects, simple CRUD
✅ Small team, need to ship fast
✅ Uncomplicated business logic
✅ Rarely change database/framework
✅ Prototype, MVP

**Example:** Personal blog, simple admin dashboard, internal tools

#### Use Clean Architecture when:

✅ Large, complex projects
✅ Rich business logic
✅ High testability needed
✅ Multiple developers working together
✅ Expected changes in database/framework
✅ Long-term maintenance

**Example:** E-commerce platform, banking system, enterprise applications

### 5.5 Evolution Path

Many projects start with MVC and evolve into Clean Architecture:

```
Stage 1: Simple MVC
  → Fast development
  → Good for MVP

Stage 2: Growing Complexity
  → Complex business logic
  → Hard to test
  → Controllers become bloated

Stage 3: Refactor to Clean Architecture
  → Extract service layer
  → Add storage abstractions
  → Improve testability
```

### 5.6 Hybrid Approach

In practice, you can mix:

```go
// Keep MVC structure but add a service layer
controllers/
  └── asset_controller.go    // Thin controller
services/                     // Business logic
  └── asset_service.go
models/
  └── asset.go
repositories/                 // Data access
  └── asset_repository.go
```

This is a **good middle ground** for many projects!

---

## 6. ANTI-PATTERNS TO AVOID

### ❌ Mixing Layers

```go
// BAD: Handler contains business logic and SQL
func (h *Handler) CreateAsset(w http.ResponseWriter, r *http.Request) {
    var req Request
    json.NewDecoder(r.Body).Decode(&req)

    // Business logic in handler - WRONG!
    if req.Name == "" {
        http.Error(w, "name required", 400)
        return
    }

    // SQL in handler - WRONG!
    db.Exec("INSERT INTO assets VALUES (?, ?)", req.Name, req.Type)
}
```

### ❌ Wrong Dependency Direction

```go
// BAD: Model depends on storage
type Asset struct {
    ID   string
    DB   *sql.DB  // WRONG! Entity should not know about the database
}
```

### ❌ Skipping Layers

```go
// BAD: Handler calls storage directly
func (h *Handler) CreateAsset(w http.ResponseWriter, r *http.Request) {
    asset := parseRequest(r)
    h.storage.Create(asset)  // WRONG! Skipping the service layer
}
```

---

## 7. CHECKLIST

When implementing a new feature, follow this checklist:

- [ ] **Model**: Define the domain entity in `model/`
- [ ] **Storage Interface**: Add method to storage interface
- [ ] **Storage Implementation**: Implement in `memory/` or `postgres/`
- [ ] **Service**: Add the business logic method in `service/`
- [ ] **Handler**: Add the HTTP handler in `handler/`
- [ ] **Main**: Wire up in `main.go`
- [ ] **Test**: Write tests for each layer

---

## 8. EVOLVING THE ARCHITECTURE

### Session 2: Basic Structure

```
cmd/server/main.go
internal/
  ├── handler/
  ├── service/
  ├── storage/memory/
  └── model/
```

### Session 3: Add Database

```
+ internal/storage/postgres/
+ migrations/
```

### Session 5: Add Tests

```
+ test/
+ *_test.go files
```

### Session 6: Add Frontend

```
+ web/
+ Dockerfile
+ docker-compose.yml
```

---

## 📚 REFERENCES

- [The Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Go Project Layout](https://github.com/golang-standards/project-layout)
- [Dependency Injection in Go](https://blog.drewolson.org/dependency-injection-in-go)
