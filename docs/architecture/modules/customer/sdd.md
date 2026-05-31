# Software Design Document — Customer Module

**Module:** `@medusajs/customer`  
**Version:** Medusa v2.15.4  
**Status:** Stable

---

## 1. Purpose and Scope

This SDD describes the internal design of the Customer module: its data model, service architecture, repository layer, and domain event strategy. It is intended for engineers contributing to or extending the module.

---

## 2. Data Model

### 2.1 Entity Relationship

```
Customer ──────────────── CustomerAddress
  │  (1:N)                  (belongs_to)
  │
  └── CustomerGroupCustomer (pivot)
        │
      CustomerGroup
```

### 2.2 Entity Definitions

#### Customer
```sql
CREATE TABLE customer (
  id           TEXT PRIMARY KEY,          -- prefix: cus_
  company_name TEXT,
  first_name   TEXT,
  last_name    TEXT,
  email        TEXT,
  phone        TEXT,
  has_account  BOOLEAN DEFAULT FALSE,
  metadata     JSONB,
  created_by   TEXT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ
);
-- Unique partial index: email + has_account (non-deleted)
CREATE UNIQUE INDEX ON customer (email, has_account) WHERE deleted_at IS NULL;
```

#### CustomerGroup
```sql
CREATE TABLE customer_group (
  id         TEXT PRIMARY KEY,            -- prefix: cusgroup_
  name       TEXT NOT NULL,
  metadata   JSONB,
  created_by TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE UNIQUE INDEX ON customer_group (name) WHERE deleted_at IS NULL;
```

#### CustomerGroupCustomer (pivot)
```sql
CREATE TABLE customer_group_customer (
  id                  TEXT PRIMARY KEY,
  customer_id         TEXT NOT NULL REFERENCES customer(id),
  customer_group_id   TEXT NOT NULL REFERENCES customer_group(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);
```

#### CustomerAddress
```sql
CREATE TABLE customer_address (
  id                    TEXT PRIMARY KEY,  -- prefix: cuaddr_
  address_name          TEXT,
  is_default_shipping   BOOLEAN DEFAULT FALSE,
  is_default_billing    BOOLEAN DEFAULT FALSE,
  company               TEXT,
  first_name            TEXT,
  last_name             TEXT,
  address_1             TEXT,
  address_2             TEXT,
  city                  TEXT,
  country_code          TEXT,
  province              TEXT,
  postal_code           TEXT,
  phone                 TEXT,
  metadata              JSONB,
  customer_id           TEXT NOT NULL REFERENCES customer(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at            TIMESTAMPTZ
);
```

### 2.3 Cascade Behaviour
- Deleting a `Customer` **cascades** to `CustomerAddress` rows.
- Deleting a `Customer` **detaches** from `CustomerGroup` (pivot rows removed, groups preserved).
- Deleting a `CustomerGroup` **detaches** its customers.

---

## 3. Service Architecture

### 3.1 Class Hierarchy

```
ICustomerModuleService (interface)
  └── CustomerModuleService extends MedusaService<{...}>
        ├── customerService_          (IMedusaInternalService<Customer>)
        ├── customerAddressService_   (IMedusaInternalService<CustomerAddress>)
        ├── customerGroupService_     (IMedusaInternalService<CustomerGroup>)
        └── customerGroupCustomerService_ (IMedusaInternalService<CustomerGroupCustomer>)
```

`MedusaService<T>` is a generated base class that wires up standard CRUD methods (`list`, `listAndCount`, `retrieve`, `create`, `update`, `delete`, `softDelete`, `restore`) for each entity in the type map using the corresponding injected internal service.

### 3.2 Dependency Injection

All dependencies are registered in Medusa's IoC container and injected via the constructor:

| Dependency | Type | Purpose |
|---|---|---|
| `baseRepository` | `DAL.RepositoryService` | Root ORM session / unit-of-work |
| `customerService` | `IMedusaInternalService` | Low-level CRUD for `Customer` |
| `customerAddressService` | `IMedusaInternalService` | Low-level CRUD for `CustomerAddress` |
| `customerGroupService` | `IMedusaInternalService` | Low-level CRUD for `CustomerGroup` |
| `customerGroupCustomerService` | `IMedusaInternalService` | Pivot table management |

### 3.3 Transaction Strategy

Public methods (e.g. `createCustomers`) are decorated with `@InjectManager()`, which resolves a database manager from the shared context or creates a new one. Protected implementation methods (e.g. `createCustomers_`) use `@InjectTransactionManager()` to run inside the active transaction.

```ts
@InjectManager()
async createCustomers(data, @MedusaContext() ctx = {}) {
  return await this.createCustomers_(data, ctx)
}

@InjectTransactionManager()
protected async createCustomers_(data, @MedusaContext() ctx = {}) {
  return await this.customerService_.create(toArray(data), ctx)
}
```

### 3.4 Group Membership Methods

`addCustomerToGroup` / `removeCustomerFromGroup` delegate to `customerGroupCustomerService_` to create or soft-delete pivot rows. Duplicate membership is handled by checking for existing non-deleted pivot entries before insertion.

---

## 4. Repository Layer

All internal services extend `MedusaInternalService`, which provides:

- `list(filters, config, ctx)` — applies filter/pagination/select/relations
- `create(data[], ctx)` — bulk insert via ORM
- `update(criteria, data, ctx)` — bulk update
- `softDelete(ids[], ctx)` — sets `deleted_at = now()`
- `restore(ids[], ctx)` — clears `deleted_at`

The `baseRepository` provides the ORM entity manager. No custom repository classes are needed for this module beyond the generated base.

---

## 5. Joiner Configuration

The module declares a `ModuleJoinerConfig` that exposes its entities to Medusa's **remote query** layer:

```ts
{
  serviceName: Modules.CUSTOMER,
  alias: [
    { name: "customer", args: { entity: "Customer" } },
    { name: "customer_group", args: { entity: "CustomerGroup" } },
    { name: "customer_address", args: { entity: "CustomerAddress" } },
  ],
  primaryKeys: ["id", "email"],
  linkableKeys: { customer_id: "Customer", customer_group_id: "CustomerGroup" },
}
```

This allows other modules to join customer data via `useQueryGraphStep` without importing the Customer module directly.

---

## 6. Domain Events

Events are emitted post-transaction via `@EmitEvents()` on public mutating methods. Event payloads carry only the entity ID to keep the event bus message small; consumers can hydrate full data via the service.

| Method | Event |
|---|---|
| `createCustomers` | `customer.created` |
| `updateCustomers` | `customer.updated` |
| `deleteCustomers` | `customer.deleted` |
| `createCustomerGroups` | `customer_group.created` |
| `updateCustomerGroups` | `customer_group.updated` |
| `deleteCustomerGroups` | `customer_group.deleted` |

---

## 7. Validation

- **Email uniqueness** is enforced at the database level via a partial unique index `(email, has_account) WHERE deleted_at IS NULL`.
- **Group name uniqueness** is enforced via a partial unique index `(name) WHERE deleted_at IS NULL`.
- Input validation (required fields, format) is performed by the API layer (zod schemas) before reaching the service.

---

## 8. Migrations

| Migration | Change |
|---|---|
| `20240124154000` | Initial schema |
| `20240524123112` | Add `created_by` to customer and group |
| `20240602110946` | Add searchable indexes |
| `20241211074630` | Add `metadata` to address |
| `20251010130829` | Soft-delete support adjustments |
