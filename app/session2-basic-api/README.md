# 🔨 Session 2: Basic API Development

## Objectives

- ✅ Implement Clean Architecture with 4 layers
- ✅ CRUD operations: Create, Read (List, Get by ID)
- ✅ In-memory storage with thread-safety
- ✅ Understand dependency injection and interfaces

## Comparison with Session 1

| Session 1       | Session 2           |
| --------------- | ------------------- |
| Single main.go file | 4 separate layers   |
| Hello World     | Full CRUD API       |
| No data storage | In-memory storage   |
| Monolithic      | Clean Architecture  |

## Architecture Overview

```
Request Flow:
HTTP Request
  → Handler (parse JSON, HTTP concerns)
    → Service (business logic, validation)
      → Storage (data persistence)
        → Model (domain entities)
```

## Key Changes

### 1. Entity Layer (`internal/model/`)

**New Files:**

- `asset.go` - Asset struct and constants
- `errors.go` - Custom error types

**Key Points:**

- Pure domain logic, no dependencies
- Struct tags for JSON marshalling
- Constants for type safety

### 2. Storage Layer (`internal/storage/`)

**New Files:**

- `storage.go` - Interface definition
- `memory/memory.go` - In-memory implementation

**Key Points:**

- Interface for flexibility (swap implementations)
- Thread-safety with sync.RWMutex
- Why use interfaces? → Session 3 will swap to a database!

### 3. Service Layer (`internal/service/`)

**New Files:**

- `asset_service.go` - Business logic
- `service.go` - Service interface

**Key Points:**

- Validation logic
- UUID generation
- Business rules (default status = active)
- Dependency injection (injects Storage interface)

### 4. Handler Layer (`internal/handler/`)

**New Files:**

- `asset_handler.go` - HTTP handlers for assets
- `health_handler.go` - Health check (refactored from main)
- `handler.go` - Handler registry

**Key Points:**

- HTTP-specific code only
- JSON parsing and encoding
- Status codes (201, 400, 404, 500)
- Dependency injection (injects Service)

### 5. Main (`cmd/server/main.go`)

**Changes:**

- Wire up all dependencies
- Register routes
- Remove business logic (moved to layers)

## API Endpoints

| Method | Path         | Description      | Status      |
| ------ | ------------ | ---------------- | ----------- |
| GET    | /health      | Health check     | ✅          |
| POST   | /assets      | Create asset     | ✅          |
| GET    | /assets      | List all assets  | ✅          |
| GET    | /assets/{id} | Get single asset | ✅          |
| PUT    | /assets/{id} | Update asset     | 🔜 Homework |
| DELETE | /assets/{id} | Delete asset     | 🔜 Homework |

## Testing

### 1. Health Check

```bash
curl http://localhost:8080/health
```

### 2. Create Asset

```bash
curl -X POST http://localhost:8080/assets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "example.com",
    "type": "domain"
  }'
```

### 3. List Assets

```bash
curl http://localhost:8080/assets
```

### 4. Get Single Asset

```bash
# Replace <id> with actual UUID from create response
curl http://localhost:8080/assets/<id>
```

**Summary**

- Validation BEFORE business logic
- UUID auto-generation
- Default values (status = active)
- Timestamps auto-set
- Service doesn't know HOW data is stored (memory? DB?)

- Handler only handles HTTP concerns
- No business logic here!
- Status codes: 201 for created, 400 for bad request
- Helper functions: RespondJSON, RespondError

- Each layer has single responsibility
- Easy to test each layer independently
- Easy to swap storage implementation
- Clear separation of concerns

## Resources

- Review: CLEAN_ARCHITECTURE.MD sections 2-4
- [Go Interfaces](https://go.dev/tour/methods/9)
- [Dependency Injection in Go](https://blog.drewolson.org/dependency-injection-in-go)
- [UUID package](https://github.com/google/uuid)
