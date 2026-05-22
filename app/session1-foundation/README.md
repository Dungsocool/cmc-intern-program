# 🚀 Session 1: Foundation & Project Setup

## Objectives

- ✅ Set up project structure according to Clean Architecture
- ✅ Understand Go basics and HTTP servers
- ✅ Run a Hello World API successfully

## Code Contents

### 1. Project Structure

```
session1-foundation/
├── cmd/
│   └── server/
│       └── main.go          # Entry point - Hello World server
├── internal/
│   ├── model/               # (Empty - preparing for Session 2)
│   ├── handler/             # (Empty - preparing for Session 2)
│   ├── service/             # (Empty - preparing for Session 2)
│   └── storage/             # (Empty - preparing for Session 2)
├── go.mod
└── README.md
```

### 2. Key Concepts

#### HTTP Server in Go

- `http.HandleFunc` - register handlers for routes
- `http.ListenAndServe` - start the server
- `ResponseWriter` and `Request` - handle HTTP requests/responses

#### JSON Response

- `json.NewEncoder(w).Encode()` - convert Go struct to JSON
- Content-Type header

### 3. Running the Code

```bash
# Initialize Go module
go mod init mini-asm

# Run server
go run cmd/server/main.go

# Test endpoint (another terminal)
curl http://localhost:8080/health
```

**Expected Output:**

```json
{
  "status": "ok",
  "message": "Mini ASM service is running"
}
```

## Comparison

### ❌ Bad Practice (Monolithic)

```go
// All code in one file
func main() {
    http.HandleFunc("/assets", func(w http.ResponseWriter, r *http.Request) {
        // Parse request
        // Validate
        // Business logic
        // Database query
        // Response
    })
}
```

### ✅ Clean Architecture (Preview of Session 2)

```go
// Separation of concerns
handler → service → storage → model
```

**→ Session 1 sets up the structure, Session 2 implements the layers!**

## Resources

- [Go HTTP Server Tutorial](https://gobyexample.com/http-servers)
- [Go Modules](https://go.dev/blog/using-go-modules)
- [REST API Best Practices](https://restfulapi.net/)
