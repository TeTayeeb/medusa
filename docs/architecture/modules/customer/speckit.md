# SpecKit — Customer Module

**Module:** `@medusajs/customer`  
**Version:** Medusa v2.15.4  
**Document type:** Functional & Technical Specification

---

## 1. Functional Specifications

### 1.1 Customer Account Management

**F-CUST-01: Create Customer**
- A customer can be created with or without an account (`has_account` flag).
- `email` is optional for guest customers; required for registered customers.
- `email` must be unique per `has_account = true` (partial unique index).
- `company_name` is optional; when present, indicates a B2B customer.

**F-CUST-02: Update Customer**
- Any non-ID field may be updated post-creation.
- Changing `email` must re-validate uniqueness.
- `has_account` should only be set to `true` after the Auth module confirms credential creation.

**F-CUST-03: Delete Customer**
- Soft-deletes the customer record (`deleted_at = now()`).
- Cascades soft-delete to all `CustomerAddress` records.
- Detaches (removes pivot rows) from all `CustomerGroup` memberships.

**F-CUST-04: List/Search Customers**
- Supports filtering by `email`, `first_name`, `last_name`, `company_name`, `has_account`, `created_at`.
- Supports full-text search across all searchable fields.
- Supports pagination (`skip`, `take`).
- Supports field selection and `groups`, `addresses` relation expansion.

### 1.2 Customer Groups

**F-GROUP-01: Create Group**
- `name` is required and must be unique (case-sensitive).

**F-GROUP-02: Add/Remove Members**
- `addCustomerToGroup({ customer_id, customer_group_id })` creates a pivot row.
- Idempotent — adding an already-member customer is a no-op.
- `removeCustomerFromGroup` soft-deletes the pivot row.

**F-GROUP-03: Delete Group**
- Soft-deletes the group and detaches all member pivot rows.
- Does NOT delete the member customers themselves.

### 1.3 Address Book

**F-ADDR-01: Create Address**
- Up to N addresses per customer (no enforced limit).
- `is_default_shipping` and `is_default_billing` are independent boolean flags.
- Setting a new default does NOT automatically unset the previous default — application layer or workflow must manage this.

**F-ADDR-02: Delete Address**
- Soft-deletes the address record.
- If the deleted address was the default, the default flag is gone (no automatic replacement).

---

## 2. Business Rules

| ID | Rule | Enforcement |
|---|---|---|
| BR-01 | Email + has_account combination must be unique among non-deleted customers | DB partial unique index |
| BR-02 | CustomerGroup name must be unique among non-deleted groups | DB partial unique index |
| BR-03 | Deleting a Customer cascades to all their addresses | DML cascade definition |
| BR-04 | Deleting a Customer detaches (does not delete) their groups | DML detach definition |
| BR-05 | `has_account` can only meaningfully be `true` if Auth module has a linked identity | Application convention |
| BR-06 | A customer can belong to multiple groups simultaneously | Supported via pivot entity |

---

## 3. API Contracts

### 3.1 POST /store/customers

**Request:**
```json
{
  "email": "customer@example.com",
  "first_name": "Jane",
  "last_name": "Doe",
  "phone": "+1-555-0100"
}
```

**Response `201`:**
```json
{
  "customer": {
    "id": "cus_01EXAMPLE",
    "email": "customer@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
    "phone": "+1-555-0100",
    "has_account": true,
    "created_at": "2024-01-01T00:00:00.000Z",
    "updated_at": "2024-01-01T00:00:00.000Z"
  }
}
```

**Errors:**
- `409 Conflict` — email already registered with `has_account = true`
- `400 Bad Request` — invalid email format

### 3.2 GET /admin/customers

**Query params:**
| Param | Type | Description |
|---|---|---|
| `q` | string | Full-text search across name/email/company |
| `has_account` | boolean | Filter by account status |
| `groups[]` | string[] | Filter by group ID membership |
| `offset` | number | Pagination offset (default 0) |
| `limit` | number | Page size (default 20, max 100) |
| `fields` | string | Comma-separated field projection |
| `expand` | string | Comma-separated relation expansion |

**Response `200`:**
```json
{
  "customers": [ /* CustomerDTO[] */ ],
  "count": 1,
  "offset": 0,
  "limit": 20
}
```

### 3.3 POST /admin/customer-groups/:id/customers

**Request:**
```json
{
  "customer_ids": ["cus_01A", "cus_01B"]
}
```

**Response `200`:**
```json
{ "customer_group": { "id": "cusgroup_01" } }
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `email` | Valid RFC 5322 email format (validated at API layer via zod) |
| `first_name`, `last_name` | String, max 255 chars |
| `company_name` | String, max 255 chars |
| `country_code` (address) | ISO 3166-1 alpha-2 (2-char string) |
| `postal_code` | String, max 64 chars |
| `metadata` | Valid JSON object |

---

## 5. DTOs

### CustomerDTO
```ts
{
  id: string
  company_name: string | null
  first_name: string | null
  last_name: string | null
  email: string | null
  phone: string | null
  has_account: boolean
  metadata: Record<string, unknown> | null
  created_by: string | null
  groups?: CustomerGroupDTO[]
  addresses?: CustomerAddressDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### CustomerGroupDTO
```ts
{
  id: string
  name: string
  metadata: Record<string, unknown> | null
  created_by: string | null
  customers?: CustomerDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

---

## 6. Integration Test Scenarios

| Scenario | Expected |
|---|---|
| Create customer with existing email + `has_account=true` | 409 Conflict |
| Create customer with same email as soft-deleted customer | 201 Created |
| Add customer to group twice | Idempotent — no duplicate pivot row |
| Delete customer — check addresses | Addresses also soft-deleted |
| Delete customer — check groups | Group still exists, membership removed |
| List customers with `q=jane` | Returns customers with matching name/email |
