# arc42 Architecture Documentation — Fulfillment Module

**Module:** `@medusajs/fulfillment`  
**Version:** Medusa v2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The Fulfillment Module must:
- Allow merchants to define geographic service areas hierarchically (country → province → city → zip)
- Support multiple fulfillment providers (manual, external carriers) with a pluggable interface
- Enable conditional shipping option availability via rule-based filtering
- Create and manage fulfillment records through their complete lifecycle (created → packed → shipped → delivered / canceled)
- Support label generation and tracking number storage from carrier APIs
- Be extensible without modifying core module code (open/closed principle via provider interface)

### 1.2 Quality Goals

| Quality Attribute | Priority | Scenario |
|---|---|---|
| Extensibility | Critical | New carriers can be added without changing module internals |
| Data Integrity | Critical | Delivery addresses are immutable once fulfillment is created (snapshot) |
| Traceability | High | All lifecycle transitions are timestamped and auditable |
| Geographic Flexibility | High | Support any geographic hierarchy from country to zip code |

---

## 2. Architecture Constraints

- Provider integrations must not block the main event loop; all provider calls are async
- Cross-module references (`line_item_id`, `inventory_item_id`) are not enforced at DB level
- PostgreSQL only; GeoZone postal expressions are stored as structured JSON for flexibility
- Provider package identifiers must be globally unique across the Medusa installation

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Medusa Application                              │
│                                                                        │
│  ┌──────────┐  createFulfillment()   ┌──────────────────────────────┐ │
│  │  Order   │ ──────────────────────► Fulfillment Module            │ │
│  │  Module  │                        │                              │ │
│  └──────────┘                        │  ┌──────────────────────┐   │ │
│                                      │  │ IFulfillmentProvider │   │ │
│  ┌──────────┐  listShippingOptions() │  │ (manual | fedex | ...)│  │ │
│  │  Cart/   │ ──────────────────────►│  └──────────────────────┘   │ │
│  │ Checkout │                        │                              │ │
│  └──────────┘                        │  ┌──────────────────────┐   │ │
│                                      │  │    PostgreSQL         │   │ │
│  ┌──────────┐                        │  └──────────────────────┘   │ │
│  │ Location │──module link──────────►│                              │ │
│  │  Module  │                        └──────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

The module uses a **hierarchical configuration** strategy: FulfillmentSet → ServiceZone → GeoZone forms a tree from business grouping down to geographic specificity. Shipping options hang off service zones, which keeps geographic configuration separate from pricing configuration.

Provider integration uses the **Strategy pattern**: all provider implementations share a common `IFulfillmentProvider` interface. The module service acts as a dispatcher, resolving providers from the DI container and delegating carrier-specific logic without coupling.

Address data uses the **snapshot immutability pattern**: delivery addresses are copied at fulfillment creation time, decoupling the fulfillment record from subsequent address changes.

---

## 5. Building Blocks

### Level 1: Module Structure

```
@medusajs/fulfillment
├── services/
│   ├── fulfillment-module-service.ts   # IFulfillmentModuleService implementation
│   └── fulfillment-provider.ts         # Provider resolution & delegation
├── models/
│   ├── fulfillment-set.ts
│   ├── service-zone.ts
│   ├── geo-zone.ts
│   ├── shipping-profile.ts
│   ├── shipping-option-type.ts
│   ├── shipping-option.ts
│   ├── shipping-option-rule.ts
│   ├── fulfillment-provider.ts
│   ├── fulfillment.ts
│   ├── fulfillment-item.ts
│   ├── fulfillment-label.ts
│   └── address.ts
└── migrations/
```

### Level 2: Key Components

| Component | Responsibility |
|---|---|
| `FulfillmentModuleService` | CRUD for all configuration entities + fulfillment lifecycle |
| `FulfillmentProviderService` | Resolves providers from container, delegates operations |
| Geographic hierarchy | FulfillmentSet → ServiceZone → GeoZone — spatial containment |
| Option evaluation | ShippingOptionRule evaluation for cart eligibility |

---

## 6. Runtime View

### Scenario: Create Fulfillment for Order

```
1. Admin triggers fulfillment creation for order line items
2. createFulfillmentWorkflow invokes IFulfillmentModuleService.createFulfillment(data, items, order, shippingOption)
3. Module creates Fulfillment record with lifecycle state = "created"
4. Module resolves IFulfillmentProvider for the shipping option's provider_id
5. Provider.createFulfillment(data, items, order, fulfillmentRecord) is called
6. Provider returns { data: { carrier_id, ... } }
7. Module updates fulfillment.data with provider response
8. Module creates FulfillmentItem records for each line item
9. FulfillmentAddress is created as a snapshot of the delivery address
10. Fulfillment record is persisted and returned
```

### Scenario: Create Shipment (Mark as Shipped)

```
1. Admin marks fulfillment as shipped
2. createShipmentWorkflow calls IFulfillmentModuleService.createShipment(id, labels)
3. Module sets fulfillment.shipped_at = now(), fulfillment.marked_shipped_by = userId
4. For each label: creates FulfillmentLabel with tracking_number, tracking_url, label_url
5. Provider.createShipment() is optionally called if the provider manages label lifecycle
6. Updated fulfillment is returned with labels
```

---

## 7. Deployment View

The Fulfillment Module shares the Medusa Node.js process. Fulfillment providers are also loaded in the same process but isolated via the DI container. External carrier API calls are made over HTTPS from within the provider implementations.

---

## 8. Crosscutting Concepts

### Provider Abstraction
The `IFulfillmentProvider` interface is the extension point. All carrier-specific operations (price calculation, data validation, shipment creation, label generation) are delegated. The module never contains carrier-specific code.

### Translatable Names
`ShippingOption.name` is marked `translatable()`, enabling the Translation Module to store locale-specific shipping option names without schema changes to the fulfillment tables.

### Rule-Based Availability
`ShippingOptionRule` uses the same operator vocabulary as the Promotion Module's rule engine (`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`), providing a consistent pattern across modules.

### Address Snapshot
`FulfillmentAddress` is cascade-deleted with its parent `Fulfillment`, ensuring no orphaned address records while maintaining delivery address integrity.

---

## 9. Architecture Decisions

### ADR-1: Timestamp-Based State Machine
Fulfillment lifecycle uses nullable timestamp fields rather than a status enum. This simplifies compound state queries and provides exact timing data for each transition without requiring a separate event table.

### ADR-2: Separate ShippingOptionType
`ShippingOptionType` is a separate entity (not an enum) to allow merchants to define custom delivery categories with metadata, displayed in the storefront option selector.

### ADR-3: Provider as DI Container Component
Providers are registered and resolved via the MedusaJS DI container rather than being directly instantiated by the module. This enables providers to have their own dependencies injected and simplifies testing via mock providers.

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Provider isolation | `IFulfillmentProvider` interface; providers resolved from DI container |
| Audit trail | `packed_at`, `shipped_at`, `delivered_at`, `canceled_at`, `created_by`, `marked_shipped_by` |
| Geographic coverage | 4-level geo zone hierarchy; `postal_expression` JSON for zip patterns |
| Cascade integrity | Cascade deletes prevent orphaned geo zones, items, labels |
