---
name: laravel-domain-architecture
description: >
  Domain-oriented architecture for Laravel without external package dependencies, based on concepts
  from "Laravel Beyond CRUD" (Spatie). Use this skill when the user wants to structure a Laravel project
  using DDD, domain-oriented architecture, or organize code by business concepts instead of technical
  properties. Also activate when the user asks about: Actions, Data Objects (DTOs), lean models,
  States with enums, ViewModels (Blade or API), HTTP Query builders, application layer, or scaling
  Laravel projects beyond simple CRUD. Activate for any mention of "domain layer", "application layer",
  "beyond CRUD", "domain-oriented", or requests to refactor a Laravel project into domain structure.
  Even if the user asks "how to organize a large Laravel project", this skill is relevant.
---

# Laravel Domain Architecture

Guide for creating and refactoring Laravel applications using a pragmatic domain-oriented architecture.
Based on patterns from "Laravel Beyond CRUD" (Brent Roose / Spatie), **simplified to work with pure PHP
and native Laravel features — no mandatory external packages**.

Core philosophy: **group code by business meaning, not by technical property**.

---

## Agent Instructions

- When creating or modifying **Domain layer** components (Actions, Data Objects, Models, Enums, QueryBuilders, Collections, Events) → consult `references/domain-building-blocks.md`
- When creating or modifying **Application layer** components (Controllers, ViewModels, HTTP Queries, Jobs, Requests, Resources, API Versioning) → consult `references/application-layer.md`
- Do not load both references at once — use only the one relevant to the current task
- Always follow the naming conventions and anti-patterns listed below

---

## When to Use

Projects **larger than average** — 50+ models, multiple developers, multi-year lifespan. For simple CRUD,
stick with Laravel defaults. The tipping point is when navigating the standard `app/` structure starts to hurt.

---

## Project Structure

The architecture separates code into two main layers:

```
src/
├── Domain/          ← Business logic (the "what")
│   ├── Invoices/
│   ├── Customers/
│   ├── Payments/
│   └── Shared/
├── App/             ← Infrastructure/HTTP (the "how it's exposed")
│   ├── Admin/
│   │   ├── Invoices/
│   │   └── Customers/
│   ├── Api/
│   │   ├── V1/
│   │   │   ├── Invoices/
│   │   │   └── Customers/
│   │   └── V2/
│   │       └── Invoices/
│   ├── Console/
│   └── Providers/
└── Support/         ← Shared helpers, base classes, utilities
```

### Composer Autoload

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/App/",
            "Domain\\": "src/Domain/",
            "Support\\": "src/Support/"
        }
    }
}
```

### Custom Application Class (optional)

```php
// src/App/Application.php
namespace App;

class Application extends \Illuminate\Foundation\Application
{
    protected $namespace = 'App\\';
}

