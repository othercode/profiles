---
name: structuring-projects
description: Use when creating a new Laravel project with DDD and Hexagonal Architecture, adding a new bounded context, or scaffolding a new module within an existing context.
---

## Overview

Structure Laravel applications as **bounded contexts** with **hexagonal layers**. Each bounded context is a self-contained namespace division inside `app/`, wired via a dedicated ServiceProvider. One module per aggregate root. Dependencies flow inward only.

---

## Project Layout

```text
app/
├── {BoundedContext}/
│   ├── {BoundedContext}ServiceProvider.php
│   ├── Shared/                              # Optional: cross-module concerns
│   └── {Module}/                            # One module per aggregate root
│       ├── Application/
│       ├── Domain/
│       └── Infrastructure/
└── Shared/                                  # Shared Kernel (no aggregates)
    ├── SharedServiceProvider.php
    ├── Application/
    ├── Domain/
    └── Infrastructure/
```

---

## Module Template

When creating a new module within a bounded context, scaffold the hexagonal structure. Items marked **(always)** are required; others are added when the module needs them:

```text
{Module}/
├── Application/
│   ├── Create{Entity}.php                   # Use case (always)
│   ├── Find{Entity}.php                     # Use case (always)
│   ├── Update{Entity}Information.php        # Use case (when updatable)
│   ├── Delete{Entity}.php                   # Use case (when deletable)
│   └── EventHandlers/                       # (when handling domain events)
├── Domain/
│   ├── {Entity}.php                         # Aggregate root — always
│   ├── {Entity}Factory.php                  # (when tests need factories)
│   ├── ValidationRules.php                  # (when entity has validation)
│   ├── Contracts/
│   │   └── {Entity}Repository.php           # Interface — always
│   ├── Events/                              # (when entity emits events)
│   │   ├── {Entity}Created.php
│   │   └── {Entity}Deleted.php
│   └── Exceptions/
│       ├── {Entity}Exception.php            # Base exception — always
│       └── {Entity}NotFound.php             # always
└── Infrastructure/
    ├── Http/                                # (when entity has UI/API)
    │   ├── Controllers/
    │   │   ├── Create{Entity}Controller.php
    │   │   ├── Show{Entity}Controller.php
    │   │   ├── List{Entities}Controller.php
    │   │   ├── Update{Entity}InformationController.php
    │   │   ├── Delete{Entity}Controller.php
    │   │   └── API/                         # (when both web + API exist)
    │   └── Requests/
    │       ├── {Entity}Request.php
    │       └── Filter{Entity}Request.php
    └── Persistence/
        ├── Eloquent{Entity}Repository.php   # always
        ├── Migrations/                      # always
        └── Seeders/                         # (when seeding needed)
```

---

## Architectural Rules

### 1 Module = 1 Aggregate Root

Each module directory maps to exactly one aggregate root entity. Child entities live inside the parent module's `Domain/` directory — never in their own module.

```text
# CORRECT: child entity inside parent module
Companies/Domain/Company.php           # Aggregate root
Companies/Domain/CompanyContact.php    # Child entity

# WRONG: child entity promoted to own module
Companies/Domain/Company.php
CompanyContacts/Domain/CompanyContact.php
```

### Hexagonal Layer Dependencies

| Layer | Contains | Depends On |
|---|---|---|
| **Domain** | Entities, VOs, enums, contracts, events, exceptions | Nothing (innermost) |
| **Application** | Use cases, event handlers, jobs, notifications | Domain only |
| **Infrastructure** | Controllers, repos, requests, persistence, mail | Application + Domain |

Dependencies flow **inward only**. Domain never imports from Application or Infrastructure.

### Shared Kernel (`app/Shared/`)

Cross-cutting contracts, traits, and base classes used by ALL bounded contexts. The Shared Kernel has **no aggregates** — it provides infrastructure only.

