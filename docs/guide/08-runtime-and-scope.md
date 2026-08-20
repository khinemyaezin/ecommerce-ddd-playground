# 8. Runtime, tech stack, and current scope

This chapter is the honest appendix: how you run the app, what it is built
with, and what the Commerce Platform PRD describes that **is not in this
repo**.

## Tech stack (current)

| Layer | Choice |
| --- | --- |
| Language / build | Java 21, Maven multi-module (`pom.xml`) |
| Application | Spring Boot 3.3.4, Spring Modulith 1.2.4 |
| Persistence | JPA / Hibernate, Flyway, PostgreSQL (one DB per module) |
| Mapping | MapStruct 1.5, Lombok |
| Money | Joda-Money |
| Category tree | Nested-set library (`khinemyaezin.nested-set`) |
| Security | Spring Security, JJWT 0.12, BCrypt, HttpOnly cookies |
| API | Spring MVC, Spring HATEOAS (HAL), RFC 7807 Problem Detail |
| Logging | Framework `Logger` SPI + `logger-slf4j` |
| CI | GitHub Actions: JDK 21, `./mvnw` package + test |
| Run | `mvn -pl store spring-boot:run` or Docker Compose |

Feature gates you will see in Compose / YAML: `MERCHANT_ENABLED`,
`PRICING_ENABLED`, `API_SECURITY`, per-module `*_SEED_ENABLED`.

## Run locally

```bash
mvn clean install
mvn -pl store spring-boot:run
```

Docker (needs `docker/env/dev.env` with datasource URLs and `MODULE_DATABASES`):

```bash
docker compose --env-file docker/env/dev.env -f docker-compose.yml up --build -d
```

Postgres databases expected in dev: `catalog`, `inventory`, `identity`,
`merchant`, `pricing`, `workflows`.

## Logging

Application code uses:

```java
private static final Logger log = Loggers.getLogger(CurrentClass.class);
```

not Lombok `@Slf4j`. `framework` owns the facade; `logger-slf4j` registers via
`ServiceLoader`. Backend is selected with `LOGGER_BACKEND` (default `slf4j`).
See [ADR-004](../system/ADR_004-Framework_logging_facade_&_slf4j_bridge.md).

## What is implemented

- Identity: register, login, refresh, platforms, roles, access assignments,
  invitations
- Merchant: onboarding / review lifecycle, storefronts
- Catalog: categories (nested set), products, variants, media, descriptions
- Pricing: price sets, lists, preferences, calculate
- Inventory: locations / zones / bins, stock, movements, allocation,
  catalog-driven product-variant view
- Workflows: durable instances; create / update sellable product sagas
- Cross-cutting: CQRS buses, per-module persistence + outbox, HAL APIs,
  Problem Detail errors, local JWT

## What is not implemented (do not document as architecture)

From the [Commerce Platform PRD](../Commerce_Platform.md) and missing modules:

- Cart, checkout, Cash on Delivery
- Orders, fees, settlement
- C2C offers and deals
- Customer storefront as a first-class BC
- Broker-backed outbox dispatcher
- Schema-based event payload (Avro/Protobuf) for service extraction
- OAuth2 / OIDC provider (contracts only)
- Consumer **inbox** table for exactly-once-ish side effects
- Background worker that auto-resumes parked workflows
- SSE hub Java types (`SseHub`) — specified in ADR-008/009, not in `store` sources

Those belong on a “later BCs” page, not in protocol chapters.

## Stale docs to treat as errata

| Document | Issue |
| --- | --- |
| Product BC ADR | Still mentions publishing via `ApplicationEventPublisher` after save. Repositories now write the **outbox**. |
| Create-sellable-product spec | Links `ADR_001-Create_Sellable_Product_workflow.md`, which is not in `docs/workflows/architecture/` (update ADR exists). Trust the spec + orchestrator code. |
| Product ADR status list | Mentions `IN_REVIEW`. Code `ProductStatus` is `DRAFT`, `ACTIVE`, `ARCHIVED`, `SUSPENDED`. |
| README (historical) | Pointed at a missing Documentation Guide. This `docs/guide/` tree is that guide. |

When chapter and ADR disagree, **code wins**, then fix the ADR.

## Suggested talk series

1. Why a modular monolith — this chapter + [System architecture](01-system-architecture.md)
2. Request path — [CQRS](02-cqrs.md) + [Persistence](03-module-persistence.md)
3. Reliable events — [Outbox](04-outbox.md)
4. When to orchestrate — [Orchestration](05-orchestration-and-choreography.md) + [Create sellable product](flows/create-sellable-product.md)
5. Identity is not the seller — [Security](06-security-and-scopes.md) + [Merchant onboarding](flows/merchant-onboarding.md)
6. One domain deep-dive — pick Catalog, Inventory, or Pricing
