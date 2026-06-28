# 🚀 Go Clean Architecture Skill

Generate and maintain Go backend services using **Clean Architecture**, **strict dependency rules**, **constructor injection**, and **event-driven workflows**.

This repository is not a starter template.

It is an **AI Skill** that guides code generation and keeps architectural consistency while building features.

---

# What Is This?

This skill teaches an AI assistant how to create backend features following predefined architectural rules.

Instead of manually designing structure every time, you describe the feature and the AI generates code that follows the architecture.

Example:

```text
Create Customer CRUD
```

AI generates:

```text
internal/
├── domain/customer.go
├── repository/customer/
│   ├── interface.go
│   ├── entity.go
│   ├── dto.go
│   └── postgres.go
├── usecase/customer/
├── api/http/customer/
├── events/
```

All files follow the architecture automatically.

---

# Install Skill

Clone repository:

```bash
git clone https://github.com/<your-user>/golang-clean-architecture.git
```

Add the skill to your AI coding environment.

Example:

```text
.skills/
└── golang-clean-architecture/
```

---

# How To Use

Start a conversation with your AI agent.

## Example 1 — Create a New Service

Prompt:

```text
Using golang-clean-architecture skill:

Create a Customer service.

Requirements:
- CRUD endpoints
- Postgres repository
- Event on customer creation
- Unit + integration tests
```

Generated output:

```text
internal/domain/customer.go

internal/repository/customer/
internal/usecase/customer/
internal/api/http/customer/

internal/events/customercreated.go
```

Everything follows dependency rules automatically.

---

## Example 2 — Add Feature

Prompt:

```text
Using golang-clean-architecture skill:

Add Invoice entity.

Requirements:
- Create invoice
- List invoices
- Publish InvoiceCreated event
```

Generated:

```text
domain/invoice.go

repository/invoice/

usecase/invoice/

api/http/invoice/
```

No manual architecture decisions.

---

## Example 3 — Refactor Existing Project

Prompt:

```text
Using golang-clean-architecture skill:

Move billing logic from handlers
into usecases.

Remove repository access from HTTP layer.
Generate tests.
```

Skill will:

✅ Detect architecture violations
✅ Move logic to correct layer
✅ Regenerate interfaces
✅ Update dependency wiring

---

# What The Skill Enforces

## Domain

```text
Pure business objects
No framework imports
```

## Repository

```text
Persistence only
```

## Usecase

```text
Business rules
Event publishing
```

## Handler

```text
HTTP transport only
```

Dependencies:

```text
handler
↓
usecase
↓
repository
↓
domain
```

---

# Example Workflow

Developer:

```text
Add Customer feature
```

↓

AI:

```text
Generate domain
Generate repository
Generate usecase
Generate handler
Generate tests
Wire main.go
```

↓

Developer:

```text
Review
Adjust business rules
Merge
```

---

# Validation

Before completing generation, the skill verifies:

* No circular imports
* Constructor injection
* Interface contracts
* Mock generation
* Test coverage
* Event consistency

---

# Why Use It

You focus on:

✔ Business logic
✔ Product requirements

The skill handles:

✔ Project structure
✔ Layer boundaries
✔ Boilerplate generation
✔ Consistency

---

⭐ If useful, give the repo a star.
