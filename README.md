# 🏗️ Generated Architecture

When the AI generates a feature, it follows a fixed architecture.

The goal is simple:

> Keep business logic independent from frameworks, databases, and delivery mechanisms.

Example generated feature:

```text
internal/
├── domain/
│   └── customer.go
│
├── repository/
│   └── customer/
│       ├── interface.go
│       ├── entity.go
│       ├── dto.go
│       └── postgres.go
│
├── usecase/
│   └── customer/
│       ├── crud.go
│       ├── dto.go
│       └── errors.go
│
├── api/
│   └── http/
│       └── customer/
│           ├── handler.go
│           └── dto.go
│
├── events/
│   └── customercreated.go
│
pkg/
└── broker/
```

---

# How It Works

Every request moves through layers.

```text
HTTP Request
     ↓
Handler
     ↓
Usecase
     ↓
Repository
     ↓
Database
```

Side effects happen independently:

```text
Usecase
   ↓
Publish Event
   ↓
Broker
   ↓
Subscribers
```

---

## 1. Domain → Business Model

```go
type Customer struct {
    ID uint
    Name string
}
```

Contains:

* Entities
* Constants
* Value objects
* Config

Rules:

✅ No HTTP
✅ No ORM
✅ No framework imports

Think of this as:

> "What the business is."

---

## 2. Repository → Data Layer

Repository converts domain objects into persistence models.

Example:

```text
Customer
↓
Repository
↓
Postgres
```

Responsibilities:

* Queries
* Mapping
* Pagination
* Transactions

Rules:

✅ Knows database
❌ Does not know HTTP
❌ Does not know business workflows

Think of this as:

> "How data is stored."

---

## 3. Usecase → Business Logic

This is the center of the system.

Example:

```text
Create Customer
↓
Validate
↓
Persist
↓
Publish Event
```

Responsibilities:

* Validation
* Business rules
* Orchestration
* Event publishing

Rules:

✅ Talks to interfaces
❌ Talks directly to DB
❌ Imports HTTP framework

Think of this as:

> "What the application does."

---

## 4. Handler → Transport Layer

Handler converts HTTP into usecase calls.

Example:

```text
POST /customers
↓
Parse JSON
↓
Call usecase
↓
Return response
```

Responsibilities:

* Parse requests
* Validate input
* Serialize response

Rules:

✅ Knows HTTP
❌ Knows repository
❌ Contains business logic

Think of this as:

> "How users interact."

---

## 5. Events → Async Processing

Mutations can emit events.

Example:

```text
Customer Created
↓
Broker.Publish()
↓
Analytics
↓
Notifications
↓
Read Models
```

Benefits:

* Decoupled workflows
* Async side effects
* Easier scaling

---

# Dependency Direction

Dependencies always move inward.

```text
handler
↓
usecase
↓
repository
↓
domain
```

Never:

```text
handler → repository ❌
usecase → gin ❌
domain → gorm ❌
```

---

# Why This Architecture Scales

Small project:

```text
1 entity
few files
```

Large project:

```text
50 entities
same rules
same structure
same developer experience
```

AI generates code.

Architecture keeps it maintainable.
