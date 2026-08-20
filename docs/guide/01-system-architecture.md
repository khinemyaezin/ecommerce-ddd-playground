# 1. System architecture

This application is a **modular monolith**: one Spring Boot process that you
run as `store`, with several **bounded contexts** that own their own models and
databases.

That is the current system, not a future microservice map. The module
boundaries exist so a context *could* be extracted later without rewriting the
domain.

## Picture

```mermaid
flowchart TB
    Client["Seller portal / Admin / API client"]

    subgraph Store["store — one Spring Boot process"]
        HTTP["Controllers + HATEOAS assemblers"]
        Facades["Command / Query services"]
        Buses["CommandBus / QueryBus"]
        Security["JWT + cookie filter → SecurityPrincipal"]

        subgraph Modules["Application modules (Spring Modulith)"]
            Identity["Identity"]
            Merchant["Merchant"]
            Catalog["Catalog"]
            Pricing["Pricing"]
            Inventory["Inventory"]
            Workflows["Workflows"]
        end

        Shared["shared — OPEN kernel<br/>CQRS wiring, security, exception handler"]
    end

    subgraph Data["Postgres — one database per module"]
        IdDb[(identity)]
        MerDb[(merchant)]
        CatDb[(catalog)]
        PriceDb[(pricing)]
        InvDb[(inventory)]
        WfDb[(workflows)]
    end

    Client --> HTTP
    HTTP --> Security
    HTTP --> Facades
    Facades --> Buses
    Buses --> Modules
    Identity --> IdDb
    Merchant --> MerDb
    Catalog --> CatDb
    Pricing --> PriceDb
    Inventory --> InvDb
    Workflows --> WfDb
    Modules --> Shared
```

`store` is the **composition root**. It imports infrastructure, exposes HTTP,
wires the buses, and binds each module to its datasource. Domain jars do not
start a server.

## Bounded contexts and what they own

Modules talk by **ID** and by **events**. They do not share aggregates or join
across databases.

```mermaid
flowchart LR
    Identity["Identity<br/>User, Platform,<br/>AccessAssignment"]
    Merchant["Merchant<br/>MerchantAccount,<br/>Storefront"]
    Catalog["Catalog<br/>Product, Category,<br/>variants"]
    Pricing["Pricing<br/>PriceSet, PriceList,<br/>calculatePrices"]
    Inventory["Inventory<br/>Location, stock,<br/>movements"]
    Workflows["Workflows<br/>process instances"]

    Identity -->|"grants access to merchantId"| Merchant
    Merchant -->|"merchantId only"| Catalog
    Catalog -->|"variantId / sku"| Pricing
    Catalog -->|"variant events"| Inventory
    Workflows -->|"request / completion events"| Catalog
    Workflows --> Pricing
    Workflows --> Inventory
```

| Context | Owns | Does **not** own |
| --- | --- | --- |
| Identity | Users, credentials, sessions, platforms, roles, access grants | The seller business, products, stock |
| Merchant | Merchant account lifecycle, storefront | Login, listings, warehouses |
| Catalog | Product content, variants, categories | Price amounts, on-hand quantity |
| Pricing | Price sets, campaigns, contextual calculation | Product names, stock |
| Inventory | Locations / zones / bins, stock, movements | Product copy, selling price |
| Workflows | Checkpoints and step order for multi-module writes | Domain invariants of the other contexts |

A person who logs in is a **User**. A business they sell under is a
**MerchantAccount**. A warehouse is an Inventory **Location**. Mixing those
three into one “seller” table is the design this repo refuses.

## Three jars per business context

```mermaid
flowchart BT
    Domain["{name}-domain<br/>aggregates, value objects, repository ports, policies<br/>no Spring, no JPA"]
    Infra["{name}-infrastructure<br/>JPA entities, mappers, repository adapters, outbox table"]
    App["store/{name}<br/>controllers, handlers, listeners, TX annotations"]
    Framework["framework<br/>CQRS, Id, AggregateRoot, outbox contracts"]

    Infra --> Domain
    App --> Domain
    App --> Infra
    Domain --> Framework
    Infra --> Framework
```

Dependency direction is inward: domain does not know HTTP or Hibernate.
Infrastructure implements ports. `store` orchestrates use cases.

Spring Modulith enforces the same idea **inside** `store`:

- `internal/` is private to that module.
- Other modules may only import named interfaces such as `catalog::events` or
  `catalog::api`.
- `shared` is `OPEN` so every module can use security and CQRS wiring.

Example: Catalog may depend on `workflows::events`. Inventory may depend on
`catalog::events` and `catalog::api`. Merchant depends only on `shared`.
Identity may depend on `merchant::events` so it can grant access when a
merchant is approved.

## How a typical request is composed

This is the same path you will see in the CQRS and persistence chapters.

```mermaid
sequenceDiagram
    participant C as Client
    participant Ctrl as Controller
    participant Svc as Facade service
    participant Bus as CommandBus
    participant H as Command handler
    participant Agg as Aggregate
    participant Repo as Repository adapter
    participant DB as Module database

    C->>Ctrl: HTTP + JWT/cookie
    Ctrl->>Svc: DTO
    Svc->>Bus: Command
    Bus->>H: dispatch
    H->>Agg: load / create, enforce rules
    H->>Repo: save(aggregate)
    Repo->>DB: business rows + outbox rows
    H-->>Svc: Result
    Svc-->>Ctrl: Response DTO
    Ctrl-->>C: HAL EntityModel
```

Writes go through the aggregate. Reads usually skip the aggregate and use a
query repository or projection. That split is [CQRS](02-cqrs.md).

## Why a modular monolith, not microservices

| Option | Why it was not chosen now |
| --- | --- |
| **One MVC app, one database, shared entities** | Fast to start, but Catalog would grow foreign keys into Inventory and Identity. Extraction later means rewriting the model. |
| **Microservices from day one** | Each BC would need its own deploy, auth, and ops story before the domain is stable. You would still need the same boundaries — plus network failure. |
| **Modular monolith (accepted)** | One process to run locally and in Docker. Separate datasources and Modulith checks keep the boundaries real. Outbox and workflow ports are already shaped for a later broker. |

The extra cost you **do** pay: more Spring wiring, explicit transaction
annotations, and no cross-module JPA joins. That cost is intentional.

## Why hexagonal slices, not “package by layer” only

A global `controller / service / repository` layout still lets any service
import any entity. Splitting **by bounded context first**, then by layer,
keeps language and persistence with the business that owns them.

`framework` is a **shared kernel**, not a dumping ground: CQRS buses, `Id`,
`AggregateRoot`, outbox contracts, workflow types, logging SPI. Business rules
do not live there.

## Where to look in code

| Piece | Location |
| --- | --- |
| Spring Boot entry | `store/.../EcommerceApplication.java` |
| Modulith markers | `store/.../catalog/CatalogModule.java` (and siblings) |
| Open shared module | `store/.../shared/package-info.java` |
| Named event packages | `store/.../catalog/events`, `merchant/events`, `workflows/events` |
| Domain example | `catalog-domain/.../aggregate/Product.java` |
| Infra example | `catalog-infrastructure/.../repository/jpa/impl/ProductRepositoryImpl.java` |
| Parent POM | `pom.xml` |

## Related ADRs

- [ADR-001 System architecture](../system/ADR_001-System_architecture.md)
