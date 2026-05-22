# 📄 SOFTWARE REQUIREMENT SPECIFICATION

## Mini Attack Surface Management (Mini ASM)

## 1. INTRODUCTION

### 1.1 Purpose

This document describes the requirements for the Mini Attack Surface Management (ASM) system — a service designed to manage the organization's public-facing assets.

The system serves the following purposes:

- Tracking public-facing domains, IPs, and services
- Managing operational status
- Serving as a platform to extend into security monitoring

### 1.2 Definitions

| Term  | Meaning                                     |
| ----- | ------------------------------------------- |
| Asset | A public resource (domain, IP, service)     |
| ASM   | Attack Surface Management                   |
| MVP   | Minimum Viable Product                      |

## 2. OVERALL DESCRIPTION

### 2.1 Product Perspective

The system operates independently and can be integrated with:

- Monitoring tools
- Scanner tools
- Frontend dashboard

## 3. FUNCTIONAL REQUIREMENTS

### 3.1 Asset Management

**FR-1: Create Asset**

- The system must allow creating a new asset
- Validate input (non-empty name, valid type)
- Automatically generate UUID for the asset
- Automatically set the created_at timestamp
- Default status is "active"

**FR-2: List Assets**

- The system must return a list of all assets
- Sorted by creation time (newest first)

**FR-3: Get Single Asset**

- The system must allow retrieving detailed asset information by ID
- Return 404 if not found

**FR-4: Update Asset**

- The system must allow updating asset details
- Updatable fields: name, type, status
- Do not allow changing: id, created_at
- Return 404 if not found

**FR-5: Delete Asset**

- The system must allow deleting an asset by ID
- Return 404 if not found

**FR-6: Filter Assets**

- The system must support filtering by type
- The system must support filtering by status
- Multiple filters can be combined

**FR-7: Search Assets**

- The system must support searching by name (partial match)

**FR-8: Health Check**

- The system must provide an endpoint to check the service health status
- Check database connection

## 4. NON-FUNCTIONAL REQUIREMENTS

### 4.1 Performance

- The system must handle at least 100 requests/second in a local environment
- Average response time < 200ms

### 4.2 Security

- Do not return stack traces to the client
- Perform comprehensive input validation
- Prevent application panic
- Do not expose internal structs

### 4.3 Maintainability

- Must have a clear project structure
- Separation of concerns (handler/service/storage)
- Code must adhere to clean code principles
- Clear naming conventions

### 4.4 Logging

- Log every HTTP request
- Log errors clearly
- Do not log sensitive data

### 4.5 API Design

- Use RESTful conventions
- Use appropriate HTTP status codes:
  - 200 OK - Request succeeded
  - 201 Created - Resource created successfully
  - 400 Bad Request - Invalid input
  - 404 Not Found - Resource does not exist
  - 500 Internal Server Error - Server error

### 4.6 Testing

- Unit test coverage ≥ 70%
- Integration tests for all endpoints
- Test cases for edge cases and error scenarios

## 5. DATA MODEL

### 5.1 Asset

| Field      | Type          | Description       | Required | Constraints                            |
| ---------- | ------------- | ----------------- | -------- | -------------------------------------- |
| id         | string (UUID) | Unique identifier | Yes      | Auto-generated                         |
| name       | string        | Asset name        | Yes      | 1-255 characters                       |
| type       | string        | Asset type        | Yes      | enum: domain/ip/service                |
| status     | string        | Asset status      | Yes      | enum: active/inactive, default: active |
| created_at | timestamp     | Creation time     | Yes      | Auto-generated                         |
| updated_at | timestamp     | Last update time  | No       | Auto-updated                           |

**Example:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "example.com",
  "type": "domain",
  "status": "active",
  "created_at": "2026-03-02T10:30:00Z",
  "updated_at": "2026-03-02T10:30:00Z"
}
```

## 6. PROJECT STRUCTURE

```
mini-asm/
├── cmd/
│   └── server/
│       └── main.go          # Entry point
├── internal/
│   ├── handler/             # HTTP handlers
│   ├── service/             # Business logic
│   ├── storage/             # Data access layer
│   │   ├── memory/          # In-memory implementation
│   │   └── postgres/        # Database implementation
│   └── model/               # Data models
├── test/                    # Integration tests
├── web/                     # Frontend files
├── go.mod
└── README.md
```

## 7. LEARNING OBJECTIVES

### Session 1: Foundation & Theory

- Understand software development lifecycle
- Use basic Git operations (clone, commit, push, pull)
- Master Go syntax and conventions

### Session 2: API Development Basics

- Set up Go project structure
- Implement HTTP server using the standard library
- Implement in-memory storage
- Create and test RESTful endpoints

### Session 3: Database Integration

- Connect Go application to a database
- Implement CRUD using SQL
- Use database migration tools

### Session 4: Advanced Features

- Complete REST API with full CRUD
- Implement filtering and searching
- Advanced validation

### Session 5: Quality Assurance

- Write unit tests using Go's testing package
- Integration testing strategies
- Error handling patterns

### Session 6: Integration & Deployment

- Integrate backend with a simple frontend
- API documentation using OpenAPI
- Containerize with Docker
- Deploy to local/cloud environments
