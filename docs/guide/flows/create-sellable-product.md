# Flow: create sellable product

**Intent:** create a merchant’s sellable product end-to-end — Catalog product
+ variants, wait until Inventory has projected every SKU, assign variant
prices, then create inventory items.

**Pattern:** process manager (orchestration) with `WAITING_EXTERNAL` and
fan-in. Not a single HTTP transaction across four databases.

**API:** `POST /api/v1/workflows/create-sellable-product` → **202** +
`workflowId`. Poll `GET .../{workflowId}` for status. Seller access comes from
`SecurityPrincipal`, not from a merchant id in the body.

## Why a saga

Catalog, Pricing, and Inventory each have their own database. You cannot
`JOIN` them in one JPA commit. You also cannot “just fire three commands” in
the controller: a crash after Catalog commit would leave a product with no
prices and no stock, and no owner of compensation.

## Step sequence

```mermaid
flowchart TB
    S1["1. create-product<br/>Catalog CreateProductSetCommand"]
    S2["2. ensure-product-view<br/>wait ProductVariantViewProjectedEvent per SKU"]
    S3["3. create-variant-prices<br/>Pricing per variant"]
    S4["4. create-inventory-item<br/>Inventory per stock line"]
    Done["COMPLETED"]

    S1 --> S2 --> S3 --> S4 --> Done
```

| Step | Owner | Request event | Done when |
| --- | --- | --- | --- |
| `create-product` | Catalog | `RequestCreateProductSetEvent` | `SellableProductProductCreatedEvent` |
| `ensure-product-view` | Inventory projection | *(none — wait)* | all expected SKUs projected |
| `create-variant-prices` | Pricing | `RequestCreateVariantPriceEvent` × variants | all `VariantPriceCreatedEvent` |
| `create-inventory-item` | Inventory | `RequestCreateInventoryItemEvent` × lines | all `InventoryItemCreatedEvent` |

Failure: `SellableProductStepFailedEvent` → compensate (delete product and/or
price sets). Empty compensate set → `FAILED`.

## Sequence (happy path)

```mermaid
sequenceDiagram
    participant Seller
    participant WF as Orchestrator
    participant Cat as Catalog
    participant Inv as Inventory
    participant Price as Pricing

    Seller->>WF: POST 202
    WF->>Cat: RequestCreateProductSetEvent
    Cat->>Cat: save Product + catalog outbox
    Cat->>WF: SellableProductProductCreatedEvent
    Note over Inv: Catalog domain events / sync project SKUs
    Inv->>WF: ProductVariantViewProjectedEvent
    WF->>Price: RequestCreateVariantPriceEvent
    Price->>WF: VariantPriceCreatedEvent
    WF->>Inv: RequestCreateInventoryItemEvent
    Inv->>WF: InventoryItemCreatedEvent
    WF-->>Seller: GET shows COMPLETED
```

Start is **idempotent** when `idempotencyKey` is sent: the same key returns
the existing instance instead of creating a second product set.

## What stays in each module

```mermaid
flowchart LR
    WF["Workflows: order, wait, compensate"]
    Cat["Catalog: product invariants"]
    Price["Pricing: price set invariants"]
    Inv["Inventory: stock invariants"]
    WF -.->|events| Cat
    WF -.-> Price
    WF -.-> Inv
```

The Catalog listener maps the request event to `CreateProductSetCommand` and
dispatches on the **Catalog** bus. It does not save JPA entities itself.

## In-process events vs outbox

Coordination messages (`Request*` / `*Created` / `*Failed`) are Spring
application events **in this monolith**. Aggregate facts inside Catalog still
use the **catalog outbox**. When you extract a service, put the coordination
messages on a durable bus too; do not keep “HTTP 202 + memory-only saga”.

The instance **is** durable (`WorkflowStore`). The messages between steps are
the part that is still process-local.

## Related code and docs

| Piece | Location |
| --- | --- |
| HTTP | `CreateSellableProductController` |
| Saga | `CreateSellableProductOrchestrator` |
| Catalog adapter | `CreateSellableProductCatalogEventListener` |
| Spec | [docs/workflows/create-sellable-product.md](../../workflows/create-sellable-product.md) |
| Update sibling | [docs/workflows/update-sellable-product.md](../../workflows/update-sellable-product.md) |

Protocol: [Orchestration](../05-orchestration-and-choreography.md).
