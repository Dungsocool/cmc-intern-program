# 🎓 Golang Backend & EASM Engineering Internship Program

Welcome to the **Golang Backend & EASM (External Attack Surface Management) Engineering Internship Program** repository! This repository contains a structured, hands-on curriculum designed to take engineers from foundations to building and deploying highly secure, production-ready enterprise systems using Go (Golang), ReactJS, PostgreSQL, and modern DevOps tools.

## 📂 Repository Structure

The curriculum is divided into two main sections: **Core Sessions (`app/`)** and **Exercises/Homeworks (`homeworks/`)**, supplemented by standard reference materials in **`resources/`**.

```text
├── app/
│   ├── session1-foundation/      # HTTP basics, Go syntax & REST foundations
│   ├── session2-basic-api/       # Building basic RESTful APIs in Go
│   ├── session3-database/        # Database integration with PostgreSQL & connection retry
│   ├── session4-advanced-api/    # Bulk operations, database transactions & stats
│   ├── session5-easm/            # Building EASM scan engines (IP, Port, SSL, Tech)
│   ├── session6-testing/         # Unit testing, integration testing & coverage auditing
│   └── session7-deployment/      # Docker, Docker Compose, Nginx, HTTPS & CI/CD
├── homeworks/
│   ├── Day2/                     # REST API development homework
│   └── Day3/                     # Database design & integration homework
└── resources/                    # Clean Code, Clean Architecture, Git & Web references
```

---

## 📅 Curriculum Overview

### 🔹 Session 1: Backend Engineering Foundation
* Core fundamentals of how the Web works (HTTP protocols, headers, methods, status codes).
* Go language syntax, pointers, data structures, and concurrency patterns.

### 🔹 Session 2: RESTful API Development
* Building robust RESTful APIs from scratch.
* Custom routing, JSON serialization/deserialization, query parameters, and custom middleware implementation (CORS, Logger).

### 🔹 Session 3: PostgreSQL Database Design
* Connecting Go applications to PostgreSQL using standard drivers.
* Designing relational schemas for assets and jobs.
* Implementing smart connection retry mechanisms with **Exponential Backoff**.

### 🔹 Session 4: Advanced Database & API Operations
* Handling batch creation/deletion (bulk operations).
* Implementing **Database Transactions** to guarantee the ACID properties (All-or-Nothing transactions).
* Writing complex queries for stats aggregation and paginated/filtered searches.

### 🔹 Session 5: External Attack Surface Management (EASM) Scan Engine
* Building a security recon and scanning engine:
  * **IP Scanner:** Geolocation and Autonomous System Number (ASN) lookup.
  * **Port Scanner:** High-speed concurrent TCP open port check.
  * **SSL/TLS Scanner:** Certificate expiration, issuer, and domain matching validations.
  * **Technology Identifier:** Identifying target software, frameworks, and web tech stacks.

### 🔹 Session 6: Enterprise Testing & Quality Auditing
* Writing high-quality Unit and Integration tests.
* Mocking external database connections and HTTP dependencies.
* Measuring code coverage and configuring automated checkups.

### 🔹 Session 7: Advanced Deployment, Security, and CI/CD
* Packaging the entire stack (Database, Backend, Frontend) with **Docker & Docker Compose**.
* Building secure, lightweight multi-stage Docker images.
* Configuring **Nginx Reverse Proxy** with **Let's Encrypt TLS/HTTPS** certificates.
* Setting up automated security scans (**Gosec, Gitleaks, Trivy, TruffleHog**) and SSH auto-deployment on every merge using **GitHub Actions**.

---

## 🛠️ Getting Started

To explore each session, navigate into the respective session folder under `/app` or `/homeworks` and follow the session-specific `README.md` instructions.

### Prerequisites
* Go 1.21+
* Docker & Docker Compose
* PostgreSQL client

---
*Developed as a training framework for modern, highly secure backend engineering.*