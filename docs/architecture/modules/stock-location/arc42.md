# arc42 Architecture Document — Stock Location Module

**Module:** `@medusajs/stock-location`  
**Version:** Medusa v2.15.4  
**arc42 Template Version:** 8.x

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Stock Location module must:
- Store named physical or virtual fulfillment locations.
- Associate postal addresses with locations.
- Expose location IDs for use by the Inventory module's `location_id` fields.
- Support linking to Sales Channels and Fulfillment Providers via the Module Link layer.
- Emit domain events on create/update/delete.

### 1.2 Quality Goals
| Priority | Quality Goal | Motivation |
|---|---|---|
| 1 | **Simplicity** | The module's scope is narrow — avoid feature creep |
| 2 | **Linkability** | Other modules must be able to reference location IDs reliably |
| 3 | **Operability** | Admin UI must support search and filtering of locations |

---

## 2. Architecture Constraints

- No direct FK dependencies to Inventory, SalesChannel, or Fulfillment modules.
- Cross-module relationships are managed exclusively via the Module Link layer.
- Event Bus integration is optional (graceful degradation in test environments).

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────────┐
│                     Medusa Application                     │
│                                                            │
│  [Admin API] ──────→ [Stock Location Module]               │
│                              │                             │
│                   [Module Link Layer]                      │
│                   ┌──────────┼───────────┐                 │
│             [SalesChannel]  [Inventory]  [Fulfillment]     │
│             (M:N link)    (location_id)  (provider link)   │
└────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Decision | Rationale |
|---|---|
| Two-entity model only | Location + address are sufficient. Quantity tracking belongs in Inventory. Channel assignment belongs in Module Links. |
| Address as separate entity | Allows multiple locations to (theoretically) share an address; enables independent address update. |
| Event Bus as optional dependency | Stock location operations are low-frequency admin actions; event bus failure should not block location management. |
| Upsert pattern | `upsertStockLocations` simplifies bulk import workflows that don't know whether a location already exists. |

---

## 5. Building Block View

### Level 1 — Module Structure

```
@medusajs/stock-location
├── models/
│   ├── stock-location.ts           ← DML entity (sloc_)
│   └── stock-location-address.ts   ← DML entity (laddr_)
├── services/
│   └── stock-location-module.ts    ← IStockLocationService implementation
├── schema/
│   └── index.ts                    ← GraphQL schema (if applicable)
├── joiner-config.ts
└── index.ts
```

### Level 2 — Service Method Map

```
StockLocationModuleService
  ├── createStockLocations        → create location + optional inline address
  ├── upsertStockLocations        → create or update by ID/name
  ├── updateStockLocations        → update existing location + optional address update
  ├── deleteStockLocations        → soft delete location(s)
  ├── listStockLocations          → filtered, paginated list
  ├── retrieveStockLocation       → single fetch with optional relations
  └── upsertStockLocationAddresses → standalone address upsert
```

---

## 6. Runtime View

### 6.1 Create Stock Location (Admin)

```
POST /admin/stock-locations
  body: { name: "Main Warehouse", address: { address_1: "...", country_code: "US" } }

  → createStockLocationsWorkflow
      → createStockLocationsStep (Stock Location Module)
          → stockLocationAddressService_.create(address)
          → stockLocationService_.create({ name, address_id: created.id, metadata })
      → emit stock-location.created
  → return StockLocationDTO (with address expanded)
```

### 6.2 Link to Sales Channel

```
POST /admin/sales-channels/:id/stock-locations
  body: { location_id: "sloc_01" }

  → linkSalesChannelsToStockLocationWorkflow
      → remoteLink.create({
          [Modules.SALES_CHANNEL]: { sales_channel_id },
          [Modules.STOCK_LOCATION]: { stock_location_id }
        })
```

### 6.3 Delete Stock Location

```
DELETE /admin/stock-locations/:id
  → deleteStockLocationsWorkflow
      → deleteStockLocationsStep (Stock Location Module)
          → stockLocationService_.softDelete([id])
          → if address_id exists: stockLocationAddressService_.softDelete([address_id])
      → removeRemoteLinksStep (remove SalesChannel and FulfillmentProvider links)
      → emit stock-location.deleted
```

---

## 7. Deployment View

The module is entirely in-process. No additional infrastructure is required. The optional Event Bus dependency is satisfied by whichever event bus adapter (in-memory or Redis) is configured in the application.

---

## 8. Cross-Cutting Concepts

### 8.1 Soft Deletes
Both `StockLocation` and `StockLocationAddress` support soft delete (`deleted_at`). Soft-deleting a location does not cascade to inventory levels — inventory data is preserved for historical reporting.

### 8.2 Metadata
Both entities carry `metadata JSONB` for extensibility (e.g. storing 3PL provider codes, warehouse management system IDs).

### 8.3 Searchable Fields
`name`, `address.city`, `address.country_code`, `address.province`, `address.postal_code` are all marked `.searchable()` for admin search support.

### 8.4 Event Bus Resilience
The module wraps event emission in a null-check: if `eventBusModuleService_` is not injected, events are silently skipped. This enables unit testing without an event bus.

---

## 9. Architecture Decisions

### ADR-01: Address as Separate Entity
**Status:** Accepted  
**Context:** Address data could be embedded as JSONB in the location row.  
**Decision:** Separate `StockLocationAddress` entity with its own ID and lifecycle.  
**Consequences:** Enables independent address updates; consistent with Customer module address pattern; slight join cost on reads.

### ADR-02: No Quantity Tracking in This Module
**Status:** Accepted  
**Context:** Early designs considered embedding quantity fields in the location entity.  
**Decision:** Quantity tracking is entirely in the Inventory module. Stock Location only stores location identity and address.  
**Consequences:** Clean separation; location module remains small and stable; Inventory can be replaced independently.

### ADR-03: Module Links for Channel and Provider Associations
**Status:** Accepted  
**Context:** Assigning locations to sales channels requires a relationship across two modules.  
**Decision:** Use the Module Link layer (`remote_link`) rather than a direct FK or a table in either module.  
**Consequences:** Zero coupling between modules; link table is managed by the framework; consistent with Medusa v2 architecture.

---

## 10. Risks and Technical Debt

| Risk | Severity | Mitigation |
|---|---|---|
| Orphaned inventory levels after location delete | Medium | Application workflow removes inventory levels before or after soft delete |
| Missing link cleanup on delete | Medium | `removeRemoteLinksStep` in delete workflow handles link removal |
| Address country_code not validated | Low | Validation at API layer (zod); module trusts caller |