// bootstrap/app.php
$app = (new Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
))->useAppPath('src/App');
```

> **Simple alternative**: keep `app/` and add `Domain` as a namespace inside it:
> `app/Domain/`. This is what Spatie does in most projects.

---

## Domain Layer

Each domain folder represents a **business concept**:

```
src/Domain/Invoices/
├── Actions/              # Business logic (user stories)
├── Data/                 # Typed DTOs (readonly classes)
├── Models/               # Lean Eloquent models
├── QueryBuilders/        # Custom query builders
├── Collections/          # Custom collections
├── Enums/                # Enums with behavior
├── Events/               # Domain events
├── Exceptions/           # Domain exceptions
└── Rules/                # Domain validation rules
```

→ Detailed implementation of each block in `references/domain-building-blocks.md`

---

## Application Layer

The application layer **consumes** the domain and exposes it to the end user. Group by **business concept**, not by technical type:

```
src/App/Api/V1/
├── Invoices/
│   ├── Controllers/
│   ├── Requests/
│   ├── Resources/
│   ├── ViewModels/
│   ├── Queries/
│   └── Middleware/
├── Customers/
└── Support/              # Base controllers, shared middleware
```

### API Versioning

The API is versioned at the application layer. The domain **does not change** — only controllers, resources, and requests are versioned:

```
src/App/Api/
├── V1/
│   └── Invoices/
│       ├── Controllers/InvoicesController.php
│       ├── Resources/InvoiceResource.php
│       └── Requests/StoreInvoiceRequest.php
├── V2/
│   └── Invoices/
│       ├── Controllers/InvoicesController.php    ← New controller
│       ├── Resources/InvoiceResource.php         ← Resource with different fields
│       └── Requests/StoreInvoiceRequest.php      ← Updated validation
└── Support/
```

Rules:
- **Domain layer is shared** — Actions, Models, Enums are the same across V1 and V2
- **Only version what changes** — If V2 only changes Invoices, no need to copy Customers
- **Routes per version** — `routes/api_v1.php`, `routes/api_v2.php` with prefix `api/v1`, `api/v2`
- **Deprecate, don't delete** — Keep V1 running as long as there are consumers

→ Detailed implementation in `references/application-layer.md`

---

## Naming Conventions

| Type | Suffix | Location | Example |
|---|---|---|---|
| Action | `*Action` | `Domain/*/Actions/` | `CreateInvoiceAction` |
| Data Object | `*Data` | `Domain/*/Data/` | `InvoiceData` |
| Model | — | `Domain/*/Models/` | `Invoice` |
| QueryBuilder | `*QueryBuilder` | `Domain/*/QueryBuilders/` | `InvoiceQueryBuilder` |
| Collection | `*Collection` | `Domain/*/Collections/` | `InvoiceLineCollection` |
| Enum | `*Status` / `*Type` | `Domain/*/Enums/` | `InvoiceStatus` |
| Event | `*Event` | `Domain/*/Events/` | `InvoiceCreatedEvent` |
| Exception | `*Exception` | `Domain/*/Exceptions/` | `InvalidInvoiceTransitionException` |
| Controller | `*Controller` | `App/*/Controllers/` | `InvoicesController` |
| ViewModel | `*ViewModel` | `App/*/ViewModels/` | `InvoiceDetailViewModel` |
| HTTP Query | `*IndexQuery` | `App/*/Queries/` | `InvoiceIndexQuery` |
| Resource | `*Resource` | `App/*/Resources/` | `InvoiceResource` |
| Request | `Store*Request` / `Update*Request` | `App/*/Requests/` | `StoreInvoiceRequest` |
| Job | `*Job` | `App/*/Jobs/` | `SendInvoiceMailJob` |

---

## Core Principles

1. **Data Objects** — Never pass loose arrays. Use typed `readonly` classes (pure PHP 8.2). Validation stays in `FormRequest`.
2. **Actions** — Classes with an `execute()` method. Represent user stories. Compose via constructor injection. Primary place for business logic.
3. **Lean Models** — Only relationships, casts, custom QueryBuilders, and custom Collections. No business logic.
4. **Enums with methods** — Replace the state pattern in most cases. Include `color()`, `label()`, `canTransitionTo()` directly in the enum. Transitions are done via Actions.
5. **ViewModels** — Blade: implement `Arrayable`. API: compose JSON responses from multiple sources. For simple endpoints, use Resource directly.
6. **Jobs are infrastructure** — Manage queues and retries. Business logic stays in Actions called from `handle()`.

---

## Cross-Domain Communication

Domains **can** import models and data objects from other domains directly. This architecture is
pragmatic — it doesn't require interfaces or anti-corruption layers for internal communication.

Rules:
- **Models can reference models from other domains** in relationships (`Invoice` → `Customer`)
- **Actions can receive data objects and models from other domains** as parameters
- **Avoid circular dependencies** — if A depends on B and B depends on A, extract shared logic to `Domain/Shared/`
- **Domain events** are the preferred mechanism when a domain needs to **react** to something from another domain without direct coupling
- Use `Domain/Shared/` for cross-cutting concerns (audit log, generic notifications) — keep it small

---

## Domain Service Providers

Each domain or application **can** have its own ServiceProvider when it needs to register
bindings, event listeners, or specific configurations:

```
src/App/Providers/
├── DomainServiceProvider.php       # Registers all domain providers
├── InvoiceServiceProvider.php      # Bindings and events for Invoices
└── CustomerServiceProvider.php     # Bindings and events for Customers
```

Use domain ServiceProviders when:
- The domain has event listeners to register
- Actions need custom bindings (e.g., interface → implementation)
- The domain requires specific boot configurations

For simple domains (no custom bindings), don't create a ServiceProvider — Laravel's auto-discovery
and auto-injection already handle it.

---

## Anti-patterns

Avoid these patterns when using this architecture:

- **Logic in Models** — Models don't calculate, don't send emails, don't do business validation. That goes in Actions.
- **Overly generic Actions** — `ProcessInvoiceAction` that does 10 things. Prefer specific actions: `CreateInvoiceAction`, `MarkInvoiceAsPaidAction`.
- **Repository pattern** — Eloquent already is the repository. Don't create an extra abstraction layer on top of it. Use custom QueryBuilders.
- **Interfaces for everything** — Only create interfaces when there are multiple real implementations (e.g., payment gateways). For everything else, inject the concrete class.
- **Giant Domain/Shared** — If `Shared/` grows too large, it probably has code that should be in its own domain.
- **Copying entire API to new version** — When versioning, only copy the modules that changed. V2 can import Resources from V1 if they haven't changed.

---

## Testing Strategy

- **Data Objects**: Minimal testing — the type system does the heavy lifting
- **Actions**: The most important tests. Pattern: **setup → execute → assert**
- **Models**: Test custom QueryBuilders and Collections in isolation
- **Enums**: Test behavior methods and transition rules

→ Complete examples of immutable factories and tests in `references/domain-building-blocks.md`

---

## Migration Strategy

No need to adopt everything at once. Recommended order:

1. **Data Objects** — Replace arrays with typed DTOs
2. **Actions** — Extract business logic from controllers and models
3. **Domains** — Group related code when directories become too large
4. **Enums with methods** — Replace conditionals with behavior in the enum
5. **Application Layer** — Introduce ViewModels, HTTP Queries, and module grouping
6. **API Versioning** — Structure when the API needs breaking changes

Domains can and should change over time. Start pragmatically, refactor as understanding grows.

---

## Reference Files

- `references/domain-building-blocks.md` — Data Objects, Actions, Models, Enums, QueryBuilders, Collections, Events, Cross-Domain Communication, Tests
- `references/application-layer.md` — Controllers, ViewModels (Blade and API), HTTP Queries, Jobs, Requests, Resources, API Versioning
