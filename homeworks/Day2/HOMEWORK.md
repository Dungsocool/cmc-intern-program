# 📝 Homework Assignment - Sessions 1-3

**Deadline:** Before Day 3  
**Submission Method:** Push to your personal Git repository on the `homework` branch. Invite dinhmanhtan (dmtangtnd@gmail.com) to the project. Create a pull request from the `homework` branch to `main`. Set reviewer to `dinhmanhtan`.  
**Note**: The project can be written in other languages; Go is not strictly mandatory. If you use a different language, please describe how to install and run the project.

---

## 📝 Table of Contents

- [General Requirements](#general-requirements)
- [How to Submit](#how-to-submit)
- [Exercise 1: Statistics APIs (20 pts)](#exercise-1-statistics-apis-20-pts)
  - [1.1 Get Assets Statistics](#11-get-assets-statistics)
  - [1.2 Count Assets by Filter](#12-count-assets-by-filter)
- [Exercise 2: Batch Create Assets (25 pts)](#exercise-2-batch-create-assets-25-pts)
- [Exercise 3: Batch Delete Assets (20 pts)](#exercise-3-batch-delete-assets-20-pts)
- [Exercise 4: Database Connection Retry (25 pts)](#exercise-4-database-connection-retry-25-pts)
- [Exercise 5: Database Health Check (15 pts)](#exercise-5-database-health-check-15-pts)
- [Exercise 6: Pagination & Filtering (15 pts) - BONUS](#exercise-6-pagination--filtering-15-pts---bonus)
- [Exercise 7: Search by Name (10 pts) - BONUS](#exercise-7-search-by-name-10-pts---bonus)
- [Grading Rubric](#grading-rubric)
- [Hints & Tips](#hints--tips)
- [Reference Materials](#reference-materials)
- [Bonus Challenges](#bonus-challenges)

---

## General Requirements

- Code must execute correctly without errors.
- Follow Clean Architecture patterns as taught.
- Include proper error handling.
- Testable using `curl` or Postman.

---

## How to Submit

### Requirements:
Push your code to the `homework` branch, open a PR to `main`, and add `dinhmanhtan` as reviewer.

---

## Exercise 1: Statistics APIs (20 pts)

### 1.1 Get Assets Statistics

- **Endpoint:** `GET /assets/stats`
- **Description:** Aggregates statistics of total assets grouped by type and status.
- **Response:** `200 OK`
  ```json
  {
    "total": 150,
    "by_type": {
      "domain": 50,
      "ip": 80,
      "service": 20
    },
    "by_status": {
      "active": 120,
      "inactive": 30
    }
  }
  ```

### 1.2 Count Assets by Filter

- **Endpoint:** `GET /assets/count`
- **Query params:** `?type=domain&status=active` (both optional)
- **Response:** `200 OK`
  ```json
  {
    "count": 45
  }
  ```

---

## Exercise 2: Batch Create Assets (25 pts)

**Requirement:** Add multiple assets in a single request. Must use **Database Transactions** to ensure ACID properties (All-or-Nothing). If one asset fails validation, the entire batch must rollback.

- **Endpoint:** `POST /assets/batch`
- **Request Body:**
  ```json
  {
    "assets": [
      {"name":"google.com","type":"domain"},
      {"name":"1.1.1.1","type":"ip"},
      {"name":"nginx","type":"service"}
    ]
  }
  ```
- **Response:** `201 Created`
  ```json
  {
    "created": 3,
    "assets": [
      {"id":"uuid-1","name":"google.com","type":"domain","status":"active"},
      {"id":"uuid-2","name":"1.1.1.1","type":"ip","status":"active"},
      {"id":"uuid-3","name":"nginx","type":"service","status":"active"}
    ]
  }
  ```

**Validation rules:**
- `name` cannot be empty.
- `type` must be one of: `domain`, `ip`, `service`.

**Test Cases:**
```bash
# Success case
curl -X POST http://localhost:8080/assets/batch \
  -H "Content-Type: application/json" \
  -d '{
    "assets": [
      {"name":"test1.com","type":"domain"},
      {"name":"test2.com","type":"domain"}
    ]
  }'

# Error case (invalid type) - should rollback all
curl -X POST http://localhost:8080/assets/batch \
  -H "Content-Type: application/json" \
  -d '{
    "assets": [
      {"name":"test1.com","type":"domain"},
      {"name":"test2.com","type":"invalid_type"}
    ]
  }'
# Expected: 400 Bad Request, none created
```

---

## Exercise 3: Batch Delete Assets (20 pts)

**Requirement:** Delete multiple assets concurrently.

### API Specification

- **Endpoint:** `DELETE /assets/batch`
- **Query params:** `?ids=uuid1,uuid2,uuid3`
- **Response:** `200 OK`
  ```json
  {
    "deleted": 3,
    "not_found": 0
  }
  ```

### Behavior:
- Deletes all valid IDs.
- Ignores non-existing IDs (do not return an error).
- Returns the number of successfully deleted assets and not found counts.

**Test:**
```bash
# Create test assets first
ID1=$(curl -s -X POST http://localhost:8080/assets \
  -H "Content-Type: application/json" \
  -d '{"name":"test1.com","type":"domain"}' | jq -r '.id')

ID2=$(curl -s -X POST http://localhost:8080/assets \
  -H "Content-Type: application/json" \
  -d '{"name":"test2.com","type":"domain"}' | jq -r '.id')

# Batch delete (include 1 fake ID)
curl -X DELETE "http://localhost:8080/assets/batch?ids=$ID1,$ID2,fake-uuid-123"

# Expected response:
# {"deleted": 2, "not_found": 1}

# Verify deletion
curl http://localhost:8080/assets/$ID1
# Expected: 404 Not Found
```

---

## Exercise 4: Database Connection Retry (25 pts)

**Requirement:** The Server must automatically retry connecting to the Database if the initial connection fails.

### Specification:
- Max retries: **5 times**.
- Exponential backoff: **1s ➡️ 2s ➡️ 4s ➡️ 8s ➡️ 16s**.
- Log details of each attempt.
- If all 5 attempts fail, exit the application with an error message.

### Expected Logs:
```
[*] Database connection attempt 1/5...
[!] Connection failed: connection refused. Retrying in 1s...
[*] Database connection attempt 2/5...
[!] Connection failed: connection refused. Retrying in 2s...
[*] Database connection attempt 3/5...
[+] Database connected successfully!
```

### Hints:
- Create file `internal/database/retry.go`
- Function: `ConnectWithRetry(dsn string, maxRetries int) (*sql.DB, error)`
- Exponential backoff: `time.Sleep(time.Duration(1<<uint(attempt-1)) * time.Second)`

---

## Exercise 5: Database Health Check (15 pts)

**Requirement:** Upgrade `/health` endpoint to return database status details.

### API Specification

- **Endpoint:** `GET /health`
- **Response:**
  - `200 OK` (if DB is connected)
  - `503 Service Unavailable` (if DB is down)

  ```json
  {
    "status": "ok",
    "database": {
      "status": "connected",
      "open_connections": 2,
      "in_use": 0,
      "idle": 2,
      "max_open": 25
    },
    "timestamp": "2026-03-06T10:00:00Z"
  }
  ```

### Implementation hints:
- Update `HealthHandler` to receive `*sql.DB`.
- Use `db.Ping()` to check connection.
- Use `db.Stats()` to fetch connection pool information.

**Test:**
```bash
# Normal operation
curl http://localhost:8080/health | jq

# Stop database
docker-compose stop db
sleep 2
curl http://localhost:8080/health
# Expected: 503, status="degraded", database.status="disconnected"

# Restart database
docker-compose start db
sleep 2
curl http://localhost:8080/health
# Expected: 200, status="ok", database.status="connected"
```

---

## Exercise 6: Pagination & Filtering (15 pts) - BONUS

**Requirement:** Add server-side pagination and filtering for listing assets.

### API Specification

- **Endpoint:** `GET /assets`
- **Query params:**
  - `page` (default: 1)
  - `limit` (default: 20, max: 100)
  - `type` (optional: domain, ip, service)
  - `status` (optional: active, inactive)

- **Response:**
  ```json
  {
    "data": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "total_pages": 8
    }
  }
  ```

### SQL hints:
```sql
SELECT * FROM assets
WHERE type = $1 AND status = $2
ORDER BY created_at DESC
LIMIT $3 OFFSET $4
```

**Test:**
```bash
# Page 1, 10 items
curl "http://localhost:8080/assets?page=1&limit=10"

# Filter by type
curl "http://localhost:8080/assets?type=domain"

# Combine filters
curl "http://localhost:8080/assets?page=2&limit=20&type=domain&status=active"
```

---

## Exercise 7: Search by Name (10 pts) - BONUS

**Requirement:** Search assets by name (partial match).

### API Specification

- **Endpoint:** `GET /assets/search`
- **Query params:** `q` (search query, required)
- **Response:** Array of matching assets (max 100)
- **Behavior:** Case-insensitive, partial matching.

### SQL hints:
```sql
SELECT * FROM assets
WHERE name ILIKE $1
LIMIT 100
```

**Test:**
```bash
# Search for "example"
curl "http://localhost:8080/assets/search?q=example"

# Search for ".com"
curl "http://localhost:8080/assets/search?q=.com"

# Case insensitive
curl "http://localhost:8080/assets/search?q=DOMAIN"
```

---

## Grading Rubric

| Exercise | Score | Required / Optional |
| :--- | :--- | :--- |
| Exercise 1: Statistics | 20 | Required |
| Exercise 2: Batch Create | 25 | Required |
| Exercise 3: Batch Delete | 20 | Required |
| Exercise 4: Connection Retry | 25 | Required |
| Exercise 5: Health Check | 15 | Required |
| Exercise 6: Pagination | 15 | Bonus |
| Exercise 7: Search | 10 | Bonus |
| **Total Required** | **105** | |
| **Total with Bonus** | **130** | |

---

## Hints & Tips

### Database Transactions in Go:
```go
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback() // Auto rollback if not committed

// Do operations with tx...
_, err = tx.Exec(query, args...)
if err != nil {
    return err // Rollback via defer
}

return tx.Commit() // Success
```

### Dynamic SQL with filters:
```go
conditions := []string{}
args := []interface{}{}
argIndex := 1

if typeFilter != "" {
    conditions = append(conditions, fmt.Sprintf("type = $%d", argIndex))
    args = append(args, typeFilter)
    argIndex++
}

whereClause := ""
if len(conditions) > 0 {
    whereClause = "WHERE " + strings.Join(conditions, " AND ")
}

query := fmt.Sprintf("SELECT * FROM assets %s", whereClause)
rows, err := db.Query(query, args...)
```

---

## Reference Materials

- [PostgreSQL Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- [Connection Pooling Best Practices](https://www.alexedwards.net/blog/configuring-sqldb)
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [RESTful API Design](https://restfulapi.net/)

## Bonus Challenges

1. **Rate Limiting:** Limit requests per minute per IP address.
2. **Caching:** Cache the list of assets in-memory (e.g. 5 minutes).
3. **Audit Log:** Log every CREATE/UPDATE/DELETE action into an audit table.
4. **Soft Delete:** Support a `deleted_at` timestamp instead of physically deleting.
5. **Import CSV:** Support uploading a CSV file to create multiple assets.
6. **Export CSV:** Export assets under a CSV format.
7. **Webhooks:** Trigger a webhook whenever a new asset is registered.

---

**Good luck with the assignment! Ask on the group channel if you have any questions!**
