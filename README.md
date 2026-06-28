# 🚀 Go Clean Architecture

Opinionated Go backend architecture for building scalable, maintainable services with **Clean / Onion Architecture**, **strict dependency boundaries**, **constructor-based dependency injection**, and **event-driven workflows**.

Designed from real-world backend engineering practices and codified into a repeatable development workflow.

---

## ✨ Features

🏗️ **Clean Architecture**

* Strict dependency direction
* Layer isolation
* Framework-independent business logic

🔌 **Interface at Point of Use**

* Consumers define contracts
* Reduced coupling
* Easier testing

🧩 **Constructor-Based Dependency Injection**

* No global state
* No service locators
* Explicit wiring

🎯 **Pure Domain Layer**

* No ORM
* No HTTP concerns
* No framework imports

⚡ **Event-Driven Projections**

* Publish side effects asynchronously
* In-memory event broker
* Eventual consistency

🧪 **Testing First**

* Unit tests via mocks
* Repository integration tests
* Enforced architecture validation

---

# Architecture

```text
HTTP Handler
     ↓
Usecase
     ↓
Repository
     ↓
Domain
```

Dependencies only move inward.

```text
handler → usecase → repository → domain
```

---

# Project Structure

```text
cmd/
internal/
├── api/http/
├── config/
├── domain/
├── events/
├── middleware/
├── repository/
├── usecase/
pkg/
└── broker/
migrations/
docs/
```

---

# Quick Start

Clone the repository:

```bash
git clone https://github.com/<your-username>/golang-clean-architecture.git

cd golang-clean-architecture
```

Install dependencies:

```bash
go mod tidy
```

Run:

```bash
go run cmd/main/main.go
```

Run tests:

```bash
go test ./...
```

---

# Example — Create a Customer Service

## 1. Create Domain

```go
// internal/domain/customer.go

package domain

type Customer struct {
    ID    uint
    Name  string
    Email string
}
```

---

## 2. Create Repository

```go
repo := customerrepo.NewCustomerPostgres(db)
```

Responsible for:

* Persistence
* Mapping
* Query filtering

---

## 3. Create Usecase

```go
uc := customerusecase.NewCustomerUsecase(
    repo,
    broker,
)
```

Responsible for:

* Business rules
* Validation
* Event publishing

---

## 4. Create HTTP Handler

```go
handler := customerhandler.NewHandler(uc)

handler.RegisterRoutes(router)
```

Responsible for:

* Request parsing
* Response serialization

---

## 5. Wire Everything

```go
customerRepo := customerrepo.NewCustomerPostgres(db)

customerUsecase := customerusecase.NewCustomerUsecase(
    customerRepo,
    eventBroker,
)

customerHandler := customer.NewHandler(
    customerUsecase,
)

customerHandler.RegisterRoutes(router)
```

Done 🎉

You now have:

```text
domain
↓
repository
↓
usecase
↓
handler
↓
events
```

with clear boundaries and testability.

---

# Design Rules

✅ Domain imports only standard library
✅ Constructor injection only
✅ Interface-driven architecture
✅ No repository access from handlers
✅ No ORM access from usecases
✅ Async side effects through events

---

# Testing

Run all tests:

```bash
go test ./...
```

Run usecase tests:

```bash
go test ./internal/usecase/...
```

Repository integration tests use testcontainers.

---

# Why This Exists

Most backend projects start clean and become difficult to maintain after enough features.

This project provides conventions that keep services:

* predictable
* testable
* scalable
* easier to evolve

---

# Contributing

PRs and discussions are welcome.

If you have ideas to improve architecture boundaries, testing workflows, or developer experience — open an issue.

---

⭐ If this helped you, give the repository a star.
