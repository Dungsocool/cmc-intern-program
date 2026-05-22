# 🎯 CLEAN CODE PRINCIPLES

> "Clean code is simple and direct. Clean code reads like well-written prose."  
> — Robert C. Martin

---

## 📋 TABLE OF CONTENTS

1. [**CORE PRINCIPLES**](#1-core-principles)
   - [1.1 Meaningful Names](#11-meaningful-names)
   - [1.2 Functions Should Be Compact](#12-functions-should-be-compact)
   - [1.3 Don't Repeat Yourself (DRY)](#13-dont-repeat-yourself-dry)
   - [1.4 Error Handling is a Single Responsibility](#14-error-handling-is-a-single-responsibility)
   - [1.5 Comments Should Explain WHY, Not WHAT](#15-comments-should-explain-why-not-what)

2. [**BEST PRACTICES FOR GO**](#2-best-practices-for-go)
   - [2.1 Package Design](#21-package-design)
   - [2.2 Error Handling](#22-error-handling)
   - [2.3 Interface Design](#23-interface-design)
   - [2.4 Struct Design](#24-struct-design)
   - [2.5 Testing](#25-testing)

3. [**CLEAN CODE BY LAYERS**](#3-clean-code-by-layers)
   - [Handler Layer](#handler-layer)
   - [Service Layer](#service-layer)
   - [Storage Layer](#storage-layer)

4. [**CODE SMELLS TO AVOID**](#4-code-smells-to-avoid)

5. [**REFACTORING CHECKLIST**](#5-refactoring-checklist)

6. [**BEFORE & AFTER EXAMPLES**](#6-before--after-examples)

7. [**DAILY HABITS**](#7-daily-habits)

8. [**KEY TAKEAWAYS**](#8-key-takeaways)

9. [**REFERENCES**](#9-references)

---

## 1. CORE PRINCIPLES

### 1.1 Meaningful Names

**Rules:**

- Names must reveal intent
- Avoid disinformation
- Should be pronounceable and searchable
- Do not use unnecessary abbreviations

**Examples:**

```go
// ❌ Bad
var d int  // elapsed time in days

// ✅ Good
var elapsedTimeInDays int
var daysSinceCreation int
```

### 1.2 Functions Should Be Compact

**Rules:**

- A function should do only ONE thing
- Ideal length: 5-15 lines
- Level of abstraction must be consistent
- If a function has more than 3-4 parameters, consider using a struct

**Examples:**

```go
// ❌ Bad: Does too many things
func ProcessOrder(orderId, userId, productId, qty int, addr string) error {
    // Validate user
    // Check inventory
    // Calculate price
    // Process payment
    // Update database
    // Send email
    return nil
}

// ✅ Good: Single responsibility
func ProcessOrder(order Order) error {
    if err := validateOrder(order); err != nil {
        return err
    }
    if err := checkInventory(order); err != nil {
        return err
    }
    return completeOrder(order)
}
```

### 1.3 Don't Repeat Yourself (DRY)

**Rules:**

- Every piece of knowledge must have a single representation
- Duplicate code is the root of all bugs
- Refactor when you see repeating patterns

### 1.4 Error Handling is a Single Responsibility

**Rules:**

- A function should either handle an error or do one thing, not both
- Never ignore errors
- Do not return nil when you should return an error
- Use error wrapping to preserve context

**Examples:**

```go
// ✅ Good
func CreateAsset(name, assetType string) (*Asset, error) {
    if name == "" {
        return nil, errors.New("name is required")
    }

    asset, err := buildAsset(name, assetType)
    if err != nil {
        return nil, fmt.Errorf("build asset: %w", err)
    }

    return asset, nil
}
```

### 1.5 Comments Should Explain WHY, Not WHAT

**Rules:**

- Good code is self-documenting
- Comments should explain business logic and architectural decisions
- Avoid redundant comments
- Update comments when updating code

**Examples:**

```go
// ❌ Bad: Obvious
// Loop through all assets
for _, asset := range assets {
    // ...
}

// ✅ Good: Explain why
// We process inactive assets first to free up resources
// before handling high-priority active assets
for _, asset := range inactiveAssets {
    // ...
}
```

---

## 2. BEST PRACTICES FOR GO

### 2.1 Package Design

- Organize by feature/domain, not by type
- Package name should be short, lowercase, and have no underscores
- Avoid util/common packages

### 2.2 Error Handling

- Return errors, do not panic (except in init or unrecoverable states)
- Use `errors.Is()` and `errors.As()` to check error types
- Wrap errors with context: `fmt.Errorf("operation failed: %w", err)`

### 2.3 Interface Design

- Accept interfaces, return structs
- Interfaces should be small (1-3 methods is ideal)
- Define interface where it is consumed, not where it is implemented

**Examples:**

```go
// ✅ Good: Small, focused interface
type Storage interface {
    Get(id string) (*Asset, error)
    Save(asset *Asset) error
}

// Consumer defines what it needs
func ProcessAsset(store Storage, id string) error {
    asset, err := store.Get(id)
    if err != nil {
        return err
    }
    // Process...
    return store.Save(asset)
}
```

### 2.4 Struct Design

- Zero value should be useful
- Provide constructor functions for complex initialization
- Use struct tags for serialization

### 2.5 Testing

- Table-driven tests
- Test behavior, not implementation
- Use meaningful test names: `TestFunction_Scenario_ExpectedBehavior`

---

## 3. CLEAN CODE BY LAYERS

### Handler Layer

**Focus:** HTTP concerns only

- Parse request → validate input → call service → format response
- Do not write business logic here
- Consistent error response format

### Service Layer

**Focus:** Business logic & orchestration

- Input validation (business rules)
- Coordinate operations
- Transaction management
- Error handling & logging

### Storage Layer

**Focus:** Data persistence

- Database queries
- Data mapping
- Connection management
- Transaction handling

**Example of clean separation:**

```go
// Handler: HTTP only
func (h *Handler) CreateAsset(w http.ResponseWriter, r *http.Request) {
    var req CreateAssetRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        RespondError(w, 400, "invalid JSON")
        return
    }

    asset, err := h.service.CreateAsset(req.Name, req.Type)
    if err != nil {
        RespondError(w, 400, err.Error())
        return
    }

    RespondJSON(w, 201, asset)
}

// Service: Business logic
func (s *Service) CreateAsset(name, assetType string) (*Asset, error) {
    if err := s.validateInput(name, assetType); err != nil {
        return nil, err
    }

    asset := &Asset{
        ID:     uuid.New().String(),
        Name:   name,
        Type:   assetType,
        Status: "active",
    }

    if err := s.storage.Save(asset); err != nil {
        return nil, fmt.Errorf("save asset: %w", err)
    }

    return asset, nil
}

// Storage: Data access only
func (s *Storage) Save(asset *Asset) error {
    query := `INSERT INTO assets (id, name, type, status) VALUES ($1, $2, $3, $4)`
    _, err := s.db.Exec(query, asset.ID, asset.Name, asset.Type, asset.Status)
    return err
}
```

---

## 4. CODE SMELLS TO AVOID

### 🚨 Common Bad Practices

| Bad Practice               | Why Bad                   | Fix                              |
| -------------------------- | ------------------------- | -------------------------------- |
| Long functions (>50 lines) | Hard to understand & test | Break into smaller functions     |
| Deep nesting (>3 levels)   | Cognitive overload        | Early returns, extract functions |
| Magic numbers              | Unclear meaning           | Use named constants              |
| Global variables           | Hidden dependencies       | Pass as parameters or use DI     |
| Ignored errors             | Silent failures           | Always handle errors             |
| God objects                | Too many responsibilities | Split into focused objects       |
| Dead code                  | Confusing, unmaintained   | Delete it                        |

### 🎯 Red Flags

```go
// ❌ Deep nesting
func Process(data Data) error {
    if data != nil {
        if data.IsValid() {
            if data.HasPermission() {
                if !data.IsLocked() {
                    // actual logic here
                }
            }
        }
    }
}

// ✅ Early returns
func Process(data Data) error {
    if data == nil {
        return errors.New("data is nil")
    }
    if !data.IsValid() {
        return errors.New("invalid data")
    }
    if !data.HasPermission() {
        return errors.New("no permission")
    }
    if data.IsLocked() {
        return errors.New("data is locked")
    }

    // actual logic here - no nesting
    return nil
}
```

---

## 5. REFACTORING CHECKLIST

When refactoring code, ask yourself the following questions:

### Naming

- [ ] Are variable/function names clear?
- [ ] Can the code be understood without comments?

### Functions

- [ ] Does the function do only one thing?
- [ ] Is the function length < 20 lines?
- [ ] Are there < 4 parameters?

### Complexity

- [ ] Is the nesting depth < 3 levels?
- [ ] Is there duplicate code?
- [ ] Are there magic numbers?

### Error Handling

- [ ] Are all errors handled?
- [ ] Are the error messages helpful?
- [ ] Is the error context preserved?

### Testing

- [ ] Is the code testable?
- [ ] Is the test coverage sufficient?
- [ ] Are the tests clear?

---

## 6. BEFORE & AFTER EXAMPLES

### ❌ Before: Messy Code

```go
func CA(n, t, s string, db *sql.DB) error {
    if n == "" || t == "" {
        return errors.New("invalid input")
    }
    id := uuid.New().String()
    if t != "domain" && t != "ip" && t != "service" {
        return errors.New("bad type")
    }
    tx, _ := db.Begin()
    _, e := tx.Exec("INSERT INTO assets VALUES (?, ?, ?, ?)", id, n, t, s)
    if e != nil {
        tx.Rollback()
        return e
    }
    tx.Commit()
    log.Println("Created asset: " + id)
    return nil
}
```

### ✅ After: Clean Code

```go
func CreateAsset(name, assetType string) (*Asset, error) {
    if err := validateAssetInput(name, assetType); err != nil {
        return nil, err
    }

    asset := buildAsset(name, assetType)

    if err := saveAsset(asset); err != nil {
        return nil, fmt.Errorf("failed to save asset: %w", err)
    }

    log.Info("asset created", "id", asset.ID, "name", asset.Name)
    return asset, nil
}

func validateAssetInput(name, assetType string) error {
    if name == "" {
        return ErrNameRequired
    }
    if !isValidAssetType(assetType) {
        return ErrInvalidAssetType
    }
    return nil
}

func buildAsset(name, assetType string) *Asset {
    return &Asset{
        ID:        uuid.New().String(),
        Name:      name,
        Type:      assetType,
        Status:    StatusActive,
        CreatedAt: time.Now(),
    }
}
```

**Improvements:**

- ✅ Clear function names
- ✅ Single responsibility
- ✅ Proper error handling
- ✅ No magic strings
- ✅ Structured logging
- ✅ Easy to test

---

## 7. DAILY HABITS

### When writing new code:

1. **Think before you code** - plan structure beforehand
2. **Write the simplest thing that works** - do not over-engineer
3. **One thing at a time** - complete one feature before moving to another
4. **Test as you go** - do not leave testing until the end

### When reviewing code:

1. **Readability first** - is the code easy to read?
2. **Question complexity** - is there a simpler way?
3. **Check error paths** - are errors handled correctly?
4. **Look for patterns** - is there an opportunity to refactor?

### When reading other people's code:

1. Understand the high-level structure first
2. Read tests to understand the expected behavior
3. Trace through a happy path
4. Identify edge cases and error handling

---

## 8. KEY TAKEAWAYS

1. **Clean code reads like prose** - others can understand it quickly
2. **Functions do one thing** - small, focused, well-named
3. **Handle errors explicitly** - never ignore, always add context
4. **No magic** - constants for numbers, clear names for variables
5. **Test-driven mindset** - if it is hard to test, the code has issues
6. **Refactor continuously** - leave code better than you found it

---

## 9. REFERENCES

- **Clean Code** by Robert C. Martin
- **Effective Go**: https://go.dev/doc/effective_go
- **Go Code Review Comments**: https://github.com/golang/go/wiki/CodeReviewComments
- **Uber Go Style Guide**: https://github.com/uber-go/guide

---

> "Any fool can write code that a computer can understand.  
> Good programmers write code that humans can understand."  
> — Martin Fowler