| Layer | Key Artifacts |
|---|---|
| Application | `CanGenerateIdentifiers` trait (UUID generation), `VersionManager` |
| Domain | `HasDomainEvents` trait, `Event`/`EventBus` contracts, `HasProfilePhoto` trait |
| Infrastructure | Base `ServiceProvider`, `IlluminateEventBus`, `EloquentFactory`, base controllers, middleware |

### Context-Level Shared (`{BoundedContext}/Shared/`)

Optional directory for cross-module concerns **within** a single bounded context. Use when multiple modules in the same context share value objects, validators, or routes:

```text
CompanyRegistry/
├── Shared/
│   ├── Domain/                       # VOs, enums, validators shared across modules
│   └── Infrastructure/
│       └── Http/Routes/
│           ├── web.php               # Context-level web routes
│           └── api.php               # Context-level API routes
├── Companies/
├── Industries/
└── JobOffers/
```

**Rule:** If shared only within one context, use `{BC}/Shared/`. If shared across contexts, use `app/Shared/`.

---

## Naming Conventions

### Namespaces

```text
App\{BoundedContext}\{Module}\{Layer}\...
```

```php
App\IdentityAndAccess\Users\Domain\User
App\CompanyRegistry\Companies\Application\CreateCompany
App\CustomerRelationshipManagement\Contacts\Infrastructure\Persistence\EloquentContactRepository
```

### Table Prefixes

Each bounded context uses a short prefix for its database tables. Choose a 2-4 letter lowercase abbreviation of the context name.

Example: `CustomerRelationshipManagement` → prefix `crm_` → tables `crm_contacts`, `crm_documents`.

### Classes and Files

| Artifact | Convention | Example |
|---|---|---|
| Bounded Context dir | PascalCase multi-word | `IdentityAndAccess/`, `CompanyRegistry/` |
| Module dir | PascalCase plural noun | `Users/`, `Companies/`, `Visas/` |
| ServiceProvider | `{BC}ServiceProvider` | `CompanyRegistryServiceProvider` |
| Entity (root) | Singular noun | `Company`, `User`, `Visa` |
| Entity (child) | Parent-context noun | `CompanyContact`, `Invitation` |
| All files | PSR-4, one class per file | `CreateCompany.php` |

---

## Adding a New Bounded Context

1. Create directory `app/{BoundedContext}/`
2. Create `{BoundedContext}ServiceProvider.php` extending `App\Shared\Infrastructure\ServiceProvider`
3. Register in `bootstrap/providers.php` (after `SharedServiceProvider`)
4. Choose a table prefix (2-4 letters)
5. Add modules as needed (one per aggregate)
6. Optionally create `{BC}/Shared/` for cross-module concerns and context-level routes

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Child entity in its own module | Keep inside parent module's `Domain/` |
| Module with multiple aggregate roots | Split into separate modules |
| Domain importing from Infrastructure | Use contracts in `Domain/Contracts/`, implement in Infrastructure |
| Cross-module code goes straight to `app/Shared/` | Use context-level `{BC}/Shared/` if only needed within one BC |
| All migrations in `database/migrations/` | Prefer `Infrastructure/Persistence/Migrations/` per module |
| Flat `app/` with no bounded context grouping | Group by bounded context, not by layer |

---

## Cross-Context References

Bounded contexts must not import directly from another context's Domain or Infrastructure. When Context A needs to reference an entity from Context B:

1. **Use the Shared Kernel** — Place shared contracts (interfaces, value objects, enums) in `app/Shared/Domain/` that both contexts depend on.
2. **Reference by identifier** — Store the foreign entity's ID (UUID), not the entity itself. Resolve it through the owning context's repository when needed.
3. **Communicate via domain events** — When Context A needs to react to changes in Context B, publish a domain event from B and handle it in A.

**Never:** Import `App\IdentityAndAccess\Users\Domain\User` from inside `App\CustomerRelationshipManagement\`. Instead, store `user_id` and resolve via a shared contract or event.
