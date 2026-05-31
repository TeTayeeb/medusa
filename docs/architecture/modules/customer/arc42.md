# arc42 Architecture Document — Customer Module

**Module:** `@medusajs/customer`  
**Version:** Medusa v2.15.4  
**arc42 Template Version:** 8.x

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Customer module must:
- Store customer profiles with email/phone, B2B company context, and account flag.
- Manage customer segmentation via named groups (used by pricing, promotions, and access control).
- Maintain per-customer address books with default shipping/billing flags.
- Delegate authentication (credential storage, token issuance) entirely to the Auth module.
- Integrate seamlessly with Orders, Cart, Pricing, and Promotion modules via the remote query layer.

### 1.2 Quality Goals
| Priority | Quality Goal | Motivation |
|---|---|---|
| 1 | **Data isolation** | Customer PII must not leak across tenant boundaries |
| 2 | **Composability** | Groups must be usable as rule attributes without coupling |
| 3 | **Soft-delete safety** | Deleted customers must not reuse email/account combinations |
| 4 | **Searchability** | Admin and store UIs need full-text search across name, email, company |

---

## 2. Architecture Constraints

- Must follow Medusa's **module pattern**: self-contained, IoC-injected, no direct imports from other modules.
- All cross-module data access must go through the **remote query** / Module Link layer.
- Database engine: **PostgreSQL** (via MikroORM).
- No direct HTTP calls to other services — event-driven integration only.

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────┐
│                    Medusa Application                  │
│                                                        │
│  [Admin API] ──────────────────────────────────────┐   │
│  [Store API] ──────→ [Customer Module] ←───────────┤   │
│                            │                       │   │
│                     [Auth Module]      [Order/Cart]│   │
│                      (token validation)    (links) │   │
│                                                    │   │
│              [Pricing Module]  [Promotion Module]  │   │
│              (group rule attr) (group condition)   │   │
└────────────────────────────────────────────────────────┘
```

External actors: **Storefront** (registers, logs in, manages profile), **Admin Dashboard** (manages all customers and groups).

---

## 4. Solution Strategy

| Decision | Rationale |
|---|---|
| Separate auth from profile | Auth module handles credentials; Customer handles profile. Clean separation of concerns. |
| M:N groups via pivot entity | `CustomerGroupCustomer` pivot allows efficient group membership queries with soft-delete semantics |
| Email uniqueness per `has_account` | Allows a guest email to coexist with a registered account email until account creation |
| Module Links for cross-module data | Avoids coupling at the DB level; Order/Cart reference `customer_id` via link tables |

---

## 5. Building Block View

### Level 1 — Module Structure

```
@medusajs/customer
├── models/
│   ├── customer.ts              ← DML entity definition
│   ├── customer-group.ts
│   ├── customer-group-customer.ts
│   └── address.ts
├── services/
│   └── customer-module.ts       ← ICustomerModuleService implementation
├── migrations/                  ← Ordered DB migrations
├── joiner-config.ts             ← Remote query registration
└── index.ts                     ← Module entry point + DI container setup
```

### Level 2 — Service Internals

```
CustomerModuleService
  ├── Public API layer (decorated with @InjectManager + @EmitEvents)
  │     createCustomers, updateCustomers, deleteCustomers
  │     createCustomerGroups, addCustomerToGroup, removeCustomerFromGroup
  │     createCustomerAddresses, updateCustomerAddresses, deleteCustomerAddresses
  │
  └── Internal layer (decorated with @InjectTransactionManager)
        createCustomers_, updateCustomers_,
        createCustomerGroups_, addCustomerToGroup_
        (private helpers, not on public interface)
```

---

## 6. Runtime View

### 6.1 Customer Registration (Store)

```
POST /store/customers
  → registerCustomerWorkflow
      → createCustomerStep (Customer Module: createCustomers)
      → createAuthIdentityStep (Auth Module)
      → setAuthAppMetadataStep (Auth Module: link auth_identity to customer_id)
  → return CustomerDTO
```

### 6.2 Add Customer to Group (Admin)

```
POST /admin/customer-groups/:id/customers
  → addCustomersToGroupWorkflow
      → addCustomerToGroup (Customer Module: addCustomerToGroup)
  → emit customer_group.updated
```

---

## 7. Deployment View

The Customer module runs in-process with the Medusa server. Its database tables reside in the shared PostgreSQL instance alongside all other module tables. No microservice isolation is required.

---

## 8. Cross-Cutting Concepts

### 8.1 Soft Deletes
All entities use `deleted_at TIMESTAMPTZ` for soft deletion. All queries automatically filter `WHERE deleted_at IS NULL` via MikroORM filter.

### 8.2 Searchable Fields
Fields marked `.searchable()` in the DML definition have full-text indexes generated. Searchable fields: `email`, `first_name`, `last_name`, `phone`, `company_name` on Customer; `name` on CustomerGroup; all address fields on CustomerAddress.

### 8.3 Metadata
All entities carry a `metadata JSONB` column for extensibility without schema changes.

### 8.4 Audit (`created_by`)
`Customer` and `CustomerGroup` store `created_by` to record which admin user or workflow created the record.

---

## 9. Architecture Decisions

### ADR-01: Delegate Authentication to Auth Module
**Status:** Accepted  
**Context:** Previous Medusa v1 coupled customer passwords and JWT issuance inside the customer service.  
**Decision:** Remove credential storage from the Customer module. The Auth module owns identity providers; the Customer module owns profile data. They are linked via `auth_identity.app_metadata.customer_id`.  
**Consequences:** Cleaner separation; swappable auth providers; Customer module has no crypto dependency.

### ADR-02: Pivot Entity for Group Membership
**Status:** Accepted  
**Context:** M:N relationships could be managed with a simple join table or a full entity.  
**Decision:** Use a full `CustomerGroupCustomer` DML entity with its own `id` and soft-delete, exposed via `IMedusaInternalService`.  
**Consequences:** Consistent pattern with rest of Medusa; enables future metadata on membership; slight overhead for simple queries.

### ADR-03: Email Uniqueness Only for Registered Accounts
**Status:** Accepted  
**Context:** Guest checkouts create customer records with `has_account = false`.  
**Decision:** Partial unique index `(email, has_account) WHERE deleted_at IS NULL` allows the same email as guest and account.  
**Consequences:** Prevents duplicate account registration while allowing guest reuse.

---

## 10. Risks and Technical Debt

| Risk | Severity | Mitigation |
|---|---|---|
| PII in metadata JSONB | Medium | Metadata content is application responsibility; module does not validate |
| Group name collision on restore | Low | Restore operations check uniqueness before clearing `deleted_at` |
| Large address books impact list performance | Low | Pagination enforced at API layer; index on `customer_id` |
