# SpecKit — Fulfillment Module

**Module:** `@medusajs/fulfillment`  
**Version:** Medusa v2.15.4  
**Document Type:** Functional & Technical Specification

---

## 1. Functional Requirements

### FR-1: Geographic Configuration

| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | System SHALL support creating fulfillment sets with a unique name | MUST |
| FR-1.2 | System SHALL support service zones grouped under fulfillment sets | MUST |
| FR-1.3 | System SHALL support country-level geo zones (ISO 3166-1 alpha-2 codes) | MUST |
| FR-1.4 | System SHALL support province-level geo zones | MUST |
| FR-1.5 | System SHALL support city-level geo zones | MUST |
| FR-1.6 | System SHALL support zip/postal code geo zones with expression-based matching | MUST |
| FR-1.7 | Service zones SHALL be unique by name across the system | MUST |

### FR-2: Shipping Options

| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | System SHALL support flat-rate and calculated (provider-computed) shipping prices | MUST |
| FR-2.2 | Shipping options SHALL be associated with exactly one service zone | MUST |
| FR-2.3 | Shipping options SHALL support optional provider assignment | MUST |
| FR-2.4 | Shipping options SHALL support optional shipping profile classification | SHOULD |
| FR-2.5 | Shipping option names SHALL support i18n translations | SHOULD |
| FR-2.6 | System SHALL support conditional rules that restrict option availability | MUST |
| FR-2.7 | Rules SHALL combine with AND logic on the same shipping option | MUST |
| FR-2.8 | System SHALL support arbitrary provider-specific data on shipping options (`data` field) | MUST |

### FR-3: Fulfillment Lifecycle

| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | System SHALL create fulfillment records linked to a stock location | MUST |
| FR-3.2 | Fulfillment creation SHALL invoke the configured fulfillment provider | MUST |
| FR-3.3 | System SHALL snapshot the delivery address at fulfillment creation time | MUST |
| FR-3.4 | System SHALL support marking fulfillments as packed, shipped, and delivered | MUST |
| FR-3.5 | System SHALL support fulfillment cancellation with provider notification | MUST |
| FR-3.6 | Fulfillment cancellation SHALL be prevented if already delivered | MUST |
| FR-3.7 | System SHALL record which admin user marked a fulfillment as shipped | SHOULD |

### FR-4: Label Generation

| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | System SHALL support creating shipment labels with tracking numbers | MUST |
| FR-4.2 | Labels SHALL store tracking URLs for customer-facing tracking | MUST |
| FR-4.3 | Labels SHALL store printable label URLs (PDF/image) | MUST |
| FR-4.4 | A fulfillment MAY have multiple labels (e.g., multi-package shipment) | SHOULD |

### FR-5: Provider Interface

| ID | Requirement | Priority |
|---|---|---|
| FR-5.1 | System SHALL support pluggable fulfillment providers via `IFulfillmentProvider` | MUST |
| FR-5.2 | Providers SHALL validate fulfillment option data before creation | MUST |
| FR-5.3 | Providers that support calculated pricing SHALL implement `canCalculate` and `calculatePrice` | MUST |
| FR-5.4 | Providers SHALL support return fulfillment creation | SHOULD |

---

## 2. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Provider calls are async non-blocking | < 5s timeout per provider call |
| NFR-2 | Fulfillment creation is transactional | Either all steps succeed or full rollback via workflow compensation |
| NFR-3 | Address snapshot immutability | `FulfillmentAddress` cannot be updated after creation |
| NFR-4 | Geographic query performance | GeoZone country/province/city indexes for fast address matching |
| NFR-5 | Provider isolation | A provider failure does not affect other shipping options |

---

## 3. API Specification

### POST /admin/fulfillments — Create Fulfillment

**Request Body:**
```json
{
  "location_id": "sloc_01",
  "provider_id": "manual",
  "shipping_option_id": "so_01",
  "delivery_address": {
    "first_name": "Jane",
    "last_name": "Doe",
    "address_1": "123 Main St",
    "city": "San Francisco",
    "country_code": "us",
    "postal_code": "94105"
  },
  "items": [
    {
      "line_item_id": "li_01",
      "inventory_item_id": "iitem_01",
      "quantity": 2,
      "title": "Blue T-Shirt",
      "sku": "SHIRT-BLU-M",
      "barcode": "1234567890"
    }
  ]
}
```

**Response:** `201 Created` with `FulfillmentDTO`

### POST /admin/fulfillments/:id/shipment — Create Shipment

**Request Body:**
```json
{
  "labels": [
    {
      "tracking_number": "1Z999AA10123456784",
      "tracking_url": "https://www.ups.com/track?tracknum=1Z999AA10123456784",
      "label_url": "https://cdn.example.com/labels/ful_01.pdf"
    }
  ]
}
```

### POST /admin/fulfillment-sets — Create Fulfillment Set

**Request Body:**
```json
{
  "name": "Default Shipping",
  "type": "shipping"
}
```

### POST /admin/service-zones — Create Service Zone

**Request Body:**
```json
{
  "name": "Continental US",
  "fulfillment_set_id": "fuset_01",
  "geo_zones": [
    { "type": "country", "country_code": "us" }
  ]
}
```

---

## 4. Workflow Specifications

### createFulfillmentWorkflow

**Input:**
```typescript
{
  location_id: string,
  provider_id: string,
  shipping_option_id?: string,
  delivery_address: FulfillmentAddressDTO,
  items: FulfillmentItemDTO[],
  data?: Record<string, unknown>,
  metadata?: Record<string, unknown>
}
```

**Steps:**
1. `validateFulfillmentDataStep` — calls `provider.validateFulfillmentData()`
2. `createFulfillmentStep` — creates DB record, calls `provider.createFulfillment()`

**Compensation:**
1. `cancelFulfillmentStep` — calls `provider.cancelFulfillment()` if creation partially succeeded

---

## 5. Data Validation Rules

| Field | Rule |
|---|---|
| `FulfillmentSet.name` | Unique; non-empty string |
| `ServiceZone.name` | Unique across all service zones |
| `GeoZone.type` | Must be `country`, `province`, `city`, or `zip` |
| `GeoZone.country_code` | ISO 3166-1 alpha-2 (2 lowercase characters) |
| `GeoZone.province_code` | Required when type is `province`, `city`, or `zip` |
| `ShippingOption.price_type` | `flat_rate` or `calculated` |
| `FulfillmentItem.quantity` | Must be positive |
| `FulfillmentLabel.tracking_number` | Non-empty string |

---

## 6. Error Conditions

| Error | Type | HTTP Status |
|---|---|---|
| Fulfillment set name already exists | `INVALID_DATA` | 422 |
| Service zone name already exists | `INVALID_DATA` | 422 |
| Provider not found | `NOT_FOUND` | 404 |
| Shipping option not found | `NOT_FOUND` | 404 |
| Fulfillment not found | `NOT_FOUND` | 404 |
| Cannot cancel delivered fulfillment | `NOT_ALLOWED` | 400 |
| Cannot cancel already-canceled fulfillment | `NOT_ALLOWED` | 400 |
| Provider validation failed | `INVALID_DATA` | 400 |
